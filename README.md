# cg-homelab/workflows

Reusable GitHub Actions workflows shared by the cg-homelab repositories.

| Workflow | What it does |
| --- | --- |
| [`semver-tag.yml`](.github/workflows/semver-tag.yml) | On a push to `main`, finds the merged pull request and pushes an annotated semver tag (`vX.Y.Z`) whose bump level comes from the PR's `release/*` label. |
| [`release-label-check.yml`](.github/workflows/release-label-check.yml) | On a pull request, fails when more than one `release/*` label is set and shows which bump the merge will produce. |

Both are plain bash with the preinstalled `gh` and `jq`; there are no helper
scripts, and the only action the reusable workflows use is `actions/checkout`.

## Label scheme

Put at most one of these labels on a pull request:

| Label | Effect when the PR merges |
| --- | --- |
| `release/major` | `v2.3.4` → `v3.0.0` |
| `release/minor` | `v2.3.4` → `v2.4.0` |
| `release/patch` | `v2.3.4` → `v2.3.5` |
| `release/skip` | no tag |
| none | the caller's `default_bump` (`patch` unless set) |

Rules applied by `semver-tag.yml`:

- `release/skip` wins over everything, also when other `release/*` labels are present.
- Otherwise `major` > `minor` > `patch` > `default_bump`. With more than one
  `release/*` label the run warns and uses the highest; the label check is
  what enforces "one label".
- PRs opened by a bot (`user.type == "Bot"`, e.g. Dependabot) are skipped
  unless they carry an explicit `release/*` label, or the caller sets
  `skip_bots: false`.
- Unknown `release/*` labels (typos such as `release/mayor`) are ignored with
  a warning by the tag workflow and fail the label check.

The workflows do not create labels. Create all four `release/*` labels in
each repository (or as organisation default labels).

## `semver-tag.yml`

Runs on the push produced by the merge. It:

1. Checks out the full history and all tags.
2. Skips if the pushed commit already carries a release tag (re-running a
   successful run is a no-op).
3. Looks up the merged PR with `GET /repos/{repo}/commits/{sha}/pulls`,
   preferring the PR into the pushed branch whose `merge_commit_sha` is the
   pushed commit, otherwise the most recently merged PR into the pushed branch. This works for merge
   commits, squash and rebase merges, and for stacked PRs (only the PR into
   `main` counts). It retries three times because the association can lag the
   merge. No PR found (a direct push) means skip.
4. Decides the level from the labels (see above).
5. Finds the previous tag: the highest strict `PREFIX<major>.<minor>.<patch>`
   tag reachable from the pushed commit (`git tag --merged`), by version sort
   (`v10.0.0` > `v9.0.0`). Pre-release and non-semver tags (`v1.0.0-rc1`,
   `nightly`) are ignored, and so are tags on other branches or on newer
   commits. If the highest release tag in the whole repository is a different
   one, the run warns and names both. No tag yet means `PREFIX0.0.0`. (Running
   for an old commit after a newer one was tagged therefore computes a tag that
   already exists and fails at the next step instead of tagging the older
   commit with a higher version.)
6. Pushes an annotated tag on the pushed commit, tagged by
   `github-actions[bot]`, with the message `vX.Y.Z: PR #N <title>`. If the tag
   already exists on the remote the run fails instead of moving it.
7. Optionally (`create_release: true`) creates a GitHub Release with
   generated notes, starting from the previous tag. This also runs when the
   commit was already tagged, and does nothing if the release exists, so
   re-running a run whose tag was pushed but whose release failed creates the
   missing release.
8. Writes a summary (PR, labels, level, previous → new tag, or the skip reason).

Every skip ends green with a notice and still sets all outputs.

### Inputs

| Input | Type | Default | Meaning |
| --- | --- | --- | --- |
| `default_bump` | string | `patch` | Bump when the PR has no `release/*` label: `major`, `minor` or `patch`. |
| `create_release` | boolean | `false` | Also run `gh release create <tag> --verify-tag --generate-notes` (skipped if the release exists; also on an already-tagged commit). |
| `skip_bots` | boolean | `true` | Skip bot-authored PRs that have no explicit `release/*` label. |
| `tag_prefix` | string | `v` | Prefix for new tags and filter for existing ones. Empty, or a letter followed by letters, `_` and `-` (so a tag can never start with `-`). |

### Outputs

| Output | Value |
| --- | --- |
| `tag` | The new tag. On the "already tagged" skip, the existing tag. Otherwise empty. |
| `previous_tag` | The previous release tag reachable from the commit (`PREFIX0.0.0` if none); empty when skipped. |
| `level` | `major`, `minor` or `patch`; empty when skipped. |
| `skipped` | `true` when no new tag was created, else `false`. Stays `true` on an already-tagged commit even if the run created the missing release. |

