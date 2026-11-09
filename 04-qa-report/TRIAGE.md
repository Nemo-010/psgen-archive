# psgen QA report — verification and triage

Source: `attachments/psgen-report.zip` → `psgen-report/` (README, FINDINGS.md,
FINDINGS-STATIC.md, COVERAGE.md, FIX-PLAN.md, COMPARISON.md, patches/, harness/,
evidence/).

Target: `Azathothas/psgen` at `bf80156` (only commit, tag `v0.0.1-beta.1`).
Tracker state before this pass: 13 open issues (#1–#13), 0 pull requests, labels
empty.

## How this pass was verified

Upstream was cloned and built here rather than trusting the report's binary:

```
git clone https://github.com/Azathothas/psgen; git checkout bf80156
cargo build --release            # cargo 1.99.1, libc only
```

The report's combined patch applies cleanly to `bf80156`
(`git apply --check` → clean, 15 files, +470/−119) and rebuilds. Both binaries
were kept side by side and every claim below was re-run against **this** split,
not against the report's logs. Where the cage blocks a trigger, that is stated.

Cage limits that matter for this pass: unprivileged (uid 966), no mount
namespaces, no `unshare`, no `/dev/ptmx`, `ptrace` blocked (`strace` cannot
attach). So: bind tier untestable, ptys untestable, `strace -e inject=` unusable.
Workarounds used: a two-syscall `libseccomp` launcher (`sg`, `sk`) stands in for
strace-based syscall denial and for a `KILL_PROCESS` filter.

## Verification matrix

Legend: **REPRO** reproduced here with the same mechanism as the report ·
**CONFIRMED** mechanism confirmed, different or narrower trigger ·
**CODE-READ** code and docs agree with the report, trigger unavailable here ·
**REFUTED** report is wrong or overstated · **DUPLICATE** already filed.

### Executed findings (FINDINGS.md)

| ID | Claim | Status | Evidence here |
| --- | --- | --- | --- |
| F-01 | same seed, different table one second later | REPRO | `psgen table -n 20 -s 42` twice, 2 s apart → 32 differing lines; patched → identical |
| F-02 | `stop --dry-run` kills | REPRO | 2 PIDs alive after `stop --dry-run`, rc 0, patched → rc 2 |
| F-03 | psgen's own pidfile refused next run | REPRO | second `inject --pidfile` → "not an empty file or a psgen pidfile", rc 2; patched rc 0 |
| F-04 | `getrandom` failure panics after spawn | REPRO | `sg` launcher → panic `inject.rs:555`, rc 134, no pidfile, stubs orphaned |
| F-05 | `-n` under-delivered / duplicate-padded | REPRO | `-f mail -n 50` → 8 rows; `net -n 50` → 15; `sys -n 200` → 80 unique of 200 |
| F-06 | `--local` + `--utc-offset` both applied | REPRO | TZ=Asia/Kolkata: utc+120 → 04:20, local → 07:50, both → 09:50 (should be 07:50) |
| F-07 | failed bind leaves host-invisible exe path | CODE-READ | needs the bind tier; namespace creation denied here |
| F-08 | `ttyS0` shown as `tty64` | CODE-READ | needs a serial console; no `/dev/ttyS0` in the cage |
| F-09 | postgres children are session leaders | REPRO | sid==pid on 5 of 8 rows, `Ss`; patched: all share master sid, `S` |
| F-10 | `inject --table` not PID-sorted | REPRO | rows 3254, 3252, 3249, 3253, 3250, 3251; patched sorted |
| F-11 | `--create-users` cannot repair a group-only host | CODE-READ | needs root + `useradd`; no sudo here |
| F-12 | `check --binaries` misreports | REPRO | `/usr/lib/bluetooth/bluetoothd` exists and is reported missing |
| F-13 | seccomp probe reports `EFAULT` as blocked | REPRO | `doctor` → `seccomp_user_notif blocked` on a normal kernel; patched → reachable |
| F-14 | `doctor --pidfile` disagrees with inject | REPRO | on a psgen pidfile: doctor DENY / inject rc 0; patched ALLOW |
| F-15 | `--mem-cap` help false below the floor | REPRO | `-n 16 --mem-cap 8` → 64 MiB reported against an 8 MiB cap |
| F-16 | unit names a nonexistent target | REPRO | `After=local-fs.target proc-fs-nets.target` |
| F-17 | unreadable live PID counted as gone | CODE-READ | needs `hidepid=2` + a namespace to remount; unavailable |
| F-18 | one-field line cannot stop bind/shadow rows | REPRO | one-field pidfile → `stopped 0, skipped 2`, both PIDs still `S` |
| F-19 | no-pidfile run leaks temp dirs | REPRO | `-d 1` run: 4 dirs survive the stubs |
| F-20 | `--phys-cap 1` cannot be met, exits 0 | REPRO | "estimated physical cost is about 822 MiB, above the 1 MiB budget", rc 0 |
| F-21 | `--verify --no-mem` warns, exits 0 | REPRO | warning only, rc 0 |
| F-22 | seed/profile no effect on a short `list` | REPRO | seed 1 vs 2 at `-n 5`: 0 differing lines; `-p` at 181: 0 |
| F-23 | `%MEM` truncated not rounded | REPRO | 32 of 60 rows differ from `round()` on upstream, 0 on patched |
| F-24 | `-q` hides the pidfile path | REPRO | `-q` → 0 stdout lines, pidfile path gone |
| F-25 | `--dry-run` skips the pidfile preflight | CONFIRMED | `--dry-run --pidfile <unwritable>` → rc 0 |
| F-26 | silently-ignored flag combinations | REPRO | `--no-user --create-users`, `--no-mem --mem-cap 8` print nothing on upstream |
| F-27 | reduced-capability warning is probe-driven | REFUTED as a defect | upstream did **not** warn for `--no-tty` here; report says the same. Not a finding, a clarification. |
| F-28 | `--flame` hidden | REPRO | accepted by the parser, rejected at runtime, absent from `--help` |
| F-29 | bare `invalid --disk` message | REPRO | no maximum in the message |
| F-30 | parse errors print full usage, no token | REPRO | `--help --bogus` → rc 2 + 100-line usage |
| F-31 | `stop` could signal a stale process group | CODE-READ | needs PID reuse; not reproducible on demand |
| F-32 | seccomp KILL takes down the probes and psgen | REPRO | `sk` launcher: upstream `doctor` rc 159, `inject --dry-run` rc 159; patched 0/0 |
| F-33 | shadow tier on `noexec` splits roots/children | CODE-READ | needs a `noexec` mount we can control; `/tmp` here is rw,exec |
| F-34 | pidfile basename near NAME_MAX | REPRO | 240-byte basename: pidfile not written, rc 0, PIDs dumped to stderr; patched exit 2 before spawn |
| F-35 | `stop` counts a subtree as one process | REPRO | 6-row run → `stopped 3, skipped 3`; sand cage number in report matches |
| F-36 | `bench/sand.sh` omits `PIDS` | CODE-READ | static; needs the cage recipe |
| F-37 | release workflow only `--version` on cross targets | CODE-READ | read `.github/workflows/release.yml` |
| F-40 | TTY column and argv tty token disagree | CODE-READ | needs ptys |

### Static findings (FINDINGS-STATIC.md)

The static review is 102 items (12 BUG, 22 RISK, 68 NIT). I did not re-verify
all of them; I checked the ones the report itself marks VERIFIED, plus every
item whose claim is a compilable/readable fact about this tree:

| ID | Verdict here |
| --- | --- |
| BUG-01..BUG-12 | all twelve re-checked against `src/`; **BUG-01, 02, 03, 04, 05, 06, 07, 08, 10, 11** are what FINDINGS.md F-04, F-15, F-05, F-08, F-09, F-02, F-17, F-06, F-13, F-11 already say. **BUG-09, BUG-12** are extra (below). |
| RISK-01..RISK-22 | RISK-01=F-01, 02=F-20, 04=F-33, 06=F-32, 08=F-14, 13=F-19, 14=F-18, 19/20/21/22 are extras (below). The remaining are code-read only. |
| NIT-01..NIT-68 | Spots checked: NIT-02 (live default is all rows, table is 30 — REPRO), NIT-11 (`log -n 5` prints 67 lines — REPRO), NIT-21 (empty `.psgen-stubs` survives `stop` — REPRO), NIT-24 (=F-10), NIT-33 (`--help --bogus` → rc 2 — REPRO), NIT-43 (=F-16), NIT-45, NIT-46, NIT-52, NIT-63. The rest are style/wording and are taken as read. |

Extra static items worth a tracking issue of their own (not in the 38 executed
findings):

- **BUG-09** — the TTY column and the argv tty token disagree (same as F-40).
- **BUG-12** — `initgroups`/`setgroups` failure is not reported, so a stub can
  keep the caller's supplementary groups.
- **RISK-09** — `close_range` `ENOSYS` is ignored on pre-5.9 kernels.
- **RISK-10** — `open_tree`/`move_mount` numbers are hardcoded instead of
  `libc::SYS_*`.
- **RISK-11** — `read`/`pread`/`pwrite`/`nanosleep`/`waitpid_nohang` do not retry
  `EINTR`.
- **RISK-12** — "created but not visible" is reported as a failure.
- **RISK-15** — a pidfile with no boot header silently disables the boot-id check.
- **RISK-16** — `table` can print RSS above MemTotal and `%MEM` > 100.
- **RISK-17** — synthetic PIDs are not capped at `pid_max`.
- **RISK-19** — `--utc-offset` accepts any `i64`.
- **RISK-20** — `log --lines` unbounded, `sys::out` ignores `EPIPE`.
- **RISK-21** — `load --mem-util` sizes from MemTotal, can OOM.
- **RISK-22** — inherited fd 3 config is trusted.

## Triage decisions

The tracker already carries 13 issues covering a slice of this report (issues
#1–#13 of the earlier pass: pidfile header, seed, `ps aux` fidelity, zombies,
staging dirs, smoke aborts, version string, doctor JSON, `-p`, `gen_list`
quadratic, CI, `--dry-run` plan, and a twelve-item grab bag). **Nothing is
re-filed.** This pass adds only what those 13 do not cover.

Filed by this pass (see `ISSUES/`): 12 new issues, one topic each.

Already covered, so not re-filed — cross-referenced instead:

| Report item | Existing issue |
| --- | --- |
| F-01 / RISK-01 / BUG-01(static numbering clash) | #2 |
| F-03 / BUG-… | #1 |
| F-10 / NIT-24 | #3 (3c) |
| F-17 / F-31 / F-35 / BUG-07 | #4 |
| F-19 / RISK-13 | #5 |
| none (harness) / #6a–6c | #6 |
| F-16 / NIT-43 | #13 (13.1) |
| F-21 / NIT-01 | #13 (13.3) |
| F-28 / NIT-32 | #13 (13.5) |
| F-37 / NIT-50 | #11 |
| F-22 / NIT-05 | #9 (9a) |
| NIT-25 / doctor JSON | #8 |
| NIT-10 / NIT-12 aliases | #13 (13.5) |

## Pull request

The report's patch is one combined 15-file diff mixing 27 fixes with test
changes. One PR of that shape cannot be reviewed, and it silently changes
output that other issues call correct: `%MEM` rounding (F-23) rewrites every
`table` row, and the postgres `F_SESSION` change (F-09) rewrites the corpus.
It also carries a fix that is **wrong as written** — see the PR body.

So: a single PR is opened against a fork, as the report's patch split into
reviewable commits, with the parts that need a design decision left out and
listed in the description. See `PR.md`.
