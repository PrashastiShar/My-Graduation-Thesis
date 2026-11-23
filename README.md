Low-Code Web UI Automation Testing Platform (Based on Playwright)
> A form-based test authoring platform that lets manual QA testers create, run and monitor
> Playwright UI tests without writing Python, while producing plain, readable
> Playwright + pytest scripts that automation engineers can take over.
![Python](https://img.shields.io/badge/Python-3.11%2B-blue)
![Flask](https://img.shields.io/badge/Flask-3-black)
![Playwright](https://img.shields.io/badge/Playwright-1.63-green)
![Docker](https://img.shields.io/badge/Docker-containerized-blue)
![CI](https://img.shields.io/badge/CI-GitHub%20Actions-blue)
![License](https://img.shields.io/badge/license-MIT-lightgrey)
![Test case editor](docs/images/editor.png)
---
Table of contents
Overview and motivation
Key features
System architecture
Tech stack
Project structure
Quick start
Configuration
Using the platform
CI/CD workflow
Security model
Testing the platform itself
Evaluation
Limitations
Future work
Academic context
License
---
1. Overview and motivation
Manual QA testers usually understand business workflows best: what a login, a checkout
or a form submission is supposed to do. Turning that knowledge into automated UI tests
normally needs programming skills, so test automation becomes a bottleneck owned by a
small group of SDETs. Every scenario a tester finds has to be handed over, explained and
re-implemented by someone else.

This bachelor's thesis prototype lets a tester:
-define test steps in a web form (navigate, click, type, choose from dropdowns,
-tick checkboxes, press keys, and assert text, visibility, URL and title);
-have the backend generate a standard Playwright + pytest script, checked for safety
before it runs;
-run it in an isolated Docker container for reproducible results;
-see per-step results, a screenshot of the failing step and a Playwright trace
that replays the whole run;
-export the test so GitHub Actions runs it on every push, every night, or after each
deployment of the application under test, with results stored in Aliyun OSS and
visible back in the platform.

The goal is not to replace SDETs but to shift routine test authoring towards the people who
know what to test, while the output stays plain pytest code an engineer can read, extend
and own. The project is a functional prototype, not production software
(see Limitations).

2. Key features
Feature	Description
Form-based test editor	12 actions (go to URL, click, input text, press key, select option, check/uncheck, assert visible/hidden/text/URL/title), reorderable steps, live preview of the generated script.
Robust element locators	Role + accessible name (Playwright's recommended strategy), label, placeholder, visible text, `data-testid`, or CSS selector.
Auto-waiting assertions	Generated tests use Playwright's `expect(...)` assertions, which retry until the page is ready, instead of fixed sleeps.
Safe script generation	Three layers: strict validation, every user value emitted as a Python string literal, and an AST allow-list check of the generated code. See Security model.
Secrets as variables	Write `${DEMO_PASSWORD}` in a step; the value comes from `.env` or GitHub secrets at run time and never appears in the database, the script or the report.
Background execution	Runs go to a worker pool. The UI gets a run id at once and polls the status, so the web server never blocks during a browser test.
Hardened Docker runs	Non-root, no Linux capabilities, read-only filesystem, memory/CPU/process limits, timeout, container removed afterwards.
Per-step diagnostics	Status and duration of every step, the error message, a full-page screenshot of the failed step, and a Playwright trace (DOM snapshots, network, console) on failure.
HTML and JUnit reports	`pytest-html` self-contained report with the failure screenshot embedded, plus `junit.xml` for other tools.
CI integration	Test cases exported as JSON to `test_cases/` run in GitHub Actions on push, pull request, a nightly schedule, manual dispatch or a deployment webhook.
Cloud storage with private access	Reports, screenshots and traces go to Aliyun OSS; the platform hands out short-lived signed links, so the bucket stays private.
CI results in the platform	Each run writes a `run.json` manifest; "Sync CI runs" imports CI results into the platform's history.
Bundled demo application	A small demo shop at `/demo/` serves as the system under test, so tests and CI need no external website.

3. System architecture
```
                 ┌──────────────────────────────┐
  Tester ──────▶ │ Frontend (HTML + vanilla JS) │
                 └──────────────┬───────────────┘
                                │ REST / JSON
                 ┌──────────────▼───────────────────────────────────────────┐
                 │ Flask backend                                             │
                 │  validation ─▶ script generator ─▶ AST allow-list check   │
                 │  SQLite (cases, steps, runs, step results)                │
                 │  run service (thread pool) ─▶ test runner                 │
                 └──────┬──────────────────────────────┬────────────────────┘
                        │ docker run (hardened)         │ "Export for CI"
                        ▼                               ▼
          ┌──────────────────────────┐        test_cases/*.json ──git push──┐
          │ Test container           │                                      │
          │ pytest + Playwright      │                                      ▼
          │ screenshots · trace ·    │        ┌───────────────────────────────────┐
          │ report · steps.json      │        │ GitHub Actions                     │
          └────────────┬─────────────┘        │ unit tests ─▶ build image ─▶       │
                       │                      │ run test_cases/ in the container   │
                       ▼                      └───────────────┬───────────────────┘
          ┌──────────────────────────┐                        │
          │ Storage: local disk or   │◀───────────────────────┘
          │ Aliyun OSS               │   artifacts + run.json manifest
          │ runs/<id>/run.json …     │
          └────────────┬─────────────┘
                       │ "Sync CI runs" reads manifests
                       ▼
                 back into SQLite → visible in the UI
```
Module	Code	Responsibility
Frontend configuration	`frontend/`	Editor for test cases and steps, run history, step-level results.
Validation	`backend/validation.py`	Schema, whitelisted actions/strategies/roles, length limits, reserved variables.
Script generation	`backend/services/script_generator.py`, `backend/templates/test_script.py.j2`	Maps each step to Playwright code, renders the pytest file, checks it.
Test execution	`backend/services/test_runner.py`, `backend/runner_support/conftest.py`	Prepares the run folder, runs pytest in Docker or locally, collects results.
Background runs	`backend/services/run_service.py`	Thread pool, run status lifecycle (`queued → running → passed/failed/error`).
Report storage	`backend/services/storage.py`	Local disk or Aliyun OSS, signed URLs, run manifests.
CI integration	`.github/workflows/ci.yml`, `backend/cli.py`, `backend/services/ci_sync.py`	Runs exported cases in CI and imports the results.

4. Tech stack
Layer	Technology
Backend	Python 3.11+, Flask 3
Database	SQLite (metadata only; files live in storage)
Frontend	HTML + vanilla JavaScript, no build step
Test engine	Playwright for Python (sync API), pytest, pytest-html
Script templating	Jinja2
Containerization	Docker (`mcr.microsoft.com/playwright/python` base image)
CI/CD	GitHub Actions
Cloud storage	Aliyun OSS (`oss2` SDK)

5. Project structure
```
low-code-ui-testing-platform/
├── backend/
│   ├── app.py                    # Flask app factory and entry point
│   ├── cli.py                    # init-db, import/export, generate, run (used by CI), sync-ci
│   ├── config.py                 # all settings, from environment variables / .env
│   ├── db.py                     # SQLite schema and connections
│   ├── repository.py             # data access for cases, steps, runs, step results
│   ├── validation.py             # test-case schema and rules
│   ├── routes/api.py             # REST API
│   ├── services/
│   │   ├── script_generator.py   # config → checked Playwright/pytest script
│   │   ├── test_runner.py        # executes one run (docker or local)
│   │   ├── run_service.py        # background thread pool
│   │   ├── storage.py            # local disk / Aliyun OSS, manifests, signed URLs
│   │   └── ci_sync.py            # imports CI run manifests
│   ├── templates/test_script.py.j2
│   └── runner_support/conftest.py  # page + step fixtures, screenshots, traces
├── demo_app/                     # demo shop served at /demo/ (system under test)
├── frontend/                     # index.html, js/main.js, css/style.css
├── test_cases/                   # exported test cases (JSON) that CI runs
├── tests/                        # unit, API and end-to-end tests of the platform
├── evaluation/                   # benchmark script, external scenarios, study protocol
├── docs/                         # screenshots, thesis change notes
├── docker/Dockerfile
├── .github/workflows/ci.yml
├── requirements.txt
├── .env.example
└── LICENSE
```
Created at run time (ignored by Git): `data/platform.db`, `runs/<run_id>/`, `evaluation/results/`.

6. Quick start
Prerequisites: Python 3.11+, Git, and Docker (for the default isolated executor).
```bash
git clone https://github.com/<your-username>/low-code-ui-testing-platform.git
cd low-code-ui-testing-platform

python -m venv venv
source venv/bin/activate            # Windows: venv\Scripts\activate
pip install -r requirements.txt
playwright install --with-deps chromium

cp .env.example .env                # then edit .env (see Configuration)

python -m backend.cli init-db
python -m backend.cli import test_cases/     # load the three sample test cases
```
Option A, isolated execution (default, `EXECUTOR=docker`):
```bash
docker build -t ui-test-runner -f docker/Dockerfile .
HOST=0.0.0.0 API_TOKEN=pick-a-token python -m backend.app
```
Tests run in containers, where `localhost` means the container itself. The runner
therefore rewrites `localhost` in the base URL to `host.docker.internal`, and the platform
must listen on `0.0.0.0` so the container can reach the bundled demo app. Set an
`API_TOKEN` when you do that (the UI asks for it once).
Option B, quick local execution (`EXECUTOR=local`):
```bash
EXECUTOR=local python -m backend.app
```
Tests run as a pytest subprocess on your machine: no Docker needed, less isolation.
Then open http://localhost:5000, select Login_Success_Test and click Run test.
Run the platform's own tests with `pytest`.

7. Configuration
All settings are environment variables, read from `.env` if present. See `.env.example`.
Variable	Default	Purpose
`HOST`, `PORT`	`127.0.0.1`, `5000`	Where the web server listens.
`API_TOKEN`	empty	When set, every change needs the `X-API-Token` header.
`EXECUTOR`	`docker`	`docker` (isolated) or `local`.
`DOCKER_IMAGE`	`ui-test-runner`	Image used per run.
`RUN_TIMEOUT_SECONDS`	`300`	Whole-run timeout; the container is killed after it.
`STEP_TIMEOUT_MS`	`10000`	How long each step waits for its element.
`MAX_PARALLEL_RUNS`	`2`	Size of the worker pool.
`TRACE_MODE`	`retain-on-failure`	`on`, `off` or `retain-on-failure`.
`STORAGE_BACKEND`	`local`	`local` or `oss`.
`OSS_ACCESS_KEY_ID`, `OSS_ACCESS_KEY_SECRET`, `OSS_ENDPOINT`, `OSS_BUCKET_NAME`	—	Aliyun OSS access (required for `oss`).
`OSS_PREFIX`	`ui-tests/`	Folder inside the bucket.
`OSS_URL_EXPIRY_SECONDS`	`3600`	Lifetime of signed links.
`DEMO_USERNAME`, `DEMO_PASSWORD`	`demo`, `demo-pass`	Demo app credentials, also used as test variables.
A RAM user with permission only for this bucket
(`oss:PutObject`, `oss:GetObject`, `oss:ListObjects`) is safer than the account's main key.
OSS makes browsers download HTML files opened through its default domain rather than
display them; to view reports in the browser, bind a custom domain to the bucket, or
download the report and open it locally.

8. Using the platform
Create a test case: click + New test case, enter a name (e.g. `Login_Success_Test`)
and the base URL of the application under test.
Add steps. For each step pick an action, then the element and a value where needed:
Action	Element	Value	Generated code (example)
Go to URL	—	`/demo/` or full URL	`page.goto(url('/demo/'))`
Click	yes	—	`page.get_by_role('button', name='Log in').click()`
Input text	yes	text or `${VAR}`	`page.get_by_label('Password').fill(resolve('${DEMO_PASSWORD}'))`
Press key	yes	`Enter`, `Control+A`	`page.get_by_label('Password').press('Enter')`
Select dropdown option	yes	label or value	`page.get_by_label('Language').select_option('中文')`
Check / Uncheck	yes	—	`page.get_by_label('Email notifications').check()`
Assert element visible / hidden	yes	—	`expect(page.get_by_role('heading', name='Welcome')).to_be_visible()`
Assert element contains text	yes	expected text	`expect(page.get_by_role('alert')).to_contain_text('Invalid')`
Assert URL / title contains	—	expected text	`expect(page).to_have_url(re.compile(re.escape('/dashboard')))`
Choosing an element. Prefer role + accessible name (e.g. role `button`, name
`Log in`): it is how users and screen readers see the page, and it survives most
layout changes. Use CSS selectors only as a last resort. To find a locator quickly, run
`playwright codegen http://localhost:5000/demo/`, click the element with the
Pick locator tool, and copy the role and name it suggests.
Save, then Run test. The run appears in the history and updates live.
Read the result. Every step shows passed / failed / not run with its duration. A
failed step shows the error and a screenshot. Open the trace at
https://trace.playwright.dev (drag the file in) or with `playwright show-trace trace.zip`
to step through the DOM, network and console at every moment of the run.
Export for CI writes the case to `test_cases/<name>.json`. Commit and push it to have
GitHub Actions run it.
Passed run	Failed run with screenshot
![Passed run](docs/images/run_passed.png)	![Failed run](docs/images/run_failed.png)
![Generated script preview](docs/images/generated_script.png)

9. CI/CD workflow
`.github/workflows/ci.yml` has two jobs.
1. Platform unit tests: `pytest tests` (validation, generator, injection cases, API, runner).
2. UI tests, run after the unit tests pass:
build the `ui-test-runner` image from the same Dockerfile used locally;
start the bundled demo app, unless a `base_url` was given;
run every file in `test_cases/` inside the container (`python -m backend.cli run test_cases --source ci`);
upload artifacts and a `run.json` manifest per run to Aliyun OSS when OSS secrets are
configured; always also keep them as workflow artifacts for 14 days;
fail the workflow if any UI test fails.
Triggers.
Trigger	Why
`push` to `main`, `pull_request`	A change to the platform or to `test_cases/` is verified.
`schedule` (02:00 China time)	Nightly regression, independent of code changes.
`workflow_dispatch` with `base_url`	Run the suite on demand against any deployed environment.
`repository_dispatch` type `app-deployed`	The application under test's own pipeline triggers the suite after each deployment.
To trigger it after a deployment, the application's pipeline sends:
```bash
curl -X POST -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/<owner>/<repo>/dispatches \
  -d '{"event_type":"app-deployed","client_payload":{"base_url":"https://staging.example.com"}}'
```
Getting CI results into the platform. The GitHub runner cannot reach the platform's
SQLite database, so results travel through storage instead. CI writes `runs/<id>/run.json`
next to the artifacts in OSS. Sync CI runs in the UI (or `python -m backend.cli sync-ci`)
reads new manifests and adds them to the history, linked by test-case name and marked CI,
with the commit and a link to the Actions run. For this to work, the platform itself must
use `STORAGE_BACKEND=oss` with the same bucket and prefix as CI.
Required secrets: `OSS_ACCESS_KEY_ID`, `OSS_ACCESS_KEY_SECRET`, `OSS_ENDPOINT`,
`OSS_BUCKET_NAME`; optionally `DEMO_USERNAME` / `DEMO_PASSWORD`. Without OSS secrets (e.g. on
forks) the workflow still runs and keeps results as workflow artifacts.

10. Security model
Generating code from user input and executing it is the platform's main risk. It is
handled in layers:
Validation (`validation.py`): only known actions, locator strategies and ARIA roles;
names restricted to letters, digits and `. _ - ( )`; length limits; no control characters;
key names checked; URLs must be `http(s)` or a path.
No user text as code: every user value enters the script through `repr()`, as a Python
string literal. Metadata such as the test name goes into constants, never into comments
or docstrings.
Allow-list check: the generated source is parsed with `ast`. Inside the test
function only known Playwright calls are allowed, and imports, nested functions and
classes are rejected. A generator bug that let code through is caught before the file is
written or run.
Secret isolation: tests only see the `${VARIABLES}` they reference. Platform
credentials (`OSS_*`, `API_TOKEN`, `SECRET_KEY`, `GITHUB_*`, …) are reserved and cannot be
referenced, so a test cannot type them into a website.
Container isolation: Docker runs are non-root, with all Linux capabilities dropped,
`no-new-privileges`, a read-only root filesystem, memory/CPU/process limits and a timeout.
Access: the server listens on `127.0.0.1` by default; `API_TOKEN` protects all changes
when it is exposed. Artifact paths are checked against path traversal; the frontend
inserts all user text as text, never as HTML.
The test suite includes 18 injection cases (6 payloads × 3 input positions) and a tampered
source check. This is a design-level defence, not a penetration-tested security
guarantee (see Limitations).

11. Testing the platform itself
```bash
pytest            # 84 tests in about 10 s
```
File	Covers
`tests/test_validation.py`	Schema rules, bad names, reserved variables.
`tests/test_script_generator.py`	Code for every action, golden-file comparison, injection payloads, AST check.
`tests/test_api.py`	CRUD, validation errors, background runs, export, API token, path traversal, CI sync.
`tests/test_runner.py`	Docker hardening flags, secrets not on the command line, missing variables.
`tests/test_end_to_end.py`	All sample cases in a real browser against the demo app; a failing case produces a screenshot and trace. Skipped if no browser is installed.
The golden file `tests/golden/login_success.py.golden` pins the generator's output, so
any change to the generated code shows up as a reviewable diff.

12. Evaluation
`evaluation/` contains everything needed for the evaluation chapter. See
`evaluation/README.md` for the full protocol.
RQ1 expressiveness: coverage matrix over the demo app and two public practice sites.
RQ2 reliability / RQ3 efficiency: `python -m evaluation.benchmark test_cases/ --repeat 30`
records pass rate, flaky cases, generation time, run time and platform overhead (local vs Docker).
RQ4 usability: a user study protocol (tasks, counterbalanced comparison, SUS questionnaire).
RQ5 safety: the injection test suite.

13. Limitations
This is an undergraduate thesis prototype. Known limitations:
Action coverage: no file upload, drag and drop, hover, iframes, multiple tabs or
JavaScript dialogs.
Linear tests only: no conditional steps, loops or data-driven parameters (variables
cover secrets and environments, not test data tables).
Manual locators: no built-in visual element picker; `playwright codegen` is the
suggested helper.
One-way generation: if an engineer edits a generated script, the change does not flow
back into the form, and regenerating overwrites it.
Simple access control: one shared token, no user accounts or roles.
Security not externally tested: the layered defence is designed and unit-tested, not
audited or penetration-tested.
CI sync is pull-based: CI results appear after Sync CI runs, not instantly.
SQLite and one process: suited to a team-sized prototype, not many concurrent users.

14. Future work
More actions: file upload, hover, drag and drop, iframes, tabs, dialogs.
Visual element picker embedded in the editor.
Data-driven and conditional testing: parameter tables and optional steps.
Two-way sync: import hand-edited scripts back into the form where possible.
Suites and scheduling inside the platform, not only in CI.
User accounts and roles (tester / SDET / admin).
Trend analytics: pass rate and flakiness over time per test case.
Push-based CI results: CI calls the platform's API when it is reachable.

15. Academic context
> **Title:** Design and Implementation of a Low-Code Web UI Automation Testing Platform Based on Playwright
> **Institution:** Wuhan University of Technology
> **Degree:** Bachelor of Engineering, Computer Science
> **Nature:** Academic prototype
The full thesis is available on request.
`docs/thesis-changes.md` summarises the design changes of this
version and where they belong in the thesis.

16. License
Released under the MIT License.
