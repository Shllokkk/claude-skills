---
name: bug-sprint
description: >
  Fix ERPNext bugs from the frappe/erpnext-bug-reviews sprint, one doctype/report/page
  at a time: verify from code, fix on develop, drive the greptile review loop to green,
  then raise adapted manual backports to v16/v15. Use whenever working on an issue from
  the erpnext-bug-reviews repo or GitHub Project 242, or when the user says things like
  "next bug", "fix the <doctype> issues", "raise the PR", "resolve greptile comments",
  "backport this", or "check CI on the backport".
---

# ERPNext bug-sprint workflow

**First step, every session:** as soon as the issue being worked is known, rename this
session to `bugsprint #<issueno>` via the session-title tool
(`set_session_title`, `session_id: "self"`). If several issues are in play, use the one
primarily being worked.

Spans many sessions — this skill is the contract. General branching, commit, PR-target,
attribution, SQL and comment rules live in **CLAUDE.md**; framework details in
`frappe-app-dev`; code shape in `code-style`; review lens in `quality-code-review`. This
file holds only the sprint-specific deltas. **Load `code-style` (and `frappe-app-dev`)
via the Skill tool before writing any code — see Sprint rules; this is mandatory, not a
pointer.**

## Context

- **Review repo:** `frappe/erpnext-bug-reviews` — one report per non-child doctype (child
  tables under their parent), plus reports and desk pages. Each bug was reproduced on
  `demo.localhost` with a rollback probe and confirmed by reading code. Probes live under
  `<doctype>/probes/...` and `reports/pages/probes/...` (+ a `common/stubs.py` stubbing
  commit/enqueue/sendmail/global-search).
- **Access:** write/maintain on both `frappe/erpnext` and `frappe/erpnext-bug-reviews`
  → you can tick the issue checkboxes, edit the issue body, and close/reopen issues.
- **Project board (my assigned issues):** https://github.com/orgs/frappe/projects/242/views/3
  — issues auto-move Todo → In Progress when a PR referencing them is opened.

## My assigned issues

Ordinals ("2nd issue") follow the board, ascending by number — match it with:

```bash
gh issue list --repo frappe/erpnext-bug-reviews --assignee Shllokkk --state open \
  --label module:selling --limit 100 --json number,title,labels --jq 'sort_by(.number)'
```

## Order of operations (strict)

```
develop PR  →  resolve greptile comments  →  repeat until develop is clean
            →  ONLY THEN raise manual backport PRs (v16, then v15)
```

Backports copy the *final* develop code, so fixing the same greptile issue twice is
wasted — don't open backports until the develop greptile loop has converged.

## Sprint rules

- **Before writing or editing ANY code, invoke the `code-style` skill via the Skill tool
  first** (and `frappe-app-dev` for Frappe work) — every time, in every session, even for a
  one-line fix, a test, or a backport edit. Referencing it is not enough; actually call it.
  CLAUDE.md mandates this and it keeps getting skipped — do not emit code until it's loaded.
- **"check next issue" = explain only, don't fix.** When the user says "check next issue"
  (or "check issue #N"), read the review + source, verify by code, and explain each bug in
  simple plain language — do NOT branch or edit any code. Wait for the user to say to fix
  before touching code.
- **Comments/docstrings: only when absolutely necessary, one line max.** Do NOT add
  docstrings or explanatory comments by default — add one only where the code genuinely
  can't speak for itself (a non-obvious WHY), and keep it to a single short line. Prefer no
  comment.
- **Verify by reading code, not by reproducing.** No testsite/probe runs unless the user
  explicitly asks (then use the repo's probes; also for non-obvious backports).
- Be open to design flaws / refactors, not just one-line patches.
- **One commit per bug**, clubbed into one PR; flag when a high-risk fix needs its own PR.
- **Tests:** per bug, add a minimal test (or alter the existing one) **only if needed** and only test backend behaviour and run the concerned test file locally on test sites (each bench has its own test site);
  fix + its test go in the **same commit**.
- **Develop PR description:** a couple of lines, plain human language — the underlying
  bug and the fix and also mention any additional bugs fixed. End with
  `Closes frappe/erpnext-bug-reviews#<issue>` (fully-qualified cross-repo, since the PR is on
  `frappe/erpnext`) so merging auto-closes the issue. Only the develop PR closes it — backport
  PRs keep their one-line `Manual backport of #<develop-pr>`.
- On raising a PR: tick the fixed bugs in the issue checklist and change status to In Progress.

## Greptile loop

- After each push read greptile's inline comments + confidence score. Explain the findings and
  **Fix the valid ones** and push by amending the relevant commit (never a new PR). For the rest, 
  **reply in 1–2 lines, plain language** — say it's fixed and how, or why it's
  intentional/acceptable; dismiss false positives. Repeat until no blocking P1s remain.
  The greptile fixes need to be amended to respective commits itself no new commits.

```bash
gh api repos/frappe/erpnext/pulls/<PR>/comments --jq '.[] | select(.user.login|test("greptile";"i"))'
gh api repos/frappe/erpnext/pulls/<PR>/comments -X POST -f body="..." -F in_reply_to=<comment_id>
```

## Backports (CLAUDE.md covers pinning, "develop first / never below v15", labels)

- Prefer the `backport version-x-hotfix` label only when the code **applies cleanly**.
  When the version branch diverges (different functions, missing APIs), the label gives a
  broken/conflicting backport → raise a **manual, adapted** PR and verify non-obvious
  fixes with the probes on that version's bench.
- **Manual backport PR description:** only `Manual backport of #<develop-pr>` (one line).
- Try manual backport to v16 and try adding v15 label to it. If not clean then only manually raise to v15.
- **Divergent (manually adapted) backports are their own change** — the "skip the greptile
  loop, it just copies develop" rule only holds when the code is *identical*. When the
  backport differs (qb rewrite, different fields/APIs), run its greptile loop independently
  and resolve there; don't blindly mirror develop's fix (it may not even apply). Also the test
  suite class has diverged between v15 and v16 (same as develop).
- **Prefer actually running the backport's tests on that version's bench over assuming parity
  with develop** — any field, DocType, method, API or import the change references may have
  been added on develop and be absent or different downstream (e.g. `Quotation.is_active`
  doesn't exist on v15/v16), so a copied-over filter/query fails only at runtime. Amend the
  adaptation into the relevant commit (fixup + autosquash), not a new one.

## Related memories

`feedback_bugsprint_pr_workflow`, `feedback_verify_by_code_not_reproduce`,
`feedback_backport_replicate_with_probes`.
