# MAIN_RELEASE_TOKEN

`discobox-ai/discobox`'s `main-build.yml` workflow publishes every main build
into this repository. A workflow's own `GITHUB_TOKEN` can only write to its own
repository, so it does this with a separate token, held in the
`MAIN_RELEASE_TOKEN` Actions secret of `discobox-ai/discobox`
([ADR 26-11-16-249](https://github.com/discobox-ai/discobox/blob/main/docs/adr/26-11-16-249-main-is-a-channel-built-into-its-own-repository-and-latest-is-no-channels-name.md)
§4, [discobox-ai/discobox#1280](https://github.com/discobox-ai/discobox/pull/1280)).
If the secret is missing, every main build fails at its first step, before
anything is built.

## What the token does

It is used only against `discobox-ai/discobox-main`, and only for releases:

| Step | Calls |
| --- | --- |
| `release:publish` | `gh release view`, `gh release create --draft` (the tag is created on this repository's default branch), `gh release upload --clobber`, `gh release edit --draft=false --latest` |
| `release:prune-main` | `GET /repos/discobox-ai/discobox-main/releases`, `DELETE …/releases/{id}`, `DELETE …/git/refs/tags/{tag}` |

All of these fall under the **Contents** repository permission. The ghcr image
tags are pruned with the workflow's own `GITHUB_TOKEN`, not this token, so it
needs no package permission.

## Create the token (fine-grained personal access token)

Use an account that is a member of `discobox-ai` with write access to
`discobox-ai/discobox-main`.

### The account needs Write on this repository

A fine-grained token never has more access than the account that owns it: the
**Contents** permission given to the token is checked against the account's
role on the repository, and the token gets only what both allow. **Triage** or
**Read** is not enough. With either, the token can read releases (the check
below still answers 200), but creating, uploading, editing and deleting
releases and deleting tags fail with 403 or 404. The first main build then
fails at `publish`.

For a bot account:

1. Keep it a member of `discobox-ai`, so that it can choose `discobox-ai` as
   the token's resource owner.
2. Give it **Write** on `discobox-ai/discobox-main` only
   (**Settings** → **Collaborators and teams**), directly or through a team
   that has no other repository. Its role on other repositories, including
   `discobox-ai/discobox`, doesn't matter to this token. Storing the secret
   there is an admin's step, not the bot's.
3. Optional: Write also lets the account push to `main` here. A branch
   ruleset on `main` that blocks pushes and deletions stops that without
   affecting main builds. Release creation tags the latest commit of `main`
   but never pushes to it.
4. Sign in as the bot and create the token as below.

1. Open <https://github.com/settings/personal-access-tokens/new>
   (profile picture → **Settings** → **Developer settings** →
   **Personal access tokens** → **Fine-grained tokens** →
   **Generate new token**).
2. **Token name**: `MAIN_RELEASE_TOKEN discobox-main`.
3. **Description**: `discobox-ai/discobox main-build.yml publishes and prunes releases in discobox-ai/discobox-main`.
4. **Resource owner**: `discobox-ai`. The token must belong to the
   organization; one owned by your personal account cannot reach the
   repository.
5. **Expiration**: choose a date, and note it for rotation (below). The
   organization's policy may cap how long the token can live.
6. **Repository access**: **Only select repositories** →
   `discobox-ai/discobox-main`. Select no other repository.
7. **Permissions** → **Repository permissions**:
   - **Contents**: **Read and write**.
   - **Metadata**: **Read-only** (GitHub adds this automatically and requires it).
   - Leave every other repository permission at **No access**, and leave every
     **Organization** and **Account** permission at **No access**.
8. Click **Generate token** and copy the value (`github_pat_…`). GitHub shows
   it only once.
9. If `discobox-ai` requires approval for fine-grained tokens, the token says
   **pending** and can't be used yet. An organization owner approves it at
   **discobox-ai → Settings → Personal access tokens → Pending requests**.
   If the organization blocks fine-grained tokens entirely, an owner has to
   allow them at **Settings → Personal access tokens → Settings** first.

## Store it as the secret

In `discobox-ai/discobox` (not this repository):

1. **Settings** → **Secrets and variables** → **Actions** →
   **New repository secret**.
2. **Name**: `MAIN_RELEASE_TOKEN`.
3. **Secret**: paste the token, then click **Add secret**.

Or, with `gh` logged in as an admin of `discobox-ai/discobox`:

```sh
gh secret set MAIN_RELEASE_TOKEN --repo discobox-ai/discobox
# paste the token when prompted
```

## Check it

The token can read this repository:

```sh
curl -sS -o /dev/null -w '%{http_code}\n' \
  -H "Authorization: Bearer $TOKEN" \
  https://api.github.com/repos/discobox-ai/discobox-main/releases
# 200
```

Then wait for the next green CI run on discobox `main`, or re-run the last
**main build** workflow run in `discobox-ai/discobox` → **Actions**. Its `publish` job
should create a release here, marked **Latest**.

## Rotate it

Before the token expires, generate a new one with the same settings (or use
**Regenerate token** on the existing one), then replace the secret's value as
above. The token belongs to the account that created it: if that account
leaves `discobox-ai` or loses access to this repository, main builds fail at
`publish` until someone else creates a new token.
