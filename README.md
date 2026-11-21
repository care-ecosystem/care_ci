# care_ci

Shared CI for the CARE plugin fleet. Today that is one thing: a reusable PR reviewer that every
plugin repo calls in ~20 lines.

This repo holds **mechanism** only. The review rules live in the private
`care-ecosystem/care_reviewer` and are checked out at run time, so changing a rule never touches
a workflow file here.

## Three tiers

| Tier | Where | Changes when |
|---|---|---|
| Mechanism | **here** (public) | the CI plumbing changes — cut a release (below) |
| Judgment | `care-ecosystem/care_reviewer` (private) | a review rule changes — takes effect immediately, fleet-wide |
| Local context | each plugin repo's `CLAUDE.md` | that plugin's shape changes |

This repo is public because GitHub only lets a *private* repo's reusable workflows be called by
other *private* repos in the same org, and most CARE plugin repos are public. It contains no
secrets.

> **Status: pilot, `0.0.1-beta.1`.** Callers reference `@dev`, so changes land immediately and
> nothing is pinned yet. The plugin repos are deliberately untouched.

## Onboarding a repo

1. Copy `templates/claude-review.yml` to `.github/workflows/claude-review.yml` and set `layer:`
   to `backend` or `frontend`.
2. Add a `CLAUDE.md` at the repo root, from `care_reviewer/templates/`. Follow
   `care_reviewer/rules/claude-md-guidelines.md` — under 60 lines, facts the code does not state
   plainly, and the deliberate deviations the reviewer should stop flagging.
3. Confirm the org secrets `ANTHROPIC_API_KEY` and `CARE_REVIEWER_TOKEN` reach the repo.

## What it does and does not do

- Posts inline comments per finding, plus one sticky summary comment that updates on each push.
- **Never blocks a merge.** The review step is `continue-on-error`, and the job is green even
  when the review is skipped or the API call fails.
- Skips drafts, bot-authored PRs, lockfile-only changes, and diffs over 2000 changed lines.
- **Does not review fork PRs.** GitHub withholds secrets from fork-PR runs on public repos, so
  the key is absent and the run cannot proceed. The alternative, `pull_request_target`, puts
  untrusted code on a runner that holds the credentials; not worth it here.
- Has no memory across pushes. It will not track which of its earlier findings were fixed.

## Prerequisites (org-level, once)

- The [Claude GitHub App](https://github.com/apps/claude) installed on `care-ecosystem`. The
  action authenticates its comment writes as this App, which is why the workflow needs only
  `pull-requests: read`.
- `ANTHROPIC_API_KEY` — an Anthropic Console API key, as an org Actions secret. An API key
  rather than an OAuth token because the secret is shared across repositories and an OAuth token
  is tied to one person's subscription.
- `CARE_REVIEWER_TOKEN` — a fine-grained PAT with `Contents: Read` on `care_reviewer` and
  nothing else. It expires; an expired token turns every review run red.

## Releases

Work lands on `dev`, and for now releases are cut from `dev` too — bump `VERSION`, push, and
the tag appears. The version comes from the `VERSION` file, so the tag can never disagree with
the tree.

```
dev  ──(VERSION bumped)──▶  v0.0.1-beta.1  +  beta
```

Releasing off `dev` is deliberate at this stage: there is no stable branch to protect yet, and
requiring a merge to `main` for every beta would add a step that buys nothing. Point
`release.yml` at `main` once the reviewer stabilises — the same moment the version drops its
`-beta` suffix and callers repin from `beta` to `v1`.

Two tags come out:

| Tag | |
|---|---|
| `v<VERSION>` | immutable — pin here for a reproducible run |
| `beta` | moving, points at the newest prerelease — what the fleet follows during the pilot |

A hyphen in `VERSION` (`0.0.1-beta.1`) marks it a GitHub prerelease. Dropping the hyphen
(`0.1.0`) makes it a normal release and `beta` stops moving — that is the natural moment to add
a moving `v0`/`v1` alias and repin callers to it.

Re-running the release on an existing version fails rather than overwriting the tag: bump
`VERSION` to release.

### Which ref should callers use?

| Stage | `uses: ...@` | Why |
|---|---|---|
| Now (pilot) | `dev` | tuning the rules; no release step per change |
| Once betas are cut | `beta` | one moving ref the fleet follows, releases are deliberate |
| Stable | `v1` | a workflow change becomes an explicit rollout |

Note the rules are a separate axis: `rules_ref` selects which ref of `care_reviewer` supplies
them, defaulting to `dev` while `care_reviewer`'s `main` is still empty. Rules deliberately are
not version-pinned — a rule fix should reach the fleet without a release.

## Inputs

| Input | Default | |
|---|---|---|
| `layer` | *(required)* | `backend` or `frontend` — selects the rule file |
| `max_diff_lines` | `2000` | skip above this many changed lines |
| `model` | `claude-sonnet-5` | |
| `rules_ref` | `dev` | which ref of `care_reviewer` supplies the rules |

## Cost

Public repos get free Actions minutes, so the cost is tokens. On `claude-sonnet-5` a typical
plugin PR runs roughly $0.20–0.40. That stays small for a pilot and does not stay small across
40 repos — size it before fanning out. The levers are `max_diff_lines`, `--max-turns`, the
`concurrency` cancel, and the draft/bot filters.
