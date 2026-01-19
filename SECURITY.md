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