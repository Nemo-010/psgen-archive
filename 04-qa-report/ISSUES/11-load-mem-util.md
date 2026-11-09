`psgen load --mem-util PCT` sizes from `MemTotal`, not from what is actually free, so `--mem-util 90` on a host that is already busy maps and touches more memory than exists free and can invoke the OOM killer — possibly against `psgen` itself.

## What happens

`src/activity.rs`, `run_load()`:

```rust
let total = util::mem_total_kb();
let cg = util::cgroup_mem_limit_kb();
let total = if cg > 0 { total.min(cg) } else { total };
let mem_kb = total * mem_pct as u64 / 100;
```

`mem_pct` is accepted up to 95 (`src/cli.rs` rejects above 95), so a request is at
most 95% of `MemTotal`. On a host where anything else is already using 20% of RAM,
95% of total is more than the free memory, and the touch loop faults in pages the
kernel cannot provide without reclaiming or killing something.

The help documents the total-based meaning —

```
--mem-util PCT  touch and hold this percent of MemTotal, or of the cgroup limit when lower (0..95)
```

— so the code matches the help. The problem is that the help describes a request
that is not safe to grant.

This is not testable here without risking the cage (12 GiB, shared), which is exactly
the point: a load generator should not be able to take down its own host by
following its documentation.

## What a caller expects

`--mem-util 90 -d 60` reads as "make the host look 90% utilized for a minute". A
reasonable reading is "allocate as much as is *available* so that the host looks
busy", not "allocate 90% of physical RAM regardless of what is already in use". A
load generator that OOMs the box it is measuring is not useful.

## Options

1. **Base it on `MemAvailable`** (`/proc/meminfo`) and treat the percentage as a
   fraction of the available memory: `mem_kb = avail_kb * pct / 100`. This always
   succeeds and still moves the utilisation number, which is what the flag is for.
2. **Keep the total-based meaning but refuse unsafe requests**: if
   `mem_kb > avail_kb`, warn and refuse with exit 2, unless `--force` is given. Keeps
   the documented semantics and makes the dangerous case explicit.
3. **Keep it and document the danger** in the help and in the README's Limitations
   section. Cheapest, and honestly the least useful.

I would do 2: it does not change what a caller asking for a *safe* amount gets, it
makes the existing semantics auditable, and `--force` preserves the current
behaviour for anyone who actually wants to exhaust memory.

```diff
--- a/src/activity.rs
+++ b/src/activity.rs
     let total = if cg > 0 { total.min(cg) } else { total };
     let mem_kb = total * mem_pct as u64 / 100;
+    let avail = util::mem_available_kb();
+    if avail > 0 && mem_kb > avail && !force {
+        // A request above what is free does not measure load, it invokes the
+        // OOM killer, and the victim is often this process.
+        sys::err(/* "psgen: --mem-util {pct}% is {mem} MiB, above the {avail} MiB
+                    available; use --force to attempt it anyway" */);
+        return 2;
+    }
```

`util::mem_available_kb()` does not exist yet; it is the `MemAvailable:` line of
`/proc/meminfo`, which every kernel from 3.14 has.

## Same shape, one function over: `--disk-util`

`--disk-util` already has both guards (`RLIMIT_FSIZE` and the free-space check from
issue #6's neighbourhood), so the memory side is the odd one out. Whatever is chosen
here should match what `--disk-util` does, so the two load knobs behave the same way.

## Test that would have caught it

```python
avail = int(next(l.split()[1] for l in open("/proc/meminfo") if l.startswith("MemAvailable:")))
p = run(["load", "--mem-util", "95", "-d", "1"], check=False)
# either it refuses, or it must not have asked for more than was available
if p.returncode == 0:
    fail("load --mem-util 95 on a partially-used host was granted")
```

(Run it under a cgroup with a small limit so the assertion is cheap; the invariant is
the return code, not the allocation.)

## Environment

`bf80156` built with `cargo 1.99.1`, x86-64, Linux 7.2.11. `MemTotal` 12,131,920 kB
on this host. Verified by reading the code path and by the documented help text; no
run was attempted, because the failure mode is the host going down.
