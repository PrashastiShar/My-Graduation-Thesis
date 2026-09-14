# Low-Code Web UI Automation Testing Platform (Based on Playwright)

> A form-based, no-code test authoring platform that lets manual QA testers create, run, and monitor Playwright UI automation tests — without writing a single line of Python.

[![Python](https://img.shields.io/badge/Python-3.11-blue)]()
[![Flask](https://img.shields.io/badge/Flask-Backend-black)]()
[![Playwright](https://img.shields.io/badge/Playwright-Test%20Engine-green)]()
[![Docker](https://img.shields.io/badge/Docker-Containerized-blue)]()
[![CI](https://img.shields.io/badge/CI-GitHub%20Actions-blue)]()

---

## Table of Contents

1. [Project Overview & Motivation](#1-project-overview--motivation)
2. [Key Features](#2-key-features)
3. [System Architecture](#3-system-architecture)
4. [Tech Stack](#4-tech-stack)
5. [Project Folder Structure](#5-project-folder-structure)
6. [Prerequisites](#6-prerequisites)
7. [Local Setup & Installation](#7-local-setup--installation)
8. [How to Use the Platform](#8-how-to-use-the-platform)
9. [CI/CD Workflow](#9-cicd-workflow)
10. [Demo Results](#10-demo-results)
11. [Project Limitations](#11-project-limitations)
12. [Future Improvements](#12-future-improvements)
13. [Academic Context](#13-academic-context)
14. [License](#14-license)

---

## 1. Project Overview & Motivation

In most software organizations, **manual QA testers are the people who best understand real business workflows** — they know exactly what a login flow, a checkout process, or a form submission is *supposed* to do. However, converting that knowledge into automated UI tests usually requires programming skills (Python/JavaScript) that manual testers often don't have.

As a result, automated UI test creation and maintenance is bottlenecked by a small group of SDETs / automation engineers. Every new scenario a manual tester identifies has to be handed off, explained, and re-implemented by someone else — introducing delay, communication overhead, and a hard ceiling on how fast automated test coverage can grow.

**This project is a bachelor's thesis prototype built to address that exact bottleneck.**

The platform allows a manual tester to:
- Define test steps (click element, input text, assert visible content) through a simple **web form** — no coding required.
- Have the backend **automatically generate a standard, human-readable Playwright + pytest Python script** from that configuration.
- Run the test in a **Docker container** for consistent, reproducible results.
- Get an **HTML report** with **automatic failure screenshots**.
- Have tests **automatically re-run in the cloud** (GitHub Actions) on every code push, with results persisted to **Aliyun OSS**.

The goal is not to replace SDETs, but to **shift routine test authoring left**, toward the people who already understand what needs to be tested — while still producing plain, standard Playwright/pytest scripts that any automation engineer can read, extend, or take ownership of.

This project was completed as an undergraduate Computer Science bachelor's thesis, and is explicitly scoped as a **functional prototype**, not enterprise production software (see [Limitations](#11-project-limitations)).

---

## 2. Key Features

| # | Feature | Description |
|---|---|---|
| 1 | **Low-code test configuration** | Web form UI to define test steps: click element, input text, assert visible content — no Python required. |
| 2 | **Dynamic script generation** | Template-driven engine that converts a JSON test configuration into a valid, executable Playwright + pytest Python script. |
| 3 | **Containerized execution** | Tests run inside a Docker container pre-configured with Playwright, browsers, and dependencies for consistent results across machines. |
| 4 | **Automatic failure screenshots** | On any failed step, a screenshot of the page state is captured automatically via `pytest-playwright`. |
| 5 | **HTML test reports** | Self-contained HTML reports generated via `pytest-html`, summarizing per-step pass/fail results. |
| 6 | **Cloud CI integration** | GitHub Actions workflow automatically builds the Docker image and re-runs tests on every push to the repository. |
| 7 | **Cloud artifact storage** | Test reports and failure screenshots are uploaded to Aliyun OSS, so they remain accessible after the CI runner/container is discarded. |
| 8 | **Lightweight metadata storage** | SQLite database stores test cases, test steps, and execution history (report/screenshot references only — not the files themselves). |

---

## 3. System Architecture

The platform is organized into five cooperating modules:

```
                         ┌───────────────────────────┐
                         │  Frontend Configuration    │
                         │  Module (HTML/JS Form)     │
                         └─────────────┬─────────────┘
                                       │ REST API (JSON)
                                       ▼
                         ┌───────────────────────────┐
                         │   Flask Backend            │
                         │  ┌───────────────────────┐ │
                         │  │ Script Generation      │ │
                         │  │ Module (templates)     │ │
                         │  └───────────┬───────────┘ │
                         └──────────────┼─────────────┘
                                        ▼
                         ┌───────────────────────────┐        ┌──────────────────────┐
   Code Push ──────────▶ │  GitHub Actions (Cloud CI) │──────▶ │  Test Execution        │
                         │  Module                    │        │  Module (Docker +      │
                         └───────────────────────────┘        │  Playwright + pytest)  │
                                                                └──────────┬─────────────┘
                                                                           ▼
                                                                ┌──────────────────────┐
                                                                │ Report & Screenshot   │
                                                                │ Storage Module        │
                                                                │ (HTML report gen +    │
                                                                │ Aliyun OSS upload)    │
                                                                └──────────────────────┘
```

- **Frontend Configuration Module** — form-based UI for defining/saving test cases.
- **Script Generation Module** — maps each step (click / input / assert_visible) to a Playwright code template and assembles a full pytest script.
- **Test Execution Module** — runs the generated script inside a Docker container using pytest + `pytest-playwright`.
- **Report & Screenshot Storage Module** — generates the HTML report, captures failure screenshots, uploads both to Aliyun OSS, and records references in SQLite.
- **Cloud CI Module** — GitHub Actions workflow that rebuilds the Docker image and re-runs tests automatically on every push.

> `[Insert architecture diagram image - thesis Figure 3.1]`

---

## 4. Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.11, Flask |
| Database | SQLite (lightweight metadata storage) |
| Frontend | HTML + vanilla JavaScript (no React, no Node.js) |
| Test Engine | Playwright (Python), pytest, `pytest-playwright`, `pytest-html` |
| Containerization | Docker |
| CI/CD | GitHub Actions |
| Cloud Storage | Aliyun OSS (Object Storage Service) |
| Version Control | Git / GitHub |

---

## 5. Project Folder Structure

```
low-code-ui-testing-platform/
├── backend/
│   ├── app.py                     # Flask application entry point
│   ├── routes/                    # API route definitions
│   ├── services/
│   │   ├── script_generator.py    # Template-driven Playwright/pytest script generation
│   │   ├── test_runner.py         # Docker/pytest execution logic
│   │   └── oss_uploader.py        # Aliyun OSS upload logic
│   ├── models/                    # SQLite models (test_case, test_step, test_run, failure_screenshot)
│   └── templates/
│       └── script_template.py.j2  # Base Playwright/pytest script template
│
├── frontend/
│   ├── index.html                 # Test-step configuration form
│   ├── js/
│   │   └── main.js
│   └── css/
│       └── style.css
│
├── generated_tests/                # Auto-generated Playwright/pytest scripts (runtime output)
│
├── docker/
│   └── Dockerfile                  # Test execution environment image
│
├── .github/
│   └── workflows/
│       └── run-tests.yml           # GitHub Actions CI workflow
│
├── reports/                        # Locally generated HTML reports & screenshots (runtime output)
│
├── requirements.txt
├── .env.example                    # Example environment variable file (no real secrets)
├── README.md
└── LICENSE
```

---

## 6. Prerequisites

Before setting up the project locally, make sure you have:

- **Python** 3.11+
- **pip** (Python package manager)
- **Docker** (Docker Engine / Docker Desktop)
- **Git**
- An **Aliyun OSS** account with a bucket created, plus an Access Key ID and Access Key Secret (only required if you want to test cloud storage upload)
- (Optional) A **GitHub repository** with Actions enabled, if you want to test the cloud CI pipeline

---

## 7. Local Setup & Installation

```bash
# 1. Clone the repository
git clone https://github.com/<your-username>/low-code-ui-testing-platform.git
cd low-code-ui-testing-platform

# 2. Create and activate a virtual environment
python -m venv venv
source venv/bin/activate        # On Windows: venv\Scripts\activate

# 3. Install backend dependencies
pip install -r requirements.txt

# 4. Install Playwright browser binaries
playwright install --with-deps chromium

# 5. Configure environment variables
cp .env.example .env
# Then edit .env and fill in your own values (see below)
```

### Environment Variables

> ⚠️ **Security Warning:** Never hard-code Aliyun OSS credentials (Access Key ID / Secret) or any other secrets directly in source code or commit them to Git. Always load them from environment variables or a `.env` file that is excluded via `.gitignore`. For GitHub Actions, store them as **encrypted repository secrets**, not as plain text in the workflow YAML.

`.env.example`:

```env
FLASK_ENV=development
SECRET_KEY=change-me

# Aliyun OSS configuration — DO NOT hard-code real values, use environment variables only
OSS_ACCESS_KEY_ID=your-access-key-id
OSS_ACCESS_KEY_SECRET=your-access-key-secret
OSS_ENDPOINT=your-oss-endpoint
OSS_BUCKET_NAME=your-bucket-name
```

```bash
# 6. Initialize the SQLite database
python backend/init_db.py

# 7. Build the Docker test execution image
docker build -t ui-test-runner -f docker/Dockerfile .

# 8. Run the Flask backend
python backend/app.py

# 9. Open the frontend
# Navigate to http://localhost:5000 in your browser
```

---

## 8. How to Use the Platform

1. **Open the web interface** at `http://localhost:5000`.
2. **Create a new test case** and give it a descriptive name (e.g., `Login_Success_Test`).
3. **Add test steps** using the form editor. For each step, choose an action type:
   - `Click` — specify the target element selector.
   - `Input Text` — specify the selector and the text value to enter.
   - `Assert Visible Content` — specify the selector (or page scope) and the expected visible text.
4. **Reorder or remove steps** as needed, then **save** the test case.
5. Click **Run Test** to trigger execution:
   - The backend generates the corresponding Playwright/pytest script.
   - The script runs inside the Docker test execution container.
   - On any failed step, a screenshot is captured automatically.
6. **View the results**: the platform displays the overall pass/fail status and a link to the generated HTML report (with embedded failure screenshots, if any).
7. **Push code** to the connected GitHub repository to trigger the same test suite automatically in the cloud (see [CI/CD Workflow](#9-cicd-workflow)) — reports and screenshots from CI runs are also viewable from the platform.

> `[Insert screenshot here — test step configuration form]`
> `[Insert screenshot here — test run result summary]`

---

## 9. CI/CD Workflow

The platform integrates GitHub Actions as a cloud CI layer, so that tests are continuously verified without manual triggering.

**Trigger:** Every `git push` to the `main` branch.

**Pipeline steps** (`.github/workflows/run-tests.yml`):

1. **Checkout** the repository code.
2. **Build the Docker test execution image**, using the same `Dockerfile` used for local execution — guaranteeing identical behavior between local and cloud runs.
3. **Run the generated Playwright/pytest test scripts** inside the container, using `pytest-playwright` (for browser automation and screenshot-on-failure) and `pytest-html` (for report generation).
4. **Upload results to Aliyun OSS** — the generated HTML report and any failure screenshots are pushed to a dedicated OSS bucket using the Aliyun OSS Python SDK.
5. **Record execution metadata** so results are viewable from the platform's UI, alongside locally triggered runs.

Aliyun OSS credentials used in this step are read from **GitHub Actions encrypted secrets** (`Settings → Secrets and variables → Actions`), never from the workflow file itself.

```yaml
env:
  OSS_ACCESS_KEY_ID: ${{ secrets.OSS_ACCESS_KEY_ID }}
  OSS_ACCESS_KEY_SECRET: ${{ secrets.OSS_ACCESS_KEY_SECRET }}
```

> `[Insert screenshot here — GitHub Actions workflow run log]`
> `[Insert screenshot here — Aliyun OSS bucket showing uploaded reports/screenshots]`

---

## 10. Demo Results

> `[Insert screenshot here — sample generated Playwright/pytest script]`

> `[Insert screenshot here — HTML test report, passing scenario]`

> `[Insert screenshot here — HTML test report with failure screenshot embedded]`

> `[Insert screenshot here — end-to-end login test scenario walkthrough]`

---

## 11. Project Limitations

This is an **undergraduate bachelor's thesis prototype**, not enterprise-grade production software. The following limitations are known and intentional given the project's academic scope:

- **Limited action coverage** — only three basic actions are supported: click, input text, and assert visible content. Dropdowns, file uploads, drag-and-drop, keyboard events, and multi-tab scenarios are not yet supported.
- **Manual selector input** — testers must supply valid element selectors (CSS selector / ID / text) themselves; there is no visual "point-and-click" element picker.
- **No conditional logic or data-driven testing** — test cases execute as a fixed, linear sequence of steps; branching and parameterized (data-driven) testing are not supported.
- **No user authentication or permission system** — all users currently share equal, unrestricted access to test cases, execution, and stored reports.
- **Input escaping, not security hardening** — user-supplied selectors/values are escaped only to preserve script syntax validity; this is **not** a security mechanism and has not been evaluated against adversarial or malicious input (e.g., code injection).
- **Single-application evaluation scope** — functional testing (see thesis Chapter 5) was performed against one sample application; broader validation across varied, real-world, dynamic web applications was not performed.

---

## 12. Future Improvements

Planned directions for extending this prototype beyond its current bachelor-thesis scope:

- **Advanced element recognition** — a visual "point-and-click" element picker to reduce reliance on manually supplied selectors, plus more resilient selector strategies.
- **Expanded action library** — support for dropdowns, checkboxes/radio buttons, file uploads, keyboard events, waits, and multi-tab/window handling.
- **Batch execution & task scheduling** — running multiple test cases as a suite, and scheduled/periodic execution (e.g., nightly regression) independent of code pushes.
- **User permission & role management** — authentication and role-based access control (e.g., tester / SDET / admin roles).
- **Data-driven and conditional testing** — parameterized test cases and basic conditional branching for optional/environment-dependent elements.
- **Enhanced reporting & analytics** — historical pass/fail trend views, and richer failure diagnostics (console logs, network failures) alongside screenshots.
- **Broader real-world validation** — testing against a wider variety of production-like web applications to assess robustness and generalizability.

---

## 13. Academic Context

This repository contains the implementation accompanying an undergraduate Computer Science **Bachelor's Degree Thesis**:

> **Title:** Design and Implementation of a Low-Code Web UI Automation Testing Platform Based on Playwright
> **Institution:** Wuhan University of Technology
> **Degree:** Bachelor of Engineering, Computer Science
> **Nature of project:** Academic prototype / proof of concept, developed to fulfill bachelor's thesis requirements.

The full thesis document — covering requirements analysis, system design, detailed implementation, and evaluation — is available in this repository under `/docs` (or upon request).

> `[Insert link to full thesis PDF here]`

---

## 14. License

This project is released under the [MIT License](LICENSE).

```
MIT License — free to use, copy, modify, and distribute, with attribution,
provided the original copyright and license notice is included.
```

---

**Author's note:** This project was built to explore how thoughtful, low-code tooling can lower the barrier between domain-knowledgeable QA testers and the automated tests that verify real product workflows — a problem I encountered conceptually while researching SDET/QA team structures for this thesis. Feedback, issues, and pull requests are welcome.
