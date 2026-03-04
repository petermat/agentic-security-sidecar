# Security Sidecar

Drop-in security scanning for any GitHub repository using
[GitHub Agentic Workflows](https://github.com/github/gh-aw).

Copy the `.github/workflows/` folder into your repo, compile the agentic
workflows, and choose which scans to run.  That's it.

---

## What's Included

| Scan | File | What it does |
|------|------|-------------|
| **Secrets Analysis** | `secrets-analysis.md` | Scans workflow files and source code for hardcoded credentials, template injection risks, overly broad permissions, and secret-in-output leaks. |
| **Malicious Code Scan** | `malicious-code-scan.md` | Analyses recent commits for secret exfiltration patterns, obfuscated code, reverse shells, privilege escalation, and workflow tampering. |
| **GitHub Security** | `github-security.md` | Queries Dependabot, code-scanning, and secret-scanning alerts; performs exploitability analysis for high/critical findings; creates a single tracking issue with remediation tasks. |
| **Orchestrator** | `security-orchestrator.yml` | Standard GitHub Actions workflow that lets you pick which scans to run (all, or any combination) via a dropdown. |

## Quick Start

### 1. Install gh-aw

```bash
gh extension install github/gh-aw
```

### 2. Copy workflows into your repo

```bash
# From the root of YOUR repository
cp -r path/to/security-sidecar/.github/workflows/ .github/workflows/
```

You should now have these files:

```
.github/workflows/
  security-orchestrator.yml    # plain YAML — no compilation needed
  secrets-analysis.md          # gh-aw agentic workflow (source)
  malicious-code-scan.md       # gh-aw agentic workflow (source)
  github-security.md           # gh-aw agentic workflow (source)
```

### 3. Compile the agentic workflows

```bash
gh aw compile
```

This generates `.lock.yml` files for each `.md` workflow — these are the
compiled GitHub Actions workflows that actually execute.

### 4. Set up your AI engine secret

| Engine | Secret name |
|--------|-------------|
| GitHub Copilot | `COPILOT_GITHUB_TOKEN` |
| Anthropic Claude | `ANTHROPIC_API_KEY` |
| OpenAI Codex | `OPENAI_API_KEY` |

Add the secret in **Settings > Secrets and variables > Actions**.

### 5. Commit and push

```bash
git add .github/workflows/
git commit -m "Add security sidecar workflows"
git push
```

### 6. Run it

Go to **Actions > Security Sidecar > Run workflow** and pick which scans
you want:

| Option | Scans |
|--------|-------|
| `secrets,malicious-code,github-security` | All three (default, also used by the daily schedule) |
| `secrets` | Secrets analysis only |
| `malicious-code` | Malicious code scan only |
| `github-security` | GitHub security scan only |
| `secrets,malicious-code` | Secrets + malicious code |
| `secrets,github-security` | Secrets + GitHub security |
| `malicious-code,github-security` | Malicious code + GitHub security |

## Configuration

### Schedule

The orchestrator runs daily at 06:00 UTC by default.  Edit the `cron`
expression in `security-orchestrator.yml`:

```yaml
schedule:
  - cron: '0 6 * * *'    # change this
```

### AI Engine

Each `.md` workflow currently uses `engine: claude`.  To switch engine, edit
the `engine:` field in the frontmatter:

```yaml
engine: claude    # or: copilot, codex
```

Then recompile: `gh aw compile`.

### GitHub Security Scan Inputs

The GitHub security scan accepts optional parameters when triggered manually:

| Input | Description | Default |
|-------|-------------|---------|
| `audit_date` | Deadline in YYYY-MM-DD format | _(none)_ |
| `severity_threshold` | Minimum severity: `critical`, `high`, `medium` | `high` |

### Editing Agent Instructions

The markdown body of each `.md` file (everything below the `---` frontmatter)
is loaded at **runtime**.  You can edit it directly on GitHub.com and changes
take effect on the next run — no recompilation needed.

The **frontmatter** (triggers, permissions, tools, safe-outputs) is baked
into the `.lock.yml` at compile time.  If you change frontmatter, run
`gh aw compile` again.

## Architecture

```
security-orchestrator.yml          (standard YAML — dispatches to lock files)
  |
  |-- secrets-analysis.lock.yml    (compiled from secrets-analysis.md)
  |-- malicious-code-scan.lock.yml (compiled from malicious-code-scan.md)
  |-- github-security.lock.yml    (compiled from github-security.md)
```

Each agentic workflow runs an AI agent (Copilot / Claude / Codex) inside a
sandboxed GitHub Actions job with:

- **Read-only permissions** on the agent step
- **Safe outputs** for write operations (issues, alerts)
- **Network firewall** restricting outbound traffic
- **Secret redaction** on all agent output
- **SHA-pinned actions** in the compiled lock file

## How It Works

1. The **orchestrator** parses the `scans` input and conditionally triggers
   each compiled agentic workflow via `uses:`.
2. Each `.lock.yml` boots the gh-aw runtime: MCP gateway, firewall, tools,
   and the AI engine.
3. The AI agent reads the markdown instructions, uses `bash` and `github`
   tools to analyse the repository, and produces **safe outputs** (issues,
   code-scanning alerts).
4. A summary job reports pass/fail for each scan in the workflow run.

## Updating

When upstream instructions change:

```bash
# Edit the .md file (frontmatter or body)
vim .github/workflows/secrets-analysis.md

# Recompile if you changed frontmatter
gh aw compile

# Commit both files
git add .github/workflows/secrets-analysis.md .github/workflows/secrets-analysis.lock.yml
git commit -m "Update secrets analysis workflow"
git push
```

If you only changed the **markdown body** (not frontmatter), you can skip
recompilation — the lock file loads the body at runtime.
