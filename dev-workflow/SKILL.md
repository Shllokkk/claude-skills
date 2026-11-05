---
name: dev-workflow
description: >-
  The house flow for changing Frappe/ERPNext apps on these benches: verify, fix, test,
  commit, review loops, PRs, Mergify queue and v16/v15 backports. Use whenever fixing a
  bug or making a change in an app, splitting or amending commits, writing or altering
  tests, raising or updating a PR, handling review-bot comments on a PR, checking CI,
  queueing with Mergify, or backporting — e.g. "fix it", "commit this", "raise the PR",
  "resolve the bot comments", "queue it", "backport to v15", "tests failed in CI".
---

# Dev workflow

The hard rules and skill routing live in CLAUDE.md (some hook-enforced); this skill is the
*how*.

## Start

- Check every app's branch in the bench: `for app in apps/*/; do echo "$app: $(git -C $app branch --show-current 2>/dev/null)"; done`
- Branch names are plain kebab-case, no `fix/`/`feat/` prefix. Before branching off a
  `version-x-hotfix`, `git pull upstream version-x-hotfix` so you start from current code.
- To work on an existing PR (conflicts, bot findings, a Mergify backport): `gh pr checkout <n>`
  in the bench matching the PR's **target** branch, and stay on the branch gh creates.

## Explain vs act

- **"check issue X", "check the PR/bot comments", "explain …" = explain only.** Read,
  verify by code, explain in plain language (valid / false-positive / unreachable for
  comments). Don't branch, edit, reply or resolve until the user says fix.
- **Verify by reading code by default** — trace the path to confirm the bug, then that the
  fix removes it and the happy path still works. Reproduce only when the user asks or the
  code is genuinely unclear: the issue repo's probe scripts if it has them, or a scratchpad
  script on the matching bench's test site with no `frappe.db.commit()`, ending in
  `frappe.db.rollback()`:
  `echo 'exec(open("<scratchpad>/repro.py").read())' | bench --site <test-site> console`

## Fix

- Fix the root cause, not the symptom; be open to design flaws and refactors. Flag when a
  high-risk fix deserves its own PR.
- `develop` first. Branch off `version-x-hotfix` only when the code is gone from develop.
- Before changing long-standing behaviour or forbidding a data shape, check whether merged
  code/tests already support it (`git log -S`, blame the test). **A fix that breaks a
  pre-existing test is a red flag that the fix over-reaches** — go back to the narrow root
  cause the report names (e.g. #368 Bug 3: rejecting repeated bundle components broke
  supported tests; the real fix was the stale packed-row key).
- When a signature or return shape changes, check every caller across `apps/*`. Keep
  `@frappe.whitelist` endpoints and their params — other apps may call them.

## Tests

- Add or alter a test **only if needed**; it must fail without the fix. Prefer altering an
  existing case over adding one. Backend business logic only — no framework behaviour.
  Fix + its test go in the same commit.
- **Where to run:** `testsite.localhost` (develop bench) is the user's hands-on site — no
  `run-tests` there unless asked; verify with a rolled-back call instead.
  `v16test.localhost` / `v15test.localhost` are test sites: run the touched modules
  (`bench --site <site> run-tests --module <module>`). Confirm names with `ls sites/`.
- A failure that predates the change (e.g. a missing-migration column) is not ours — check
  it against the base branch and say so.

## Code

- Comments: one line, only for a non-obvious why. No docstrings. Scan `"""` and `#` in the
  diff before handing it over.
- New documents via `frappe.new_doc("DocType")`, set fields, then `.insert()` — not
  `frappe.get_doc({"doctype": ...})`.
- Lint changed files with the repo's pinned ruff: `uvx ruff@0.2.0 check` / `format`. In zsh
  pass file lists through `| xargs` (no word-splitting of variables).

## Commits

- The user reviews uncommitted diffs in VS Code's Source Control, so hand work over
  uncommitted.
- Message: one short line (e.g. `fix: validate exchange rate endpoint`), no body, no
  attribution lines.
