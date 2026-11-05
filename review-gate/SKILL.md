---
name: review-gate
description: >-
  Problem, root-cause and scope gate for reviewing a Frappe/ERPNext diff or PR, run
  together with frappe-code-review (which owns the technical checklist). Covers: is the
  problem clear, does the fix hit the root cause or patch a symptom, could it be smaller or
  split, does it need a test, an independent second-reviewer pass, and a verdict. Use
  whenever reviewing a diff, a fix or a PR — "review the diff", "review PR 63855", "review
  like mihir", "is this fix right", or the local review step before committing or raising
  a PR.
---

# Review gate

A review protects the codebase from workarounds, bloat and code nobody can read in a
year. A change earns approval by fixing a real, understood problem at its root, with the
least code that does it.

**First step: invoke `frappe-code-review` via the Skill tool** if it isn't already loaded
this turn — don't wait to be asked; the review isn't complete without it. This skill
judges *whether the change is the right change*; `frappe-code-review` judges correctness,
security, performance, concurrency, compatibility, patches and error messages. Don't
repeat its checklist here.

## Input

- **Default:** the uncommitted diff in the app being worked on (`git -C apps/<app> diff`,
  plus `git diff upstream/<base>` if commits exist).
- **A PR** (number or URL; bare number = `frappe/erpnext`).

## 1. Gather

PR:
```bash
gh pr view <n> --repo <repo> --json number,title,body,author,baseRefName,headRefName,additions,deletions,files,closingIssuesReferences,url
gh pr diff <n> --repo <repo>
gh pr view <n> --repo <repo> --comments
gh pr checks <n> --repo <repo>
```
Read linked issues. Bot comments are claims, not findings — check each against the code
and drop rare/impossible cases.

## 2. Problem gate

Write in one or two lines: what is broken, how it's triggered, expected vs actual. If
that can't be stated from the PR/issue/conversation, stop: the verdict is **Request
changes — state the exact problem and steps**. Reviewing code for an unknown problem
wastes everyone's time.

## 3. Verify the bug and the fix

Verify and (only if needed) reproduce as `dev-workflow` → *Explain vs act* describes,
tracing the base branch for the bug and the diff for the fix.

**Develop first.** If a release-branch PR (`version-*-hotfix`) fixes a bug that also
exists on `develop` and it isn't a backport of a develop PR, the verdict is **Request
changes — fix on develop first, backport after**.

For someone else's PR: note the branch you were on before `gh pr checkout`, switch back
afterwards, and never check out over uncommitted work — stop and ask. Don't run test
suites unless asked; CI covers PRs.

## 4. Judge

In this order — the first items decide the verdict:

1. **Root cause.** Fix where the bug comes from, not where it shows up. Workaround signs:
   a special-case `if` for one input, `try/except` hiding the error, a flag to skip a
   check, a client-side fix for server data, fixing one caller when the shared function
   is wrong. Name the real root cause and where the fix belongs.
2. **Size and scope.** Unrelated refactors, formatting or extra features → separate
   commits/PRs. Suggest the split.
3. **Framework.** No overriding Frappe behaviour; validation on the server (the REST API
   skips client code). If the framework lacks something, a Frappe PR may come first.
4. **Simplicity.** Could this be half as long? Dead code, flags carrying state,
   duplicated logic, clever code, functions doing two jobs. Don't ask to copy a bad
   pattern just because it's common in the file.
5. **Tests and text.** Is a test needed, does it fail without the fix, does it alter an
   existing case; one-line comments, short PR body — judge against `dev-workflow`. An
   edited existing assertion needs a reason: it may mean real behaviour broke.

## 5. Independent pass

Spawn **one** subagent (Agent tool, `general-purpose`) with a fresh context. Give it the
diff command and the one-line problem — **not your findings**, so it looks independently:

```
Read-only review. Do not edit files, commit, push or comment.
Repo: <path>. Diff: <git diff command or gh pr diff <n> --repo <repo>>
The change claims to fix: <one-line problem>
Find bugs, missed edge cases, workarounds instead of root-cause fixes, missing permission
checks, N+1 queries, framework overrides, client-only validation and unnecessary code.
For each: file:line, a concrete failure scenario reachable through normal Desk/API use,
and the fix. If nothing is real, say "no findings".
```

Wait for its result. Validate every finding by reading the code; drop the wrong, out of
scope and unreachable ones (`dev-workflow` reachability baseline). Valid ones join your list. If the subagent fails, say so — never skip silently.

## 6. Verdict and report

Merge this skill's findings with `frappe-code-review`'s into one list.

- **Approve** — problem confirmed, fixed at the root, only nits left.
- **Request changes** — problem unclear, wrong branch, or any of items 1–4 or a
  frappe-code-review correctness/security finding fails.
- **Close and redo** — wrong approach, or fixing it means rewriting most of the diff.

```
## <PR or "Local diff">: <title>
Verdict: Approve | Request changes | Close and redo
Problem: <one line, or "unclear">
Verified: code | repro on <bench>/<site> — bug on base: yes/no — fixed: yes/no
Root cause: <one line>
Second pass: <n> findings — <k> valid, rejected: <one-line reason each> | failed: <why>

### Blocking
1. `path:line` — problem. What to do instead.

### Non-blocking
1. `path:line` — ...
```

Cap each list at 5; merge or drop the weakest. Keep the report short.

## 7. Act

- **Own diff:** fix the valid findings, then re-run both skills on the new diff; loop
  until clean.
- **Someone else's PR:** add a draft review — short, direct, one point per comment, says
  what to change, no praise padding, no mention of AI. Ask "Post this review?" and post
  only on a clear yes: inline comments via `gh api repos/<repo>/pulls/<n>/reviews` with
  `event` + `comments`; summary-only via `gh pr review <n> --repo <repo>
  --request-changes|--approve|--comment --body-file <file>`. For **Close and redo**, ask
  whether to write the alternative instead (branch off `develop`, review it with both
  skills, raise only after the user okays it, then close the original with a one-line
  link).
