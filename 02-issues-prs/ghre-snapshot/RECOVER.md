# Recovering Azathothas/psgen from this directory

Every issue and pull request is here twice: `issues/<n>.json` and
`prs/<n>.json` are the API's own answers, and the `.md` beside each is a
readable rendering with the body, the comments, the reviews and the review
comments. `MANIFEST.json` holds a SHA-256 per rendering, so a copy that lost
bytes can be spotted rather than trusted.

Nothing here needs the network:

- to read an issue, open its `.md`;
- to refile one, take the `## Body` section of its `.md` as the body and the
  `## Comments` section as the discussion;
- to check the archive is intact, run `ghre snapshot <repo> --out <dir> --verify`.

## Capture state

The archive was not asked for captures. Run this lane again with
`--save` to put a copy of every page on a host that is not this one.
