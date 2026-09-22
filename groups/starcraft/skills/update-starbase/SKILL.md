---
name: update-starbase
description: Syncs Starbase-managed project files by adding the canonical/starbase remote and merging starbase/main into the current branch. Use when updating common.mk and related shared Starcraft build/CI conventions.
allowed-tools: make git
---

# Update Starbase

## Add the remote and merge Starbase

Run the preparation script from the root of the child repository:

```bash
bash <path-to-skill>/scripts/prepare_merge.sh [BRANCH_SUFFIX]
```

- `BRANCH_SUFFIX` is optional; defaults to today's date
  (`work/update-starbase-YYYY-MM-DD`).
- The script handles steps 1–6: safe-state check, remote setup, fetch,
  work-branch creation, merge, and conflict detection.
- It exits **non-zero and prints `MERGE RESULT: merge-conflicted`** if
  the merge stops with conflicts, or exits **zero and prints
  `MERGE RESULT: merge-clean`** on success.
- In both cases the script prints a **`NEXT STEPS FOR THE AGENT`** block
  listing exactly what to do next — read it and follow it.

If the script cannot be run, follow [`references/manual_merge_steps.md`](references/manual_merge_steps.md) instead.

After the script completes, follow the `NEXT STEPS FOR THE AGENT` block it
printed. The steps below expand on each item in detail:

1. If GitHub reports conflicts with `main`, fetch the latest `origin/main` and
   redo the merge from that branch before continuing.

2. Clean up placeholder text in all merged files (both conflicted and cleanly merged):
   Scan all files that were added or modified by the merge for any remaining "Starcraft" or "Starbase" placeholder text:

   ```bash
   git diff --name-only --diff-filter=d starbase/main...HEAD | xargs -r grep -i -E "starcraft|starbase"
   ```

   Update any matches found (except external docs/style guide URLs) to use the child repository's name and purpose.

