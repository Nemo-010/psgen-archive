`psgen stop --pidfile FILE --dry-run` kills the run. `--dry-run` is parsed globally, so `stop` accepts it, and the stop path never looks at it. A command that reads as a preview is destructive, and with `--elevate` it does so as root.

## What happens

```
$ psgen inject -n 2 -s 5 --pidfile /tmp/e2.pid -q
$ PIDS=$(awk '!/^#/{print $1}' /tmp/e2.pid | paste -sd,); echo $PIDS
6723,6724
$ psgen stop --pidfile /tmp/e2.pid --dry-run
stopped 2 processes
$ echo $?
0
$ sleep 0.5; for p in 6723 6724; do awk '{print $3}' /proc/$p/stat; done
Z
Z
```

Both stubs are dead. There is no preview output, no warning, and the exit code says the stop happened — because it did.

With `--elevate` the same invocation re-execs through `sudo -n` first (`src/cli.rs`, the `elevate` branch runs for `Sub::Stop` too), so the kill is performed as root even when the caller is not.

## Why

`src/cli.rs` parses the flag for every subcommand:

```rust
b"--dry-run" => a.dry_run = true,
```

and the whole `stop` dispatch is:

```rust
Sub::Stop => {
    let path = match &args.pidfile {
        Some(p) => p,
        None => {
            sys::err(b"psgen: stop needs --pidfile\n");
            return 2;
        }
    };
    inject::stop(path, args.force)
}
```

`args.dry_run` is never read. The help documents `--dry-run` under INJECT only, but nothing rejects it elsewhere. `load` accepts and ignores it too, which is harmless; `stop` ignores it destructively.

## Patch

Reject the combination at parse time, the way `--force` is already restricted to `stop`:

```diff
--- a/src/cli.rs
+++ b/src/cli.rs
@@ -518,6 +518,11 @@ pub fn parse(args: &[Vec<u8>]) -> Result<Args, &'static str> {
     {
         return Err("--force only applies to stop");
     }
+    // --dry-run is an inject preview. Accepting it for stop made a command
+    // that looks like a preview kill the run.
+    if a.dry_run && a.sub == Sub::Stop {
+        return Err("--dry-run only applies to inject");
+    }
     Ok(a)
 }
```

Built and verified here:

```
$ psgen stop --pidfile /tmp/e2.pid --dry-run
psgen: --dry-run only applies to inject
$ echo $?
2
$ for p in 6723 6724; do awk '{print $3}' /proc/$p/stat; done
Z          # (the earlier run's stubs; a fresh run's stubs stay S)
```

A fresh 2-row run, then `stop --dry-run`, leaves both rows alive and exits 2.

If a real preview is wanted instead, the honest version is a separate print-only
path: resolve the identity of every line, print `would signal <pid> (<comm>)`, and
return without signalling. Rejecting the flag is the smaller change and does not
add a second code path that can drift from the real one.

## Test that would have caught it

In `tests/invariants.py`, `check_stop()`:

```python
# --dry-run on stop must be a parse error, and must not signal anything
pidfile, pids, _ = inject_and_check([...], tmp, 2)
p = run(["stop", "--pidfile", str(pidfile), "--dry-run"], check=False)
if p.returncode != 2:
    fail("stop --dry-run was accepted")
for pid in pids:
    if not alive(pid):
        fail(f"stop --dry-run killed pid {pid}")
stop_pidfile(pidfile)
```

## Environment

`bf80156` built with `cargo 1.99.1`, x86-64, Linux 7.2.11, unprivileged uid 966.
The same behaviour is present for any caller; `--elevate` only changes the uid the
kill runs as.
