---
name: github-triage
description: Use when triaging GitHub issues or pull requests - categorizing, labeling, organizing open items, validating if issues are still relevant, and reviewing/merging PRs. Handles taxonomy discovery, label creation, bulk tagging, issue staleness checks, and full PR review + merge workflows.
---

# GitHub Triage

Systematically categorize, validate, and act on open GitHub issues and PRs.

**Core principle:** Discover what dimensions matter for THIS project, propose a taxonomy, get approval, then tag everything. Never apply labels you haven't verified exist. When reviewing PRs or validating issues, use parallel sub-agents for efficiency.

## Full Triage Workflow

```
1. Label all issues and PRs (Phases 1-5)
2. Validate open issues against codebase (Phase 6)
3. Review and merge open PRs (Phase 7)
```

---

## Phase 1: Discovery

Pull all open issues and PRs in parallel. Scan titles, bodies, and comments for recurring patterns.

```bash
# Run these in parallel
gh issue list --repo OWNER/REPO --state open --json number,title,body,labels,createdAt --limit 500
gh pr list --repo OWNER/REPO --state open --json number,title,body,labels,isDraft --limit 500
gh label list --repo OWNER/REPO --json name,description,color --limit 100
```

**Look for:**
- Platform mentions (Windows, macOS, Linux, WSL)
- Tool/harness names (varies by project — read carefully, don't guess)
- Type signals (bug templates, "how do I", "feature request", error logs)
- Severity signals (crash, security, data loss, cosmetic)
- Status signals (stale, waiting for response, needs reproduction)
- Root cause signals (upstream dependency bug, user environment, plugin bug)

## Phase 2: Propose Taxonomy

Present discovered dimensions and proposed label values. Match existing label style:
- If repo uses flat labels (`windows`, `bug`), stay flat
- If repo uses prefixed labels (`platform:windows`, `type:bug`), use prefixes
- If starting fresh, recommend prefixed (self-documenting, filterable)

**Present as a table including proposed per-item mappings:**

```
Dimension: type     → bug, enhancement, documentation
Dimension: area     → area:docker, area:devcontainer-spec, area:testing, area:tooling
Dimension: priority → priority:high (bugs that block users)
Dimension: status   → status:needs-review, status:blocked, status:stale

| #   | Title                        | Labels                          |
|-----|------------------------------|---------------------------------|
| #21 | mountpoint is outside rootfs | bug, priority:high, area:docker |
| #12 | HTTPS tarball support        | enhancement, area:devcontainer  |
...
```

Showing the per-item mapping upfront lets the user approve taxonomy AND assignments in one step.

**Do not proceed until user approves.**

## Phase 3: Create Labels

Create ONLY approved labels. Check each one first — `gh issue edit --add-label` silently auto-creates labels that don't exist, which pollutes the label namespace.

```bash
# Create only if missing
gh label create "label-name" --repo OWNER/REPO --description "description" --color "hex"
```

**Never let `gh issue edit --add-label` auto-create labels.**

## Phase 4: Tag

Apply labels from the approved taxonomy. Batch all commands — no need to re-read between labeling.

```bash
gh issue edit NUMBER --repo OWNER/REPO --add-label "label1,label2"
gh pr edit NUMBER --repo OWNER/REPO --add-label "label1,label2"
```

**Confidence rule:** Apply `needs-categorization` when uncertain. A wrong label is worse.

**Skip closed issues** unless explicitly asked.

## Phase 5: Labeling Report

```
## Triage Summary
Tagged: 16 issues, 6 PRs

| Label                  | Count |
|------------------------|-------|
| enhancement            | 13    |
| bug                    | 7     |
| area:devcontainer-spec | 10    |
| area:docker            | 9     |
| priority:high          | 1     |
```

---

## Phase 6: Validate Open Issues Against Codebase

Go through open issues **oldest-first** and check whether each has been resolved in the current codebase. Close resolved ones with a comment.

```bash
# Get issues sorted oldest-first
gh issue list --repo OWNER/REPO --state open --json number,title,body,createdAt --limit 500 \
  | python3 -c "import json,sys; issues=sorted(json.load(sys.stdin), key=lambda x: x['number']); ..."
```

**Validation approach:**
- Search the codebase for the feature/fix described in the issue
- For features: look for the key function names, config fields, or struct fields mentioned
- For bugs: look for the specific error message, code path, or fix in git log
- Use parallel searches across multiple issues for speed

**Key searches to run in parallel:**
```bash
grep -r "featureName\|FeatureName" /path/to/repo --include="*.go" -l
git log --oneline | grep -i "keyword"
```

**When closing a resolved issue:**
```bash
gh issue close NUMBER --repo OWNER/REPO --comment "Verified by Claude Code: [specific explanation of where/how it's implemented, with file paths and line numbers where helpful]. — Claude [model], Claude Code [version], session [session-id]"
```

**Leave open if:**
- Feature is genuinely not implemented (grep finds nothing)
- Bug has no associated fix in git log or code
- Issue is a discussion/vision item that's ongoing

**Be precise:** Cite specific files, function names, or commits when closing. Don't close based on a vague sense that "it seems implemented."

---

## Phase 7: PR Review and Merge

For each open PR, follow this sequence: **security review → code review + local tests → merge**. Never check out or run PR code before the security review completes.

### Step 1: Check mergeability
```bash
for pr in 30 29 27; do
  gh pr view $pr --repo OWNER/REPO --json mergeable,mergeStateStatus
done
```

### Step 2: Security review FIRST (before any local checkout)

Invoke the `pr-security-review` agent on every PR before touching local code. The agent has read-only tools (`Read`, `Grep`, `Glob`, `WebFetch`) and cannot be compromised by what it analyzes.

Provide the agent:
- The PR number and repo
- The full diff (fetch from `https://github.com/OWNER/REPO/pull/NUMBER.diff` or via `gh pr diff NUMBER`)
- The PR author's name and whether they are a known contributor

**Security verdict gates:**
| Verdict | Action |
|---------|--------|
| ✅ SAFE | Proceed to Step 3 |
| ⚠️ REVIEW NEEDED | Present findings to user. If the fix is clear and small (e.g. missing input validation), **offer to apply it yourself** and ask the user whether to fix-then-merge or bounce back to the author. Wait for explicit direction before touching local code. |
| 🚫 BLOCK | Do not check out or run code. Report to user and stop. |

### Step 3: For each mergeable, security-cleared PR, launch two agents in parallel

**Agent 1 — Code Review (Explore agent):**
- Read the full diff (`gh pr diff NUMBER`)
- Read the changed files in the repo at their current state
- Check all callers of modified functions (blast radius)
- Verify correctness against the spec/docs
- Flag edge cases, missing tests, security issues
- Return: approve / approve-with-notes / request-changes

**Agent 2 — Test Runner (general-purpose agent):**
- Check out the PR branch **locally**: `gh pr checkout NUMBER --repo OWNER/REPO`
- Run lint **locally**: `make lint` (or project equivalent)
- Run tests **locally**: `make test` (or project equivalent)
- Do NOT rely on CI status — always run locally regardless of what CI shows
- `git checkout main` when done
- Return: lint pass/fail, test pass/fail, any new failures vs. pre-existing

**Why local?** CI may be broken, rate-limited, or testing a stale base. Local runs catch issues CI misses and confirm the code works on the actual machine it will be merged from.

### Step 3: Merge decision

| Code Review | Tests | Action |
|-------------|-------|--------|
| Approve | Pass | **Present findings to user. Ask: "Shall I merge?"** |
| Approve-with-notes | Pass | **Present findings + notes to user. Ask: "Shall I merge?"** |
| Approve | Fail (pre-existing only) | **Present findings to user, note pre-existing failures. Ask: "Shall I merge?"** |
| Approve | Fail (new failures) | Do not merge, report to user |
| Request changes | Any | Do not merge, report to user |

**Conflicts:** If `mergeStateStatus` is `DIRTY`, do not merge. Report to user with description of what conflicts.

> **MANDATORY: Never merge autonomously.**
> After code review and tests complete — even when all checks are green — Claude MUST stop and present findings to the user. State the PR number, the code review verdict, and the test results. Then ask explicitly: **"Shall I merge PR #N?"** Wait for an affirmative reply ("yes", "go ahead", "merge it", etc.) before running any merge command. This rule applies unconditionally, regardless of how clear-cut the result appears.

### Fork branch limitation: manual squash when you can't push to the author's branch

When a PR comes from a fork (i.e., `gh pr view N --json headRepositoryOwner` returns someone other than the repo owner), you cannot push lint fixes or other improvements to the PR branch.

**Instead of `gh pr merge --squash`** (which would merge the PR as-is without your fixes), do a manual squash:

```bash
# 1. Confirm it's a fork
gh pr view N --repo OWNER/REPO --json headRepositoryOwner -q '.headRepositoryOwner.login'

# 2. Fetch the PR branch locally
git fetch origin pull/N/head:pr-N

# 3. Pull all changed files onto main
git checkout main
git checkout pr-N -- file1.go file2.go ...   # only the files changed by the PR

# 4. Apply any fixes you need (lint, etc.)

# 5. Commit as a squash (summarize all PR changes in one message)
git commit -m "fix: ... (#N)\n\nCo-authored-by: Author <Author@users.noreply.github.com>"

# 6. Push
git push origin main

# 7. Close the PR manually with a comment explaining what you did
gh pr close N --repo OWNER/REPO --comment "Merged manually as squash commit <SHA> with <fix applied on top>. ..."
```

**Co-author attribution:** Always include the PR author in the commit via `Co-authored-by:` so their contribution appears in the repo's contributor graph.

### Missing test coverage: write the tests yourself

When code review identifies that new code lacks test coverage (e.g. a new function has no unit tests), do not just request changes from the author. Instead:

1. Write the tests yourself on the PR branch
2. Push them to the PR branch with a commit message explaining what you added and why
3. Proceed to the merge decision as normal

Only request changes from the author if the code itself has correctness issues, not just missing tests. Missing tests are your problem to fix, not a reason to block the PR.

### New function added by a PR? Verify the call site exists

When a PR adds a new function that's supposed to fire at runtime (a warning, a hook, a side effect), grep for callers across the codebase before merging. It's easy to add the function and forget to wire it up.

```bash
grep -rn "functionName" pkg/ cmd/ --include="*.go"
```

If the function is defined but never called outside its own file, flag it and wire it in yourself before merging.

### Superseded PR check: run lint/tests on main before stealing code

Before extracting fixes from a stale PR, run `make lint` and `make test` on main first. Many fixes may already be on main from earlier work. This avoids duplicating effort and tells you exactly what's still outstanding.

### Architectural alternatives: prototype before commenting

When you have a meaningful architectural concern or an alternative approach worth considering:

- Do not leave a review comment that only describes the idea in words
- Actually prototype it: create a branch off the PR branch, implement the alternative, write tests, verify they pass
- Push the prototype branch
- Leave a PR comment that includes:
  - A link to the prototype branch
  - A tradeoff table comparing both approaches (e.g. mount count, complexity, scope, future-proofing)
  - An explicit invitation to discuss

Do NOT block the PR on an architectural alternative. It is a discussion, not a blocker. The original PR proceeds to the normal merge decision independently.

### The prototype branch pattern

```bash
# 1. Get onto the PR branch
gh pr checkout <number>

# 2. Branch off it
git checkout -b prototype/<short-description>

# 3. Implement the alternative approach

# 4. Run the full test suite — it must pass before you push
make test   # or project equivalent

# 5. Commit with a clear message explaining the approach
git commit -m "prototype: <description of alternative approach>"

# 6. Push the prototype branch
git push origin prototype/<short-description>

# 7. Return to main
git checkout main

# 8. Leave a comment on the PR with the branch link and tradeoff comparison
gh pr comment <number> --body "..."
```

**Tradeoff table format:**

```
| Dimension        | Current approach | Prototype approach |
|------------------|------------------|--------------------|
| Mount count      | ...              | ...                |
| Complexity       | ...              | ...                |
| Scope of change  | ...              | ...                |
| Future-proofing  | ...              | ...                |
```

### Merging multiple PRs: rebase each one after the previous lands

When merging more than one PR in sequence, the second PR's branch was based on the old main — not the main that includes the first PR's changes. After merging PR A, you must rebase PR B's branch before merging it:

```bash
git checkout pr-B-branch
git rebase main          # rebase onto main that now includes PR A
# resolve any conflicts
npm run build            # or project equivalent — rebuild generated files
git checkout main
git merge --squash pr-B-branch
```

Conflicts from sequential merges are almost always in **generated files** (bundled dist, lock files, source maps). Resolve them by accepting the current main's version and rebuilding — never try to hand-merge a minified bundle.

### Step 4: Merge and thank the author

```bash
gh pr merge NUMBER --repo OWNER/REPO --squash
git pull origin main
```

**Always leave a comment thanking the author and identifying yourself:**

```bash
gh pr comment NUMBER --repo OWNER/REPO --body "Thanks @author! [1-2 sentence summary of what the PR does and why it's good].

— Claude [model], Claude Code [version], session [session-id]"
```

**Identification format:** Always include:
- Model name (e.g., `Claude Opus 4.6`)
- Harness + version (e.g., `Claude Code 2.1.56`)
- Session ID (if available)

To get version and session:
```bash
claude --version       # e.g., "2.1.56 (Claude Code)"

# Session ID is passed as --session-id to the Claude process; read it from the parent:
ps -p $PPID -o args= | grep -oE -- '--session-id [^ ]+' | awk '{print $2}'
```

`$CLAUDE_SESSION_ID` is not reliably set as an environment variable. Use the `ps` command above instead. If it returns empty (Claude was launched without an explicit session ID), omit the session ID from the comment rather than leaving a blank placeholder.

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Labeling without checking existing labels | Always run `gh label list` first |
| Letting `gh issue edit` auto-create labels | Create labels explicitly in Phase 3 |
| Confusing similar tool names | Read full issue body, not just title |
| Tagging closed issues | Skip unless explicitly asked |
| Applying labels with low confidence | Use `needs-categorization` instead |
| Skipping taxonomy approval | Always get approval before tagging |
| Closing an issue based on partial evidence | Grep for specific function/struct names, don't guess |
| Checking out PR code before security review | Always run pr-security-review agent first — it has read-only tools for safety |
| Merging without testing | Always run code review + test agents in parallel first |
| **Merging without user confirmation** | **NEVER merge autonomously. Always present findings and ask "Shall I merge PR #N?" — wait for an explicit yes before running any merge command, even when all checks are green** |
| Merging conflicting PRs | Check `mergeStateStatus` — never merge `DIRTY` |
| Missing identification in PR comments | Always include model, Claude Code version, session ID |
| Reviewing PRs sequentially | Launch code review and test agents in parallel |
| Thanking with wrong session ID | Get it via `ps -p $PPID -o args= \| grep -oE -- '--session-id [^ ]+' \| awk '{print $2}'`; omit if empty |
| Requesting changes for missing tests only | Write the tests yourself on the PR branch and push them |
| Leaving an architectural alternative as a comment only | Build a prototype branch, run tests, push it, then comment with branch link + tradeoff table |
| Blocking a PR on an architectural discussion | Prototype discussions are non-blocking; PR proceeds to normal merge decision |
| Pushing lint fixes to a fork PR branch | You can't — detect with `headRepositoryOwner`, then use manual squash pattern |
| Merging second PR without rebasing after first landed | Always rebase each PR branch onto current main before merging; conflicts will be in generated files — resolve by rebuilding |
| Merging a PR that adds a function never called | Grep for callers; if none exist outside the file, wire it up before merging |
| Duplicating work from a stale PR | Run lint/tests on main first to see what's actually still outstanding |
