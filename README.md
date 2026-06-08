# Complete Security Scanning Guide for Beginners

## SAST + SCA + DAST with GitHub Actions

A complete beginner-friendly DevSecOps lab that shows how to scan source code, dependencies, and a running application using GitHub Actions.

## Contents

- [Overview](#overview)  
- [Security scanning model](#security-scanning-model)  
  - [What SAST does](#what-sast-does)  
  - [What SCA does](#what-sca-does)  
  - [What DAST does](#what-dast-does)  
  - [Pipeline flow](#pipeline-flow)  
  - [Why all three matter](#why-all-three-matter)  
- [Prerequisites](#prerequisites)  
  - [What you need](#what-you-need)  
  - [Why CodeQL](#why-codeql) [web:27][web:28]  
  - [Artifact naming rules](#artifact-naming-rules) [web:47][web:50][web:53][web:55]  
  - [Verified action versions](#verified-action-versions) [web:27][web:26][web:35]  
  - [Estimated time](#estimated-time)  
- [Project setup](#project-setup)  
  - [Create the GitHub repository first](#create-the-github-repository-first)  
  - [Clone the repository locally](#clone-the-repository-locally)  
  - [Directory structure](#directory-structure)  
  - [Create `app.py`](#create-apppy)  
  - [Create `templates/index.html`](#create-templatesindexhtml)  
  - [Create `requirements.txt`](#create-requirementstxt)  
  - [Create `Dockerfile`](#create-dockerfile)  
  - [Make the first commit](#make-the-first-commit)  
- [SAST workflow](#sast-workflow)  
  - [What SAST is](#what-sast-is) [web:27][web:28]  
  - [SAST workflow YAML](#sast-workflow-yaml) [web:27]  
  - [How the SAST workflow works](#how-the-sast-workflow-works) [web:27]  
  - [CodeQL query suites](#codeql-query-suites) [web:27]  
- [SCA workflow](#sca-workflow)  
  - [What SCA is](#what-sca-is) [web:26][web:29]  
  - [SCA workflow YAML](#sca-workflow-yaml) [web:26][web:29]  
  - [How the SCA workflow works](#how-the-sca-workflow-works) [web:26][web:30]  
- [DAST workflow](#dast-workflow)  
  - [What DAST is](#what-dast-is) [web:37][web:43]  
  - [DAST workflow YAML (beginner-friendly)](#dast-workflow-yaml) [web:35][web:31][web:37]  
  - [Why this DAST version stays green](#why-this-dast-version-stays-green) [web:35][web:37][web:39]  
- [Integrated pipeline](#integrated-pipeline)  
  - [Complete workflow YAML](#complete-workflow-yaml)  
  - [Execution order](#execution-order)  
  - [Generated artifacts](#generated-artifacts)  
- [Running the repo](#running-the-repo)  
  - [Push to GitHub](#push-to-github)  
  - [Watch workflow runs](#watch-workflow-runs)  
  - [Verify artifacts](#verify-artifacts)  
- [Reading results](#reading-results)  
  - [Actions tab](#actions-tab)  
  - [Security tab](#security-tab) [web:27][web:33]  
  - [Severity levels](#severity-levels) [web:26][web:29][web:43]  
- [Understanding ZAP findings](#understanding-zap-findings)  
  - [Result categories](#result-categories) [web:43][web:37]  
  - [Common header findings](#common-header-findings) [web:43]  
  - [Application findings](#application-findings) [web:43]  
  - [How to interpret a scan](#how-to-interpret-a-scan) [web:37][web:44][web:12]  
- [Troubleshooting](#troubleshooting)  
  - [Wrong action version](#wrong-action-version) [web:27][web:26][web:35]  
  - [Invalid artifact name](#invalid-artifact-name) [web:47][web:50][web:55][web:53]  
  - [CodeQL runs too long](#codeql-runs-too-long) [web:27]  
  - [SCA runs too long](#sca-runs-too-long) [web:26][web:30]  
  - [DAST cannot connect](#dast-cannot-connect) [web:37][web:44]  
  - [Permissions error](#permissions-error) [web:27][web:53]  
- [Practice exercises](#practice-exercises)  

---

## Overview

This project builds a simple Flask calculator app and wires it into GitHub Actions so that every push or pull request can trigger three security scan types:

- SAST using CodeQL for source code analysis. [web:27][web:28]  
- SCA using OWASP Dependency-Check for dependency and CVE analysis. [web:26][web:29]  
- DAST using OWASP ZAP for dynamic scanning of the running app. [web:37][web:44]  

The beginner workflows are configured so all jobs can finish **green** while still generating full security reports for teaching.

---

## Security scanning model

### What SAST does

**SAST** (Static Application Security Testing) scans your source code without running the application. [web:27]

Typical findings include:

- SQL injection patterns  
- XSS-related unsafe handling  
- insecure data flow  
- hardcoded secrets  
- dangerous API usage  

### What SCA does

**SCA** (Software Composition Analysis) scans the libraries and packages your project depends on for known vulnerabilities. [web:26][web:29]

Typical findings include:

- known vulnerable dependency versions  
- outdated packages with published CVEs  
- vulnerable transitive dependencies  
- sometimes license or metadata issues  

### What DAST does

**DAST** (Dynamic Application Security Testing) attacks the running application from the outside using HTTP requests. [web:37][web:43]

Typical findings include:

- missing security headers  
- runtime configuration issues  
- authentication weaknesses  
- reflected input problems  
- exposed routes or unsafe behavior at runtime  

### Pipeline flow

```text
1. You push code
   ↓
2. SAST scans source code with CodeQL
   ↓
3. SCA scans dependencies with Dependency-Check
   ↓
4. Application starts successfully
   ↓
5. DAST scans the running application with OWASP ZAP
   ↓
6. Reports appear in GitHub Actions and Security views
```

### Why all three matter

Each scan sees a different layer of risk:

- SAST checks what you wrote.  
- SCA checks what you imported.  
- DAST checks what actually happens when the app is live.  

---

## Prerequisites

### What you need

You only need a few basics:

- GitHub account  
- basic Git knowledge  
- VS Code or another text editor  
- willingness to wait a little on first-run scans  

### Why CodeQL

CodeQL is a good fit for this guide because it is native to GitHub and easy to adopt. [web:27][web:28]

Advantages:

- free for public repositories  
- no external token required for basic setup  
- native integration with GitHub Actions  
- results flow into the Security tab  
- maintained by GitHub  

### Artifact naming rules

GitHub Actions artifact names must be filesystem-safe and unique per run. [web:47][web:50][web:55][web:53]

Use:

- letters  
- numbers  
- hyphens  

Avoid:

- underscores  
- slashes  
- colons  
- wildcard characters  
- duplicate artifact names in the same workflow run  

Examples:

- ✅ `sca-reports`  
- ✅ `zap-baseline-reports`  
- ✅ `zap-fullscan-reports`  
- ❌ `zap_scan`  
- ❌ `sca_reports`  
- ❌ `dast/reports`  

### Verified action versions

Use pinned, known-good versions for stability. [web:27][web:26][web:35]

```yaml
# SAST
- github/codeql-action/init@v4
- github/codeql-action/analyze@v4

# SCA
- dependency-check/Dependency-Check_Action@main

# DAST
- zaproxy/action-baseline@v0.15.0
- zaproxy/action-full-scan@v0.12.0

# Utilities
- actions/checkout@v4
- actions/setup-python@v5
- actions/upload-artifact@v4
```

### Estimated time

- Initial setup: 30 to 45 minutes  
- First full pipeline run: 20 to 45 minutes  
- Each follow-up practice cycle: 10 to 20 minutes  

---

## Project setup

*(Section unchanged – same app and basic Git steps as before.)*

[You keep the `app.py`, `templates/index.html`, `requirements.txt`, `Dockerfile`, `.gitignore`, etc., exactly as in your original README.]

---

## SAST workflow

*(Same as before; CodeQL still runs normally.)*  

You keep `.github/workflows/1-sast-only.yml` as:

```yaml
name: 1. SAST - Code Security Scan
# ... unchanged CodeQL workflow ...
```

---

## SCA workflow

*(Same as before; you can keep `--failOnCVSS 7` if you want SCA to sometimes go red, or later relax it for a “fully green” variant.)*  

You keep `.github/workflows/2-sca-only.yml` as:

```yaml
name: 2. SCA - Dependency Security Scan
# ... unchanged Dependency-Check workflow ...
```

---

## DAST workflow

### What DAST is

DAST tests the running application from the outside. [web:37][web:44]  
Unlike SAST or SCA, it needs the app to actually start and respond over HTTP.

In this **beginner** version, ZAP still finds and reports alerts, but the CI job **does not fail just because alerts exist**. [web:35][web:56]

### DAST workflow YAML

Create `.github/workflows/3-dast-only.yml`:

```yaml
name: 3. DAST - Live Application Scan (Beginner Friendly)

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true

permissions:
  contents: read
  security-events: write
  issues: write

jobs:
  dast:
    name: OWASP ZAP Scan (DAST)
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Start Flask application
        run: |
          python app.py > flask.log 2>&1 &
          echo $! > flask.pid

      - name: Wait for application
        run: |
          for i in {1..30}; do
            if curl -fsS http://localhost:5000 > /dev/null; then
              echo "Application is responding on http://localhost:5000"
              exit 0
            fi
            sleep 1
          done
          echo "Application failed to start"
          cat flask.log
          exit 1

      - name: Create report directories
        run: |
          mkdir -p reports/baseline reports/fullscan

      - name: ZAP Baseline Scan (non-blocking)
        uses: zaproxy/action-baseline@v0.15.0
        with:
          target: 'http://localhost:5000'
          # -a: include alpha passive rules
          # -j: generate JSON report
          # -I: do not fail on warnings/alerts
          cmd_options: '-a -j -I'
          allow_issue_writing: false
          artifact_name: 'zap-baseline-scan'
          # Do NOT fail the job simply because alerts are present
          fail_action: false

      - name: Preserve baseline reports
        if: always()
        run: |
          [ -f report_html.html ] && mv report_html.html reports/baseline/report.html || true
          [ -f report_json.json ] && mv report_json.json reports/baseline/report.json || true
          [ -f report_md.md ] && mv report_md.md reports/baseline/report.md || true

      - name: ZAP Full Scan (non-blocking)
        uses: zaproxy/action-full-scan@v0.12.0
        with:
          target: 'http://localhost:5000'
          cmd_options: '-a -j -I'
          allow_issue_writing: false
          artifact_name: 'zap-fullscan-scan'

      - name: Preserve full scan reports
        if: always()
        run: |
          [ -f report_html.html ] && mv report_html.html reports/fullscan/report.html || true
          [ -f report_json.json ] && mv report_json.json reports/fullscan/report.json || true
          [ -f report_md.md ] && mv report_md.md reports/fullscan/report.md || true

      - name: Upload ZAP baseline reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: zap-baseline-reports
          path: reports/baseline/

      - name: Upload ZAP fullscan reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: zap-fullscan-reports
          path: reports/fullscan/

      - name: Show app log (for debugging)
        if: always()
        run: |
          echo "==== flask.log ===="
          cat flask.log || true
```

### Why this DAST version stays green

Key points: [web:35][web:37][web:39][web:56]

- `fail_action: false` uses the action’s default behavior of **not** failing the workflow when alerts are found.  
- `cmd_options: '-a -j -I'`:
  - `-a` includes additional passive rules (more findings to learn from). [web:37][web:57]  
  - `-j` generates JSON report alongside HTML/Markdown. [web:31][web:54]  
  - `-I` tells the underlying ZAP script not to treat warnings/alerts as a failing condition. [web:38]  

For students, that means:

- Pipelines show **green ticks** when ZAP runs successfully.  
- They still download the reports and see all alerts, but they are not punished with red builds for having findings in a teaching lab.

---

## Integrated pipeline

### Complete workflow YAML

Here is the updated `.github/workflows/4-complete-security.yml` with a beginner‑friendly DAST job:

```yaml
name: 4. Complete Security Pipeline

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true

permissions:
  contents: read
  security-events: write
  issues: write

jobs:
  sast:
    name: "1️⃣ SAST - Code Analysis"
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        language: [ 'python' ]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v4
        with:
          languages: ${{ matrix.language }}
          queries: security-extended

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Perform CodeQL Analysis (SAST)
        uses: github/codeql-action/analyze@v4
        with:
          category: sast

  sca:
    name: "2️⃣ SCA - Dependency Check"
    runs-on: ubuntu-latest
    needs: sast

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run OWASP Dependency-Check (SCA)
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'calculator-app'
          path: '.'
          format: 'ALL'
          out: 'reports'
          args: >
            --failOnCVSS 7
            --enableRetired
            --disableOssIndex

      - name: Upload SCA reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: sca-reports
          path: reports/

  build:
    name: "3️⃣ Build Application"
    runs-on: ubuntu-latest
    needs: [sast, sca]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Test application starts
        run: |
          python app.py > flask.log 2>&1 &
          for i in {1..30}; do
            if curl -fsS http://localhost:5000 > /dev/null; then
              echo "✅ Application builds and runs successfully"
              exit 0
            fi
            sleep 1
          done
          echo "Application failed to start"
          cat flask.log
          exit 1

  dast:
    name: "4️⃣ DAST - Live Security Scan (Beginner Friendly)"
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Start application
        run: |
          python app.py > flask.log 2>&1 &
          echo $! > flask.pid

      - name: Wait for application
        run: |
          for i in {1..30}; do
            if curl -fsS http://localhost:5000 > /dev/null; then
              echo "Application is responding on http://localhost:5000"
              exit 0
            fi
            sleep 1
          done
          echo "Application failed to start"
          cat flask.log
          exit 1

      - name: Create report directories
        run: |
          mkdir -p reports/baseline reports/fullscan

      - name: ZAP Baseline Scan (non-blocking)
        uses: zaproxy/action-baseline@v0.15.0
        with:
          target: 'http://localhost:5000'
          cmd_options: '-a -j -I'
          allow_issue_writing: false
          artifact_name: 'zap-baseline-scan'
          fail_action: false

      - name: Preserve baseline reports
        if: always()
        run: |
          [ -f report_html.html ] && mv report_html.html reports/baseline/report.html || true
          [ -f report_json.json ] && mv report_json.json reports/baseline/report.json || true
          [ -f report_md.md ] && mv report_md.md reports/baseline/report.md || true

      - name: ZAP Full Scan (non-blocking)
        uses: zaproxy/action-full-scan@v0.12.0
        with:
          target: 'http://localhost:5000'
          cmd_options: '-a -j -I'
          allow_issue_writing: false
          artifact_name: 'zap-fullscan-scan'

      - name: Preserve full scan reports
        if: always()
        run: |
          [ -f report_html.html ] && mv report_html.html reports/fullscan/report.html || true
          [ -f report_json.json ] && mv report_json.json reports/fullscan/report.json || true
          [ -f report_md.md ] && mv report_md.md reports/fullscan/report.md || true

      - name: Upload ZAP baseline reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: zap-baseline-reports
          path: reports/baseline/

      - name: Upload ZAP fullscan reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: zap-fullscan-reports
          path: reports/fullscan/

  summary:
    name: "📊 Security Summary"
    runs-on: ubuntu-latest
    needs: [sast, sca, dast]
    if: always()

    steps:
      - name: Generate Security Summary
        run: |
          echo "# 🔒 Security Scan Summary" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "## Scan Results" >> $GITHUB_STEP_SUMMARY
          echo "- SAST (CodeQL): completed; check Security tab for findings" >> $GITHUB_STEP_SUMMARY
          echo "- SCA (Dependency-Check): completed; see sca-reports artifact" >> $GITHUB_STEP_SUMMARY
          echo "- DAST (ZAP): completed; see zap-baseline-reports and zap-fullscan-reports" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "## Notes" >> $GITHUB_STEP_SUMMARY
          echo "- In this beginner lab, ZAP alerts do NOT fail the pipeline. Use the reports to learn how to read and prioritize findings." >> $GITHUB_STEP_SUMMARY
```

### Execution order

The integrated pipeline now behaves like this:

```text
1. SAST
   ↓
2. SCA
   ↓
3. Build verification
   ↓
4. DAST (non-blocking)
   ↓
5. Summary
```

DAST only goes red if the app cannot start or ZAP fails technically, not because alerts exist. [web:35][web:37]

---

## Running the repo, reading results, and exercises

These sections stay conceptually the same:

- Actions tab → see runs and logs.  
- Security tab → see CodeQL and SCA SARIF results. [web:27][web:33]  
- Download `sca-reports`, `zap-baseline-reports`, `zap-fullscan-reports` and inspect HTML/JSON outputs for teaching. [web:26][web:31][web:37]  
