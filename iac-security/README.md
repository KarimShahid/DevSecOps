# 🛡️ IaC Security Scanning with Trivy

Automated Infrastructure-as-Code (IaC) security scanning using [Trivy](https://trivy.dev/) by Aqua Security. This pipeline scans CloudFormation templates for misconfigurations before they reach AWS, acting as a security gate on every push and pull request.

---

## 📁 Project Structure

```
.
├── .github/
│   └── workflows/
│       └── trivy-iac-scan.yml   # GitHub Actions pipeline
├── sast/
│   └── aws.yaml                 # CloudFormation template (scanned)
└── README.md
```

---

## 🔍 What is Trivy?

Trivy is an open-source, all-in-one security scanner by Aqua Security. In this project it is used in **config mode**, which scans IaC files (CloudFormation, Terraform, Kubernetes, Dockerfile, etc.) for security misconfigurations — without connecting to any cloud provider or deploying anything.

---

## ⚙️ Pipeline Overview

The pipeline is defined in `.github/workflows/trivy-iac-scan.yml` and runs automatically on:

| Trigger | Condition |
|---|---|
| `push` | Changes to `sast/aws.yaml` on `main` or `develop` |
| `pull_request` | Any PR targeting `main` |
| `workflow_dispatch` | Manual trigger from the GitHub Actions UI |

### Pipeline Steps

```
Checkout → Table Scan → JSON Report → Upload Artifact → Gate (fail on HIGH/CRITICAL)
```

| Step | Purpose | Fails Pipeline? |
|---|---|---|
| **Checkout** | Clones repo so Trivy can access files | — |
| **Table Scan** | Prints all findings (LOW→CRITICAL) to Actions log | ❌ No |
| **JSON Report** | Generates machine-readable `trivy-results.json` | ❌ No |
| **Upload Artifact** | Saves JSON report for download (30 days retention) | ❌ No |
| **Gate Step** | Fails pipeline if any HIGH or CRITICAL finding exists | ✅ Yes |

---

## 🚨 Scan Results Example

Below is a sample of findings detected from `aws.yaml`:

```
aws.yaml (cloudformation)
=========================
Tests: 28 (SUCCESSES: 14, FAILURES: 14)
Failures: 14 (HIGH: 13, CRITICAL: 1)
```

### Findings Summary

| Severity | Count | Example Issues |
|---|---|---|
| 🔴 CRITICAL | 1 | Security group allows unrestricted egress (`0.0.0.0/0`) |
| 🟠 HIGH | 13 | Public S3 bucket, unencrypted EBS/RDS, open SSH ingress, no IMDSv2 |

### Key Findings Breakdown

**S3 Bucket (`PublicBucket`)**
- `AccessControl: PublicRead` exposes bucket to the internet
- No `PublicAccessBlockConfiguration` set
- No KMS encryption (customer managed key)

**Security Group (`OpenSecurityGroup`)**
- Ingress open on port 22 (SSH) and all ports `0–65535` from `0.0.0.0/0`
- Egress unrestricted to all IPs on all protocols *(CRITICAL)*

**EC2 Instance (`InsecureInstance`)**
- IMDSv2 not enforced — vulnerable to SSRF credential theft
- Root block device has no encryption

**RDS Instance (`OpenDB`)**
- No storage encryption
- `PubliclyAccessible: true` — reachable outside VPC

**EBS Volume (`InsecureVolume`)**
- `Encrypted: false` explicitly set

---


## 📥 Downloading the Scan Report

After each pipeline run:

1. Go to **Actions** tab in GitHub
2. Click the relevant workflow run
3. Scroll to **Artifacts** at the bottom
4. Download `trivy-scan-report` (contains `trivy-results.json`)

The JSON report includes full details: rule ID, severity, description, affected line numbers, and remediation links for every finding.

---

## 🎛️ Configuration

To adjust severity thresholds, edit `.github/workflows/trivy-iac-scan.yml`:

```yaml
# Change which severities fail the pipeline
severity: HIGH,CRITICAL   # Options: LOW, MEDIUM, HIGH, CRITICAL
exit-code: 1              # 1 = fail pipeline, 0 = report only
```

To scan a different file or directory:
```yaml
scan-ref: sast/aws.yaml   # Change to any file or folder path
```

---

## 🔗 References

- [Trivy Documentation](https://trivy.dev/latest/docs/)
- [Trivy GitHub Action](https://github.com/aquasecurity/trivy-action)
- [Aqua Vulnerability Database (AVD)](https://avd.aquasec.com/misconfig/)
- [CloudFormation Security Best Practices](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/security-best-practices.html)