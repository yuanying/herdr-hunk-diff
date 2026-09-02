# About this fork

A fork of [jhochenbaum/herdr-hunk-diff](https://github.com/jhochenbaum/herdr-hunk-diff),
carrying one feature on top of upstream releases.

## Why

`review:branch` builds its range as `<base>...HEAD`. A three-dot range compares two
commits, so uncommitted edits and untracked files never appear in the review — which is
exactly what you want to look at when reviewing what an agent just wrote.

The plugin offers no way to change that range: `takesRef` on a review action controls
whether the pane recovers a ref from the review index, not whether a caller can supply
one. Driving `hunk diff <base>` directly from a herdr keybinding avoids the range problem
but loses everything the plugin manages — `send-review` cannot find the agent pane,
`close-review` and `reload` have no record to work from, and a `type = "pane"` binding
has no `placement` option so it opens fullscreen.

Hence the fork.

## What it adds

`review.branch_scope`, a `[review]` setting:

| Value               | Range           | Includes                              |
| ------------------- | --------------- | ------------------------------------- |
| `commits` (default) | `<base>...HEAD` | committed work only                   |
| `worktree`          | `<base>`        | committed, uncommitted, and untracked |

It applies to `review:branch` and to the branch diff `auto` selects, so both agree.
The default preserves upstream behaviour.

Everything lives in one commit touching `src/config.ts`, `src/target.ts`,
`tests/config.test.ts`, `tests/target.test.ts` and `README.md`. The mode stays `branch`,
so no downstream module changes.

## Versioning

Tags are `v<upstream version>-fork.<N>`:

```
v0.1.0-fork.1     upstream 0.1.0, first fork build
v0.1.0-fork.2     same upstream, fork-only change
v0.2.0-fork.1     followed upstream 0.2.0, N resets
```

`.github/upstream-release` records which upstream release `main` currently sits on.

`package.json` and `herdr-plugin.toml` keep upstream's version. Upstream bumps those on
every release, so editing them here would guarantee a conflict every single time. The
fork's version lives only in the tag.

### Do not mark releases as pre-release or draft

The consumer ([yuanying/devbox](https://github.com/yuanying/devbox)) tracks this fork with
Renovate's `github-releases` datasource:

- a **draft** release is ignored entirely
- a **pre-release** is `isStable: false` and gets filtered by `ignoreUnstable`

Either one silently stops update PRs from appearing. That is why `release.yml` creates
releases rather than leaving it to hand.

Renovate is told to read `fork.<N>` as a `build` component rather than a semver
prerelease, via `versioning=regex:` in the consumer's Dockerfile annotation. A plain
semver reading would treat `v0.1.0-fork.1` as older than `v0.1.0` and would refuse to
move to `v0.2.0-fork.1`.

## Operating it

**Upstream released something** — `sync-upstream.yml` runs weekly (or on demand). It
merges the upstream _tag_ into a branch, runs the full check, and then either releases
automatically or stops and opens a PR:

| Situation               | What happens                                                  |
| ----------------------- | ------------------------------------------------------------- |
| merge conflict          | job fails with resolution steps in the summary                |
| check/build/test fails  | draft PR, nothing released                                    |
| **fork delta is empty** | PR, nothing released — upstream may have absorbed the feature |
| otherwise               | merges to `main`, tags, and releases                          |

**Changing the fork's own code** — commit to `main`, then run **Release fork build**
manually. `N` increments on its own.

Releasing is safe to automate because a release only reaches a machine after a Renovate PR
is merged in devbox and its image is rebuilt by hand. Both are human decisions.

**`SYNC_TOKEN`** — `sync-upstream.yml` runs on a repository secret of that name rather
than on `GITHUB_TOKEN`. Upstream changes `.github/workflows/ci.yml` on most releases, and
GitHub refuses any push from a GitHub App token whose commits touch `.github/workflows/` —
which `GITHUB_TOKEN` is. There is no way around it from inside the workflow: `workflows`
is not one of the permissions the `permissions:` key can grant. Without the secret the
sync job stops on its first push with `refusing to allow a GitHub App to create or update
workflow`. Make it a fine-grained PAT scoped to this repository alone:

| Permission    | Access         |
| ------------- | -------------- |
| Contents      | Read and write |
| Pull requests | Read and write |
| Workflows     | Read and write |

```
gh secret set SYNC_TOKEN --repo yuanying/herdr-hunk-diff
```

Fine-grained tokens expire; when this one does, the sync fails on the `Require SYNC_TOKEN`
step. `release.yml` needs none of this — it only pushes a tag pointing at a commit `main`
already has, so `GITHUB_TOKEN` is enough there.

## Retiring the fork

If upstream gains an equivalent setting:

1. In devbox, point the annotation back at `jhochenbaum/herdr-hunk-diff`, drop the
   `versioning=regex:` part, and set the ARG to the upstream version. Renovate cannot do
   this hop for you because the versioning scheme changes.
2. Rename the setting in the plugin config if upstream chose a different name. If it
   matches, nothing to do — `pick()` falls unknown keys back to the default, so a leftover
   key is inert.
3. Archive this repository. Existing tags stay resolvable, so old images still build.

The plugin id is deliberately unchanged, so no keybinding or config path moves.

To offer the feature upstream instead, branch from `upstream/main` and cherry-pick the
feature commit — it is kept as a single commit for exactly this reason.
