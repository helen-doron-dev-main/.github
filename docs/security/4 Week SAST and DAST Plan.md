Here's a complete, staged implementation plan tailored for **ISO 27001 audit compliance** with Semgrep + OWASP ZAP.  This focuses on **demonstrable security controls** rather than comprehensive coverage.

---

## **ISO 27001 Compliance Focus**

For ISO audits, you need to demonstrate: 
- ✅ **A. 14.2. 1** - Secure development policy (documented process)
- ✅ **A.14.2.5** - Secure system engineering principles (SAST in CI/CD)
- ✅ **A.14.2.8** - System security testing (DAST on deployed apps)
- ✅ **A.12.6.1** - Management of technical vulnerabilities (tracking & remediation)

**You don't need perfect security — you need documented, repeatable processes with evidence.**

---

## **Implementation Stages Overview**

| Stage | Timeline | Goal |
|-------|----------|------|
| **Stage 1** | Week 1 | Foundation - Documentation & basic SAST |
| **Stage 2** | Week 2-3 | SAST on all repos |
| **Stage 3** | Week 4 | DAST on web-exposed applications |
| **Stage 4** | Ongoing | Evidence collection & remediation tracking |

---

## **Stage 1: Foundation (Week 1)**

### 1.1 Create Security Policy Documentation

Create a `SECURITY.md` in your organization's `.github` repository (applies to all repos):

````markdown name=.github/SECURITY.md
# Security Policy

## Secure Development Lifecycle

This organization implements automated security testing as part of our SDLC in compliance with ISO 27001 (A.14.2).

### Static Application Security Testing (SAST)
- **Tool**: Semgrep
- **Frequency**: Every pull request and weekly full scans
- **Scope**: All application source code
- **Rules**:  OWASP Top 10, language-specific security rules

### Dynamic Application Security Testing (DAST)
- **Tool**: OWASP ZAP
- **Frequency**: Weekly baseline scans, monthly full scans
- **Scope**: All web-exposed applications and APIs

### Vulnerability Management
- **Critical/High**:  Remediate within 30 days
- **Medium**:  Remediate within 90 days
- **Low**:  Tracked and reviewed quarterly

## Reporting a Vulnerability

Please report security vulnerabilities to security@yourcompany.com. 

## Evidence & Audit Trail

- Scan results are retained as GitHub Actions artifacts (90 days)
- Findings are tracked as GitHub Issues with `security` label
- Remediation is tracked via pull request references

_Last reviewed: 2026-01-13_
_Next review: 2026-07-13_
````

### 1.2 Create Reusable Workflow Templates

Create reusable workflows in your `.github` repository that all repos can reference:

```yaml name=.github/workflows/reusable-semgrep.yml
name: Reusable Semgrep SAST

on:
  workflow_call:
    inputs: 
      additional_configs:
        description: 'Additional Semgrep rule configs'
        required: false
        type: string
        default: ''

jobs:
  semgrep: 
    name:  SAST Scan
    runs-on: ubuntu-latest
    container:
      image: semgrep/semgrep
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Semgrep
        run: |
          semgrep scan \
            --config auto \
            --config p/security-audit \
            --config p/secrets \
            --config p/owasp-top-ten \
            ${{ inputs.additional_configs }} \
            --sarif --output semgrep. sarif \
            --json --output semgrep.json
        continue-on-error: true
      
      - name: Generate Summary
        if: always()
        run: |
          echo "## 🔒 SAST Scan Results" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Tool:** Semgrep" >> $GITHUB_STEP_SUMMARY
          echo "**Date:** $(date -u +%Y-%m-%dT%H:%M:%SZ)" >> $GITHUB_STEP_SUMMARY
          echo "**Commit:** ${{ github.sha }}" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          if [ -f semgrep.json ]; then
            TOTAL=$(cat semgrep.json | jq '.results | length')
            HIGH=$(cat semgrep.json | jq '[.results[] | select(.extra.severity == "ERROR")] | length')
            MEDIUM=$(cat semgrep.json | jq '[.results[] | select(.extra.severity == "WARNING")] | length')
            echo "| Severity | Count |" >> $GITHUB_STEP_SUMMARY
            echo "|----------|-------|" >> $GITHUB_STEP_SUMMARY
            echo "| High | $HIGH |" >> $GITHUB_STEP_SUMMARY
            echo "| Medium | $MEDIUM |" >> $GITHUB_STEP_SUMMARY
            echo "| Total | $TOTAL |" >> $GITHUB_STEP_SUMMARY
          fi
      
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: semgrep.sarif
        continue-on-error: true
      
      - name: Upload artifacts (audit evidence)
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: sast-report-${{ github.sha }}
          path: |
            semgrep.sarif
            semgrep.json
          retention-days: 90  # ISO audit evidence retention
```