### Caller

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]

permissions:
  contents: write       # push the tag (and create the release)
  pull-requests: read   # look up the merged PR

jobs:
  tag:
    uses: cg-homelab/workflows/.github/workflows/semver-tag.yml@v1.0.0
    # with:
    #   default_bump: patch
    #   create_release: true
    #   skip_bots: true
    #   tag_prefix: v
```

Using the outputs in a later job:

```yaml
  publish:
    needs: tag
    if: needs.tag.outputs.skipped == 'false'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Released ${{ needs.tag.outputs.tag }}"
```

The job runs in the concurrency group `semver-tag-<owner>/<repo>-<tag_prefix>`,
so callers with different prefixes in one repository do not queue behind or
cancel each other. Do not use that group name on your own jobs.

## `release-label-check.yml`

Reads the labels from the pull request event payload (no API calls, no
permissions needed), fails if more than one `release/*` label is present
(`release/skip` counts) or a `release/*` label is unknown, and writes the
bump the merge will produce to the step summary, for example "minor" or
"skip (no tag)". It has no inputs; for an unlabelled PR it reports the default
bump, since it cannot see the tag workflow's `default_bump`.

### Caller

```yaml
# .github/workflows/label-check.yml
name: Label check

on:
  pull_request:
    types: [opened, synchronize, reopened, labeled, unlabeled]

permissions:
  contents: read

jobs:
  release-label:
    uses: cg-homelab/workflows/.github/workflows/release-label-check.yml@v1.0.0
```

The `labeled` and `unlabeled` types are what make the check re-run when the
labels change. Runs for the same PR share the concurrency group
`release-label-check-<PR number>` and a newer run cancels an older one, so
rapid label changes cannot finish out of order. To block merging on it, require the `release-label / Release
label` check in a branch ruleset.

## Permissions

A reusable workflow cannot grant itself more than the caller's token has.
The caller must set:

| Workflow | Caller permissions |
| --- | --- |
| `semver-tag.yml` | `contents: write`, `pull-requests: read` |
| `release-label-check.yml` | none beyond the default read (`contents: read` is enough) |

If the caller grants less, the run fails at startup with a message like
"The nested job 'tag' is requesting 'contents: write, pull-requests: read',
but is only allowed ..." listing the requested permissions.

## Limitations

- **Tags do not trigger other workflows.** The tag is pushed with the
  built-in `GITHUB_TOKEN`, and events created with that token do not start new
  workflow runs, so an `on: push: tags:` workflow will not fire. The same
  holds for the Release created with `create_release: true`: `on: release:`
  workflows do not fire either. Chain work through the `outputs` in the same
  caller workflow instead (see above).
- **PRs merged by a workflow are never tagged.** If another workflow merges a
  PR with `GITHUB_TOKEN` (for example `gh pr merge` in an auto-merge job), the
  merge produces no push run at all. GitHub's native auto-merge, enabled by a
  person, is attributed to that person and is tagged normally.
- **Fast merges are batched.** Runs queue on the concurrency group. If a third
  merge arrives while one run is tagging and a second is waiting, GitHub
  cancels the waiting one. The run that survives tags its own commit, which
  also contains the cancelled PR's changes, so nothing is left unreleased;
  the batch just gets one bump instead of two. The cancelled PR's label is not
  honoured: a cancelled `release/major` PR behind a surviving unlabelled PR
  yields a patch. Either accept the lower bump, or put the higher label on the
  surviving PR, delete its tag and re-run that run.
- **The label check can go stale.** A label added or removed by another
  workflow using `GITHUB_TOKEN` does not trigger a `labeled`/`unlabeled` run,
  so the check keeps its previous result until the PR is updated.

## Pinning and updates

Pin consumers to an exact tag (`@v1.0.0`), not a branch, and let Dependabot
raise the pin. Dependabot's `github-actions` ecosystem updates `uses:`
references to reusable workflows as well as actions (but not `docker://`
references, which is why the actionlint image pin in this repo's `ci.yml` is
bumped by hand):

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

A major version of this repo means a breaking change to an input, an output,
the label scheme or the required permissions; read the release before merging
such a bump.

## Versioning of this repository

This repository tags itself with its own workflow:
[`release.yml`](.github/workflows/release.yml) calls
`./.github/workflows/semver-tag.yml` by local path (so no bootstrap tag is
needed) and [`label-check.yml`](.github/workflows/label-check.yml) runs the
label check on its own PRs. [`ci.yml`](.github/workflows/ci.yml) runs
[actionlint](https://github.com/rhysd/actionlint) on every push to `main` and
every PR.

The first PR carries `release/major`, which makes the first tag `v1.0.0`; that
is the version consumers pin. After that, label each PR as described above.

To lint locally:

```sh
docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:latest -color
```
