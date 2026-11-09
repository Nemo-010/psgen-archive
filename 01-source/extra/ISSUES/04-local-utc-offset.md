`table --local` and `table --utc-offset` are both applied, so the START column is shifted twice when the system timezone is not UTC. The result is neither local time nor UTC plus the offset.

## What happens

With `TZ=Asia/Kolkata` (UTC+5:30), on the same second:

```
$ psgen table -f dev -n 4 -s 1 | awk '{print $9}'                 # default: UTC
02:20 02:19 02:20 Oct31
$ psgen table -f dev -n 4 -s 1 --utc-offset 120 | awk '{print $9}'
04:20 04:19 04:20 Oct31                                          # UTC+2, correct
$ psgen table -f dev -n 4 -s 1 --local | awk '{print $9}'
07:50 07:49 07:50 Oct31                                          # local, correct
$ psgen table -f dev -n 4 -s 1 --local --utc-offset 120 | awk '{print $9}'
09:50 09:49 09:50 Oct31                                          # local + 2h: wrong
$ TZ=Asia/Kolkata date +%H:%M; date -u +%H:%M
07:52
02:22
```

`--local` alone is right (07:50 ≈ local 07:52), `--utc-offset 120` alone is right
(04:20 ≈ UTC 02:22 + 2h), and the two together give 09:50 — local time shifted by
another two hours. The help lists both flags with no exclusion, so there is no way
for a caller to know which one wins, and the answer is "both".

## Why

`src/table.rs`, `start_field()`:

```rust
let now = (sys::now_secs() as i64).saturating_add(offset_min.saturating_mul(60));
let start = now.saturating_sub(age as i64);
...
if local {
    let ok = unsafe { !libc::localtime_r(&t, &mut tm).is_null() };
```

`offset_min` is added to `now` *before* `start` is computed, and `localtime_r` then
converts the already-shifted timestamp to the system timezone. Two conversions, one
of them unwanted.

## Patch (verified)

`--local` selects the system timezone, so a numeric offset must not be applied on
top of it. Make local win, and say so in the help:

```diff
--- a/src/cli.rs
+++ b/src/cli.rs
@@ -727,6 +727,16 @@ fn pidfile_parent(path: &[u8]) -> &[u8] {
+/// The START offset to hand to the table. `--local` selects the system
+/// timezone, so a UTC offset would shift a second time; local wins.
+fn start_offset(args: &Args) -> i64 {
+    if args.local {
+        0
+    } else {
+        args.offset_min
+    }
+}
```

and pass `start_offset(&args)` in all three `table::render` call sites (`table`,
`table --live`, `inject --table`). Help:

```diff
-      --local         use the system timezone for the START column
+      --local         use the system timezone for the START column; overrides
+                      --utc-offset
```

Verified with the same four commands:

```
default(utc) 02:20 02:19 02:20 Oct31
utc+120      04:20 04:19 04:20 Oct31
local        07:50 07:49 07:50 Oct31
local+120    07:50 07:49 07:50 Oct31     <- now equals --local
```

The alternative — reject the combination with exit 2 — is defensible too, but
`--local --utc-offset 0` is a reasonable thing to write, and "local wins" is what
the flag names imply.

## Related, same function: `--utc-offset` is unbounded

`src/cli.rs` parses an arbitrary `i64` and `src/table.rs` multiplies it by 60 with
`saturating_mul`. Nothing bounds it, so the START column can be a year with 20
digits, which shifts every column after it:

```
$ psgen table -n 2 --utc-offset 9223372036854775807 | head -2
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.3  0.1  22144 16952 ?        Ss   Nov25  38:31 /sbin/init
$ psgen table -n 1 --utc-offset 100000 | head -2
root           1  0.1  0.1  24504 16488 ?        Ss   Jan 8  12:50 /sbin/init
```

(`100000` minutes is 69 days; the row says January.) Bound it to ±1440 minutes and
reject the rest with exit 2, or document the unit and the range in the help.

## Test that would have caught it

```python
env = dict(os.environ, TZ="Asia/Kolkata")
loc = run(["table", "-f", "dev", "-n", "4", "-s", "1", "--local"], env=env).stdout
both = run(["table", "-f", "dev", "-n", "4", "-s", "1", "--local",
            "--utc-offset", "120"], env=env).stdout
if loc != both:
    fail("--local and --utc-offset were both applied")
p = run(["table", "-n", "1", "--utc-offset", "9223372036854775807"], check=False)
if p.returncode != 2:
    fail("an absurd --utc-offset was accepted")
```

## Environment

`bf80156` built with `cargo 1.99.1`, x86-64, Linux 7.2.11, unprivileged uid 966,
`TZ=Asia/Kolkata` (UTC+5:30) for the double-shift case. Reproduces with any
non-UTC zone; with `TZ=UTC` the two offsets coincide and the defect is invisible,
which is why it is easy to miss in testing.
