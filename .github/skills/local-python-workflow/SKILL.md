---
name: local-python-workflow
description: 'Set up this repository locally with uv or Python venv, install project and backend requirements, run focused or full pytest checks, and start the API safely on loopback. Use for local setup, Python environment, uv, venv, test, pytest, or isolated staging requests.'
argument-hint: 'Describe whether you need environment setup, tests, or a local API smoke test.'
---

# Local Python Workflow

## When to Use

- Set up a local Python environment with `uv` or `venv`.
- Install project and backend dependencies.
- Run focused or full pytest suites locally.
- Start the API for an isolated local smoke test.

## Safety Rules

- Python 3.11 is the recommended common denominator for this repository. `pyproject.toml` requires at least 3.11; `setup.py` contains a conflicting, outdated declaration.
- Never use production secrets or real exchange credentials for local API tests. Do not create a `.env` file containing real credentials.
- Bind the API only to `127.0.0.1`. Do not start live-trading scripts.
- Do not use runtime monkeypatches as product fixes. Report version/import failures and fix the source or dependency setup explicitly.
- Do not change dependencies or lockfiles unless the task includes that work.

## Set Up with uv

PowerShell from the repository root:

```powershell
uv python install 3.11
uv venv --python 3.11 .venv
.\.venv\Scripts\Activate.ps1
uv pip install -r requirements.txt -r backend/requirements.txt "bcrypt<5"
```

If PowerShell blocks activation, install into and use the environment directly:

```powershell
uv pip install --python .venv\Scripts\python.exe -r requirements.txt -r backend/requirements.txt "bcrypt<5"
.\.venv\Scripts\python.exe --version
```

## Set Up with Python venv

PowerShell:

```powershell
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt -r backend/requirements.txt "bcrypt<5"
```

If the Python launcher is unavailable, use `python -m venv .venv` only after confirming `python --version` is 3.11.

POSIX shell:

```bash
uv python install 3.11
uv venv --python 3.11 .venv
source .venv/bin/activate
uv pip install -r requirements.txt -r backend/requirements.txt 'bcrypt<5'
```

Without `uv`:

```bash
python3.11 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt -r backend/requirements.txt 'bcrypt<5'
```

The requirements combine scientific/ML packages with backend packages and may trigger large installations. For fully reproducible builds, establish one unified dependency and lockfile workflow first.

## Tests and Diagnostics

Check the interpreter and dependency consistency first:

```powershell
python --version
python -m pip check
```

Focused tests:

```powershell
python -m pytest -q tests/test_live_engine.py tests/test_secrets_manager.py
```

Full root suite:

```powershell
python -m pytest -q tests
```

`tests/conftest.py` imports NumPy, Pandas, and Torch at collection time, so even focused root tests require the relevant root dependencies.

The CI command `pytest backend/tests -v` currently points to a directory that is absent from the repository. Do not report that test path as a passing gate until the tests are added or the workflow is corrected.

## Start the API Locally

Start only after fixing the known import blockers. The current API may fail under Pydantic 2 at `Field(regex=...)`; the profiling module also imports `get_question_text`, which is missing from the reviewed questionnaire module.

PowerShell with temporary dummy values and empty exchange credentials:

```powershell
$env:SECRET_KEY = [Convert]::ToBase64String([System.Security.Cryptography.RandomNumberGenerator]::GetBytes(32))
$env:ADMIN_USERNAME = 'local-stage-admin'
$env:ADMIN_PASSWORD = [Guid]::NewGuid().ToString('N') + [Guid]::NewGuid().ToString('N')
$env:BINANCE_API_KEY = ''
$env:BINANCE_API_SECRET = ''
$env:BINANCE_TESTNET = 'true'
$env:ENVIRONMENT = 'staging'
python -m uvicorn backend.main:app --host 127.0.0.1 --port 8765
```

Stop the server with `Ctrl+C` and verify port 8765 is free after testing. Do not transfer temporary workarounds into source code.

## Known Repository Mismatches

- Python support conflicts between `pyproject.toml` (>=3.11), the README, and `setup.py` (>=3.8).
- Backend dependencies are pinned separately; root and backend requirements are not represented by a unified lockfile.
- `backend/main.py` imports `src.config`, but the Compose build context contains only `backend/`.
- Compose sets `BINANCE_SECRET_KEY`, while the code reads `BINANCE_API_SECRET`.
- The reviewed backend does not start without resolving the Pydantic and profiling import errors.

Do not silently work around these mismatches. State version differences, missing modules, and unavailable test paths in the results.
