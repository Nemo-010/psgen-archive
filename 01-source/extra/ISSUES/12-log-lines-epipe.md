`psgen load --lines` is unbounded and `sys::out` ignores a closed pipe, so `log --lines` can be asked for an impossible amount of output and `log ... | head` keeps generating after the reader has gone.

## What happens

With `--delay 0`, the whole output is accumulated in memory before the single final write:

```
$ psgen log -m weblog --lines 1000000000 > /dev/null
(allocates until the box gives up)
```

With a positive delay, `sys::out` discards the write error, so a reader that exits
early does not stop the loop:

```
$ psgen log -m weblog --lines 100000 --delay 100 | head -1
192.168.68.116 - - [...] "GET /static/js/app.js HTTP/1.1" 200 38 "-" "..."
$ # the pipeline returns immediately, but psgen keeps formatting lines
$ # for the full 100000 * 100 ms = ~2.8 hours
```

## Why

`src/activity.rs`, `run_log()`:

```rust
let lines = args.log_lines;
let mut emitted = Buf::new();
for i in 0..lines {
    ...
    if args.log_delay_ms > 0 && i + 1 < lines {
        sys::out(emitted.as_bytes());
        emitted = Buf::new();
    }
}
```

With `--delay 0` the `emitted` buffer is never flushed inside the loop, so it grows
to the size of the whole output. With a delay, each flush ignores the result.

`src/sys.rs`:

```rust
pub fn out(buf: &[u8]) {
    let _ = io::stdout().write_all(buf);
}
```

`write_all` returns `EPIPE` once the reader is gone, and the error is dropped. The
process also does not die from `SIGPIPE`, because Rust sets `SIG_IGN` for it at
startup, so there is nothing else to stop the loop.

## Patch

Three small changes, each independent:

```diff
--- a/src/activity.rs
+++ b/src/activity.rs
-    let lines = args.log_lines;
+    // A log generator has no reason to produce more than this, and the
+    // buffer below holds it all when --delay is 0.
+    let lines = args.log_lines.min(1_000_000);
     let mut emitted = Buf::new();
     for i in 0..lines {
@@
         if args.log_delay_ms > 0 && i + 1 < lines {
-            sys::out(emitted.as_bytes());
+            if !sys::out_checked(emitted.as_bytes()) {
+                return 0;   // the reader is gone; EPIPE is not an error
+            }
             emitted = Buf::new();
         }
```

```diff
--- a/src/sys.rs
+++ b/src/sys.rs
 pub fn out(buf: &[u8]) {
     let _ = io::stdout().write_all(buf);
 }
+
+/// Write to stdout, reporting a closed pipe so a generator can stop.
+pub fn out_checked(buf: &[u8]) -> bool {
+    match io::stdout().write_all(buf) {
+        Ok(()) => true,
+        Err(e) => e.kind() != io::ErrorKind::BrokenPipe,
+    }
+}
```

and flush the buffer periodically when `--delay 0` as well, so the memory bound is
the flush interval rather than the whole output:

```rust
if emitted.len() >= 64 * 1024 {
    if !sys::out_checked(emitted.as_bytes()) { return 0; }
    emitted = Buf::new();
}
```

## Related: `--lines 0` and `--speed 0`

`psgen log --lines 0` prints nothing and exits 0, which is a silent success for a
request that cannot mean anything. `--speed 0` is rejected for `log` only, in
`run_log()`; `load` and the other subcommands accept and ignore it. Validate both
once in `parse()` so the behaviour is the same for every subcommand, and decide
whether 0 lines is an error (I would make it one, matching `--wait --duration 0`).

## Test that would have caught it

```python
# a closed reader must stop the generator promptly
p = subprocess.Popen([BIN, "log", "-m", "weblog", "--lines", "100000",
                      "--delay", "10"], stdout=subprocess.PIPE)
p.stdout.readline()
p.stdout.close()
try:
    p.wait(timeout=5)
except subprocess.TimeoutExpired:
    p.kill()
    fail("log kept generating after its reader exited")

# an absurd line count must be refused or capped
p = run(["log", "--lines", "1000000000"], check=False, timeout=10)
if p.returncode == 0 and len(p.stdout) > 10 ** 8:
    fail("log --lines 1000000000 produced an unbounded amount of output")
```

## Environment

`bf80156` built with `cargo 1.99.1`, x86-64, Linux 7.2.11, unprivileged uid 966.
The `--lines 1000000000` case was not run to completion here (it is the failure
mode); the `head -1` case was observed to keep the process alive after the reader
exited.
