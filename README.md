# Complete Security Scanning Guide for Beginners

## SAST + SCA + DAST with GitHub Actions

A beginner-friendly DevSecOps lab that shows how to scan source code, dependencies, and a running application using GitHub Actions.

You will:

- Build a simple Flask calculator app
- Add three security workflows (SAST, SCA, DAST)
- Run a complete pipeline on every push / PR
- Learn to read reports while most jobs stay green

---

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
  - [Why CodeQL](#why-codeql)
  - [Artifact naming rules](#artifact-naming-rules)
  - [Node.js 24 warning and env](#nodejs-24-warning-and-env)
  - [Verified action versions](#verified-action-versions)
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
  - [What SAST is](#what-sast-is)
  - [SAST workflow YAML](#sast-workflow-yaml)
  - [How the SAST workflow works](#how-the-sast-workflow-works)
  - [CodeQL query suites](#codeql-query-suites)
- [SCA workflow](#sca-workflow)
  - [What SCA is](#what-sca-is)
  - [SCA workflow YAML](#sca-workflow-yaml)
  - [How the SCA workflow works](#how-the-sca-workflow-works)
- [DAST workflow](#dast-workflow)
  - [What DAST is](#what-dast-is)
  - [DAST workflow YAML (beginner-friendly)](#dast-workflow-yaml-beginner-friendly)
  - [Why this DAST version stays green](#why-this-dast-version-stays-green)
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
  - [Security tab](#security-tab)
  - [Severity levels](#severity-levels)
- [Understanding ZAP findings](#understanding-zap-findings)
  - [Result categories](#result-categories)
  - [Common header findings](#common-header-findings)
  - [Application findings](#application-findings)
  - [How to interpret a scan](#how-to-interpret-a-scan)
- [Troubleshooting](#troubleshooting)
- [Practice exercises](#practice-exercises)

---

## Overview

This project builds a simple Flask calculator app and wires it into GitHub Actions so that every push or pull request can trigger three types of security scans:

- SAST using CodeQL for source code analysis  
- SCA using OWASP Dependency-Check for dependency and CVE analysis  
- DAST using OWASP ZAP for dynamic scanning of the running app  

For beginners, the DAST job is configured to **stay green** even when ZAP finds alerts. You still get full reports to learn from, but you avoid “red build” anxiety in early sessions.

---

## Security scanning model

### What SAST does

**SAST** (Static Application Security Testing) scans your source code without running the application.

Typical findings include:

- SQL injection patterns
- XSS-related unsafe handling
- insecure data flow
- hardcoded secrets
- dangerous API usage

### What SCA does

**SCA** (Software Composition Analysis) scans the libraries and packages your project depends on.

Typical findings include:

- known vulnerable dependency versions
- outdated packages with published CVEs
- vulnerable transitive dependencies
- sometimes license or metadata issues

### What DAST does

**DAST** (Dynamic Application Security Testing) attacks the running application from the outside over HTTP.

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

- GitHub account
- basic Git knowledge
- VS Code or another text editor
- willingness to wait a bit on first-run scans

No security background is required.

### Why CodeQL

CodeQL is a good fit for this guide because it is native to GitHub and easy to adopt:

- free for public repositories
- no external token required for basic setup
- native integration with GitHub Actions
- results flow into the Security tab
- maintained by GitHub

### Artifact naming rules

GitHub Actions artifact names should be simple and safe.

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

### Node.js 24 warning and env

GitHub is deprecating Node.js 20 for running JavaScript-based actions. To avoid a noisy warning and be future-proof, each workflow in this lab sets:

```yaml
env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true
```

This tells GitHub to run actions like `actions/checkout`, `actions/setup-python`, and `actions/upload-artifact` on Node.js 24 instead of Node.js 20.  
Students do not need to change this; it simply keeps the Actions logs clean and up to date.

### Verified action versions

Use pinned, known-good versions for stability.

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

- Initial setup: 30–45 minutes
- First full pipeline run: 20–45 minutes
- Each follow-up practice cycle: 10–20 minutes

---

## Project setup

### Create the GitHub repository first

Create the repository on GitHub **before** creating local files.

Suggested settings:

- Repository name: `calculator-security-demo`
- Visibility: public (simplest for this lab)
- Initialize with README: optional

Copy the HTTPS clone URL when the repo is created.

### Clone the repository locally

```bash
git clone https://github.com/YOUR_USERNAME/calculator-security-demo.git
cd calculator-security-demo
```

If Git asks for your identity later:

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

### Directory structure

Target layout:

```text
calculator-security-demo/
├── app.py
├── requirements.txt
├── templates/
│   └── index.html
├── Dockerfile
└── .github/
    └── workflows/
        ├── 1-sast-only.yml
        ├── 2-sca-only.yml
        ├── 3-dast-only.yml
        └── 4-complete-security.yml
```

### Create `app.py`

```python
from flask import Flask, render_template, request, jsonify

app = Flask(__name__)

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/calculate', methods=['POST'])
def calculate():
    try:
        data = request.get_json()
        num1 = float(data['num1'])
        num2 = float(data['num2'])
        operation = data['operation']

        if operation == 'add':
            result = num1 + num2
        elif operation == 'subtract':
            result = num1 - num2
        elif operation == 'multiply':
            result = num1 * num2
        elif operation == 'divide':
            if num2 == 0:
                return jsonify({'error': 'Cannot divide by zero'}), 400
            result = num1 / num2
        else:
            return jsonify({'error': 'Invalid operation'}), 400

        return jsonify({'result': result})
    except (ValueError, KeyError):
        return jsonify({'error': 'Invalid input'}), 400

if __name__ == '__main__':
    # Demo-only setting. Do not use debug=True in production.
    app.run(host='0.0.0.0', port=5000, debug=True)
```

### Create `templates/index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Simple Calculator</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      max-width: 400px;
      margin: 50px auto;
      padding: 20px;
      background-color: #f0f0f0;
    }
    .calculator {
      background: white;
      padding: 20px;
      border-radius: 8px;
      box-shadow: 0 2px 10px rgba(0,0,0,0.1);
    }
    input {
      width: 100%;
      padding: 10px;
      margin: 10px 0;
      border: 1px solid #ddd;
      border-radius: 4px;
      box-sizing: border-box;
    }
    button {
      width: 48%;
      padding: 10px;
      margin: 5px 1%;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      background-color: #4CAF50;
      color: white;
      font-size: 16px;
    }
    button:hover {
      background-color: #45a049;
    }
    #result {
      margin-top: 20px;
      padding: 15px;
      background-color: #e8f5e9;
      border-radius: 4px;
      font-size: 20px;
      text-align: center;
    }
    .error {
      background-color: #ffebee;
      color: #c62828;
    }
  </style>
</head>
<body>
  <div class="calculator">
    <h1>Simple Calculator</h1>
    <input type="number" id="num1" placeholder="Enter first number" step="any">
    <input type="number" id="num2" placeholder="Enter second number" step="any">

    <button onclick="calculate('add')">Add (+)</button>
    <button onclick="calculate('subtract')">Subtract (-)</button>
    <button onclick="calculate('multiply')">Multiply (×)</button>
    <button onclick="calculate('divide')">Divide (÷)</button>

    <div id="result"></div>
  </div>

  <script>
    async function calculate(operation) {
      const num1 = document.getElementById('num1').value;
      const num2 = document.getElementById('num2').value;
      const resultDiv = document.getElementById('result');

      if (!num1 || !num2) {
        resultDiv.className = 'error';
        resultDiv.textContent = 'Please enter both numbers';
        return;
      }

      try {
        const response = await fetch('/calculate', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({
            num1: parseFloat(num1),
            num2: parseFloat(num2),
            operation: operation
          })
        });

        const data = await response.json();

        if (response.ok) {
          resultDiv.className = '';
          resultDiv.textContent = `Result: ${data.result}`;
        } else {
          resultDiv.className = 'error';
          resultDiv.textContent = `Error: ${data.error}`;
        }
      } catch (error) {
        resultDiv.className = 'error';
        resultDiv.textContent = 'Connection error';
      }
    }
  </script>
</body>
</html>
```

### Create `requirements.txt`

```txt
flask==2.0.1
werkzeug==2.0.1
```

These versions are intentionally old enough to help demonstrate SCA behavior.

### Create `Dockerfile`

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .
COPY templates/ templates/

EXPOSE 5000

CMD ["python", "app.py"]
```

### Make the first commit

```bash
echo "__pycache__/" > .gitignore
echo "*.pyc" >> .gitignore
echo ".pytest_cache/" >> .gitignore

git add .
git commit -m "Initial commit: Calculator web app"
```

---

## SAST workflow

### What SAST is

SAST examines code without running it. In this guide, GitHub CodeQL is used because it integrates directly with GitHub Actions and the Security tab.

### SAST workflow YAML

Create `.github/workflows/1-sast-only.yml`:

```yaml
name: 1. SAST - Code Security Scan

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

jobs:
  analyze:
    name: CodeQL Code Scan (SAST)
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        language: [ 'python' ]

    steps:
      - name: Checkout repository
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
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v4
        with:
          category: sast
```

### How the SAST workflow works

This workflow:

1. checks out the repository  
2. initializes CodeQL  
3. sets up Python  
4. installs dependencies  
5. runs CodeQL analysis and sends results to the Security tab  

### CodeQL query suites

Common query packs:

- `security-extended` — recommended for a training lab  
- `security-minimal` — faster, less coverage  
- `security-and-quality` — adds general code quality checks  

---

## SCA workflow

### What SCA is

SCA checks your packages and transitive dependencies for known vulnerabilities.

### SCA workflow YAML

Create `.github/workflows/2-sca-only.yml`:

```yaml
name: 2. SCA - Dependency Security Scan

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

jobs:
  sca:
    name: OWASP Dependency-Check (SCA)
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

      - name: Run OWASP Dependency-Check
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

      - name: Upload Dependency-Check reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: sca-reports
          path: reports/

      - name: Upload SARIF to GitHub Security (if present)
        uses: github/codeql-action/upload-sarif@v4
        if: always() && hashFiles('reports/dependency-check-report.sarif') != ''
        with:
          sarif_file: reports/dependency-check-report.sarif
          category: sca
```

### How the SCA workflow works

Key settings:

- `project: 'calculator-app'` gives the scan a readable name  
- `path: '.'` scans the repository root  
- `format: 'ALL'` generates multiple report formats  
- `--failOnCVSS 7` sets a failure threshold at CVSS 7 and above  
- `--disableOssIndex` avoids extra analyzer warnings in a beginner lab  

Reports are uploaded even if the scan fails, which is useful for teaching.

---

## DAST workflow

### What DAST is

DAST tests the running application from the outside. Unlike SAST or SCA, it needs the app to start successfully first.

In this beginner version, ZAP:

- still runs baseline and full scans  
- still produces full HTML/JSON/Markdown reports  
- does **not fail the job** just because alerts exist  

### DAST workflow YAML (beginner-friendly)

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

- `fail_action: false` ensures the ZAP Baseline action does **not** fail the job due to alerts.  
- `cmd_options: '-a -j -I'`:
  - `-a`: include extra passive rules (more findings to review)  
  - `-j`: also generate a JSON report  
  - `-I`: do not treat warnings/alerts as a failing exit code  

The only reasons for a red DAST job now are:

- the Flask app fails to start  
- the runner or ZAP container actually breaks  

---

## Integrated pipeline

### Complete workflow YAML

Create `.github/workflows/4-complete-security.yml`:

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

---

## Running the repo

### Push to GitHub

```bash
git add .
git commit -m "Add security workflows"
git push origin main
```

### Watch workflow runs

- Open the **Actions** tab.
- Look for:
  - `1. SAST - Code Security Scan`
  - `2. SCA - Dependency Security Scan`
  - `3. DAST - Live Application Scan (Beginner Friendly)`
  - `4. Complete Security Pipeline`

Most jobs should be green unless something is actually broken (e.g., app won’t start).

### Verify artifacts

From any run:

- Scroll to **Artifacts**
- Download:
  - `sca-reports`
  - `zap-baseline-reports`
  - `zap-fullscan-reports`

Open the HTML reports in a browser to review findings.

---

## Reading results

### Actions tab

- Shows per-workflow status and logs.
- Good for debugging failing steps and checking if the app started.

### Security tab

- Aggregates CodeQL (SAST) and any SARIF uploaded from tools like Dependency-Check.
- Lets you filter by severity, tool, and branch.

### Severity levels

- CodeQL: error / warning / note.
- Dependency-Check: CVSS-based (e.g., 7+ is often “high”).
- ZAP: High / Medium / Low / Informational.

Explain to students that “green pipeline” does not mean “no issues”; it only means “tools ran successfully”.

---

## Understanding ZAP findings

### Result categories

ZAP often reports:

- Missing or weak security headers
- Cookie flags (HttpOnly, Secure)
- Information disclosure via headers or error pages
- Input and output handling issues

### Common header findings

For this HTTP-only Flask demo, typical alerts:

- Missing Strict-Transport-Security header
- Missing Content Security Policy
- Missing X-Content-Type-Options or X-Frame-Options

These are good starting points for discussions on secure defaults and reverse proxy configuration.

### Application findings

As the app grows, ZAP may find:

- Parameters without server-side validation
- Reflected content without output encoding
- Unprotected endpoints

### How to interpret a scan

Suggested approach for students:

1. Count High / Medium / Low / Informational alerts.
2. Start with the highest severity issues.
3. Read the description and “Solution” text in the report.
4. Map each alert to a code or configuration change to try.

---

## Troubleshooting

Common issues:

- **App won’t start**: check `flask.log` output in the workflow logs.
- **ZAP cannot connect**: confirm `http://localhost:5000` is correct and the wait loop passes.
- **Artifact name errors**: remove underscores or slashes from artifact names.
- **Permissions errors**: ensure `security-events: write` and `contents: read` are set.

---

## Practice exercises

- Run each workflow individually (SAST, SCA, DAST) and identify at least one finding.
- Use the integrated pipeline and compare results across scans.
- Fix one SCA or SAST issue, push again, and observe how the results change.
- Use ZAP reports to identify which issues are headers vs application code.
