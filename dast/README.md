# DAST - ZAP Scan on Juice Shop

Automated Dynamic Application Security Testing (DAST) using [OWASP ZAP](https://www.zaproxy.org/) against [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/), integrated into GitHub Actions.

---

## What is DAST?

DAST (Dynamic Application Security Testing) scans a **live, running application** from the outside by sending real HTTP requests to it — the same way an attacker would. Unlike SAST which reads source code, DAST finds vulnerabilities that only appear at runtime, such as misconfigured HTTP headers, CORS issues, and XSS/SQLi behaviour.

---

## Where does DAST fit in the pipeline?

```
Code pushed      → SAST (source code scan)
IaC pushed       → IaC scan (Dockerfile, k8s manifests)
Image built      → Container image scan (Trivy)
App deployed     → DAST ← this workflow
```

---

## Workflow Overview

**File:** `.github/workflows/zap-dast.yml`

The workflow is **manually triggered** (`workflow_dispatch`) only — it never runs automatically on push or PR. When triggered, you choose which scan type to run from a dropdown.

### Scan Types

| Option | Description | Approx. Time |
|---|---|---|
| `baseline` | Passive scan — spiders the app and observes traffic, no active attacks | ~3–5 min |
| `full` | Active scan — fires real attack payloads (SQLi, XSS, etc.) at the app | ~15–30 min |
| `both` | Runs both jobs in parallel | ~15–30 min |

---

## How to Run

1. Go to your repository on GitHub
2. Click the **Actions** tab
3. Select **DAST - ZAP Scan on Juice Shop** from the left sidebar
4. Click **Run workflow**
5. Select your scan type from the dropdown
6. Click **Run workflow**

---

## How it Works

Both jobs follow the same steps:

```
1. Spin up Juice Shop in a Docker service container (port 3000)
2. Wait for Juice Shop to be ready (polls every 5s)
3. Fix workspace permissions (so ZAP container can write reports)
4. Run ZAP scan against http://localhost:3000
5. Upload reports as a downloadable artifact
```

Juice Shop runs entirely inside the GitHub Actions runner — no external hosting needed. ZAP and Juice Shop communicate directly between containers on the runner machine.

---

## Reports

After the workflow completes, reports are available under the **Artifacts** section of the workflow run:

| Artifact Name | Contents |
|---|---|
| `zap-baseline-reports` | Reports from the baseline scan |
| `zap-full-reports` | Reports from the full active scan |

Each artifact contains:

| File | Format | Description |
|---|---|---|
| `report_html.html` | HTML | Human-readable report, open in a browser |
| `report_json.json` | JSON | Machine-readable, useful for scripting |
| `report_xml.xml` | XML | Compatible with CI reporting tools |
| `report_sarif.sarif` | SARIF | Standard format for security tools |

Reports are retained for **30 days** then automatically deleted by GitHub.

---

## Understanding the Results

ZAP categorises findings by risk level:

| Level | Meaning |
|---|---|
| **High** | Serious vulnerability, fix immediately |
| **Medium** | Should be fixed, poses real risk |
| **Low** | Minor issue, fix when possible |
| **Informational** | No action needed, just observations |

### Known findings on Juice Shop

Since Juice Shop is intentionally vulnerable, the following warnings are expected and can be ignored in this context:

- **CSP Header Not Set** — no Content-Security-Policy header
- **Cross-Domain Misconfiguration** — CORS set to `*`
- **Dangerous JS Functions** — `bypassSecurityTrustHtml()` used in Angular code
- **Deprecated Feature-Policy Header** — old header used instead of `Permissions-Policy`
- **Timestamp Disclosure** — Unix timestamps leaked in responses

---

## Known Issue

The `zaproxy/action-baseline` and `zaproxy/action-full-scan` actions attempt to upload their own internal artifact named `zap_scan`, which is rejected by GitHub's artifact API with a `400 Bad Request`. This is a **bug in the ZAP action itself** and does not affect the scan or your reports.

`continue-on-error: true` is set on the ZAP steps to prevent this bug from failing the pipeline. Your reports are always uploaded correctly by the explicit upload step.

---

## Workflow Configuration

```yaml
fail_action: false        # don't fail the job if ZAP finds vulnerabilities
allow_issue_writing: false # don't open GitHub Issues for findings
continue-on-error: true   # suppress the known zap_scan artifact bug
retention-days: 30        # how long artifacts are kept
```

To make the pipeline **fail when vulnerabilities are found**, change `fail_action` to `true`.