```yaml name=.github/workflows/reusable-zap-baseline.yml
name: Reusable ZAP Baseline Scan

on:
  workflow_call:
    inputs:
      target_url:
        description: 'Target URL to scan'
        required:  true
        type: string
      scan_type:
        description: 'Scan type:  baseline or api'
        required: false
        type: string
        default: 'baseline'
      openapi_url:
        description: 'OpenAPI spec URL (for API scans)'
        required: false
        type: string

jobs:
  zap: 
    name:  DAST Scan
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run ZAP Baseline Scan
        if: inputs.scan_type == 'baseline'
        uses: zaproxy/action-baseline@v0.14.0
        with:
          target:  ${{ inputs.target_url }}
          fail_action:  false
          allow_issue_writing: true
          artifact_name: dast-report
      
      - name: Run ZAP API Scan
        if:  inputs.scan_type == 'api'
        uses: zaproxy/action-api-scan@v0.9.0
        with:
          target: ${{ inputs.openapi_url }}
          format: openapi
          fail_action: false
          allow_issue_writing:  true
          artifact_name: dast-api-report
      
      - name: Generate Summary
        if: always()
        run: |
          echo "## 🌐 DAST Scan Results" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Tool:** OWASP ZAP" >> $GITHUB_STEP_SUMMARY
          echo "**Target:** ${{ inputs.target_url }}" >> $GITHUB_STEP_SUMMARY
          echo "**Date:** $(date -u +%Y-%m-%dT%H:%M:%SZ)" >> $GITHUB_STEP_SUMMARY
          echo "**Type:** ${{ inputs.scan_type }}" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "See artifacts for detailed report." >> $GITHUB_STEP_SUMMARY
      
      - name: Upload evidence
        uses: actions/upload-artifact@v4
        if:  always()
        with:
          name: dast-report-${{ github.run_id }}
          path: |
            report_html.html
            report_json.json
          retention-days: 90
```

---

## **Stage 2: SAST on All Repos (Week 2-3)**

### 2.1 Node.js Monolithic Backend

```yaml name=.github/workflows/security. yml
# For:  backend-api repo (Node.js monolith)
name: Security Scans

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'  # Weekly Monday 2 AM UTC

concurrency:
  group: security-${{ github.ref }}
  cancel-in-progress: true

jobs:
  sast:
    name: SAST - Semgrep
    uses: your-org/. github/.github/workflows/reusable-semgrep.yml@main
    with:
      additional_configs: '--config p/javascript --config p/typescript --config p/nodejs'
```

**Node.js-specific Semgrep rules to include:**
- `p/javascript` - JS security patterns
- `p/typescript` - TS security patterns  
- `p/nodejs` - Node.js specific (e.g., path traversal, injection)
- `p/jwt` - JWT security issues
- `p/express` - Express.js vulnerabilities (if using Express)

---

### 2.2 Web Applications (Frontend)

```yaml name=.github/workflows/security. yml
# For: frontend web app repos
name: Security Scans

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'

concurrency:
  group: security-${{ github.ref }}
  cancel-in-progress: true

jobs:
  sast: 
    name: SAST - Semgrep
    uses:  your-org/.github/.github/workflows/reusable-semgrep.yml@main
    with:
      additional_configs: '--config p/javascript --config p/typescript --config p/react --config p/html'
```

**Web app-specific rules:**
- `p/react` - React security (XSS, dangerouslySetInnerHTML)
- `p/html` - HTML injection patterns
- `p/javascript` - General JS security

---

### 2.3 Flutter App (iOS, Android, Web, Desktop)

```yaml name=.github/workflows/security.yml
# For: Flutter app repo
name: Security Scans

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'

concurrency:
  group: security-${{ github.ref }}
  cancel-in-progress: true

jobs:
  sast: 
    name: SAST - Semgrep
    uses: your-org/.github/.github/workflows/reusable-semgrep.yml@main
    with:
      additional_configs: '--config p/dart --config p/secrets'
```

**Flutter/Dart-specific focus:**
- `p/dart` - Dart language security
- `p/secrets` - Hardcoded API keys, tokens (critical for mobile apps!)

**Additional mobile security considerations for ISO:**

