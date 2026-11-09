`-n/--count` is documented as "rows or commands", but for four of the ten families the request is silently under-delivered, and for the families that do reach the count the remainder is filled with unlimited duplicates of one template. Nothing warns.

## What happens

Families whose templates all have `inst <= 1` cannot reach the requested count. The generator spins until its guard expires and returns fewer rows:

```
$ for f in mail mon sto net; do
    printf '%-5s inject -n 50 -> %s rows; table -n 50 -> %s; list -n 50 -> %s\n' \
      $f "$(psgen inject -f $f -n 50 --dry-run | grep -c .)" \
         "$(psgen table -f $f -n 50 | tail -n +2 | wc -l)" \
         "$(psgen list -f $f -n 50 | wc -l)"
  done
mail  inject -n 50 -> 8 rows; table -n 50 -> 8; list -n 50 -> 8
mon   inject -n 50 -> 16 rows; table -n 50 -> 16; list -n 50 -> 16
sto   inject -n 50 -> 9 rows; table -n 50 -> 9; list -n 50 -> 9
net   inject -n 50 -> 15 rows; table -n 50 -> 15; list -n 50 -> 15
```

`mail` has 8 templates, `sto` 9, `net` 15, `mon` 16, and every one of them has
`inst = 1`, so the uniqueness check rejects all of them after the first pass.

When a family *does* have a repeatable template, the same loop pads the count by
repeating it without limit. `sys` has exactly one (`sshd: {user} [priv]`, `inst = 2`):

```
$ psgen table -f sys -n 200 | tail -n +2 | wc -l
200
$ psgen table -f sys -n 200 | tail -n +2 \
    | awk '{c=""; for(i=11;i<=NF;i++) c=c" "$i; print c}' | sort -u | wc -l
88
$ psgen table -f sys -n 200 | tail -n +2 | grep -c 'sshd: user \[priv\]'
120
```

120 of the 200 rows are the same two commands repeated. `dev` is excluded from the
uniqueness check entirely, so `table -f dev -n 5000` repeats the 33 fixed dev
commands ~150 times each.

The help says `-n, --count N  rows or commands (list: all templates, table: 30, inject: 12)`;
the caps are documented (2000 / 5000 / 500) but the shortfall and the padding are not.

## Why

`src/gen.rs`, `gen_table()` (same shape in `gen_inject()` and `gen_list()`):

```rust
while rows.len() < want && guard < want * 40 + 1000 {
    let idx = pick_index(rng, &weights, opts.family, &used);
    if corpus::CORPUS[idx].inst <= 1
        && corpus::CORPUS[idx].fam != corpus::FAM_DEV
        && used.contains(&idx)
    {
        guard += 1;
        continue;
    }
```

`inst` is doing two jobs: "how many instances this template represents" (a real
`nginx: worker` count) and "may this template be picked again". `inst = 2` therefore
means both "emit two rows together" and "an unlimited repeat licence", which is why
`sys` fills up with `sshd ... [priv]` pairs. Families with no `inst > 1` template
have no repeat licence at all, so they simply run out.

## Patch (warning only, verified)

The count semantics need a decision (below), but the silence does not. Warn on
stderr whenever a family supplies fewer rows than requested:

```diff
--- a/src/cli.rs
+++ b/src/cli.rs
@@ -886,6 +886,18 @@ pub fn run() -> i32 {
             let mut cmds = gen::gen_list(&mut rng, &opts);
+            if let Some(f) = args.family {
+                let want = args.count.min(2000);
+                if args.count > 0 && cmds.len() < want {
+                    let msg = format!(
+                        "psgen: family {} supplies {} rows; {} were requested\n",
+                        corpus::family_name(f),
+                        cmds.len(),
+                        want
+                    );
+                    sys::err(msg.as_bytes());
+                }
+            }
```

(the same block in the `table` and `inject` arms, with their own caps). Verified:

```
$ psgen list -f mail -n 50 >/dev/null
psgen: family mail supplies 8 rows; 50 were requested
$ psgen table -f mail >/dev/null
psgen: family mail supplies 8 rows; 30 were requested
$ psgen list -f sys -n 50 >/dev/null
(nothing: sys can satisfy the request)
```

## Decision needed

The warning is the floor, not the fix. Pick one:

1. **Short return, exit 0** (what the warning implies). Update the help to say the
   count is a maximum and that a family may supply fewer.
2. **Error**, exit 2, when the family cannot supply `-n`. Consistent with the other
   preflight refusals, but breaks scripts that ask for more than a family has.
3. **Add `F_REPEAT`** to the templates that may legitimately appear more than once
   (nginx workers, `sshd [priv]`, containerd shims) and stop using `inst > 1` as a
   repeat licence. This is the only option that fixes the `sys` padding as well, and
   it is the one I would pick: it separates the two meanings of `inst` and makes the
   count honest for every family.

Under option 3, `sys -n 200` would warn and return ~80 rows rather than 200 rows of
which 120 are duplicates. If padding is wanted, `--repeat` can allow it explicitly.

## Test that would have caught it

```python
# a family that cannot supply the count must say so, and must not pad
for fam, n in (("mail", 50), ("mon", 50), ("sto", 50), ("net", 50)):
    p = run(["table", "-f", fam, "-n", str(n)])
    rows = p.stdout.splitlines()[1:]
    if len(rows) >= n:
        fail(f"{fam} claimed {n} rows")
    if "supplies" not in p.stderr:
        fail(f"{fam} shortfall was silent")
# and the padding case
rows = run(["table", "-f", "sys", "-n", "200"]).stdout.splitlines()[1:]
cmds = [" ".join(r.split()[10:]) for r in rows]
if len(set(cmds)) < len(cmds) // 2:
    fail(f"sys -n 200 is {len(cmds)-len(set(cmds))} duplicate rows")
```

## Environment

`bf80156` built with `cargo 1.99.1`, x86-64, Linux 7.2.11, unprivileged uid 966.
Family template counts from `src/corpus.rs`: `mail` 8, `sto` 9, `net` 15, `mon` 16,
`sys` 32 (one with `inst > 1`), `dev` 33 (excluded from the check).
