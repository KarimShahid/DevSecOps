# SonarQube SAST

Static Application Security Testing (SAST) using SonarQube, integrated with GitHub Actions.

---

## Prerequisites

- SonarQube Server configured and running
- Web application for testing
- GitHub repository access

---

## 1. Login

Log in to your SonarQube instance.

![Login](images/Screenshot_2026-02-17_at_2.54.15_PM.png)

---

## 2. Create a Project

From the dashboard, create a new project.

![Create Project](images/Screenshot_2026-02-17_at_2.55.54_PM.png)

### Enter Project Name and Project Key

Provide a unique project name and key to identify your project within SonarQube.

![Project Name and Key](images/Screenshot_2026-02-17_at_2.57.04_PM.png)

### Select Global Settings

SonarQube follows the **Clean as You Code** methodology — only code changed since the last project version is considered *New Code* and evaluated against the Quality Gate.

![Global Settings](images/Screenshot_2026-02-17_at_2.59.36_PM.png)

### Select GitHub Actions as the Analysis Method

Choose GitHub Actions as the CI/CD method to trigger scans automatically on push and pull requests.

![GitHub Actions Method](images/Screenshot_2026-02-17_at_3.05.56_PM.png)

---

## 3. Configure GitHub Secrets and Workflow

### Add Secrets

SonarQube will provide a token and host URL. Add these to your GitHub repository under **Settings → Secrets and variables → Actions**.

| Secret Name | Description |
|---|---|
| `SONAR_TOKEN` | Authentication token from SonarQube |
| `SONAR_HOST_URL` | URL of your SonarQube server |

![Add Secrets](images/Screenshot_2026-02-17_at_3.06.39_PM.png)

### Create the Workflow File

Create the following workflow file in your repository:
```bash
.github/workflows/build.yml
```

![Workflow File](images/Screenshot_2026-02-17_at_3.11.46_PM.png)

![Workflow Configuration](images/9679cc66-87e2-46be-80c5-13cd8192e358.png)

---

## 4. Check GitHub Actions

Once pushed, the workflow will trigger automatically. Go to the **Actions** tab in your repository to monitor the scan progress.

![GitHub Actions Run](images/Screenshot_2026-02-17_at_3.29.06_PM.png)

---

## 5. View Results in SonarQube

After the workflow completes, navigate back to your SonarQube project to view the full analysis report including bugs, vulnerabilities, code smells, and coverage.

![SonarQube Results](images/Screenshot_2026-02-17_at_3.29.39_PM.png)