```yaml name=.github/workflows/flutter-security.yml
name: Flutter Security Checks

on:
  push: 
    branches: [main]
  pull_request:
    branches:  [main]

jobs:
  sast:
    name: SAST - Semgrep
    runs-on: ubuntu-latest
    container:
      image: semgrep/semgrep
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Semgrep
        run: |
          semgrep scan \
            --config auto \
            --config p/dart \
            --config p/secrets \
            --config p/owasp-top-ten \
            --sarif --output semgrep.sarif \
            --json --output semgrep.json
        continue-on-error:  true
      
      - name:  Check for hardcoded secrets (critical for mobile)
        run: |
          echo "## 🔑 Secrets Detection" >> $GITHUB_STEP_SUMMARY
          # Check for common mobile app secrets
          if grep -rn "api[_-]key\|apikey\|secret\|password\|token" --include="*.dart" lib/ 2>/dev/null | grep -v "test\|mock\|example" | head -20; then
            echo "⚠️ Potential hardcoded secrets found - review required" >> $GITHUB_STEP_SUMMARY
          else
            echo "✅ No obvious hardcoded secrets detected" >> $GITHUB_STEP_SUMMARY
          fi
        continue-on-error: true
      
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: sast-report-${{ github. sha }}
          path: |
            semgrep.sarif
            semgrep.json
          retention-days: 90
```

---

### 2.4 Unity Apps (C#)

```yaml name=.github/workflows/security.yml
# For: Unity game/app repos
name: Security Scans

on:
  push: 
    branches: [main]
  pull_request:
    branches: [main]
  schedule: 
    - cron: '0 2 * * 1'

concurrency:
  group:  security-${{ github.ref }}
  cancel-in-progress:  true

jobs:
  sast:
    name: SAST - Semgrep
    runs-on: ubuntu-latest
    container:
      image: semgrep/semgrep
    
    steps:
      - uses:  actions/checkout@v4
      
      - name: Run Semgrep for Unity/C#
        run: |
          semgrep scan \
            --config auto \
            --config p/csharp \
            --config p/secrets \
            --config p/owasp-top-ten \
            --exclude='Library' \
            --exclude='Temp' \
            --exclude='Logs' \
            --exclude='*.meta' \
            --sarif --output semgrep.sarif \
            --json --output semgrep.json
        continue-on-error: true
      
      - name: Generate Summary
        if: always()
        run: |
          echo "## 🎮 Unity SAST Results" >> $GITHUB_STEP_SUMMARY
          if [ -f semgrep.json ]; then
            TOTAL=$(cat semgrep.json | jq '.results | length')
            echo "Total findings: $TOTAL" >> $GITHUB_STEP_SUMMARY
          fi
      
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: sast-report-${{ github. sha }}
          path: |
            semgrep.sarif
            semgrep.json
          retention-days: 90
```

**Unity-specific notes:**
- Exclude Unity auto-generated folders (`Library/`, `Temp/`, `Logs/`)
- Focus on `Assets/Scripts/` directory
- `p/csharp` covers common C# vulnerabilities
- `p/secrets` catches hardcoded keys (important if Unity app connects to backend)

---

## **Stage 3: DAST on Web-Exposed Apps (Week 4)**

### 3.1 Backend API - DAST

```yaml name=.github/workflows/dast.yml
# For: backend-api repo
name: DAST Scans

on:
  schedule:
    - cron: '0 3 * * 1'  # Weekly Monday 3 AM UTC
  workflow_dispatch:
    inputs: 
      target_url:
        description: 'API URL to scan'
        required: true
        default: 'https://staging-api.yourapp.com'

jobs:
  dast-api:
    name: DAST - API Scan
    uses: your-org/.github/.github/workflows/reusable-zap-baseline.yml@main
    with:
      target_url: ${{ github.event.inputs.target_url || 'https://staging-api.yourapp.com' }}
      scan_type: 'api'
      openapi_url: 'https://staging-api.yourapp.com/api-docs/swagger.json'
  
  dast-baseline:
    name: DAST - Baseline Scan
    uses: your-org/.github/.github/workflows/reusable-zap-baseline.yml@main
    with:
      target_url: ${{ github.event.inputs.target_url || 'https://staging-api.yourapp.com' }}
      scan_type: 'baseline'
```

---

### 3.2 Web Apps - DAST

