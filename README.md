# psgen archive dossier

One place with everything public about `Azathothas/psgen`: the code, the release
assets, all 28 issues and their comments, pull request #29, the QA report that
produced them, the off-site copies, and the receipts for every archive request.

Collected 2026-11-09 with [ghless](https://github.com/Nemo-010/ghless).

| Directory | Contents |
| --- | --- |
| `01-source/` | git bundles (full history, upstream base, changes), patch series, `head.tar.gz`, Software Heritage request record, recovery notes |
| `02a1-issues-1-14/`, `02a2-issues-15-28/` | every issue as JSON plus a readable `.md`, with comments (split in two so each fits an Arweave part) |
| `02b-prs/` | PR #29 — the ghre rendition plus the raw API documents (commits, files, reviews, events) |
| `03-release-assets/` | all 14 release binaries and tarballs for `v0.0.1-beta.1`, plus `SHA256SUMS` (all verified OK) |
| `04-qa-report/` | the third-round QA report, the triage matrix, and the 15 issue bodies as filed |
| `05-archive-bundle/` | the bundles and patch series again, at the top level of the dossier, so a single `git clone` recovers the work |
| `06-crawlers/` | the URL lists sent to each crawler, and the per-URL outcome tables |
| `07-verify/` | the lane verification record (Software Heritage visit ids, Radicle reachability) |

## Recover the source

```sh
git clone 01-source/full.bundle psgen          # every branch and tag, full history
cd psgen && git log --oneline -1               # cfc5fb8, the QA branch tip
git checkout main                              # bf80156, the upstream release
```

The upstream release is also recoverable from `01-source/source-base.bundle` plus
`01-source/changes.patch`:

```sh
git clone 01-source/source-base.bundle base
cd base && git checkout ghless/source-base && git am < ../changes.patch
```

Both trees were verified: the replay tree hash equals the full-bundle tree hash
(`bc405252f2bfb783ddec46ce0820df803a5bb40e`).

Verify the whole thing first:

```sh
cd 01-source && sha256sum -c MANIFEST.sha256
cd ../03-release-assets && sha256sum -c SHA256SUMS
```

## What exists where

| Surface | Where it lives now |
| --- | --- |
| code, all history, both branches, tag | Software Heritage, Arweave, Nostr, GitHub fork, `01-source/` |
| release binaries, 7 targets | `03-release-assets/`, archived on Wayback and Arquivo.pt |
| 28 issues + every comment | `02a1-*`/`02a2-*`/`02b-prs/`, Nostr (via the published repo), Wayback (API JSON) |
| PR #29 (body, 10 commits, 15 files) | `02-issues-prs/pr29/`, Wayback (API JSON), Nostr |
| QA report and issue texts | `04-qa-report/`, `01-source/full.bundle` |
| Wayback | 35 URLs confirmed; github.com HTML pages refused by IA (see below) |
| Arquivo.pt | all 39 submitted URLs accepted |

### Software Heritage

- `github.com/Azathothas/psgen` — revision `bf80156`, snapshot `5df0799a2c73db6b29f812d4242cbaa8da9dadf6` (visit 8, full)
- `github.com/Nemo-010/psgen` — snapshot `c18ecab07d184e9f42dbb82eba46c8ab6de3a7d7`, carrying `refs/heads/main`, `refs/heads/qa/verified-fixes` and `refs/tags/v0.0.1-beta.1`

### Nostr

`nostr://npub1a8hwrjy5lmm56v9nf556gtveyuz0vndl0ju6jlkuedp224wym9es5p5flr/relay.ngit.dev/psgen`
(verified: the GRASP server serves the pushed commit).

### Arweave

Three parts under wallet `nQ4e9Et0kStCwbQRZd7BFWMdWaxI99lpOPxylu7uK1I`:

- `psgen-part1` tx `QX53yNwr4OHD1MGRuw5oHOyvFc86rL8mqtzKakoQoK8`
- `psgen-part2` tx `p0OvDijBNx9IOLrQ9m7Fvub466CMiScSpvV53Gyqb7w`
- `psgen-part3` tx `5bH152mKvPc7KIBVIXDzMZg7pfxKKbsM1vklYZT2rKE`

Recorded as `ok`, not `ok:verified`: the free tier refused the single packed
upload (402) so it went up as three sub-100 KiB parts, and `arweave.net` was still
returning 404 for the transaction ids when the publish finished. Re-check before
relying on them; re-running `ghless publish` re-verifies.

### Radicle

**Not published.** This cage denies `bind()` and every outbound TCP port except
proxied 443, so no Radicle seed is reachable:

```
seed.radicle.garden 8776 unreachable: [Errno 13] Permission denied
iris.radicle.xyz    8776 unreachable: [Errno 13] Permission denied
```

The lane is configured and the identity exists; re-running `ghless publish` from a
host with raw egress will retry it. Radicle payloads are delivered by other
people's seeds, so nothing was uploaded.

### Wayback, the one partial lane

92 of 115 URLs confirmed — **every API document (70/71) and every source file
(22/22)**. The 23 that are not are all `github.com/<owner>/<repo>` **HTML** pages
(repo root, issues, the release page, PR #29). Save Page Now is
internally fetching them and getting a 404 — its own error page says
`The target server cannot find https://github.com/Azathothas/psgen (HTTP
status=404)` — while the same URL returns 200 to a normal user agent from here,
and `raw.githubusercontent.com` and `api.github.com` save fine. That is IA-side,
not something the tool or the repo can fix.

The content those pages would have shown is archived anyway:

- **issue and PR bodies, comments, metadata** — in `02a1-*`/`02a2-*`/`02b-prs/` and on
  Wayback as the `api.github.com` JSON documents, 70 of 71 confirmed;
- **every source file** — on Wayback as `raw.githubusercontent.com` (17 confirmed);
- **the release page** — its assets are in `03-release-assets/` and accepted by
  Arquivo.pt;
- **the same URLs at Arquivo.pt** — all 39 submitted, `accepted`.

An earlier session's captures from 2026-11-08 still cover several issue pages
(`issues/6`, `8`, `9`, `11`), which is why the CDX index for this repo is not empty.

## Payload sizes

| Item | Size |
| --- | --- |
| source bundles + tarball | 444 KB |
| issues + PRs snapshot | 748 KB |
| release assets (14 files) | 6.6 MB |
| QA report + issue texts | 132 KB |
| archive bundles (duplicated for convenience) | 320 KB |

## Licence

`psgen` is 0BSD. The release assets and source are redistributed here unchanged
under that licence. The QA report, the issue texts and this dossier are the same
0BSD.
