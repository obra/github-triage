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
| ⚠️ REVIEW NEEDED | Present findings to user. Wait for explicit approval before proceeding. |
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
| Approve | Pass | Merge |
| Approve-with-notes | Pass | Merge, note in comment |
| Approve | Fail (pre-existing only) | Merge, note pre-existing failures |
| Approve | Fail (new failures) | Do not merge, report to user |
| Request changes | Any | Do not merge, report to user |

**Conflicts:** If `mergeStateStatus` is `DIRTY`, do not merge. Report to user with description of what conflicts.

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
- Session ID (from `$CLAUDE_SESSION_ID`)

To get version and session:
```bash
claude --version       # e.g., "2.1.56 (Claude Code)"
echo $CLAUDE_SESSION_ID
```

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
| Merging conflicting PRs | Check `mergeStateStatus` — never merge `DIRTY` |
| Missing identification in PR comments | Always include model, Claude Code version, session ID |
| Reviewing PRs sequentially | Launch code review and test agents in parallel |
| Thanking with wrong session ID | Get it from `$CLAUDE_SESSION_ID` at time of comment |
