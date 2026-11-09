# Azathothas/psgen snapshot

29 objects: 28 issues, 1 pull requests.

One readable file per object beside its raw JSON.
`MANIFEST.json` carries a SHA-256 per rendering, and `RECOVER.md` says
how to refile from this directory.

## Issues

- [#1](issues/1.md) open - A pidfile written by psgen is rejected on the next run (header compare is off by one)
- [#2](issues/2.md) open - A seed is not reproducible across seconds: child-row ages use an uptime-sized modulus
- [#3](issues/3.md) open - table --live and inject --table do not match ps aux: raw newlines, no <defunct>, unsorted --table, columns shifted by a wide VSZ
- [#4](issues/4.md) open - stop counts zombies as stopped, and reports success before the signal lands
- [#5](issues/5.md) open - --duration runs leave their staging directories behind (shadow copy, --disk data) when nobody runs stop
- [#6](issues/6.md) open - tests/smoke.sh aborts instead of skipping on a host with a finite RLIMIT_FSIZE, no pty, or a non-reaping PID 1
- [#7](issues/7.md) open - The version string is hardcoded eight times, and stop parses the pidfile header with the version in it
- [#8](issues/8.md) open - doctor --json emits checks as an array of single-key objects
- [#9](issues/9.md) open - -p/--profile is advertised for every command but only table and inject use it, and other options are silently ignored
- [#10](issues/10.md) open - gen_list's duplicate check is quadratic (list -n 2000 takes 21 ms where 2000 rows should cost a few)
- [#11](issues/11.md) open - CI and release: add fmt/clippy, smoke the cross targets under qemu, publish the crate, verify the unit
- [#12](issues/12.md) open - inject --dry-run prints two columns and no plan (tier, exe, VSZ/RSS/CPU, threads, temp dirs)
- [#13](issues/13.md) open - Twelve small defects and doc mismatches found while testing every flag (unit target, --verify, check --binaries, log aliases, uid fallback, @self, repo metadata)
- [#14](issues/14.md) open - inject --pidfile panics after every stub is spawned when getrandom fails, leaving the run unstoppable
- [#15](issues/15.md) open - stop --dry-run is accepted and ignored, so a preview kills the run
- [#16](issues/16.md) open - -n is silently under-delivered for four families and padded with unlimited duplicates elsewhere
- [#17](issues/17.md) open - --local and --utc-offset are both applied, so START is shifted twice
- [#18](issues/18.md) open - PostgreSQL background workers and client backends are injected as session leaders
- [#19](issues/19.md) open - doctor reports seccomp_user_notif blocked on every normal kernel (EFAULT is read as blocked, ENOSYS as reachable)
- [#20](issues/20.md) open - --create-users cannot repair a host that has the group but not the user (both useradd attempts pass --user-group)
- [#21](issues/21.md) open - check --binaries reports an existing absolute path as missing (the directory is stripped before the check)
- [#22](issues/22.md) open - A one-field pidfile line cannot stop a bind-tier or shadow-tier row, and stop reports success anyway
- [#23](issues/23.md) open - load --mem-util sizes from MemTotal, so a safe-looking request can invoke the OOM killer
- [#24](issues/24.md) open - log --lines is unbounded and sys::out ignores EPIPE, so a closed reader does not stop the generator
- [#25](issues/25.md) open - The stub trusts any inherited fd 3 with the config magic, which is code execution and a root file write under a permissive sudo policy
- [#26](issues/26.md) open - A pidfile basename near NAME_MAX passes the preflight and fails after the stubs are spawned
- [#27](issues/27.md) open - The printed systemd unit cannot be restarted: stop leaves the pidfile and ExecStop fails when it is absent
- [#28](issues/28.md) open - table can print RSS above MemTotal, %MEM above 100 and PIDs above pid_max

## Pull requests

- [#29](prs/29.md) open - Fix the verified defects from the third psgen beta-test round (pidfile panic, stop accounting, seed, postgres, probes, check, unit, tests)
