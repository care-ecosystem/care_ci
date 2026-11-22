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

Full runbook, including the failure modes: `care_reviewer/ONBOARDING.md`. Short version:

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
- `CLAUDE_CODE_OAUTH_TOKEN` — generated with `claude setup-token`, as an org Actions secret.
  Works on Pro, Max, Ultra, Team and Enterprise.

  Know what this means before relying on it: the token is tied to the **subscription of whoever
  ran the command**. Reviews draw on that person's rate limits, so a busy review day throttles
  their own Claude Code; the token dies if they leave or their seat changes; and Anthropic's
  docs recommend a Console API key instead for a secret shared across repositories. Fine for a
  pilot, worth revisiting before the full fleet. Swapping to an API key later is a two-line
  change: `anthropic_api_key` in place of `claude_code_oauth_token`.
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
| `model` | org variable, else `claude-haiku-4-5` | see below |
| `rules_ref` | `dev` | which ref of `care_reviewer` supplies the rules |

### Changing the model

Resolution order: the caller's `model:` input, then the org variable `CARE_REVIEW_MODEL`, then
`claude-haiku-4-5`.

Set the **org variable** (a variable, not a secret — the model name isn't sensitive) to move the
whole fleet at once, with no commit here and no change in any caller:

```
gh variable set CARE_REVIEW_MODEL --org care-ecosystem --body claude-sonnet-5
```

A single repo can override it by passing `model:` in its caller, which is the cheap way to run
one noisy repo on a stronger model without touching the rest.

Current options, cheapest first: `claude-haiku-4-5`, `claude-sonnet-5`, `claude-opus-5`.

## Cost

Public repos get free Actions minutes, and with `CLAUDE_CODE_OAUTH_TOKEN` the model usage goes
against the **subscription**, not API billing — so there is no per-review dollar figure. The
scarce resource is the token owner's rate limit.

That changes what the model choice is for. Picking `claude-haiku-4-5` over `claude-sonnet-5`
does not save money here; it stretches the rate limit further and leaves more headroom for the
token owner's own Claude Code. Start on Haiku, and move up only if the reviews are visibly
missing things.

If this later switches to an API key, per-token billing applies and the dollar figures matter
again: roughly $0.20–0.40 per plugin PR on Sonnet 5, less on Haiku.

Either way the volume levers are the same: `max_diff_lines`, `--max-turns`, the `concurrency`
cancel, and the draft/bot filters.
