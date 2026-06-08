# DevSecOps Lab Handout  
SAST + SCA + DAST with GitHub Actions

This lab wires a simple Flask calculator app into GitHub Actions so every push runs:

- SAST with CodeQL (source code). [web:27][web:28]
- SCA with OWASP Dependency-Check (dependencies). [web:26][web:29]
- DAST with OWASP ZAP (running app). [web:28][web:31]

You will copy–paste the files, push to GitHub, and then read / fix security findings.

---

## 1. Mental model

- **SAST (CodeQL)**: scans your code without running it, looking for dangerous patterns.
- **SCA (Dependency-Check)**: scans your `requirements.txt` for known CVEs.
- **DAST (ZAP)**: hits the running app over HTTP and looks for security problems.

Pipeline flow:

```text
Push code → SAST → SCA → Build app → DAST → Summary
```

---

## 2. Setup steps

### 2.1 Create repo on GitHub

- Name: `calculator-security-demo`
- Visibility: public is easiest for this lab

### 2.2 Clone locally

```bash
git clone https://github.com/YOUR_USERNAME/calculator-security-demo.git
cd calculator-security-demo
```

If needed:

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

### 2.3 Target layout

Make your repo look like this:

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

---

## 3. Application files

### 3.1 `app.py`

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

### 3.2 `templates/index.html`

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

### 3.3 `requirements.txt`

```txt
flask==2.0.1
werkzeug==2.0.1
```

(Old on purpose so SCA finds issues.) [web:26][web:29]

### 3.4 `Dockerfile`

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

---

## 4. Workflows

Place these files in `.github/workflows/`.

### 4.1 `1-sast-only.yml` — SAST (CodeQL)

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

env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true

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

### 4.2 `2-sca-only.yml` — SCA (Dependency-Check)

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

The `--failOnCVSS 7` flag makes the job fail if any vulnerability has CVSS ≥ 7. [web:26][web:30]

### 4.3 `3-dast-only.yml` — DAST (ZAP)

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

### 4.4 `4-complete-security.yml` — integrated pipeline

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

---

## 5. After you push

1. Commit and push:

   ```bash
   git add .
   git commit -m "Add security lab files"
   git push origin main
   ```

2. Open **GitHub → Actions**

   - Check the runs for:
     - `1. SAST - Code Security Scan`
     - `2. SCA - Dependency Security Scan`
     - `3. DAST - Live Application Scan`
     - `4. Complete Security Pipeline`

3. Download artifacts:

   - SCA: `sca-reports` (Dependency-Check HTML/JSON). [web:26][web:30]
   - DAST: `zap-baseline-reports`, `zap-fullscan-reports` (ZAP HTML/JSON). [web:31]

4. Open the **Security** tab to see SAST and SCA results in GitHub’s UI. [web:27][web:33]

---

## 6. Mini-tasks

- Make one pipeline run **fail** and identify the reason (SAST / SCA / DAST).
- Fix one issue (code, dependency, or header) and re-run.
- Try to get the full `4. Complete Security Pipeline` workflow **green**.