3. Document change provenance on each file:
   For every file added, deleted, or modified by the merge, make review comments on the GitHub PR explaining the provenance of the changes. Additionally, post inline review comments pointing out specific custom changes (e.g., removing a duplicate directive or fixing a type ignore).
   Files that already existed in the child repository and merged cleanly without conflicts or custom changes do not need a comment.
   **Provenance and rationale for manual changes belong exclusively in PR review comments, never as comments inside the source file.** If you catch yourself writing "why a bot changed this" as a `#`/`//` comment in code, stop and move it to a PR review comment instead. This applies just as much to config files like `docs/conf.py`: e.g. explaining *why* only specific `sphinx_toolbox` submodules are loaded (instead of the top-level package) belongs in a PR file-level comment, not a multi-line `#` block above the `extensions` entries.
   The comments must follow these guidelines:
   - **Prefix Template**: Each comment must begin with a robot emoji and a prefix in square brackets announcing that a bot wrote it, along with the model and harness. E.g. `🤖 [BEEP BOOP, A BOT WROTE THIS COMMENT - <model>, <harness>]`.
     Example: `🤖 [BEEP BOOP, A BOT WROTE THIS COMMENT - Gemini 3.5 Flash (High), antigravity]`
   - **Provenance Descriptions**:
     - For new files: `"new file from starbase"`
     - For modified files: `"file updated from starbase"`
     - For renamed or moved files: `"file moved in starbase"`
     - For complex/mixed files (e.g. `uv.lock` or `pyproject.toml`): Provide specific intermediate/complex details (e.g. `"file updated from starbase (child-owned file, regenerated locally based on merged dependencies)"`).
   - **Implementation via GitHub API**:
     - Post file-level comments on the PR using the REST API (`POST /repos/{owner}/{repo}/pulls/{pull_number}/comments`) with the `"subject_type": "file"` parameter so that a specific line number is not required.
     - Post inline line-level review comments for specific code changes (specifying `"line"` and `"side": "RIGHT"`) to highlight specific modifications made (such as resolving duplicate extensions or custom linter ignores).
     - **Handling Rate Limits**: When posting a batch of comments, enforce a delay (e.g., 4-5 seconds) between calls to avoid GitHub's spam rate limiter (`was submitted too quickly`). Implement an automatic backoff/retry (e.g., sleeping 30 seconds upon hitting a rate limit) to guarantee all comments are registered.
   - **Own-invention workarounds require a suggestion, not a direct push**: for
     a change that is neither sourced from `starbase/main` nor an obvious
     conflict resolution — for example, a hand-written workaround like adding
     an `export SPHINX_OPTS := ... -j 1` override to fix a `--fail-on-warning`
     failure caused by a Sphinx extension's parallel-read warning — do not
     push the change directly into the merge commit. Instead, push the merge
     without it, then leave an inline review comment at the relevant location
     using the standard robot-prefix template that explains the problem and
     proposes the fix as an actual GitHub suggestion (a fenced
     ` ```suggestion ` block), so a human reviewer can review and apply it
     explicitly rather than the bot self-approving its own invented fix.
   - **Skip comments on routine mechanical regenerations**: a purely mechanical,
     expected side effect of the merge (for example, `uv.lock` being
     regenerated after merging dependency changes) does not need its own
     provenance comment unless something about that specific regeneration is
     non-obvious (e.g. an unexpected transitive version change worth
     flagging). Reviewers have called out such comments as unnecessary noise;
     when in doubt, prefer silence over a comment that just restates the
     expected mechanical outcome.
   - **Cite precedent with a link, not just a claim**: when a provenance or
     custom-change comment says a resolution "matches" or was "already used
     in" another repository's Starbase merge (for example, the same
     `Makefile` lint-target conflict resolved the same way in
     `craft-application`), the comment must link the actual PR where that
     precedent was set, and briefly note any discussion that happened there
     — not just assert the precedent by name.

## Output

Report one of:
- `merge-clean`: Starbase merged with no conflicts.
- `merge-conflicted`: merge stopped with conflicts, including the file list from `git status --short`.

## Pre-PR validation (required)

Before creating a PR, ensure repository checks pass:

```bash
make format
make lint
make docs
make test-fast
```

`make lint` builds the docs as part of linting, but run `make docs` too so
docs-specific failures are caught and reported clearly on their own, not just
as a side effect of the lint target.

Do not create the PR until these commands complete successfully.

## Creating preparation PRs (as needed)

You won't know upfront which fixes need their own PR — this only becomes
clear while resolving conflicts and running validation. Watch for it
throughout the merge, not just at the end.

Some fixes discovered while merging or validating are not actually part of
the Starbase sync itself — they're pre-existing or unrelated issues that the
merge happened to surface. Bundling these into the merge PR makes it harder
to review and obscures which changes actually came from Starbase. Recognized
triggers include:

- A follow-up fix that only became necessary *because of* an earlier
  follow-up fix (a chain reaction), rather than because of the Starbase merge
  itself.
- A dependency or version bump driven by something unrelated to the sync
  (e.g. a security-scanner finding against a pinned test fixture) — the bump
  is legitimate, but it isn't a Starbase change and stands on its own merits.
- A purely mechanical reformatting diff produced by a newly introduced
  formatter or linter (e.g. `tombi`, `shfmt`) with no logic changes — split
  the reformat-only diff into its own PR so the merge PR's diff isn't
  dominated by noise unrelated to the actual content change, and so the
  formatting-only change can be reviewed and reverted independently.
- Starbase introducing a new formatter or linter that this repository doesn't
  yet run, **but only when it's default-enabled in Starbase** (i.e. it's
  included as a dependency of Starbase's `lint` or `format` `common.mk`
  targets, so every repo picks it up automatically rather than opting in) —
  whether or not it also requires a purely mechanical reformat, per the point
  above. Open a prep PR that just enables the new tool (adds it to
  `common.mk`/pre-commit/CI as applicable) and makes the changes needed to
  pass it clean, scoped to that tool alone. This keeps "we're now enforcing
  X" as its own reviewable decision, separate from the rest of the Starbase
  sync. Optional/opt-in Starbase tooling that this repository chooses not to
  adopt does not need a prep PR.
- A large, net-new file generated from a template (e.g. `AGENTS.md` derived
  from `AGENTS.lib.md`/`AGENTS.app.md`) — even though it's part of the
  Starbase scaffolding, its size and repo-specific content make it worth
  reviewing as its own unit rather than folding into the broader merge diff.
- A latent bug the new tooling now catches that isn't itself a Starbase file
  or convention.

When you identify such a change:
1. **Recognize the signal**: ask whether the fix would still be needed on
   `main` even without the Starbase merge, or whether it exists only to
   patch a side effect of another fixup you just made, or matches one of the
   trigger categories above. If any of these are true, it likely belongs in
   its own PR rather than as a merge-PR follow-up commit.
2. **Split it out**: create a new branch from `origin/main` (not the merge
   branch), apply just that fix, and open it as a **draft PR** against `main`
   describing the fix on its own merits — not as a Starbase merge follow-up.
3. **Stack it before the merge PR**: base the preparation PR on `main` so it
   can be reviewed and merged independently. Once it lands (or while it's
   still pending review), rebase the merge branch onto the updated `main` so
   the fix is picked up naturally and drops out of the merge PR's diff and
   follow-up-fix commit list.
   - If a second, related preparation fix comes up before the first one has
     merged (e.g. it depends on the first fix, or the two are easier to
     review as an ordered sequence), don't pile both onto the same branch or
     open unrelated parallel PRs from `main`. Instead use GitHub's
     [stacked pull requests](https://docs.github.com/en/pull-requests/how-tos/stacked-pull-requests):
     branch the second fix from the first preparation PR's branch and open it
     as a draft PR targeting that branch, so GitHub renders the dependency
     chain and each PR's diff stays scoped to just its own fix. Keep chaining
     additional preparation PRs the same way if more come up.
4. **Don't block on it**: if a preparation PR (or stack) hasn't merged yet,
   continue the Starbase merge work; rebase again once each one lands. Note
   pending preparation PRs in the merge PR description so reviewers know why
   a fix they might expect to see isn't there.
   - Keep every preparation PR in **draft** for its entire life until it is
     ready to merge — do not mark one ready for review just because the
     merge PR itself is progressing.
   - Whenever you open, update, or check in on a preparation PR, also report
     it directly to the user/operator (not only in the merge PR description):
     list every open preparation PR by title with a direct link, and its
     current status (draft, awaiting review, merged, etc.), so they always
     have an up-to-date view without having to dig through the merge PR's
     description themselves.
5. **Use the same provenance conventions**: preparation PRs still use the
   robot-prefix comment template and merge-commit-message rules where
   applicable — they are otherwise ordinary PRs, not Starbase-merge-specific.
6. **Link the source template when the PR is the agent's own original work
   derived from one**: for a preparation PR whose content is written/adapted
   by the agent from a specific Starbase template or scaffold file (e.g. an
   `AGENTS.md` written from `AGENTS.lib.md`/`AGENTS.app.md`), the PR
   description must name that exact source file and link directly to it at
   the commit/ref of `starbase/main` it was derived from, so a human reviewer
   can open both side-by-side and compare them line-for-line. A vague
   reference to "the Starbase template" is not sufficient — link the actual
   file.
7. **Link the originating Starbase PR for major changes**: when a preparation
   PR is about a major reorganization (e.g. moving `docs/.sphinx/` to
   `docs/_dev/`), the addition of a significant new file (e.g. a new
   scaffold/config file introduced by Starbase), or the addition of a new
   linter/formatter (per the "default-enabled tool" trigger above), the PR
   description must link the specific `starbase` PR that introduced that
   change upstream, in addition to any source-file link required above — not
   just describe the change in prose. This gives reviewers the full upstream
   context and discussion, not just the resulting diff.

Do not create the merge PR's ready-for-review PR (mark it out of draft) until
every preparation PR it depends on has been merged.

## PR labeling (required)

When opening the PR, apply the label:
- `PR: Merge`

## PR review and CI

- Open the PR as a draft, request a Copilot review while it is still draft, and
  keep iterating until the review is clean enough to mark ready.
- Check the PR's CI status before handing it off.
- If a reviewer's requested change contradicts a rule in this skill (the
  file-ownership map, a conflict-resolution rule, or any other documented
  rule), see
  [`references/reviewer_feedback_conflicts.md`](references/reviewer_feedback_conflicts.md)
  before acting on it.
- If CI is still running or likely to fail, start a background agent to watch
  the PR checks and report failures so they can be fixed promptly.
- You do not need to wait for the full matrix to finish before fixing failures
  in jobs that are already failing or have enough signal to act on.
- While the PR is still in draft (fixing initial CI failures and addressing
  the first round of review comments), make separate commits for each
  follow-up fix rather than amending; this keeps the history of what changed
  and why easy to review incrementally.
- Squash the follow-up-fix commits back into the single merge commit twice,
  at two distinct points:
  1. Once CI is green and the PR is ready to come out of draft: squash all
     follow-up-fix commits made so far into the original merge commit, then:
     - **Non-interactive harness**: mark the PR ready for review.
     - **Interactive harness**: do *not* mark the PR ready. Instead, send the
       operator a message with the draft PR URL and a brief summary (what was
       merged, any conflicts resolved, any custom changes made) and stop.
       Leave promoting the PR to the operator.
  2. Right before the final merge into the base branch: squash any further
     follow-up commits made during the ready-for-review round (e.g., fixes
     from human reviewers) back into that same single merge commit.
  Between these two squash points, follow-up fixes should again land as
  separate commits, not be squashed continuously.
- **The provenance/documentation rules in "Document change provenance on each
  file" apply for the entire life of the PR, not just the initial merge.**
  Any manual code change made while fixing CI failures or responding to review
  feedback — including one-line bugfixes — must be explained via a PR review
  comment (inline on the changed line, using the same robot-prefix template),
  never as a comment added to the source file itself. Do not add explanatory
  `#`/`//` comments to files to document *why a bot made this change*; source
  comments are only for genuinely non-obvious code, not for change provenance.
- Post each of these follow-up-fix review comments (and remember the
  robot-prefix template) in the same turn you push the corresponding fix —
  don't defer it, and don't wait to be reminded.

## Refreshing a stale merge PR (required when either main has moved on)

If significant time has passed since the merge PR was opened and either
`origin/main` or `starbase/main` has gained new commits, do not layer another
merge commit on top of the existing one. Instead, rebuild the merge commit
from the current heads of both branches while preserving every follow-up-fix
commit already pushed to the PR:

1. `git fetch origin --prune && git fetch starbase --prune` to get both
   branches' latest state.
2. Create a fresh branch from the current `origin/main` (do not reuse the old
   merge base): `git checkout -b <new-branch> origin/main`.
3. `git merge --no-ff --no-commit starbase/main` and resolve conflicts using
   the same file-ownership map and decisions as the original merge. Diff the
   new merge tree against the old merge commit
   (`git diff <new-working-tree> <old-merge-commit> --stat`) to catch any
   conflict-map resolutions, placeholder-text cleanups, or lint-rule fixes
   that the new merge silently skipped (for example, because
   `starbase/main` didn't touch a file that the old merge still needed to
   modify, so git raises no conflict for it at all — reapply those changes
   manually from the old merge commit).
4. Commit the merge using the same commit message template as any other
   Starbase merge (see "Merge commit message format"), updating the date.
5. Cherry-pick every follow-up-fix commit from the old branch, in order, onto
   the new merge commit: `git cherry-pick <fix-1> <fix-2> ...`. These commits
   must be preserved, not redone from scratch or squashed away.
6. Re-run the pre-PR validation commands (`make format`, `make lint`,
   `make docs`; `make test-fast` at your discretion) against the rebuilt
   branch before pushing.
7. Force-push the rebuilt branch to the existing PR branch (use
   `--force-with-lease` against the branch's current remote tip for safety).
8. Update the PR title's date to match the new merge date (see "Merge commit
   message format" for the title/subject format). `gh pr edit --title` can
   spuriously fail on repos with legacy Projects (classic) boards; if it
   errors, fall back to
   `gh api repos/{owner}/{repo}/pulls/{number} -X PATCH -f title="..."` and
   confirm the new title with a follow-up `gh pr view --json title`.
9. Post a PR comment (standard robot-prefix template) noting that history was
   rewritten and force-pushed, and summarizing what changed on each side
   (new commits pulled in from `origin/main` and/or `starbase/main`) and
   confirming the follow-up-fix commits were preserved.

## Merge commit message format

When finalizing the Starbase merge commit, include the merge date as an
ISO 8601 date only (`YYYY-MM-DD`) in the commit message.

Do not include time or timezone.

Example:
- `chore(merge): update starbase (2026-06-18)`

The commit body must include a concise overview of the Starbase changes that
were pulled in (for example: build-system updates, workflow changes, docs
tooling changes, or dependency-management adjustments).

Use this template:

```text
chore(merge): update starbase (<ISO-8601-DATE: YYYY-MM-DD>)

Merge `starbase/main` into this branch and sync Starbase-managed updates.

Overview of Starbase changes pulled in:
- <high-level change area 1>
- <high-level change area 2>
- <high-level change area 3>
- <high-level change area 4>

Conflict resolution applied:
- Kept child-repo version of `<child-owned-file-1>` (`--ours`).
- Kept child-repo version of `<child-owned-file-2>` (`--ours`).
- Took full `<source-of-truth-file-1>` from `starbase/main` (source-of-truth file).
- Took full `<source-of-truth-file-2>` from `starbase/main` (source-of-truth file).
- For this `<library|application>` repo:
  - kept `<agents-template-file-to-delete-1>` deleted,
  - kept `<agents-template-file-to-delete-2>` deleted,
  - kept `AGENTS.md` authoritative and applied relevant updates from `<agents-template-reference-file>`.
```

## Conflict resolution

When resolving merge conflicts, consult
[`references/file_ownership.md`](references/file_ownership.md) for the
complete ownership map and decision rules, including:

- **Conflict rule 1** – File ownership map (always take starbase, always take
  child, and mixed-ownership rules by file/section).
- **Conflict rule 2** – Python type annotation modernisation (when to apply
  `X | Y`, `tuple[X, Y]`, etc., and Python version caveats).
- **Conflict rule 3** – AGENTS template handling (library vs. application
  repositories).

If a reviewer's PR feedback asks for something that contradicts one of these
rules (or any other rule in this skill), see
[`references/reviewer_feedback_conflicts.md`](references/reviewer_feedback_conflicts.md)
for how to resolve it — don't silently follow one side without recording why.