```yaml name=.github/workflows/dast.yml
# For: frontend web app repos
name: DAST Scans

on:
  schedule:
    - cron:  '0 4 * * 1'  # Weekly Monday 4 AM UTC
  workflow_dispatch:
    inputs: 
      target_url:
        description: 'Web app URL to scan'
        required: true
        default: 'https://staging. yourwebapp.com'

jobs:
  dast: 
    name: DAST - Web App Scan
    uses: your-org/.github/.github/workflows/reusable-zap-baseline.yml@main
    with:
      target_url: ${{ github.event.inputs.target_url || 'https://staging.yourwebapp.com' }}
      scan_type: 'baseline'
```

---

### 3.3 Flutter Web - DAST (if web target is deployed)

```yaml name=.github/workflows/dast.yml
# For: Flutter app repo (web deployment only)
name: DAST Scans

on:
  schedule:
    - cron: '0 5 * * 1'
  workflow_dispatch: 

jobs:
  dast: 
    name: DAST - Flutter Web
    uses: your-org/.github/.github/workflows/reusable-zap-baseline.yml@main
    with:
      target_url: 'https://staging.yourflutterapp.com'
      scan_type: 'baseline'
```

---

### 3.4 Unity Apps - No DAST Required

Unity apps are client-side applications.  DAST applies to the **backend API** they connect to (covered in 3.1).

**For ISO audit, document this:**

```markdown name=docs/security/unity-apps-security.md
# Unity Applications Security Testing

## Scope

Unity applications are client-side applications distributed via app stores. 
Dynamic Application Security Testing (DAST) is not applicable to client-side 
applications.

## Security Controls Applied

1. **SAST (Static Analysis)**:  Semgrep scans on every PR and weekly
   - Ruleset: p/csharp, p/secrets, p/owasp-top-ten
   - Evidence: GitHub Actions artifacts retained 90 days

2. **Backend API Security**: All Unity apps communicate with our backend API, 
   which undergoes weekly DAST scans (see backend-api security documentation)

3. **Secrets Management**: Semgrep secrets detection prevents hardcoded 
   credentials in client applications

4. **Dependency Scanning**: (Future consideration for Stage 5)

_Last reviewed: 2026-01-13_
```

---

## **Stage 4: Evidence Collection & Tracking (Ongoing)**

### 4.1 Issue Tracking Template

Create an issue template for security findings:

```yaml name=.github/ISSUE_TEMPLATE/security-finding.yml
name: Security Finding
description: Track a security vulnerability finding
labels: ["security", "vulnerability"]
body: 
  - type: dropdown
    id: severity
    attributes:
      label: Severity
      options:
        - Critical
        - High
        - Medium
        - Low
    validations:
      required: true
  
  - type: dropdown
    id: source
    attributes:
      label:  Detection Source
      options:
        - SAST (Semgrep)
        - DAST (OWASP ZAP)
        - Manual Review
        - External Report
    validations:
      required: true
  
  - type: input
    id: cwe
    attributes:
      label: CWE ID
      description: Common Weakness Enumeration ID (if applicable)
      placeholder: "CWE-79"
  
  - type: textarea
    id: description
    attributes:
      label: Description
      description:  Describe the vulnerability
    validations:
      required: true
  
  - type: textarea
    id: remediation
    attributes:
      label: Remediation Plan
      description: How will this be fixed?
  
  - type: input
    id: target-date
    attributes:
      label:  Target Remediation Date
      description: Based on severity SLA
      placeholder: "2026-02-13"
```

### 4.2 Monthly Security Report Workflow

```yaml name=.github/workflows/security-report.yml
# Place in your org's .github repo
name: Monthly Security Report

on:
  schedule: 
    - cron: '0 6 1 * *'  # First day of each month at 6 AM UTC
  workflow_dispatch:

jobs: 
  generate-report:
    name: Generate Security Report
    runs-on:  ubuntu-latest
    
    steps:
      - name: Generate Report
        uses: actions/github-script@v7
        with:
          script:  |
            const { owner, repo } = context.repo;
            
            // Get all security issues
            const issues = await github.rest.issues.listForRepo({
              owner,
              repo,
              labels: 'security',
              state: 'all',
              since: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString()
            });
            
            const open = issues.data.filter(i => i.state === 'open').length;
            const closed = issues.data.filter(i => i.state === 'closed').length;
            
            const report = `
            # Monthly Security Report
            **Period:** ${new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString().split('T')[0]} to ${new Date().toISOString().split('T')[0]}
            **Generated:** ${new Date().toISOString()}
            
            ## Summary
            | Metric | Count |
            |--------|-------|
            | Open Security Issues | ${open} |
            | Closed Security Issues | ${closed} |
            | Total Findings | ${issues.data.length} |
            
            ## ISO 27001 Compliance Status
            - ✅ A. 14.2. 1 - Secure development policy:  Documented
            - ✅ A. 14.2.5 - SAST implemented in CI/CD pipeline
            - ✅ A.14.2.8 - DAST implemented for web applications
            - ✅ A.12.6.1 - Vulnerability tracking via GitHub Issues
            
            ## Evidence Locations
            - SAST Reports: GitHub Actions Artifacts (90-day retention)
            - DAST Reports: GitHub Actions Artifacts (90-day retention)
            - Remediation Tracking: GitHub Issues with 'security' label
            `;
            
            console.log(report);
            
            // Create summary
            await core.summary.addRaw(report).write();
      
      - name: Upload Report
        uses: actions/upload-artifact@v4
        with:
          name: security-report-${{ github.run_id }}
          path: ${{ github.workspace }}
          retention-days: 365  # Keep reports for 1 year for audit
```

