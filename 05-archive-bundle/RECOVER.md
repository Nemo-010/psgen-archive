# Recovering psgen

Archived 2026-11-09T06:36:47Z from /workspace/psgen at commit cfc5fb804c2cf705a81cc0374e6066ca8b41ddf2.
Upstream: https://github.com/Azathothas/psgen.git

    git clone full.bundle psgen          # every branch and tag, full history
    git clone source-base.bundle base    # the upstream commit you started from (if present)
    git -C base checkout ghless/source-base
    git -C base am < ../changes.patch    # replay your changes on that base (if present)
    tar xzf head.tar.gz                  # HEAD without history

If uncommitted.patch or untracked.tar.gz exist, the checkout had changes that were never committed:
    git -C psgen apply ../uncommitted.patch ; tar xzf untracked.tar.gz -C psgen

Verify first: sha256sum -c MANIFEST.sha256
