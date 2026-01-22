# Flie Overview

A short description for every file

---

[docs/4 Week SAST and DAST Plan.md](../docs/4%20Week%20SAST%20and%20DAST%20Plan.md)  
This document is the master rollout plan for building ISO 27001/27701 audit evidence using automated SAST and DAST. It defines four implementation stages over a month, tying each stage to specific ISO control objectives and emphasizing repeatable, documented processes rather than perfect coverage.

It also serves as a template source for the workflows in this repo, embedding example GitHub Actions YAML for Semgrep and OWASP ZAP. The text explicitly calls out required artifacts, data classification, privacy impact notes, and retention policies, which are essential for auditability. In practice, this file is the blueprint for all other files in the workspace.

---

[docs/security/SECURITY.md](../docs/security/SECURITY.md)  
This is the organization’s security policy for secure development and testing. It defines SAST and DAST cadence, scope, tools, and remediation SLAs, plus how findings are tracked for audit evidence.

It also incorporates ISO 27701 privacy evidence requirements—data classification and privacy impact tracking—so that the security program produces audit‑ready artifacts. The last‑reviewed and next‑reviewed dates provide governance traceability.

---

[docs/security/unity-apps-security.md](../docs/security/unity-apps-security.md)  
This document is a scope statement for Unity apps. It explains why DAST is not applicable to client‑side Unity distributions and establishes SAST as the primary control.

It also documents dependency and privacy considerations, including that Unity apps are treated as non‑PII by default unless they explicitly collect user data. This is used as evidence for the ISO 27701 privacy posture of Unity‑based products.

---

[ISSUE_TEMPLATE/security-finding.yml](../ISSUE_TEMPLATE/security-finding.yml)  
This GitHub issue template standardizes security findings across repos. It captures `severity`, `source`, and `CWE`, plus narrative `description` and `remediation` fields, ensuring consistent reporting.

It is primarily filled in by humans when creating a new Security Finding issue in GitHub (developers, security reviewers, or triage). It is not automatically populated by default.

It also embeds a required privacy impact field aligned to ISO 27701, and provides a target remediation date placeholder tied to SLA expectations. No workflow explicitly references this template file; It is only for developers to use.

---

[.zap/rules.tsv](../.zap/rules.tsv)  
This file customizes OWASP ZAP alert handling for baseline scans. Each row maps a ZAP alert ID to a policy action like `IGNORE` or `WARN`, with a short description.

The purpose is to reduce noise for known, accepted issues (for example, missing CSP headers in certain environments) while still surfacing higher‑value alerts. It acts as a tuning layer for the ZAP workflows.

---

[workflows/reusable-semgrep.yml](../workflows/reusable-semgrep.yml)  
This is the core reusable SAST workflow. It runs Semgrep in a container, supports optional include/exclude patterns, and generates JSON and SARIF outputs with a step summary.

It also uploads SARIF to GitHub code scanning (optional) and stores scan artifacts for audit evidence. Inputs like `data_classification` and `privacy_impact` are printed into the summary to provide ISO 27701 privacy context for each run.

---

[workflows/reusable-zap-baseline.yml](../workflows/reusable-zap-baseline.yml)  
This is the core reusable DAST workflow for OWASP ZAP baseline or API scans. It validates inputs (like requiring `openapi_url` for API scans), runs the correct ZAP action, and records evidence.

It supports optional context files, authentication parameters, rule tuning via `rules_file_name`, and custom ZAP command options. The job writes a summary including privacy and data classification and uploads reports as long‑retention artifacts.

---

[workflows/security-report.yml](../workflows/security-report.yml)  
This workflow generates a monthly security report using `actions/github-script`. It queries issues labeled `security`, counts open and closed findings over the last 30 days, and produces an ISO‑aligned narrative summary.

It writes the report to the job summary and uploads it as an artifact with long retention (365 days), serving as audit evidence. It also references evidence locations for SAST/DAST artifacts and policy documentation.

---

[workflows/sast-unity.yml](../workflows/sast-unity.yml)  
This workflow is a Unity‑specific SAST runner. It calls the reusable Semgrep workflow with C# and secrets rules and excludes Unity build folders (`Library`, `Temp`, `Logs`) and `.meta` files.

It also declares data classification as `None` and states the privacy impact assumption for Unity clients. The exclusions improve scan speed and reduce noise typical to Unity project layouts.

---

[workflows/sast-node.yml](../workflows/sast-node.yml)  
This workflow provides a Node.js‑specific SAST entry point. It calls the reusable Semgrep workflow with JavaScript, TypeScript, and Node.js rules plus the OWASP Top 10 pack.

It includes concurrency controls to avoid duplicate scans per branch and records privacy/data classification notes relevant to backend APIs. This lets backend repos adopt SAST by using a consistent, simple wrapper.

---

[workflows/sast-react.yml](../workflows/sast-react.yml)  
This workflow provides a React web app SAST entry point. It calls the reusable Semgrep workflow with JS/TS/React/HTML rules plus OWASP Top 10.

It also uses concurrency controls and includes privacy notes appropriate for frontend clients. Like the Node workflow, it standardizes how React repos plug into the shared SAST process.

---

[workflows/sast-flutter.yml](../workflows/sast-flutter.yml)  
This workflow is tailored for Flutter repos. It runs Semgrep with Dart and secrets rules plus OWASP Top 10 via the reusable workflow, and adds a lightweight grep‑based secrets check for Dart files.

The additional secrets check is meant to catch obvious mobile‑app credential issues. It writes findings to the step summary but does not fail the build, keeping it audit‑friendly while reducing friction.

---

[workflows/dast-backend.yml](../workflows/dast-backend.yml)  
This workflow is a DAST wrapper for backend APIs. It triggers both an API scan (OpenAPI‑driven) and a baseline scan using the reusable ZAP workflow.

It supports optional authentication context and rule customization, while including privacy and data classification fields for ISO 27701 evidence. It is intended to be called by API repos as a single entry point.

---

[workflows/dast-web.yml](../workflows/dast-web.yml)  
This workflow is a DAST wrapper for web frontends. It calls the reusable ZAP workflow for baseline scans against a target URL, with optional auth context and rule tuning.

It standardizes how frontend repos run OWASP ZAP, while also capturing privacy and data classification notes for audit evidence.

---

[workflows/zap-full.yml](../workflows/zap-full.yml)  
This workflow runs the full ZAP scan action, which is deeper and more time‑intensive than baseline. It supports command options (like time limits) and allows auto‑issue creation.

It uploads the HTML report as an artifact for audit evidence and is intended for scheduled or less frequent deep assessments.

---

[workflows/semgrep-sast-template.yml](../workflows/semgrep-sast-template.yml)  
This is a minimal Semgrep SAST template. It performs a standard scan, uploads JSON artifacts, and optionally uploads SARIF for GitHub code scanning.

It is suitable for teams that want a quick SAST baseline without the expanded inputs and summary logic of the reusable workflow.

---

[workflows/semgrep-optimized.yml](../workflows/semgrep-optimized.yml)  
This workflow implements an optimized Semgrep approach with two modes: diff scanning on PRs and full scanning on main. It uses `baseline_commit` to scope diff scans, which can significantly reduce runtime.

It’s gated by a repository check (`github.repository == 'your-org/your-repo'`) and uploads results as artifacts. This is meant for performance‑sensitive repos that want faster feedback while retaining full scans where needed.