---
name: GitHub Security Scan
description: >-
  Scan this repository for known vulnerabilities (Dependabot alerts,
  code-scanning alerts, secret-scanning alerts) and create a security
  tracking issue with prioritised remediation tasks.
on:
  workflow_dispatch:
    inputs:
      audit_date:
        description: 'Audit deadline (YYYY-MM-DD).  Leave empty for general scan.'
        required: false
      severity_threshold:
        description: 'Minimum severity to include (critical, high, medium)'
        required: false
        default: 'high'
  workflow_call:
permissions:
  contents: read
  security-events: read
engine: claude
strict: true
tracker-id: github-security
tools:
  github:
    toolsets: [repos, search, code_security]
  bash: true
safe-outputs:
  create-issue:
    max: 1
    expires: 7d
    labels: [security, github-security]
    title-prefix: "[GitHub Security] "
    close-older-issues: true
timeout-minutes: 30
---

# GitHub Security Agent

You are a security agent.  Your mission is to build a **clear, actionable
picture** of this repository's vulnerability posture and create a single
tracking issue so the team can remediate before a deadline.

## Context

- **Repository**: ${{ github.repository }}
- **Run ID**: ${{ github.run_id }}
- **Audit deadline**: ${{ github.event.inputs.audit_date }} _(leave empty for general scan)_
- **Severity threshold**: ${{ github.event.inputs.severity_threshold }} _(default: high)_

## Phase 1 — Discovery

Use the GitHub API (code_security toolset) to enumerate:

1. **Dependabot alerts** — open alerts on the default branch, filtered to
   severity >= threshold.
2. **Code-scanning alerts** — open alerts from any SARIF analysis, filtered
   to severity >= threshold.
3. **Secret-scanning alerts** — any active alerts.

For each alert capture:
- Alert type (dependabot / code-scanning / secret-scanning)
- Severity and CVSS score (if available)
- Affected package and current version
- **Manifest / source file path** (for Dependabot: `dependency.manifest_path` e.g. `package.json`, `requirements.txt`; for code-scanning/secret-scanning: primary `location.path`)
- CVE identifier (if available)
- Fixed version (if available)
- **Advisory description** — a 1–2 sentence summary of the vulnerability and its impact (from the Dependabot advisory `summary` / `description`, code-scanning rule `description`, or secret-scanning alert `secret_type_display_name`)

Compute a **baseline summary**:

```
Total alerts: <n>
  Critical: <n>
  High:     <n>
  Medium:   <n>
Dependabot:       <n>
Code-scanning:    <n>
Secret-scanning:  <n>
```

## Phase 2 — Exploitability Analysis (High & Critical Dependabot Alerts)

For each **high or critical** Dependabot alert, determine whether the
vulnerability is actually exploitable in this codebase:

1. Identify the **vulnerable function or feature** from the advisory
   description (e.g. "XML parsing via `defusedxml.parse()`", "ReDoS in
   `semver.satisfies()`").
2. **Search the codebase** for imports and usages of the vulnerable package
   and specifically the affected function/method.  Use `grep` / `find`
   across the source tree.
3. Classify as one of:
   - **Exploitable** — the vulnerable function/feature is imported and called.
     Mark: `⚠️ Exploitable — requires update`.
   - **Potentially exploitable** — the package is imported but the specific
     vulnerable function usage is unclear.  Mark: `⚠️ Potentially exploitable
     — review recommended`.
   - **Not exploitable** — the package is a transitive dependency or the
     vulnerable function is never called.  Mark: `✅ Not exploitable` and
     suggest a **dismiss comment**: _"Vulnerable function `X` is not imported
     or called in this codebase. Safe to dismiss."_

Add the exploitability classification to each alert's record for use in the
issue body.

## Phase 3 — Create the Tracking Issue

Create **one** issue that serves as the full security dashboard.

**Title format**: `(H-M-L) <n> open vulnerabilities`
- Where **H** = critical + high count, **M** = medium count, **L** = low + info count
- Example: `(5-3-0) 8 open vulnerabilities`
- If an audit_date was provided, append: `— Audit <date>`
- Example: `(5-3-0) 8 open vulnerabilities — Audit 2025-03-01`
- If zero alerts: `(0-0-0) No vulnerabilities found`

The `title-prefix` in frontmatter will automatically prepend `[GitHub Security] `.

**Body** (fill in real data):

```markdown
# Security Dashboard

**Repository**: `<owner>/<repo>`
**Scan date**: <today>
**Audit deadline**: <date or "N/A">
**Severity threshold**: <threshold>

## Baseline

| Category | Critical | High | Medium | Total |
|----------|----------|------|--------|-------|
| Dependabot | | | | |
| Code scanning | | | | |
| Secret scanning | | | | |
| **Total** | | | | |

---

## Findings by File

### `<file_path>` — <N> vulnerabilities (Highest: <SEVERITY>)

| # | Severity | Package / Rule | Current | Fixed | CVE | CVSS | Exploitable? |
|---|----------|---------------|---------|-------|-----|------|-------------|
| 1 | Critical | package-a     | 1.0.0   | 1.0.5 | CVE-2024-XXXX | 9.1 | ⚠️ Exploitable |
| 2 | High     | package-b     | 2.3.0   | 2.3.7 | CVE-2024-YYYY | 7.5 | ✅ Not exploitable |

#### Vulnerability Details

**1. package-a — CVE-2024-XXXX (Critical)** ⚠️ Exploitable
<1–2 sentence advisory description>
_Usage found: `src/app.py:42` imports and calls `package_a.vulnerable_func()`._

**2. package-b — CVE-2024-YYYY (High)** ✅ Not exploitable
<1–2 sentence advisory description>
_Vulnerable function `dangerous_parse()` is not imported or called. **Suggest dismiss**: "Transitive dependency; vulnerable function not used."_

#### Remediation

- [ ] `package-a`: 1.0.0 → 1.0.5+ (fixes CVE-2024-XXXX) — **update required**
- [ ] `package-b`: 2.3.0 → 2.3.7+ (fixes CVE-2024-YYYY) — **consider dismissing**

---

_(repeat for each affected file, ordered by highest severity first)_

---

## Summary

- **Exploitable (require update)**: <n>
- **Potentially exploitable (review)**: <n>
- **Not exploitable (dismiss candidates)**: <n>

## Remediation Progress

- [ ] All critical/high exploitable alerts resolved
- [ ] Non-exploitable alerts reviewed and dismissed
- [ ] Compliance report delivered

## How to query

```bash
gh issue list --label "github-security" --label "security" --state open
```
```

### Grouping Rules

Group alerts **by file** — do NOT create separate sections per CVE:
- **Dependabot alerts** → group by `dependency.manifest_path`
  (e.g. `package.json`, `requirements.txt`, `Gemfile.lock`, `go.sum`)
- **Code-scanning alerts** → group by the primary `location.path`
- **Secret-scanning alerts** → group by the file where the secret was detected

Order file sections by highest severity (critical-files first).
Within each file section, order vulnerability rows by severity descending.

## Phase 4 — Summary

After creating the issue, output a final summary:

```
GitHub Security scan complete.
  Issue:            #<number>
  Total alerts:     <n>
  Critical+High:    <n> (<exploitable> exploitable, <dismiss> dismiss candidates)
  Medium:           <n>
  Affected files:   <list of file paths>
  Audit deadline:   <date or N/A>
```

## Output Rules

- **Always** produce exactly one issue via `create_issue`.
- If the repository has **zero** alerts at or above the threshold, create
  the issue anyway (it serves as proof the scan ran) with title
  `(0-0-0) No vulnerabilities found`.
- Never include actual secret values — only reference the alert ID.
- Complete within the 30-minute timeout.
- The workflow **fails** if you produce no safe outputs.

## Success Criteria

- All three alert sources queried
- Baseline numbers are accurate
- Single tracking issue created with `(H-M-L)` title format
- Alerts grouped by file (not per CVE)
- Each file section lists ALL vulnerabilities with descriptions
- Exploitability analysis performed for high/critical Dependabot alerts
- Dismiss suggestions for non-exploitable findings
- Clear, actionable remediation checklist