- **While converging:** one commit; amend the tip and `git push --force-with-lease` each
  round.
- **When clean:** split into one commit per bug (fix + its test): `git reset --soft <base>`
  then stage per bug. When bugs share a file, re-apply each bug's edits in turn
  (`git add -p` isn't available).
- **Amend a non-tip commit:** `git commit --fixup=<sha>` then
  `GIT_SEQUENCE_EDITOR=true GIT_EDITOR=true git rebase -i --autosquash <base>`; resolve
  conflicts by hand, don't abort.

## PRs

- `gh pr create --repo frappe/<repo> --base <develop|version-x-hotfix>` (the bench's line).
- Description: 2–3 plain lines — the bug, the fix, any other bugs fixed. End with
  `Closes <owner/repo#issue>` when an issue tracks the work (fully qualified across repos).
  Backport PRs: only `Manual backport of #<develop-pr>`.
- Report PR links in chat; don't comment PR mappings on tracking issues.

## Review loops

1. **Local, before the PR:** `review-gate` + `frappe-code-review` on
   `git diff upstream/<base>`. Fix, re-run the touched tests, loop until clean.
2. **PR review bot** (whichever one the repo runs) posts inline findings, often with a
   score. Verify every claim against the code — premises are often wrong. Fix valid ones
   by amending; for false-positive or unreachable ones reply in 1–2 lines and resolve.
   Re-trigger the bot the way it documents (usually a mention comment); loop until it
   passes or the rest are soundly dismissed.
   ```bash
   gh api repos/<owner>/<repo>/pulls/<PR>/comments --jq '.[] | select(.user.type == "Bot")'
   gh api repos/<owner>/<repo>/pulls/<PR>/comments -X POST -f body="..." -F in_reply_to=<comment_id>
   # resolve a thread: resolveReviewThread(input:{threadId}) via gh api graphql
   ```
- **Reachability baseline:** fix what a real user or normal integration can hit, even if
  rare. Dismiss findings that need invalid config, hand-crafted API calls or corrupted data.
- **Order:** converge develop first, then raise the backports and converge them together
  (dedupe v16 and v15 findings into one list). A backport finding in shared logic gets
  fixed on develop too and carried to every branch. Security/permission fixes go to every
  backport even if the bot flagged only one; adapt when a develop-only helper is missing.

## Merge queue and CI

- Only light checks run on open. When asked, comment `@mergifyio queue` on each PR
  (develop + backports) to batch it into the heavy suite. The `merge-alone` label runs the
  suite alone, then merges.
- On a CI failure, first decide if it's ours: map failing tests to the diff, and check
  whether the base branch or other PRs fail the same way.

## Backports

- Never below v15: from a develop PR check v16 + v15; from a v16 PR, v15.
- **Label** (`backport version-x-hotfix`) only for bugs and small refactors that apply
  cleanly — Mergify does the rest. Check upfront by cherry-picking onto the target in its
  bench (not `git apply --check`; the `modified` line gives false conflicts).
- **Manual, adapted PR** otherwise, in the version bench: branch `<name>-v16` (see Start),
  apply the develop commits and **leave them uncommitted for review** (`cherry-pick -n`, or per-commit picks followed by a mixed
  `git reset <base>`).
  - On conflicts, transplant only the PR's hunks; never copy surrounding newer-branch code.
  - Check every field, doctype, helper and API the change uses exists downstream
    (`git show upstream/version-x-hotfix:<path>`); e.g. `Quotation.is_active` is
    develop-only and broke a v15/v16 `get_list`. Test base classes differ (v15 uses
    `FrappeTestCase`), and so do frappe signatures (v15 `server_action` sends one `args`).
  - Run the touched tests on that bench's test site; replicate non-obvious bugs there first.
  - A divergent backport is its own change — review it on its own.
- After the user reviews and asks: split per bug, rebase on the latest hotfix, push to
  `origin`, raise the PR, queue if asked.
- **Mergify backport PR conflicts:** check it out as in Start, fix, amend and force-push.
