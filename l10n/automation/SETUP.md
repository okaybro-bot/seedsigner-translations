# l10n translations bridge: operator setup

`l10n-translations-bridge.yml` brings Transifex translations into this repository
as one pull request per locale. It runs in a bot-owned fork of this repo, never in
this repo itself: it stays inert here because it activates only when
`L10N_TRANSLATIONS_UPSTREAM` is set and a push lands on the Transifex write branch,
neither of which is true upstream.

All configuration below therefore lives on the BOT FORK, never on this repository.
Setting these on the upstream repo would make it try to open pull requests into
itself.

## Topology

- `SeedSigner/seedsigner-translations` (this repo): the upstream the bridge opens
  PRs into. The automation never gains write access to it.
- `<bot>/seedsigner-translations`: a real fork (same fork network, required for
  cross-fork PRs) where the bridge runs. Transifex commits translated catalogs here
  in direct-commit mode, on the write branch.

The source `l10n/messages.pot` is mirrored into the bot fork's default branch by the
main repo's automation; Transifex reads its source from there.

Flow: Transifex writes `l10n/<locale>/LC_MESSAGES/messages.po` to the bot fork's
write branch -> the bridge compares it against this repo's base branch -> for each
locale whose translations changed it opens or updates one PR here; locales that have
converged with upstream have their PR closed.

For testing as an external contributor, stand everything up under your own accounts:
your own fork of this repo plays the upstream, and a bot account's fork plays
`<bot>/seedsigner-translations`.

## Repository variables (on the bot fork)

Settings -> Secrets and variables -> Actions -> Variables. All three are required.

| Variable | Example | Notes |
| --- | --- | --- |
| `L10N_TRANSLATIONS_UPSTREAM` | `SeedSigner/seedsigner-translations` | The repo PRs are opened into. Empty means the bridge no-ops. |
| `L10N_TRANSLATIONS_BRANCH` | `dev` | The upstream base branch PRs target. |
| `L10N_TRANSLATIONS_WRITE_BRANCH` | `transifex-writes` | The branch Transifex commits to. Must equal the literal `branches:` filter in the workflow's `on.push` (GitHub Actions cannot read variables in `on:`). |

The bot fork itself is not configured anywhere: the workflow derives it from the
repository it runs in. Locales are not configured either; they are discovered by
scanning `l10n/*/LC_MESSAGES/messages.po`, so a new language needs no edits.

## Repository secrets (on the bot fork)

The bridge authenticates as a GitHub App (no long-lived PATs) and mints short-lived,
least-privilege tokens at runtime. Two logical roles:

- Fork role: `contents: write` on the bot fork; pushes the rolling branches.
- PR role: `pull-requests: write` on the upstream repo; opens the PRs.

| Secret | Value |
| --- | --- |
| `L10N_TR_FORK_CLIENT_ID` | Fork App Client ID |
| `L10N_TR_FORK_PRIVATE_KEY` | Fork App private key (`.pem` contents) |
| `L10N_TR_PR_CLIENT_ID` | PR App Client ID |
| `L10N_TR_PR_PRIVATE_KEY` | PR App private key (`.pem` contents) |

### Production: two Apps

Because no upstream repo may ever be granted `contents: write`, production uses two
separate Apps: a PR App carrying only `Pull requests: Read & write` installed on the
upstream repo, and a Fork App carrying only `Contents: Read & write` (plus
`Workflows: Read & write` if the fork can fall behind upstream) installed on the bot
fork. The upstream App never holds `contents` access at all.

### Testing: one App

For a single-maintainer test you may install one App on both stand-in repos, with
`Contents`, `Pull requests`, `Metadata` (and `Workflows`) read/write, and put its
Client ID and key in all four secrets. The workflow still down-scopes each minted
token (the PR token to pull-requests, the fork token to contents), so behavior
matches production; the only relaxation is that the single test App could write more
than it is asked to.

## Transifex

Configure the official Transifex GitHub integration against the bot fork in
direct-commit mode:

- Source: the repo-root `l10n/messages.pot` (mirrored in by the main repo's
  automation). The checked-in `.tx/config` expresses the source path relative to a
  submodule mountpoint; the GitHub integration roots at the repository, so configure
  its source as the repo-root `l10n/messages.pot`.
- Translations: `l10n/<lang>/LC_MESSAGES/messages.po`, with the `zh-Hans ->
  zh_Hans_CN` language map, committed to the write branch.

## First-time PR CI approval

This repo's `tests.yml` runs on `pull_request`. The first PR from the bot fork hits
GitHub's first-time-contributor gate; a maintainer approves it once in the Actions
tab, after which runs from that fork proceed automatically.
