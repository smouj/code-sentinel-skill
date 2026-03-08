---
name: Code Sentinel
version: 2.1.0
description: Advanced code security scanner for detecting vulnerabilities in commits and PRs
maintainer: security-team
tags: [security, scanning, code, vulnerabilities, saastools]
dependencies: [git, semgrep, trufflehog, bandit, gitleaks, docker]
requires_network: true
---

# Code Sentinel

## Purpose

Code Sentinel performs deep static analysis (SAST), secrets detection, and dependency scanning on code commits and pull requests. It integrates into CI/CD pipelines to catch:

- Hardcoded credentials (API keys, tokens, passwords)
- SQL injection, XSS, command injection patterns
- Insecure deserialization and unsafe functions
- Vulnerable dependencies (via SBOM analysis)
- Misconfigurations in Dockerfiles, K8s manifests, IaC

**Real use cases:**
- Blocking PR merges when high-severity vulnerabilities are detected
- Automated security reports in GitHub/GitLab PR comments
- Nightly full-repository scans for drift detection
- Compliance verification (PCI-DSS, SOC2) with evidence generation
- Developer feedback via pre-commit hooks for rapid remediation

## Scope

### Commands

`sentinel scan` - Scan specific files, directories, or commits

```bash
# Scan staged changes
sentinel scan --staged

# Scan specific commit
sentinel scan --commit abc123

# Scan PR diff (when in CI)
sentinel scan --pr 42

# Scan entire repo
sentinel scan --path . --full

# Custom rules
sentinel scan --rules custom-rules/ --severity high,critical

# Output formats
sentinel scan --format sarif  # for GitHub Code Scanning
sentinel scan --format json   # for custom parsers
sentinel scan --format html   # for human review
```

`sentinel config` - Configure scanner profiles

```bash
# Set project security profile
sentinel config set profile webapp   # includes OWASP Top 10
sentinel config set profile mobile   # mobile-specific rules
sentinel config set profile api      # API-focused checks

# Configure ignore patterns
sentinel config set ignore "*.min.js,dist/*"

# Set severity thresholds
sentinel config set threshold high   # fail on high+ issues
```

`sentinel fix` - Apply automated fixes for low-risk issues

```bash
# Interactive fix mode
sentinel fix --interactive

# Auto-fix specific rule
sentinel fix --rule hardcoded-secret --file src/config.py

# Dry-run first
sentinel fix --dry-run
```

`sentinel report` - Generate and manage security reports

```bash
# Generate executive summary
sentinel report --type executive --output security-summary.html

# Export compliance evidence
sentinel report --compliance pci-dss --format pdf

# Compare scans (drift detection)
sentinel report --baseline scan-001.json --current scan-002.json
```

`sentinel daemon` - Run as background service for real-time scanning

```bash
# Start daemon (daemon mode)
sentinel daemon --watch . --port 8080

# Connect IDE plugin to daemon
sentinel daemon --ide-integration
```

## Work Process

### 1. Pre-scan Preparation

```bash
# Clone target repository
git clone <repo-url>
cd <repo>

# Load configuration
sentinel config load .sentinel.yml  # project-specific config
sentinel config load ~/.sentinel/profiles/default.yml  # global config

# Verify toolchain
sentinel doctor  # checks semgrep, trufflehog, docker are installed
```

### 2. Select Scan Target

By commit range:
```bash
# Scan commits since last release
sentinel scan --since v1.2.0

# Scan commits in current branch not in main
sentinel scan --branch feature/login --against main
```

By file type:
```bash
# Scan only Python secrets
sentinel scan --lang python --rule HARDCODED_SECRET

# Scan container images
sentinel scan --docker-image myapp:latest
```

### 3. Execute Multi-Engine Scan

Sentinel runs sequential analysis:

1. **Secrets Detection** (truffleHog + gitleaks)
   - Scans for AWS keys, GitHub tokens, Slack webhooks
   - Entropy analysis for high-entropy strings
   - Regex patterns for 200+ secret formats

2. **SAST** (semgrep with custom rulesets)
   - OWASP Top 10 rules
   - Language-specific vulnerabilities
   - Custom organization rules

3. **SBOM + Dependency Check** (if package.json/requirements.txt present)
   - Generate CycloneDX SBOM
   - Query vuln databases (NVD, GitHub Advisory)
   - Check for known CVEs

