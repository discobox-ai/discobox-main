# discobox-main

This repository holds only releases built from `main` of
[discobox-ai/discobox](https://github.com/discobox-ai/discobox). It has no
source code; the source, issues and pull requests live in
[discobox-ai/discobox](https://github.com/discobox-ai/discobox).

Each green commit on discobox `main` is built (coalesced, so not every commit)
and published here as a release named `v<next patch>-main.<N>.g<sha7>`, marked
latest. Builds older than fourteen days are deleted; the newest is always kept.
Every release's tag points at this placeholder commit, so the tag itself carries
no meaning: the source commit is in the version and the release notes.

Install the newest main build with:

```sh
curl -sSfL https://main.discobox.ai | sh
```

or pass `--channel main` to the installer. Main builds are for trying main,
not for staying on a commit.

See ADR
[26-11-16-249](https://github.com/discobox-ai/discobox/blob/main/docs/adr/26-11-16-249-main-is-a-channel-built-into-its-own-repository-and-latest-is-no-channels-name.md)
for the design.
