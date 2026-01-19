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