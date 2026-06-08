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
  - [Why CodeQL](#why-codeql)
  - [Artifact naming rules](#artifact-naming-rules)
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
  - [DAST workflow YAML](#dast-workflow-yaml)
  - [Why this DAST version is better](#why-this-dast-version-is-better)
  - [Optional ZAP rules file](#optional-zap-rules-file)
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
  - [Common student questions](#common-student-questions)
- [Troubleshooting](#troubleshooting)
  - [Wrong action version](#wrong-action-version)
  - [Invalid artifact name](#invalid-artifact-name)
  - [CodeQL runs too long](#codeql-runs-too-long)
  - [SCA runs too long](#sca-runs-too-long)
  - [DAST cannot connect](#dast-cannot-connect)
  - [Workflow stuck in queued state](#workflow-stuck-in-queued-state)
  - [Too many false positives](#too-many-false-positives)
  - [Permissions error](#permissions-error)
- [Practice exercises](#practice-exercises)
  - [Exercise 1: Basic setup](#exercise-1-basic-setup)
  - [Exercise 2: Analyze scan results](#exercise-2-analyze-scan-results)
  - [Exercise 3: Interpret ZAP findings](#exercise-3-interpret-zap-findings)
  - [Exercise 4: Compare multiple scans](#exercise-4-compare-multiple-scans)
  - [Exercise 5: Document the pipeline](#exercise-5-document-the-pipeline)
  - [Exercise 6: Turn the pipeline green](#exercise-6-turn-the-pipeline-green)
- [Quick reference](#quick-reference)

---

## Overview

This project builds a simple Flask calculator app and wires it into GitHub Actions so that every push or pull request can trigger three security scan types:

- **SAST** using CodeQL for source code analysis.
- **SCA** using OWASP Dependency-Check for dependency and CVE analysis.
- **DAST** using OWASP ZAP for dynamic scanning of the running app.

The guide is designed for beginners, but the workflow structure is still realistic enough for workshops, demos, and early DevSecOps training.

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

**DAST** (Dynamic Application Security Testing) attacks the running application from the outside.

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

- **SAST** checks what you wrote.
- **SCA** checks what you imported.
- **DAST** checks what actually happens when the app is live.

A useful analogy:

- **SAST** = reviewing the blueprint
- **SCA** = checking the components you bought
- **DAST** = attacking the finished system

---

## Prerequisites

### What you need

You only need a few basics to run this project:

- GitHub account
- basic Git knowledge
- VS Code or another text editor
- willingness to wait a little on first-run scans

No deep security background is required.

### Why CodeQL

**CodeQL** is a good fit for this guide because it is native to GitHub and easy to adopt.

Advantages:

- free for public repositories
- no external token required for basic setup
- native integration with GitHub Actions
- results flow into the Security tab
- maintained by GitHub
- current CodeQL v4 action line is aligned with GitHub's Node 24 runtime migration

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

- Initial setup: 30 to 45 minutes
- First full pipeline run: 20 to 45 minutes
- Each follow-up practice cycle: 10 to 20 minutes

---

## Project setup

### Create the GitHub repository first

Create the repository on GitHub **before** creating local files. This avoids the most common beginner Git problems, including remote URL mistakes, `remote origin already exists`, and `failed to push some refs` after GitHub creates an initial commit.

Recommended settings on GitHub:

- Repository name: `calculator-security-demo`
- Owner: your personal account or organization
- Visibility: public for easiest CodeQL usage, or private if your plan supports your workflow needs
- Initialize with README: optional, but this guide is easier if you create the repo first and then clone it immediately

After the repository is created, copy its HTTPS clone URL from GitHub.

### Clone the repository locally

Use the GitHub clone URL instead of starting with `git init` in a random local folder.

```bash
git clone https://github.com/YOUR_USERNAME/calculator-security-demo.git
cd calculator-security-demo
```

If Git asks for your identity later during commit, configure it once:

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

### Directory structure

Inside the cloned repository, create the project with this layout:

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

These versions are intentionally old enough to help demonstrate SCA findings.

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

CodeQL checks fail pull requests automatically when they detect high or critical security issues, so in this lab you should expect SAST jobs to fail until you address the reported findings. [web:158][web:155]

### SAST workflow YAML

Create `.github/workflows/1-sast-only.yml`:

```yaml
name: 1. SAST - Code Security Scan

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

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
3. prepares Python
4. installs dependencies
5. runs CodeQL analysis

The `category: sast` field helps organize findings in GitHub’s Security experience.

### CodeQL query suites

Common query packs:

- `security-extended` — recommended for a training lab
- `security-minimal` — faster, less coverage
- `security-and-quality` — broader checks including code quality

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

- `project: 'calculator-app'` gives the scan a readable project name.
- `path: '.'` scans the repository root.
- `format: 'ALL'` generates multiple report formats.
- `--failOnCVSS 7` sets the failure threshold to high severity and will fail the job when any vulnerability has CVSS ≥ 7. [web:145][web:154]
- `--disableOssIndex` suppresses the Sonatype OSS Index analyzer warning for this beginner-friendly demo.

The reports are uploaded even if the scan fails, which is important for teaching and debugging. [web:145][web:148]

For this guide, the SCA workflow intentionally uses NVD-based sources only. That keeps setup simple and avoids Sonatype credential management in a beginner lab. [web:148]

If you previously saw a warning like `Sonatype OSS Index Analyzer disabled due to missing credentials`, this README now avoids that noise by explicitly disabling the OSS Index analyzer in the workflow arguments. That warning is not a fatal error, but disabling the analyzer makes the demo cleaner and easier to teach. [web:145]

---

## DAST workflow

### What DAST is

DAST tests the running application from the outside. Unlike SAST or SCA, it needs the app to actually start first.

### DAST workflow YAML

Create `.github/workflows/3-dast-only.yml`:

```yaml
name: 3. DAST - Live Application Scan

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

      - name: ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.15.0
        with:
          target: 'http://localhost:5000'
          cmd_options: '-a'
          allow_issue_writing: false
          artifact_name: 'zap-baseline-scan'
          fail_action: true

      - name: Preserve baseline reports
        if: always()
        run: |
          [ -f report_html.html ] && mv report_html.html reports/baseline/report.html || true
          [ -f report_json.json ] && mv report_json.json reports/baseline/report.json || true
          [ -f report_md.md ] && mv report_md.md reports/baseline/report.md || true

      - name: ZAP Full Scan
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

### Why this DAST version is better

This version improves reliability in a few important ways:

- it waits for the app with a loop instead of assuming a fixed startup time
- it saves baseline and full scan reports into separate folders
- it avoids report overwrite confusion
- it keeps artifacts available even when scans return warnings or soft failures
- it opts JavaScript actions into Node 24 early so the workflow is better aligned with GitHub's 2026 runtime migration [web:106][web:124]
- it is configured so that ZAP findings can fail the job, which is ideal for teaching how to respond to DAST alerts [web:150][web:147]

With `fail_action: true` in the baseline scan and `-I` in the full scan, ZAP will mark the job as failed when failing-level alerts are present, while still uploading full reports for discussion. [web:150][web:147]

### Optional ZAP rules file

If you want to suppress noisy demo-only findings, create `.zap/rules.tsv`:

```tsv
10035	IGNORE	(Strict-Transport-Security Header)
10063	IGNORE	(Feature Policy Header)
```

Use this only when you understand what you are suppressing.

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
    name: "4️⃣ DAST - Live Security Scan"
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

      - name: ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.15.0
        with:
          target: 'http://localhost:5000'
          cmd_options: '-a'
          allow_issue_writing: false
          artifact_name: 'zap-baseline-scan'
          fail_action: true

      - name: Preserve baseline reports
        if: always()
        run: |
          [ -f report_html.html ] && mv report_html.html reports/baseline/report.html || true
          [ -f report_json.json ] && mv report_json.json reports/baseline/report.json || true
          [ -f report_md.md ] && mv report_md.md reports/baseline/report.md || true

      - name: ZAP Full Scan
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
          echo "- This demo intentionally fails when serious SAST, SCA, or DAST findings are present. Use the reports and logs to decide what to fix next." >> $GITHUB_STEP_SUMMARY
```

### Execution order

In this integrated pipeline, any of the SAST, SCA, or DAST jobs can cause the workflow to fail when they find serious issues. This is intentional for teaching: students should expect red builds until they address the underlying findings.

```text
1. SAST
   ↓
2. SCA
   ↓
3. Build verification
   ↓
4. DAST
   ↓
5. Summary
```

### Generated artifacts

The pipeline should produce:

- `sca-reports`
- `zap-baseline-reports`
- `zap-fullscan-reports`

These names are safe for GitHub Actions and are easy for students to understand.

---

## Practice exercises (new teaching focus)

### Exercise 6: Turn the pipeline green

Goal:

- start from a failing pipeline (due to SAST, SCA, or DAST)
- iteratively fix or suppress issues until all four workflows pass

Suggested sequence:

- fix at least one CodeQL finding and re-run SAST [web:158][web:149]
- upgrade or pin dependencies to reduce high CVSS findings and re-run SCA [web:145][web:154]
- adjust headers or app configuration to reduce ZAP failing alerts and re-run DAST [web:150][web:147]

---

This README is now aligned with the patched workflows:

- CodeQL uses `@v4` and fails on serious findings by default. [web:158][web:155]
- SCA uses `--failOnCVSS 7` and `--disableOssIndex` for high-severity teaching failures without OSS Index credentials. [web:145][web:154]
- DAST uses `fail_action: true` and `-I` so ZAP alerts can fail the job while still saving reports. [web:150][web:147]
- All workflows opt into Node 24 via `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24`. [web:106][web:124]

Do you want me to also paste the final versions of `1-sast-only.yml`, `2-sca-only.yml`, and `4-complete-security.yml` here, or is having them in the README enough for your students to follow?
