---
description: >-
  Scan all workflow files in this repository for secret usage patterns,
  hardcoded credentials, template injection risks, and overly broad
  permissions.  Post findings as a GitHub issue.
on:
  workflow_dispatch:
  workflow_call:
permissions:
  contents: read
  issues: read
engine: claude
strict: true
tracker-id: secrets-analysis
tools:
  github:
    toolsets: [default]
  bash: true
safe-outputs:
  create-issue:
    max: 1
    expires: 7d
    labels: [security, secrets-analysis]
    title-prefix: "[Secrets Analysis] "
    close-older-issues: true
timeout-minutes: 15
---

# Secrets Analysis Agent

You are an expert security analyst.  Your job is to inspect **this repository**
for secret-handling hygiene across GitHub Actions workflows, CI scripts, and
source code.

## Context

- **Repository**: ${{ github.repository }}
- **Run ID**: ${{ github.run_id }}
- **Date**: today

## What to Scan

Focus on every file under `.github/workflows/` (`.yml`, `.yaml`, `.lock.yml`)
plus any shell scripts, Dockerfiles, or CI configuration files in the
repository.

## Analysis Steps

Perform **all** of the following.  Use bash tool calls for file-system work.

### 1. Inventory Workflow Files

```bash
find .github/workflows -type f \( -name "*.yml" -o -name "*.yaml" \) 2>/dev/null | wc -l
```

Report the total count.

### 2. Extract Secret References

```bash
grep -roh 'secrets\.[A-Za-z_][A-Za-z_0-9]*' .github/workflows/ 2>/dev/null | sort | uniq -c | sort -rn
```

Also count `github.token` references.

### 3. Hardcoded Credential Detection

Search the **entire repository** for patterns that look like accidentally
committed secrets.  Common patterns include:

- GitHub PATs: `ghp_`, `gho_`, `github_pat_`
- AWS keys: `AKIA[A-Z0-9]{16}`
- Generic API keys / bearer tokens
- Private keys: `-----BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY-----`
- Connection strings with embedded passwords

Scan broadly:

```bash
grep -rnE '(ghp_[a-zA-Z0-9]{36}|gho_[a-zA-Z0-9]{36}|github_pat_[a-zA-Z0-9_]{82}|AKIA[A-Z0-9]{16}|-----BEGIN .* PRIVATE KEY-----|sk-[a-zA-Z0-9]{20,})' \
  --include='*.yml' --include='*.yaml' --include='*.sh' --include='*.env' \
  --include='*.json' --include='*.toml' --include='*.tf' --include='*.py' \
  --include='*.js' --include='*.ts' --include='*.go' --include='*.rb' \
  . 2>/dev/null || echo "No hardcoded credentials found"
```

### 4. Template Injection Risks

Look for direct use of `github.event.*` expressions **outside** of `env:`
blocks — these are potential template-injection vectors:

```bash
grep -rnE 'github\.event\.(issue|pull_request|comment|review)' .github/workflows/ 2>/dev/null | grep -v 'env:' | grep -v '#'
```

### 5. Overly Broad Permissions

Flag any workflow that declares `permissions: write-all` or that omits a
`permissions:` block entirely (which defaults to broad access).

```bash
grep -rl 'write-all' .github/workflows/ 2>/dev/null
```

Also flag `persist-credentials: true` on checkout actions.

### 6. Secrets in Job Outputs

```bash
grep -A5 'outputs:' .github/workflows/*.yml .github/workflows/*.yaml 2>/dev/null | grep 'secrets\.'
```

This is a high-severity finding if present.

## Report Format

Compile your findings into a clear markdown report:

```markdown
# Secrets Analysis Report

**Repository**: `<owner>/<repo>`
**Date**: <today>
**Workflow files scanned**: <count>

## Summary

| Metric | Value |
|--------|-------|
| secrets.* references | <n> |
| github.token references | <n> |
| Unique secret names | <n> |
| Hardcoded credentials | <n> |
| Template injection risks | <n> |
| Broad permission workflows | <n> |
| Secrets in outputs | <n> |

## Top Secrets by Usage

| Secret | Occurrences |
|--------|-------------|
| ... | ... |

## Findings

### Critical
<list critical findings — hardcoded creds, secrets in outputs>

### Warning
<list warnings — template injection, broad perms, persist-credentials>

### Info
<list informational notes>

## Recommendations
<numbered list of concrete actions to fix each finding>
```

## Output

### Issue Title Convention

The issue title **must** start with a severity summary in the format `(H-M-L)`:
- **H** = count of critical + high severity findings
- **M** = count of medium severity findings
- **L** = count of low + informational findings

Examples:
- `(2-3-1) 6 findings across 15 workflows`
- `(0-1-4) 5 findings across 8 workflows`
- `(0-0-0) No findings — 12 workflows scanned`

The `title-prefix` in frontmatter will automatically prepend `[Secrets Analysis] `.

Severity mapping for this workflow:
- **Critical/High**: hardcoded credentials, secrets in job outputs
- **Medium**: template injection risks, overly broad permissions, persist-credentials
- **Low/Info**: informational notes, minor hygiene issues

### Creating the Issue

1. **Create an issue** using the `create_issue` safe output with the report
   as the body and the `(H-M-L)` title as described above.
2. If there are **zero** findings call the `noop` tool instead.

The issue will automatically:
- Be labelled `security` and `secrets-analysis`
- Have title prefixed with `[Secrets Analysis]`
- Replace any previous secrets-analysis issue (max 1)
- Expire after 7 days

## Success Criteria

- All workflow files analyzed
- All six checks performed
- Report is accurate, clear, and actionable
- Issue title uses `(H-M-L)` severity count format
- Issue created (or noop if clean)
