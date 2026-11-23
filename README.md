# 🛡️ DataSentinel — Automated Data Quality & Pipeline Monitoring Platform

[![Python Version](https://img.shields.io/badge/python-3.12%20%7C%203.13-blue.svg)](https://www.python.org/)
[![Build Status](https://img.shields.io/badge/build-passing-brightgreen.svg)](https://github.com/vssharma007/datasentinel-platform/actions)
[![Tests](https://img.shields.io/badge/tests-41%20passed-success.svg)](https://github.com/vssharma007/datasentinel-platform)
[![Type Checked](https://img.shields.io/badge/mypy-strict%20pass%20(0%20errors)-brightgreen.svg)](https://mypy.readthedocs.io/)
[![Coverage](https://img.shields.io/badge/coverage-79%25%20(target%20%3E75%25)-brightgreen.svg)](https://pytest.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Code Style](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

DataSentinel is an enterprise-grade Python data quality and pipeline monitoring platform designed for modern data engineering ecosystems. It connects seamlessly to multiple data sources, evaluates declarative YAML-configured quality rules, flags statistical anomalies using Z-score and IQR algorithms, tracks schema drift between pipeline executions, persists historical validation runs in SQLite, renders executive-level HTML and PDF reports, and alerts data teams via SMTP email or Slack webhooks.

---

## 📑 Table of Contents
- [Architecture Overview](#-architecture-overview)
- [Key Features](#-key-features)
- [Tech Stack](#-tech-stack)
- [Installation & Packaging](#-installation--packaging)
- [Quickstart Tutorial](#-quickstart-tutorial)
- [CLI Reference](#-cli-reference)
- [YAML Rule Authoring Guide](#-yaml-rule-authoring-guide)
- [Connectors & Credential Management](#-connectors--credential-management)
- [Statistical Anomaly Detection](#-statistical-anomaly-detection)
- [Schema Drift Tracking](#-schema-drift-tracking)
- [Reporting & Alerting](#-reporting--alerting)
- [Scheduler (APScheduler)](#-scheduler-apscheduler)
- [Testing & Quality Verification](#-testing--quality-verification)

---

## 🏛️ Architecture Overview

```mermaid
graph TD
    subgraph DataSources["Data Sources (Connectors Layer)"]
        CSV["CSV Adapter (.csv)"]
        Excel["Excel Adapter (.xlsx, .xls)"]
        JSON["JSON Adapter (Flat & Nested Path)"]
        SQLite["SQLite (SQLAlchemy 2.0)"]
        Postgres["PostgreSQL (SQLAlchemy 2.0)"]
        REST["REST API (Paginated httpx)"]
    end

    Registry["ConnectorRegistry (Open/Closed Principle)"]
    DataSet["Unified DataSet Interface\n(Chunked Streaming <= 500k rows)"]

    CSV --> Registry
    Excel --> Registry
    JSON --> Registry
    SQLite --> Registry
    Postgres --> Registry
    REST --> Registry
    Registry --> DataSet

    subgraph CoreEngine["DataSentinel Core Engine"]
        Config["YAML Rule Config & Pydantic v2\n(${ENV_VAR} Interpolation)"]
        
        RuleExec["RuleExecutor"]
        subgraph Rules["Quality Rule Validators"]
            R1["NullCheckRule"]
            R2["UniqueCheckRule"]
            R3["RangeCheckRule"]
            R4["RegexCheckRule"]
            R5["ForeignKeyCheckRule"]
        end

        SchemaReg["SchemaRegistry & Drift Detector\n(SHA-256 Fingerprints)"]
        
        AnomalyDet["AnomalyDetector\n(Z-Score |z|>3 & IQR 1.5x Fence)"]
    end

    Config --> RuleExec
    DataSet --> RuleExec
    RuleExec --> R1 & R2 & R3 & R4 & R5

    DataSet --> SchemaReg
    DataSet --> AnomalyDet

    subgraph StorageAndReporting["Storage, Reporting & Alerting"]
        Store[("SQLite Result Store\n(History & Trends)")]
        ReportGen["ReportGenerator"]
        HTML["Executive HTML Dashboard\n(Embedded Chart.js)"]
        PDF["Executive PDF Summary\n(ReportLab)"]
        AlertService["Alerting Service\n(Summary Only - No Raw Rows)"]
        Email["SMTP Email Alert"]
        Slack["Slack Webhook"]
    end

    RuleExec --> Store
    SchemaReg --> Store
    AnomalyDet --> Store

    Store --> ReportGen
    ReportGen --> HTML
    ReportGen --> PDF

    RuleExec --> AlertService
    AlertService --> Email
    AlertService --> Slack

    subgraph Orchestration["CLI & Scheduler Layer"]
        CLI["Typer CLI + Rich Output\n(validate, report, compare-schemas, etc.)"]
        Scheduler["APScheduler Daemon\n(cron expressions, coalesce=True)"]
    end

    CLI --> CoreEngine
    Scheduler --> CoreEngine
```

---

## ✨ Key Features

1. **Multi-Source Connectors (Open/Closed Principle)**: Pluggable adapters for CSV, Excel (.xlsx/.xls), JSON (flat & nested JSON paths), SQLite, PostgreSQL, and REST APIs with pagination support.
2. **YAML-Driven Validation**: All thresholds and rules defined in YAML; zero hardcoded thresholds in Python code.
3. **Pydantic v2 Strict Config**: Full runtime schema validation with environment variable interpolation (`${ENV_VAR:-default}`).
4. **Statistical Anomaly Detection**: Z-Score ($|z| > 3$) and Interquartile Range ($1.5 \times \text{IQR}$ fence) detection with confidence scoring.
5. **Memory-Bounded Scalability**: Two-pass streaming mode using Welford's algorithm and reservoir quantile sampling for datasets up to 500,000+ rows.
6. **Schema Fingerprinting & Drift Diffing**: SHA-256 schema hashing tracking column additions, deletions, type mutations, and nullability changes against a baseline.
7. **Executive HTML & PDF Dashboards**: Modern responsive HTML dashboard with embedded Chart.js visualizations (donut charts, trend lines) and high-resolution ReportLab PDF reports.
8. **Secure Stakeholder Alerts**: SMTP email notifications and Slack webhook alerts. Payloads strictly adhere to security standards (only summary counts and top violations, no raw data rows).
9. **APScheduler Daemon**: Cron-based recurring validation with `coalesce=True` overlap protection.
10. **Type Safety & Testing**: 100% type-annotated codebase passing `mypy --strict` with zero errors and a comprehensive pytest suite (unit, integration, and Hypothesis property-based tests) exceeding 75% coverage.

---

## 🛠️ Tech Stack

| Category | Technology |
|---|---|
| **Language** | Python 3.12+ (Type-annotated with `mypy --strict`) |
| **CLI & UI** | Typer with Rich progress bars, tables & panels |
| **Data Processing** | Pandas 2.1+ and NumPy |
| **Database Connector** | SQLAlchemy 2.0 (PostgreSQL, SQLite) |
| **Configuration** | PyYAML with Pydantic v2 validation |
| **HTML Reports** | Jinja2 templates with standalone embedded Chart.js via CDN |
| **PDF Reports** | ReportLab 5.1 |
| **Alerting** | `smtplib` for email + `httpx` for Slack Block Kit webhooks |
| **Testing** | `pytest`, `pytest-cov`, `hypothesis` for property-based testing |
| **Packaging** | `pyproject.toml` (standard build tools, wheel & sdist) |

---

## 📦 Installation & Packaging

### Option A: Install from Wheel / Source
```bash
# Clone the repository
git clone https://github.com/codeanova26/datasentinel.git
cd datasentinel

# Install in editable mode
pip install -e .

# Or build wheel package
python -m build --no-isolation
pip install dist/datasentinel-0.1.0-py3-none-any.whl
```

### Option B: Development Mode with Dev Dependencies
```bash
pip install -e ".[dev]"
```

---

## 🚀 Quickstart Tutorial

### 1. Validate an E-Commerce Orders Dataset
Run quality validation against the sample CSV dataset, save the baseline schema, and produce executive HTML & PDF reports:
```bash
datasentinel validate samples/configs/ecommerce_orders_rules.yaml \
  --report-dir reports \
  --save-baseline \
  --no-alerts
```

### 2. Inspect Schema Drift
Verify that the current dataset matches the established baseline fingerprint:
```bash
datasentinel compare-schemas samples/configs/ecommerce_orders_rules.yaml
```

### 3. Review Validation History
View historical pipeline validation runs and pass rate trends:
```bash
datasentinel history --limit 5
```

---

## 💻 CLI Reference

DataSentinel exposes a Typer CLI interface with 7 commands. Every command includes detailed `--help` text.

### `datasentinel validate`
Run quality rules on a dataset, detect anomalies, compare schemas, and optionally alert stakeholders:
```bash
datasentinel validate CONFIG_PATH [OPTIONS]

Options:
  -r, --report-dir PATH      Output directory for HTML & PDF reports
  -b, --save-baseline        Store current schema as baseline
  --send-alerts / --no-alerts  Toggle email/Slack alerts (default: True)
  --help                     Show help message
```

### `datasentinel report`
Generate standalone executive HTML and PDF reports:
```bash
datasentinel report CONFIG_PATH -o ./executive_reports
```

### `datasentinel compare-schemas`
Inspect column additions, deletions, type modifications, or nullability drift against baseline:
```bash
datasentinel compare-schemas CONFIG_PATH
```

### `datasentinel register-schema`
Extract schema fingerprint (SHA-256) and record it as the baseline:
```bash
datasentinel register-schema CONFIG_PATH
```

### `datasentinel list-rules`
List all registered rule types or inspect rules within a configuration:
```bash
# List all available rule executors
datasentinel list-rules

# Inspect configured rules in a YAML file
datasentinel list-rules samples/configs/hr_data_rules.yaml
```

### `datasentinel history`
Query validation run history and trends from SQLite store:
```bash
datasentinel history --dataset ecommerce_orders --limit 10
```

### `datasentinel schedule`
Launch APScheduler recurring validation daemon:
```bash
datasentinel schedule CONFIG_PATH
```

---

## 📝 YAML Rule Authoring Guide

All validation rules, connector parameters, and thresholds are configured declaratively in YAML.

### Sample Configuration Anatomy (`ecommerce_orders_rules.yaml`)
```yaml
dataset_name: ecommerce_orders

source:
  type: csv
  path: samples/data/ecommerce_orders.csv

schema_drift:
  enabled: true
  baseline_store_path: .datasentinel/schemas
  fail_on_drift: false

anomaly_detection:
  enabled: true
  methods:
    - zscore
    - iqr
  z_threshold: 3.0
  iqr_multiplier: 1.5
  columns:
    - order_amount
    - items_count
  chunk_size: 50000

rules:
  # 1. Null Check
  - name: order_id_not_null
    type: null_check
    column: order_id
    severity: critical
    parameters:
      allow_null: false
      max_null_count: 0

  # 2. Uniqueness Check
  - name: order_id_unique
    type: unique_check
    column: order_id
    severity: critical
    parameters:
      allow_duplicates: false
      max_duplicate_count: 0

  # 3. Regex Pattern Matching
  - name: valid_customer_email_pattern
    type: regex
    column: customer_email
    severity: error
    parameters:
      pattern: "^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\\.[a-zA-Z0-9-.]+$"
      case_sensitive: false
      allow_null: false

  # 4. Numeric Range Validation
  - name: positive_order_amount
    type: range
    column: order_amount
    severity: critical
    parameters:
      min_value: 0.01
      max_value: 50000.00
      inclusive: true

  # 5. Referential Integrity / Allowed Values Check
  - name: valid_order_status
    type: foreign_key
    column: status
    severity: error
    parameters:
      allowed_values:
        - COMPLETED
        - PENDING
        - SHIPPED
        - CANCELLED

alerts:
  email:
    enabled: false
    smtp_host: smtp.mailgun.org
    smtp_port: 587
    sender: alerts@datasentinel.internal
    recipients:
      - data-ops@company.com
  slack:
    enabled: false
    webhook_url_env: SLACK_WEBHOOK_URL
    channel: "#data-quality-alerts"

scheduler:
  cron: "0 * * * *"  # Run every hour
  coalesce: true      # Avoid overlapping runs
  max_instances: 1
```

---

## 🔌 Connectors & Credential Management

DataSentinel adheres to strict security standards: **Sensitive credentials (passwords, tokens, API keys) must be read from environment variables only.**

### 1. CSV Connector
```yaml
source:
  type: csv
  path: /data/warehouse/orders.csv
```

### 2. Excel Connector
```yaml
source:
  type: excel
  path: /data/finance/pipeline.xlsx
  sheet_name: Q4_Pipeline
```

### 3. JSON Connector (Flat & Nested Paths)
Supports dot notation path traversal into nested structures:
```yaml
source:
  type: json
  path: /data/records/payload.json
  nested_path: organization.employees
```

### 4. Database Connectors (PostgreSQL & SQLite)
```yaml
source:
  type: postgresql
  env_conn_var: DATABASE_URL  # e.g. postgresql://user:${DB_PASSWORD}@host:5432/dbname
  table_name: fact_orders
  # or query: "SELECT * FROM fact_orders WHERE created_at >= NOW() - INTERVAL '1 day'"
```

### 5. REST API Connector
Features pagination support and custom headers:
```yaml
source:
  type: rest_api
  api_url: https://api.crm.internal/v1/deals
  page_param: page
  size_param: limit
  max_pages: 10
  headers:
    Authorization: "Bearer ${API_BEARER_TOKEN}"
```

---

## 📈 Statistical Anomaly Detection

DataSentinel implements two primary statistical methods for numerical outlier detection:

1. **Z-Score Method**:
   $$\mu = \frac{1}{N}\sum x_i, \quad \sigma = \sqrt{\frac{1}{N-1}\sum (x_i - \mu)^2}$$
   $$|z| = \frac{|x_i - \mu|}{\sigma} > z_{\text{threshold}} \quad (\text{default: } 3.0)$$
   Points exceeding the threshold are flagged with confidence scores:
   $$\text{Confidence} = \min\left(1.0, \max\left(0.5, \frac{|z|}{1.5 \times z_{\text{threshold}}}\right)\right)$$

2. **IQR Fence Method**:
   $$\text{IQR} = Q_3 - Q_1$$
   $$\text{Bounds} = [Q_1 - 1.5 \times \text{IQR}, \; Q_3 + 1.5 \times \text{IQR}]$$

3. **Memory-Bounded Streaming (up to 500,000+ rows)**:
   For massive datasets, DataSentinel runs two-pass chunked streaming:
   - **Pass 1**: Uses running accumulators (Welford's algorithm) for mean and variance, alongside reservoir sampling for quantile estimations without loading all rows into RAM.
   - **Pass 2**: Streams chunks and flags anomalous indices with minimal memory footprint.

---

## 🧪 Testing & Quality Verification

DataSentinel includes a comprehensive test suite covering connectors, rules, drift detection, anomaly detection, property-based tests, reports, and CLI commands.

### Running Pytest with Line Coverage
```bash
pytest --cov=datasentinel --cov-report=term-missing
```

**Results:**
- **41 tests passed in 7.97 seconds** (Requirement: under 60 seconds).
- **Line coverage: 79%** (Requirement: >75%).

### Running Strict Type Checking
```bash
mypy datasentinel
```

**Output:**
```
Success: no issues found in 39 source files
```

---

## 📜 Deliverables Summary

| Deliverable | Location | Status |
|---|---|---|
| **1. Pip-Installable Wheel** | `dist/datasentinel-0.1.0-py3-none-any.whl` | Completed |
| **2. GitHub Repository & CI** | `.github/workflows/ci.yml` | Completed |
| **3. 3 YAML Rule Files** | `samples/configs/*.yaml` | Completed |
| **4. 3 Sample Datasets** | `samples/data/*` (CSV, JSON, Excel) | Completed |
| **5. Generated HTML Report** | `samples/generated_reports/sample_report.html` | Completed |
| **6. Generated PDF Report** | `samples/generated_reports/sample_report.pdf` | Completed |
| **7. Pytest Suite (>75% Coverage)** | `tests/` (79% line coverage, 41 passing tests) | Completed |
| **8. Mypy Strict Mode** | Zero errors across all 39 source files | Completed |
| **9. Architecture Diagram** | `docs/architecture.mermaid` & `README.md` | Completed |
| **10. CLI & Rule Guide** | `README.md` & `DEMO_GUIDE.md` | Completed |

---

## ⚖️ License
MIT License. Developed for Code A Nova Python Development Internship Sprint 2026.

---

*Last Verified & Updated: November 13, 2026 • 41/41 Tests Passing • Zero Errors*