4. **IaC Scanning** (checkov/terrascan if IaC files detected)
   - Terraform, CloudFormation, K8s, Dockerfile security

### 4. Triaging & Deduplication

```bash
# Interactive triage
sentinel triage --interactive  # mark as false-positive, accept risk, etc.

# Auto-deduplicate
sentinel dedupe --strategy hash  # merge identical findings across commits
```

### 5. Report Generation

Output includes:
- **Summary**: Count by severity, category, file
- **Findings**: Line numbers, code snippets, CWE IDs, fix suggestions
- **Evidence**: Commit hash, author, timestamp
- **Remediation**: Specific code changes, OWASP references, CVSS scores

Sample JSON output:
```json
{
  "scan_id": "sentinel-2024-03-15-abc123",
  "repo": "myorg/myapp",
  "commit": "abc123def456",
  "findings": [
    {
      "severity": "critical",
      "rule": "hardcoded-aws-secret",
      "file": "src/aws.py",
      "line": 42,
      "code": "AWS_SECRET = 'AKIAIOSFODNN7EXAMPLE'",
      "confidence": "high",
      "cwe": "CWE-798",
      "fix": "Use AWS Secrets Manager or environment variable",
      "exploitability": 9.1
    }
  ]
}
```

### 6. Integration Hooks

**GitHub Actions** (`.github/workflows/sentinel.yml`):
```yaml
- name: Run Code Sentinel
  uses: code-sentinel/action@v2
  with:
    sarif-output: true
    fail-on-severity: high
- name: Upload SARIF
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: sentinel-results.sarif
```

**GitLab CI** (`.gitlab-ci.yml`):
```yaml
sentinel_scan:
  script:
    - sentinel scan --pr $CI_MERGE_REQUEST_IID --format sarif
  artifacts:
    paths: [sentinel-results.sarif]
    expire_in: 1 week
```

## Golden Rules

1. **Never disable scanning** – Always run sentinel on PRs targeting main/staging
2. **Treat secrets as critical** – Any hardcoded credential must block merge
3. **Fix in source, not in config** – If rule generates false-positive, improve the rule, not the ignore list
4. **Review findings within 4 hours** – Security findings have SLA based on severity:
   - Critical: 1 hour
   - High: 4 hours
   - Medium: 24 hours
   - Low: 1 week
5. **Maintain custom rules as code** – Store organization-specific rules in `security/rules/` and version them
6. **Sign all commits** – Enforce GPG-signed commits to prevent tampering
7. **Rotate if secret leaked** – If sentinel finds exposed credential, rotate immediately, regardless of age
8. **Scan everything** – Include scripts, tests, docs (secrets often hide there)

## Examples

### Example 1: Pre-commit Hook for Python Project

`.git/hooks/pre-commit`:
```bash
#!/bin/bash
# Code Sentinel pre-commit hook

echo "🔍 Running Code Sentinel on staged changes..."

# Only scan staged Python/JS files
STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACMR | grep -E '\.(py|js|ts)$')
if [ -z "$STAGED_FILES" ]; then
  exit 0
fi

# Run sentinel
sentinel scan --files "$STAGED_FILES" --format json --output /tmp/sentinel-precommit.json

# Check for failures
if jq -e '.findings[] | select(.severity | test("critical|high"))' /tmp/sentinel-precommit.json > /dev/null; then
  echo "❌ Critical/High vulnerabilities detected. Commit blocked."
  sentinel report /tmp/sentinel-precommit.json --format pretty
  exit 1
fi

echo "✅ No blocking vulnerabilities found."
exit 0
```

### Example 2: PR Comment Automation

GitHub Action step:
```yaml
- name: Comment PR with findings
  run: |
    sentinel scan --pr ${{ github.event.pull_request.number }} --format json > results.json
    
    CRITICAL=$(jq '[.findings[] | select(.severity=="critical")] | length' results.json)
    HIGH=$(jq '[.findings[] | select(.severity=="high")] | length' results.json)
    
    if [ "$CRITICAL" -gt 0 ] || [ "$HIGH" -gt 0 ]; then
      gh pr comment ${{ github.event.pull_request.number }} \
        --body "🚨 **Code Sentinel found $CRITICAL critical and $HIGH high severity vulnerabilities.**\n\nRun \`sentinel scan --pr ${{ github.event.pull_request.number }}\` locally to view details."
    fi
```