---

## **Complete Implementation Checklist**

### Stage 1 - Foundation (Week 1)
- [ ] Create `SECURITY. md` in `.github` repo
- [ ] Create reusable Semgrep workflow
- [ ] Create reusable ZAP workflow
- [ ] Create security finding issue template

### Stage 2 - SAST (Week 2-3)
| Repo Type | Workflow | Rules | Done |
|-----------|----------|-------|------|
| Node.js Backend | `security. yml` | `p/nodejs`, `p/javascript`, `p/typescript` | [ ] |
| Web App 1 | `security.yml` | `p/javascript`, `p/typescript`, `p/react` | [ ] |
| Web App 2 | `security.yml` | `p/javascript`, `p/typescript`, `p/react` | [ ] |
| Web App 3 | `security.yml` | `p/javascript`, `p/typescript`, `p/react` | [ ] |
| Flutter App | `security.yml` | `p/dart`, `p/secrets` | [ ] |
| Unity App 1 | `security.yml` | `p/csharp`, `p/secrets` | [ ] |
| Unity App 2 | `security.yml` | `p/csharp`, `p/secrets` | [ ] |
| Unity App 3 | `security.yml` | `p/csharp`, `p/secrets` | [ ] |
| Unity App 4 | `security.yml` | `p/csharp`, `p/secrets` | [ ] |
| Unity App 5 | `security.yml` | `p/csharp`, `p/secrets` | [ ] |
| Unity App 6 | `security.yml` | `p/csharp`, `p/secrets` | [ ] |
| Unity App 7 | `security.yml` | `p/csharp`, `p/secrets` | [ ] |

### Stage 3 - DAST (Week 4)
| Target | Scan Type | Frequency | Done |
|--------|-----------|-----------|------|
| Backend API | API Scan + Baseline | Weekly | [ ] |
| Web App 1 (staging) | Baseline | Weekly | [ ] |
| Web App 2 (staging) | Baseline | Weekly | [ ] |
| Web App 3 (staging) | Baseline | Weekly | [ ] |
| Flutter Web (staging) | Baseline | Weekly | [ ] |

### Stage 4 - Evidence & Tracking (Ongoing)
- [ ] Monthly security report workflow
- [ ] Security issue tracking process documented
- [ ] Artifact retention verified (90 days)

---

## **ISO Audit Evidence Summary**

When auditors ask for evidence, point them to: 

| ISO Control | Evidence Location |
|-------------|-------------------|
| A.14.2.1 - Secure development policy | `SECURITY.md` in `.github` repo |
| A.14.2.5 - SAST in SDLC | GitHub Actions workflow runs + artifacts |
| A.14.2.8 - System security testing | ZAP DAST reports in artifacts |
| A.12.6.1 - Vulnerability management | GitHub Issues with `security` label |
| Evidence retention | 90-day artifacts, monthly reports (365 days) |

---

## **Estimated Minutes Usage (Final)**

| Scan Type | Repos | Frequency | Min/Run | Monthly Minutes |
|-----------|-------|-----------|---------|-----------------|
| SAST (PRs) | 10 | ~200 PRs | 5 min | 1,000 min |
| SAST (scheduled) | 10 | 4/month each | 5 min | 200 min |
| DAST baseline | 5 targets | Weekly (20/mo) | 10 min | 200 min |
| DAST API | 1 target | Weekly (4/mo) | 15 min | 60 min |
| **Total** | | | | **~1,460 min** |

✅ **Fits within GitHub Free tier (2,000 min/month)**

---

Would you like me to help you create any of these files in a specific repository, or explain any part of this plan in more detail? 