# Creating Preparation PRs

Use this guide when you encounter fixes, adjustments, or additions during the Starbase merge or validation that are not strictly part of the Starbase sync itself.

Bundling pre-existing or unrelated changes into the Starbase merge PR obscures the merge diff and complicates review. These changes belong in separate **Preparation PRs** based on `origin/main` that are stacked or merged prior to finalizing the merge PR.

---

## 1. Triggers for a Preparation PR

Open a separate preparation PR if a change falls into any of these categories:

- **Unrelated or pre-existing fixes**: A latent bug or issue exposed by new tooling that is not a Starbase file or convention, or a fix that would still be needed on `main` without the Starbase sync.
- **Chain-reaction fixes**: A follow-up fix that only became necessary *because of* an earlier follow-up fix, rather than because of the Starbase merge itself.
- **Unrelated dependency or version bumps**: A bump driven by an external factor (e.g. security scanner finding against a pinned test fixture) rather than the Starbase sync.
- **Purely mechanical reformatting**: Diff produced by a newly introduced formatter or linter (e.g. `tombi`, `shfmt`) with no logic changes. Keep the formatting diff separate so it can be reviewed and reverted independently.
- **New default-enabled tooling from Starbase**: Starbase introduces a new formatter or linter that is default-enabled in Starbase (i.e. part of Starbase's `lint` or `format` `common.mk` targets).
  - Open a prep PR that enables the tool (in `common.mk`/pre-commit/CI) and addresses its findings alone.
  - *Note:* Optional/opt-in Starbase tooling not adopted by this repository does *not* need a prep PR.
- **Large net-new template-derived files**: Net-new files generated from a template (e.g. `AGENTS.md` derived from `AGENTS.lib.md`/`AGENTS.app.md`). Their size and repository-specific content warrant an independent review unit.

---

## 2. Preparation PR Lifecycle & Rules

1. **Branch from `origin/main`**:
   - Create a fresh branch from `origin/main` (not the merge branch):
     ```bash
     git checkout -b fix/<descriptive-name> origin/main
     ```
   - Apply only the isolated fix and open a **draft PR** against `main`. Describe the fix on its own merits, not as a Starbase merge follow-up.

2. **Handle dependent fixes via stacked PRs**:
   - If a second preparation fix depends on the first, do not combine them or open parallel PRs from `main`.
   - Branch the second fix from the first preparation PR's branch and open a draft PR targeting that branch ([GitHub Stacked PRs](https://docs.github.com/en/pull-requests/how-tos/stacked-pull-requests)).

3. **Keep preparation PRs in DRAFT**:
   - **The agent must NEVER mark a preparation PR ready for review or merge it.** Only the human operator promotes PRs out of draft.
   - Once CI is green and reviews are addressed, report the PR to the operator and wait for them to promote and merge it.

4. **Report open preparation PRs to the operator**:
   - Whenever opening, updating, or checking on preparation PRs, report them directly in your response to the user/operator.
   - Include direct links, titles, and current status (e.g., draft, CI running, ready for operator promotion).

5. **Rebase the merge branch**:
   - Do not block ongoing Starbase merge work while waiting for prep PRs to merge.
   - Note pending preparation PRs in the merge PR description.
   - Once a preparation PR merges into `main`, rebase the merge branch onto the updated `origin/main` so the fix drops out of the merge PR diff.
   - Do not mark the Starbase merge PR ready for review until all dependent preparation PRs are merged.

---

## 3. Description & Provenance Requirements

Preparation PR descriptions must include explicit provenance links:

- **Link the source template**:
  If the PR content was adapted by the agent from a Starbase template (e.g. `AGENTS.md` from `AGENTS.lib.md`/`AGENTS.app.md`), the description must name the exact source file and link directly to it at the specific commit/ref of `starbase/main` from which it was derived.
- **Link the originating Starbase PR**:
  If the PR introduces a major reorganization (e.g. moving `docs/.sphinx/` to `docs/_dev/`), a significant new file, or a default-enabled linter/formatter, the description must link the specific upstream `starbase` PR that introduced the change.
- **Bot attribution**:
  Follow standard robot-prefix comment templates (`🤖 [BEEP BOOP...]`) and commit message formats where applicable.