### Example 3: False Positive Triage Script

`scripts/triage-findings.sh`:
```bash
#!/bin/bash
SCAN_ID=$1
MARK_AS=$2  # "false-positive", "accept-risk", "wontfix"

# Get all findings
sentinel findings list --scan $SCAN_ID --format json > /tmp/findings.json

# Loop through and mark
jq -c '.[]' /tmp/findings.json | while read -r finding; do
  ID=$(echo "$finding" | jq -r '.id')
  sentinel findings update $ID --status $MARK_AS --reason "Reviewed by $USER"
done

echo "Triage complete for scan $SCAN_ID"
```

## Verification

```bash
# 1. Verify installation
sentinel --version  # should output 2.1.0
sentinel doctor     # checks all dependencies

# 2. Test scan on known vulnerable code
curl -s https://raw.githubusercontent.com/OWASP/CheatSheetSeries/master/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.md > test_vuln.py
sentinel scan test_vuln.py
# Should detect SQL injection patterns

# 3. Verify GitHub integration (if configured)
sentinel test-github --repo myorg/testrepo
# Should post test comment to PR

# 4. Check SARIF output validity
sentinel scan . --format sarif > results.sarif
jq '.' results.sarif  # should parse without error
```

## Rollback

If a scan incorrectly blocks a merge or generates false positives:

1. **Temporary override** (for urgent deployments):
```bash
# Bypass sentinel check for specific commit
git commit --no-verify -m "Urgent fix"
# OR set environment variable
SENTINEL_SKIP=true git push
```

2. **Rollback sentinel rule update** (if recent rule change caused issues):
```bash
# List recent config changes
sentinel config history

# Restore previous config
sentinel config restore --revision abc123 --file .sentinel.yml

# Disable problematic rule set
sentinel config disable-rules --rule-id XSS-001
```

3. **Cleanup corrupted scan data**:
```bash
# Remove specific scan results
sentinel scans delete --id sentinel-2024-03-15-xyz789

# Reset local cache (forces fresh scan)
rm -rf ~/.cache/sentinel/
```

4. **Revert daemon state**:
```bash
sentinel daemon stop
# Restore daemon config from backup
cp ~/.sentinel/backups/daemon-config-2024-03-14.yml ~/.sentinel/daemon.yml
sentinel daemon start
```

## Dependencies & Requirements

**System:**
- Python 3.9+ (for semgrep, custom scripts)
- Docker (for container scanning)
- Git 2.30+

**Python packages** (installed via `pip install codesentinel`):
- semgrep>=1.50.0
- truffleHog>=3.0.0
- gitleaks>=8.18.0
- pyyaml, jsonschema, tabulate

**Optional:**
- trivy (for container image scanning)
- checkov (for IaC)
- bandit (Python-specific)
- npm audit (JavaScript)

**Environment variables:**
- `SENTINEL_API_KEY` – For commercial rule feed (default: reads from `~/.sentinel/api-key`)
- `SENTINEL_CONFIG_PATH` – Override config location (default: `.sentinel.yml`)
- `SENTINEL_CACHE_DIR` – Cache directory (default: `~/.cache/sentinel/`)
- `SENTINEL_GITHUB_TOKEN` – For GitHub API access (for PR comments, advisory data)

## Troubleshooting

**Issue**: `sentinel scan` hangs on large repo
**Fix**: Exclude directories with `sentinel config set ignore "node_modules,build,dist"` or use `--max-files 1000`

**Issue**: False positives on test fixtures
**Fix**: Mark directory as test in config: `sentinel config set test-dirs "test,tests,__test__"`

**Issue**: SARIF file rejected by GitHub
**Fix**: Validate SARIF schema: `sentinel validate-sarif results.sarif` and ensure version 2.1.0

**Issue**: No secrets detected in known vulnerable file
**Fix**: Update regex patterns: `sentinel rules update --source official` and verify `trufflehog` version >= 3.0

**Issue**: High memory usage on monorepo
**Fix**: Use incremental scan: `sentinel scan --changed-since origin/main --workers 2` (limit workers)

**Issue**: Docker scanning fails with permission denied
**Fix**: Add user to docker group: `sudo usermod -aG docker $USER` and restart session
```