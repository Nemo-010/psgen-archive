# psgen — third QA round: verification and triage

Greetings from Port Edwards. Neucom Info ran this pass; the fix set is the kind of
thing Neucom Publishing would want a changelog for, and Neucom Space would want a
post-mortem for.

**Input**: `attachments/psgen-report.zip` → `psgen-report/` (README, FINDINGS.md
38 executed findings, FINDINGS-STATIC.md 102 static items, COVERAGE.md, FIX-PLAN.md,
COMPARISON.md, `patches/psgen-qa-fixes.patch` + a binary, `harness/`, `evidence/`).

**Target**: `Azathothas/psgen` at `bf80156` (tag `v0.0.1-beta.1`).

**Tracker before this pass**: 13 open issues (#1–#13), 0 pull requests.

## What I did

1. Cloned the repo and rebuilt `bf80156` here (`cargo 1.99.1`, `libc` only) instead of
   trusting the report's binary. Built a second binary from the report's patch.
2. Re-ran every executed finding against that pair. Where the cage blocks the trigger
   I said so and used a substitute (see *Cage*).
3. Triaged against #1–#13: **nothing was re-filed**. Findings already tracked got a
   comment with my evidence; the rest became new issues.
4. Filed **15 new issues** (#14–#28), one topic each, every one actionable without a
   follow-up question.
5. Opened **one pull request**: the report's patch, unmodified in substance, split into
   9 commits, plus the three test-harness fixes from #6 (without them the patch still
   aborts the suite on a constrained host).

## Verification matrix

Legend: **REPRO** = reproduced here with the same mechanism · **CONFIRMED** = mechanism
confirmed, different trigger · **CODE-READ** = code and docs agree, trigger unavailable ·
**REFUTED** = report is wrong.

| Report finding | Status | Evidence here |
| --- | --- | --- |
| F-01 seed not reproducible | REPRO | `table -n 20 -s 42` twice, 2 s apart → 32 differing lines; fixed → identical |
| F-02 `stop --dry-run` kills | REPRO | both PIDs `Z` after a "preview", rc 0; fixed → rc 2, nothing signalled |
| F-03 psgen's own pidfile refused | REPRO | second `inject --pidfile` → rc 2; fixed → rc 0 |
| F-04 `getrandom` panics after spawn | REPRO | seccomp launcher → panic at `inject.rs:555`, rc 134, no pidfile, 2 orphaned stubs |
| F-05 `-n` shortfall / duplicate padding | REPRO | `-f mail -n 50` → 8 rows; `net` → 15; `sys -n 200` → 88 unique of 200 |
| F-06 `--local` + `--utc-offset` | REPRO | TZ=Asia/Kolkata: utc+120=04:20, local=07:50, both=09:50 |
| F-07 child exe is a host-invisible path | CODE-READ | needs the bind tier; no mount namespaces in this cage |
| F-08 `ttyS0` → `tty64` | CODE-READ | needs a serial console |
| F-09 postgres workers are `Ss` | REPRO | 5 of 8 rows `sid == pid`; fixed → all in the postmaster's session |
| F-10 `inject --table` unsorted | REPRO | `3254 3252 3249 3253 3250 3251`; already tracked in #3c |
| F-11 `--create-users` group-only host | CODE-READ | needs root + `useradd`; no sudo here |
| F-12 `check --binaries` misreports | REPRO | `/usr/lib/bluetooth/bluetoothd` exists, reported missing |
| F-13 seccomp probe `EFAULT` | REPRO | probe gives `EFAULT`, doctor says `blocked`; fixed → `reachable` |
| F-14 `doctor --pidfile` vs inject | REPRO | on a psgen pidfile: doctor DENY while inject accepts |
| F-15 `--mem-cap` help false below floor | REPRO | `-n 16 --mem-cap 8` reports 64 MiB against an 8 MiB cap |
| F-16 unit names a nonexistent target | REPRO | `After=local-fs.target proc-fs-nets.target` |
| F-17 unreadable live PID counted as gone | CODE-READ | needs `hidepid=2` in a private mount namespace |
| F-18 one-field pidfile cannot stop bind/shadow | REPRO | `stopped 0 processes, skipped 2`, both PIDs still `S` |
| F-19 no-pidfile runs leak temp dirs | REPRO | 4 dirs and 4 files survive the stubs |
| F-20 `--phys-cap 1` cannot be met, rc 0 | REPRO | "about 822 MiB, above the 1 MiB budget", rc 0 |
| F-21 `--verify --no-mem` warns only | REPRO | warning, rc 0 |
| F-22 seed/profile inert on a short `list` | REPRO | seed 1 vs 2 at `-n 5` → 0 lines |
| F-23 `%MEM` truncated | REPRO | 32 of 60 rows differ from `round()`; fixed → 0 |
| F-24 `-q` hides the pidfile path | REPRO | 0 stdout lines |
| (extra) `log` keeps running after its reader exits | REPRO | still alive 4.5 s after `head -1` exited; peak RSS 7.8/9.5/73.7 MiB at 2k/20k/200k lines |
| F-25 `--dry-run` skips the pidfile preflight | CONFIRMED | `--dry-run --pidfile <unwritable>` → rc 0 |
| F-26 flags that cannot take effect were silent | REPRO | `--no-user --create-users`, `--no-mem --mem-cap 8` printed nothing |
| F-27 reduced-capability warning | **REFUTED** | see below |
| F-28 `--flame` hidden | REPRO | parsed, rejected at runtime, absent from `--help` |
| F-29 bare `invalid --disk` | REPRO | no maximum in the message |
| F-30 parse errors print full usage | REPRO | `--bogus` → 89 lines, no token |
| F-31 `stop` could signal a stale group | CODE-READ | needs PID reuse |
| F-32 seccomp KILL takes down psgen | REPRO | `doctor` and `inject --dry-run` rc 159; fixed → rc 0 |
| F-33 noexec shadow tier | CODE-READ | cannot create a `noexec` mount here |
| F-34 pidfile basename near NAME_MAX | REPRO | boundary between 225 and 230 bytes; 2 stubs spawned, rc 1 |
| F-35 `stop` counts a subtree as one | REPRO | 40-row run → 33/35/34; fixed → 40/40/40 |
| F-36 `bench/sand.sh` omits `PIDS` | CODE-READ | static |
| F-37 release workflow only `--version` | CODE-READ | read `release.yml` |
| F-40 TTY column vs argv token | CODE-READ | needs ptys |

### The one refutation

**F-27** says the static review's `--no-tty` complaint was wrong and that upstream
"printed no reduced-capability line for `--no-tty --no-user`". On this host upstream
*does* print it:

```
$ psgen inject -n 2 --no-tty --no-user --dry-run
psgen: reduced capability: pty=no (run `psgen doctor` for the errno)
```

which is `!c.pty || (c.root && !c.user_switch)` firing on the probe result, exactly
what the static review said. So the refutation is wrong, and F-27 is not a finding —
but the change it motivates (condition the warning on what the caller asked for) is
still worth having, so it stays in the patch.

### The one claim that is conditional

The report's README says `make test` passes after the patch. On a host with a finite
hard `RLIMIT_FSIZE` or no `/dev/ptmx` it does not: the patch does not touch
`tests/invariants.py` for those, so the suite still aborts in `check_load` and then in
`check_injection`. The PR therefore carries issue #6's three harness fixes as its last
commit. With them the suite is green here.

### Static review (102 items)

I re-checked all twelve BUGs and the RISKs that the report marks VERIFIED, plus every
item whose claim is a readable fact in this tree. Ten of the twelve BUGs are the same
ten findings as F-04/15/05/08/09/02/17/06/13/11. The extras that found their own homes:

| Item | Where |
| --- | --- |
| BUG-09 TTY token vs pty | folded into the report's F-40; needs ptys, not filed separately |
| BUG-12 `initgroups` failure unreported | noted in #25 as a related stub hardening |
| RISK-09 `close_range` `ENOSYS` | not filed; one-line fix, mentioned in the PR notes |
| RISK-10 hardcoded `open_tree`/`move_mount` | not filed; use `libc::SYS_*` |
| RISK-11 `EINTR` not retried | not filed; same one-line shape as `read_file` |
| RISK-12 "created but not visible" | folded into #20 |
| RISK-15 pidfile without a boot header | not filed; fits #1's neighbourhood |
| RISK-16 RSS above MemTotal | #28 |
| RISK-17 PIDs above `pid_max` | #28 |
| RISK-19 unbounded `--utc-offset` | #17 |
| RISK-20 unbounded `log --lines` | #24 |
| RISK-21 `load --mem-util` can OOM | #23 |
| RISK-22 fd 3 config trust | #25 |

Spot-checked NITs, with results: NIT-02 (`table --live` prints all 562 rows, `table`
prints 30 — REPRO), NIT-11 (`log -n 5` prints 67 lines — REPRO), NIT-21 (empty
`.psgen-stubs` survives `stop` — REPRO), NIT-33 (`--help --bogus` → rc 2 — REPRO),
NIT-43 = F-16, NIT-24 = F-10.

## Triage: new issues

Filed: **#14–#28**. Each is standalone, has a repro or an explicit CODE-READ label, a
patch where the patch is small enough to verify, a test that would have caught it, and
the exact environment.

| Issue | Severity | Patch in PR | Verified |
| --- | --- | --- | --- |
| #14 `getrandom` panic after spawn | P0 destructive | yes | repro |
| #15 `stop --dry-run` kills | P0 destructive | yes | repro |
| #16 `-n` shortfall / padding | P1 | warning only | repro |
| #17 `--local` + `--utc-offset`, unbounded offset | P1 | yes | repro |
| #18 postgres session leaders | P1 | yes | repro |
| #19 seccomp probe errnos | P2 | yes | repro |
| #20 `--create-users` group-only host | P1 | yes | code-read |
| #21 `check --binaries` misreports | P2 | yes | repro |
| #22 one-field pidfile cannot stop bind/shadow | P2 | doc only | repro |
| #23 `load --mem-util` can OOM | P2 | no | code-read |
| #24 `log --lines` / `EPIPE` | P2 | no | code-read |
| #25 fd 3 config trust | security | no | code-read |
| #26 pidfile basename near NAME_MAX | P2 | yes | repro |
| #27 unit cannot be restarted | P2 | partial | repro + guest |
| #28 impossible values in `table` | P2 | no | repro |

Not re-filed because already tracked: F-01→#2, F-03→#1, F-10→#3c, F-17/F-31/F-35→#4,
F-19→#5, #6a–c→#6, F-16→#13.1, F-21→#13.3, F-28→#13.5, F-37→#11, F-22→#9,
doctor JSON→#8. Each got a comment with fresh evidence where I had it (#1, #2, #3, #4,
#5, #6, #10).

## The pull request

Branch `qa/verified-fixes` in `Nemo-010/psgen`, nine commits by area. `git diff
main..HEAD` is byte-identical to the report's patch except for the three
`tests/invariants.py` hunks from #6.

Closes #1, #2, #4, #6, #14, #15, #17, #18, #19, #20, #21, #26.

Review passes done on it:

1. **Content equality**: diffed the branch against the report's patch — 589 changed
   lines both sides, identical set. Nothing added or dropped by accident.
2. **Per-commit build**: checked out each of the nine commits and ran
   `cargo build --release`; all clean, so it bisects.
3. **Suite**: `sh tests/smoke.sh` → `invariants: ok`, `smoke: ok` (including the
   patch's own `check_regressions()`).
4. **Differential**: 250 randomised invocations across every subcommand, upstream vs
   branch — zero exit-code mismatches; plus 9 targeted before/after assertions
   (seed, `stop --dry-run`, seccomp verdict, `%MEM`, local/utc, postgres, `check`,
   unit, 40-row stop).
5. **Operational**: three 40-row stop cycles all report 40; pidfile reuse works twice;
   `stop` still removes its temp dirs; `cargo clippy` still reports the same 23
   warnings as `bf80156`; `cargo fmt --check` fails identically before and after (the
   hits are pre-existing, in files the PR does not touch).

Left out on purpose, with the reasoning in the issue: #16's count semantics, #22's
one-field form, #23/#24/#25/#27/#28. Those need a decision, not a patch.

## Cage

Unprivileged (uid 966), no mount namespaces, no `unshare`, no `/dev/ptmx`, `ptrace`
blocked. Substitutes used, kept in `/workspace/qa`:

- `sg.c` — `libseccomp` launcher returning `EPERM` for `getrandom`; stands in for
  `strace -e inject=getrandom:error=EPERM`, which needs `CAP_SYS_PTRACE`.
- `sk.c` — `KILL_PROCESS` filter for `memfd_create`, `bpf`, `seccomp`; reproduces the
  "probe kills psgen" case that needs a hardening profile.
- `cmds.txt`, `ev/` — the differential matrix and the captured runs.

## Files

| Path | Contents |
| --- | --- |
| `psgen-qa-round3-report.md` | this report |
| `TRIAGE.md` | the same triage, kept as a working note |
| `ISSUES/*.md` | the 15 issue bodies exactly as filed |
| `qa/` | the verification harness and captured evidence |
| `report/psgen-report/` | the submitted bundle, extracted |
| `psgen/` | the clone; branch `qa/verified-fixes` is the PR |
