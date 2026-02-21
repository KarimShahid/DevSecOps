# 🐳 Container Security Scanning with Trivy

Automated container security scanning using [Trivy](https://trivy.dev/) by Aqua Security. This pipeline scans the `container-security/` directory on every push and pull request, covering Dockerfile misconfigurations and Docker image CVE vulnerabilities.

---

## 📁 Directory Structure

```
container-security/
├── Dockerfile            # Scanned for misconfigurations (Job 1)
├── docker-compose.yml    # ⚠️ Not supported by Trivy (see Known Limitations)
└── index.html            # Static web app served by nginx
```

---

## ⚙️ Pipeline Overview

The pipeline is defined in `.github/workflows/trivy-container-scan.yml` and runs automatically on:

| Trigger | Condition |
|---|---|
| `push` | Changes to `container-security/**` on `main` or `develop` |
| `pull_request` | Any PR targeting `main` |
| `workflow_dispatch` | Manual trigger from GitHub Actions UI |

---

## 🔀 Jobs

The pipeline runs **2 jobs in parallel** — each on its own fresh runner:

```
Pipeline Triggers
       ↓
┌──────────────────────┐     ┌──────────────────────┐
│  scan-container-     │     │  scan-image          │
│  config              │     │                      │
│  ~30 seconds         │     │  ~2-3 minutes        │
└──────────────────────┘     └──────────────────────┘
```

---

### Job 1 — Scan Container Config (`scan-container-config`)

Scans config files in `container-security/` for misconfigurations using `trivy config`.

| Step | Purpose | Fails Pipeline? |
|---|---|---|
| Checkout | Clones repo to runner | — |
| List files | Debug step — confirms files visible to Trivy | — |
| Table Scan | Prints all findings (LOW→CRITICAL) to Actions log | ❌ No |
| JSON Report | Generates `trivy-config-results.json` | ❌ No |
| Upload Artifact | Saves JSON report (30 day retention) | ❌ No |
| Gate | Fails pipeline if CRITICAL misconfiguration found | ✅ Yes |

**What Trivy checks in the Dockerfile:**
- Running container as root (no `USER` instruction)
- Missing `HEALTHCHECK` instruction
- Use of `latest` tag instead of pinned version
- Secrets baked into image layers
- Privileged operations

---

### Job 2 — Scan Built Docker Image (`scan-image`)

Builds the Docker image locally on the runner then scans it for CVEs using `trivy image`.

| Step | Purpose | Fails Pipeline? |
|---|---|---|
| Checkout | Clones repo to runner | — |
| Docker Build | Builds `song-of-the-day:<commit-sha>` locally | — |
| Table Scan | Prints all CVEs (LOW→CRITICAL) to Actions log | ❌ No |
| JSON Report | Generates `trivy-image-results.json` | ❌ No |
| Upload Artifact | Saves JSON report (30 day retention) | ❌ No |
| Gate | Fails pipeline if CRITICAL CVE with available fix found | ✅ Yes |

**What Trivy checks in the image:**
- OS package vulnerabilities (Alpine, Debian, Ubuntu packages)
- Language dependency vulnerabilities (npm, pip, etc.)
- Hardcoded secrets inside image layers
- Known CVEs matched against the Aqua vulnerability database

**Key flags used:**
- `--platform linux/amd64` — matches the platform defined in docker-compose.yml
- `--ignore-unfixed` — skips CVEs with no available patch, avoiding false blockers
- Image tagged with `${{ github.sha }}` — unique per commit, no caching confusion

---

## 🚨 Scan Results

### Job 1 — Dockerfile Findings

```
Dockerfile (dockerfile)
=======================
Tests: 27 (SUCCESSES: 25, FAILURES: 2)
Failures: 2 (HIGH: 1, LOW: 1, CRITICAL: 0)
```

| Rule | Severity | Issue |
|---|---|---|
| DS-0002 | 🟠 HIGH | No `USER` instruction — container runs as root |
| DS-0026 | 🔵 LOW | No `HEALTHCHECK` instruction |

Pipeline **passed** — no CRITICAL findings, gate did not trigger.

### Job 2 — Image CVE Findings

```
song-of-the-day (alpine 3.23.3)
================================
Vulnerabilities: 0 | Secrets: 0
```

Image is clean. `nginx:alpine` is a minimal, well-maintained base image with a very small attack surface.

---

## 🔧 How to Fix Dockerfile Findings

**DS-0002 — Add a non-root USER:**
```dockerfile
# Create a non-root user and switch to it
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

**DS-0026 — Add a HEALTHCHECK:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost/ || exit 1
```

---

## ⚠️ Known Limitations

**docker-compose.yml is not scanned by Trivy.**

Trivy v0.69.1 does not support docker-compose as a misconfiguration scan target. All three files (`Dockerfile`, `docker-compose.yml`, `index.html`) are confirmed present in the runner but Trivy consistently detects `num=1` — only picking up the Dockerfile.

**Workaround:** Use [Checkov](https://www.checkov.io/) alongside Trivy to cover compose file scanning:

```yaml
- name: Scan docker-compose.yml with Checkov
  uses: bridgecrewio/checkov-action@master
  with:
    file: container-security/docker-compose.yml
    framework: dockerfile
    soft_fail: false
```

---

## 📥 Downloading Scan Reports

After each pipeline run:

1. Go to **Actions** tab in GitHub
2. Click the relevant workflow run
3. Scroll to **Artifacts** at the bottom
4. Download `trivy-config-report` or `trivy-image-report`

Both are JSON files containing full details — rule ID, severity, description, affected line numbers, and remediation links.

---

## 🎛️ Configuration

To adjust severity thresholds, edit `.github/workflows/trivy-container-scan.yml`:

```yaml
severity: CRITICAL          # Gate threshold — what fails the pipeline
exit-code: 1                # 1 = fail, 0 = report only
ignore-unfixed: true        # Skip CVEs with no fix available (image scan only)
```

---

## 🔗 References

- [Trivy Documentation](https://trivy.dev/latest/docs/)
- [Trivy GitHub Action](https://github.com/aquasecurity/trivy-action)
- [Aqua Vulnerability Database (AVD)](https://avd.aquasec.com/)
- [Docker Security Best Practices](https://docs.docker.com/develop/security-best-practices/)
- [nginx:alpine Docker Hub](https://hub.docker.com/_/nginx)
