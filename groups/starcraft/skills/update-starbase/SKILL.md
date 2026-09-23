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
   git diff --name-only --diff-filter=d HEAD^1 HEAD | xargs -r grep -i -E "starcraft|starbase"
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

Some fixes discovered while merging or validating are pre-existing or unrelated issues (e.g. latent bugs, mechanical formatting diffs, new default-enabled tooling, or large net-new template files). Bundling these into the merge PR obscures the merge diff.

When you encounter an unrelated issue, split it into a separate draft preparation PR against `origin/main` before finalizing the merge. Consult [`references/preparation_prs.md`](references/preparation_prs.md) for trigger criteria, stacked PR workflows, lifecycle requirements (draft-only, operator promotion), and provenance link rules.

## PR labeling (required)

When opening the PR, apply the label:
- `PR: Merge`

## PR review and CI

- Open the PR as a draft, request a Copilot review while it is still draft, and
  keep iterating until the review is clean enough for the operator to mark it
  ready.
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
     follow-up-fix commits made so far into the original merge commit, then
     send the operator a message with the draft PR URL and a brief summary
     (what was merged, any conflicts resolved, any custom changes made) and
     stop. **The agent must never mark any PR ready for review itself** —
     regardless of harness type (interactive or non-interactive) — only the
     operator takes a PR out of draft. Leave promoting the PR to the
     operator.
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
merge commit on top of the existing one. Follow
[`references/refresh_stale_pr.md`](references/refresh_stale_pr.md) to rebuild the
merge commit and preserve all existing follow-up commits.

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
