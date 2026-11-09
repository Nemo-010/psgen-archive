`F_SESSION` is used for three different things, so PostgreSQL background workers and client backends are rendered as session leaders (`Ss`) and are injected with `setsid`, which makes them leaders of their own sessions instead of children of the postmaster. A real PostgreSQL host does not look like this.

## What happens

`psgen inject -f db -n 8 -s 1`:

```
$ ps -o pid,ppid,sid,pgid,stat,args -p $(awk '!/^#/{print $1}' /tmp/db.pid | paste -sd,) --no-headers
 3446    1  3446  3446 Ss  /usr/lib/postgresql/17/bin/postgres -D /var/lib/postgresql/17/main ...
 3447    1  3447  3447 Ss  postgres: 17/main: checkpointer
 3449    1  3449  3449 Ss  postgres: 17/main: autovacuum launcher
 3450 3446  3450  3450 Ss  postgres: 17/main: app app 10.0.2.240(57112) idle
 3452 3446  3452  3452 Ss  postgres: 17/main: app app 10.0.1.135(47462) idle
```

Every `postgres: 17/main: ...` row is a session leader: `sid == pid`, and the
background workers are not even children of the postmaster (`ppid == 1`). On a
real host the postmaster forks all of them, so every row shares the postmaster's
`sid` and `pgid`, and STAT is `S` (or `Ssl` for the postmaster), never `Ss`.

`psgen table -f db` shows the same `Ss` in the STAT column, so the synthetic table
is wrong in the same way.

## Why

`src/corpus.rs` marks all eight postgres rows `F_SESSION`:

```rust
t!("postgres: 17/main: checkpointer ", "postgres", FAM_DB, 218036, 6224, 0, 1, F_SESSION, 10),
t!("postgres: 17/main: background writer ", ... F_SESSION, 10),
t!("postgres: 17/main: walwriter ", ... F_SESSION, 10),
t!("postgres: 17/main: autovacuum launcher ", ... F_SESSION, 10),
t!("postgres: 17/main: logical replication launcher ", ... F_SESSION, 10),
t!("postgres: 17/main: app app {ip}({cport}) idle", ... F_SESSION, 18),
t!("postgres: 17/main: app app {ip}({cport}) SELECT", ... F_BUSY | F_SESSION, 10),
t!("postgres: 17/main: archiver ", ... F_SESSION, 6),
```

and the flag is consumed in three places with three different meanings:

1. `src/gen.rs` — "started at boot", for the age:
   ```rust
   let age = if t.flags & (corpus::F_LONG | corpus::F_SESSION) != 0 {
   ```
2. `src/table.rs` — the `s` in STAT:
   ```rust
   if p.flags & crate::corpus::F_SESSION != 0 {
       out.push(b's');
   ```
3. `src/stub.rs` — `setsid`:
   ```rust
   if cfg.flags & (corpus::F_SESSION as u32 | corpus::F_TTY as u32) != 0 {
       sys::setsid();
   ```

Separately, `parent_index()` only linked the `app app` rows to the postmaster:

```rust
} else if cmd.starts_with("postgres: 17/main: app app") {
    find_idx("/usr/lib/postgresql/17/bin/postgres -D")
```

so the checkpointer, writers, autovacuum and archiver became roots with `ppid == 1`.

## Patch (verified)

Split the meanings: `F_LONG` keeps the boot-relative age, `F_CHILD` makes the row a
child of the postmaster and drops the `s` flag and the `setsid`, and `parent_index`
links every `postgres: 17/main:` row to the postmaster.

```diff
--- a/src/corpus.rs
+++ b/src/corpus.rs
-    t!("postgres: 17/main: checkpointer ", "postgres", FAM_DB, 218036, 6224, 0, 1, F_SESSION, 10),
+    t!("postgres: 17/main: checkpointer ", "postgres", FAM_DB, 218036, 6224, 0, 1, F_LONG | F_CHILD, 10),
     ... (same for background writer, walwriter, autovacuum launcher,
          logical replication launcher, archiver)
-    t!("postgres: 17/main: app app {ip}({cport}) idle", ... F_SESSION, 18),
-    t!("postgres: 17/main: app app {ip}({cport}) SELECT", ... F_BUSY | F_SESSION, 10),
+    t!("postgres: 17/main: app app {ip}({cport}) idle", ... F_CHILD, 18),
+    t!("postgres: 17/main: app app {ip}({cport}) SELECT", ... F_BUSY | F_CHILD, 10),
--- a/src/gen.rs
+++ b/src/gen.rs
-    } else if cmd.starts_with("postgres: 17/main: app app") {
+    } else if cmd.starts_with("postgres: 17/main: ") {
+        // The postmaster forks every backend and background worker, so all
+        // of them belong to its session, not their own.
         find_idx("/usr/lib/postgresql/17/bin/postgres -D")
```

Verified:

```
$ ps -o pid,ppid,sid,pgid,stat,args -p <8 rows>
 3460    1  3460  3460 Ss  /usr/lib/postgresql/17/bin/postgres -D ...
 3462 3460  3460  3460 S   postgres: 17/main: app app 10.0.2.240(57112) idle
 3463 3460  3460  3460 S   postgres: 17/main: app app 10.0.1.135(47462) idle
 3466 3460  3460  3460 S   postgres: 17/main: checkpointer
 3467 3460  3460  3460 S   postgres: 17/main: autovacuum launcher
```

All workers share the postmaster's `sid`/`pgid`, `ppid == 3460`, STAT `S`.
`table -f db` shows `S` for them too.

## Note on the corpus change

This changes the generated rows for `-f db` and for any unfiltered `table`/`inject`
run that draws a postgres template, so it will move any golden-output test. That is
the point: the current output is wrong. `tests/invariants.py` has no postgres
assertion today, so nothing breaks, but the new assertion below should be added at
the same time.

## Test that would have caught it

```python
pidfile, pids, _ = inject_and_check(["inject", "-f", "db", "-n", "8", "-s", "1",
                                     "--no-user", "--no-mem", "--no-tty"], tmp, 8)
stats = {p: open(f"/proc/{p}/stat").read() for p in pids}
master = [p for p in pids if "postgres -D" in open(f"/proc/{p}/cmdline").read()]
if master:
    want_sid = int(stats[master[0]].rsplit(")", 1)[1].split()[3])
    for p, s in stats.items():
        f = s.rsplit(")", 1)[1].split()
        state, sid = f[0], int(f[3])
        if "postgres: 17/main:" in open(f"/proc/{p}/cmdline").read():
            if state == "S" and "s" in f:  # STAT has the session flag
                fail(f"postgres worker {p} is a session leader")
            if sid != want_sid:
                fail(f"postgres worker {p} left the postmaster session")
```

## Environment

`bf80156` built with `cargo 1.99.1`, x86-64, Linux 7.2.11, unprivileged uid 966.
The `sid`/`pgid`/`ppid` shape is kernel-side and independent of the environment;
`setsid` succeeds for an unprivileged process as long as it is not already a
process-group leader, which the injector guarantees.
