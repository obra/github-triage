# github-triage

A Claude Code plugin for systematic GitHub issue and PR triage, with a security-gated PR review workflow.

## Components

### Skill: `github-triage`

Full triage workflow across 7 phases:
- **Phase 1–5**: Discover patterns, propose label taxonomy, create labels, tag issues/PRs, summarize
- **Phase 6**: Validate open issues against the codebase (oldest-first), close resolved ones with evidence
- **Phase 7**: Security-gated PR review — security analysis → parallel code review + local tests → merge + author credit

### Agent: `github-triage:pr-security-review`

Read-only security analysis agent (tools: Read, Grep, Glob, WebFetch — no execution, no writes).

Analyzes PR diffs before any local code checkout for:
- Malware and backdoors
- Supply chain attacks and typosquatting
- Credential theft and data exfiltration
- CI/CD and build script poisoning
- Test weaponization

Returns: ✅ SAFE / ⚠️ REVIEW NEEDED / 🚫 BLOCK

## Installation

```bash
claude plugin marketplace add /path/to/github-triage --name github-triage-dev
claude plugin install github-triage@github-triage-dev
```

## Usage

Use the `github-triage` skill for full triage sessions, or invoke `github-triage:pr-security-review` directly via the Task tool before checking out PR code.
