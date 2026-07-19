---
name: python-dev
description: Python development standards for zero-fabrication, test-driven work with strict quality gates. Use when writing or modifying any Python project — enforces venv-only setup, per-file real integration tests, ruff linting, the `run` facade, and prohibited-pattern rules. Read `gui.md` before any GUI code and `GOTCHAS.md` when debugging mysterious failures.
---

# Python Development Standards

**Zero-Fabrication | Test-Driven | Zero-Tolerance**

Real implementations, rigorous per-file testing, no fabrication. These rules are non-negotiable.

## Before You Start

- **GUI work of ANY kind** (desktop app, window, dialog, widget, display, visualization; new app, adding visual components, styling/layout/color/font changes, icons/images) — read and follow `gui.md` in this skill directory FIRST. It fixes the shared framework (PySide6), visual language, and architecture for all desktop apps on this system.
- **Debugging mysterious failures** — read `GOTCHAS.md` in this skill directory. Covers FastAPI, SQLAlchemy, SQLite, Playwright, Python stdlib, Claude SDK, Pyright, macOS/PyObjC, ML models, Web APIs, services/processes, and ffmpeg.

---

## Repository & Project Layout

**Required structure**
```
README.md
requirements.txt
.gitignore
run              # Python file with argparse delegation (no .py), executable
src/             # all source code
output/          # gitignored runtime outputs
  testing/       # gitignored test output/logs/artifacts
local/           # gitignored large downloads/artifacts
```

**Directory constraints**
- Shallow structure preferred. Once you use subdirectories, place peer modules at the same nesting level.
- Max 20 files per directory.
- No experimental scripts or alt versions anywhere; use branches for iteration.
- **Access `output/` and `local/` relative to the code, never the current working directory.** The caller's cwd is never a thing. Resolve paths with `Path(__file__).parent` or similar.

**Example `.gitignore`**
```
output/
local/
.venv/
__pycache__/
*.pyc
```

---

## Virtual Environment

**Always use `venv`.** No conda, poetry, or pipenv.

**`run` is OUTSIDE the venv.** It runs with the ambient system Python and does three things only:
1. Create the venv if missing (`python3 -m venv .venv`)
2. Install requirements into the venv (`pip install -r requirements.txt`)
3. Delegate everything into the venv via `os.execv`

Callers never create, activate, or think about the venv — they call `./run` or `~/bin/projectname` from anywhere. The venv is a local detail `run` manages transparently.

---

## Run Facade (Minimal Delegation Only)

`run` is a **Python file named `run`** (no `.py`, executable). It is a **pure facade** — no work, no argument parsing, no business logic. Its only jobs: ensure the venv exists, ensure requirements are installed, and pass ALL arguments through to the real entry point inside the venv. It must **never** parse arguments, implement subcommands, or contain logic beyond bootstrapping. All of that lives in `src/`.

**Example `run`:**
```python
#!/usr/bin/env python3
# run — pure bootstrap facade. No logic here. Everything delegates to src/main.py.
import os, subprocess, sys
from pathlib import Path

SCRIPT_DIR = Path(__file__).resolve().parent
VENV = SCRIPT_DIR / ".venv"
REQS = SCRIPT_DIR / "requirements.txt"
PYTHON = VENV / "bin" / "python"

if not VENV.exists():
    subprocess.run([sys.executable, "-m", "venv", str(VENV)], check=True)

if REQS.exists():
    subprocess.run([str(PYTHON), "-m", "pip", "install", "-q", "-r", str(REQS)], check=True)

# Hand off entirely — this process is replaced, no return
os.execv(str(PYTHON), [str(PYTHON), str(SCRIPT_DIR / "src" / "main.py")] + sys.argv[1:])
```

`src/main.py` handles all argparse, subcommands, and logic. `run` only readies the venv, then disappears via `execv`.

**`~/bin` wrapper** — also create a global-access wrapper:
```bash
# ~/bin/myproject (executable, 2 lines)
#!/bin/bash
exec ~/src/myproject/run "$@"
```

---

## Coding Standards

**Naming**
- Never use a name not in the dictionary.
- Prefer `snake_case` for all identifiers — especially in the database.

**Process title** — `pip install setproctitle`; every app's `main` starts with:
```python
setproctitle.setproctitle('meaningful title')
```

**Commenting**
- No docstrings. Regular comments only.
- Every class/function ends with a **comment block**: a line, a short human name, and a concise why + key intent in prose.
- Keep functions tiny and obviously correct. A function with a loop should mostly loop and call a named helper. A multi-step function delegates each step to named helpers.

**Comment-block example**
```python
# ##################################################################
# clean repository
# restore the working tree to a fresh-clone state by
# removing untracked files, resetting changes, and ensuring only
# canonical files remain.
def cleanRepository() -> None:
    run_shell("git reset --hard")
    run_shell("git clean -xfd")
    ensure_only_expected_files()
```

**Static, namespaced utilities**
```python
# ##################################################################
# text operations namespace
# stateless text helpers grouped for discoverability
class TextOperations:
    # ##################################################################
    # wrap
    # makes sure text is split into lines of less than this width, but
    # without breaking words unless there is no other choice
    @staticmethod
    def wrap(text: str, width: int) -> str:
        return textwrap.fill(text, width)
```

---

## DRY in Practice

Centralize repeated patterns: logging, color output, constants, conversions. After writing code, extract commonality, then migrate other files to the new helper.

```python
# ##################################################################
# print header
# make sure headers are always cyan
def print_header(text: str) -> None:
    print(f"{Fore.CYAN}{text}{Style.RESET_ALL}")
    logger.info(f"----{text}")
```

---

## Error Handling

- Catch exceptions only where you can make a meaningful decision:
  - **Entry points** — fail the task, show the error.
  - **Long processing loops** — log context, continue to next item.
- Elsewhere, catch only to add context and re-raise.
- For network work, use standard backoff utilities kept in one reusable module.
- Never log or raise a constant string; always add context.

**Processing-loop example**
```python
# ##################################################################
# process batch
# ensure one bad item doesn't abort a long batch while logging context for triage
def process_batch(items: list[str]) -> None:
    for item in items:
        try:
            process_one(item)
        except Exception as err:
            logger.error("process_one failed item=%r err=%s", item, err)
            continue
```

**Entry-point exception policy**
```python
# ##################################################################
# top-level entry
# single place to convert exceptions into exit codes and logs
def main() -> int:
    try:
        run_pipeline()
        return 0
    except Exception as err:
        logger.exception("Pipeline failed: %s", err)
        return 1
```

---

## Async / asyncio Gotchas

**Never cache an `asyncio.Semaphore`/`Lock`/`Event`/`Queue` as a module-level singleton if the program calls `asyncio.run()` more than once.** These primitives bind to the event loop alive when first *awaited*; a second `asyncio.run()` creates a NEW loop, and reusing the old object raises `RuntimeError: <Semaphore …> is bound to a different event loop`. Classic trigger: a multi-step CLI where each step is its own `asyncio.run(step())` but they share a global `_sem` (e.g. an LLM concurrency limiter). Fix — build the primitive lazily **per running loop**:
```python
_sems: dict = {}
def _semaphore() -> asyncio.Semaphore:
    loop = asyncio.get_running_loop()
    sem = _sems.get(loop)
    if sem is None:
        sem = asyncio.Semaphore(N)
        _sems[loop] = sem
    return sem
```
— or create it inside the top-level coroutine and thread it down. Prefer one `asyncio.run()` per process where you can; this bug is a symptom of many.

---

## Secrets & Configuration

**Secrets**
- Only from **keyring** or **AWS Parameter Store**.
- The literal word `keyring` must **never appear in tests**.
- If secret retrieval fails, allow natural failure; do not override or stub.

**Configuration**
- Internal apps: avoid config files. Encode base configuration in code.
- Environment-specific values via environment variables only.

```python
# ##################################################################
# get api key
# centralized secret access with explicit failure; sane default via env override
def get_api_key(name: str) -> str:
    key = keyring.get_password("app", name)
    if not key:
        raise RuntimeError(f"Missing secret for {name}")
    return key

DEFAULT_TIMEOUT_SEC = int(os.getenv("APP_TIMEOUT_SEC", "15"))
```

---

## Testing Policy (Real, Integration-First, Pytest, Per-File Only)

- Each `x.py` has `x_test.py` beside it.
- **Pytest only**; no `__main__` blocks.
- All tests are real end-to-end or integration tests. No smoke checks.
- **Run only the specific test(s) for the file you are working on.** Do NOT run the full suite yourself — `dazpycheck` runs everything at the end. Example while developing `src/text/wrap.py`: `pytest src/text/wrap_test.py::test_wrap_text`.
- Write all test logs and artifacts to `output/testing/`.
- **Expensive external actions**: design a plugin interface with at least two real implementations (e.g. Physical vs In-Memory) and run the same test suite on both. In production, wire either.
- **LLM/costly calls**: use **memoization** keyed by full parameters. Identical calls hit cache; single runs stay fully real.

**Per-file run (tool behavior)**
```bash
# Good: focused
pytest -q src/text/wrap_test.py::test_wrap_text --maxfail=1 --disable-warnings > output/testing/wrap_test.log 2>&1

# Bad: full-suite duplication during development
pytest
```

**Plugin pattern**
```python
# ##################################################################
# check printer
# this prints out an actual check, or cheque - we define it as an
# interface so we can have several different implementations
class CheckPrinter:
    # ##################################################################
    # check printer
    # this prints out an actual check, or cheque
    def print_check(self, data: dict) -> None: ...

# ##################################################################
# check printer physical
# for use when sending out cheques to clients
class CheckPrinterPhysical(CheckPrinter):
    def print_check(self, data: dict) -> None:
        send_to_usb_printer(data)

# ##################################################################
# check printer memory
# for testing, uat and simulation, we don't want real checks to be
# printed out - so we just print them to memory and log them
class CheckPrinterMemory(CheckPrinter):
    def __init__(self) -> None:
        self.printed: list[dict] = []

    def print_check(self, data: dict) -> None:
        self.printed.append(data)

# ##################################################################
# tests of CheckPrinter
# any time we have an interface we want tests that test each implementation, to make sure that
# they all work exactly the same
@pytest.mark.parametrize("impl_cls", [CheckPrinterPhysical, CheckPrinterMemory])
def test_prints_identically(impl_cls):
    impl = impl_cls()
    impl.print_check({"amount": 100})
    # assertions…
```

**LLM memoization**
```python
from functools import lru_cache

# ##################################################################
# llm_complete
# we memoize this because calls to the llm are expensive - this is occasionally useful in production, but
# particularly helps with the speed of tests - if ever time we call the llm in a test we use the same prompt
# then the test suite only ever ends up calling the llm once
@lru_cache(maxsize=256)
def llm_complete(model: str, prompt: str) -> str:
    return call_llm(model=model, prompt=prompt)
```

---

## Linting, Warnings, and `dazpycheck`

- Treat all warnings as errors, including deprecations.
- Line length is 120.
- Run the linter after each file, then run only that file's tests (logging to `output/testing/`).
- At task end, run `dazpycheck` (via `./run check`) — the full test suite and final gates.
- **No commits** without a full `dazpycheck` pass. `git status` must be clean.

```bash
ruff check --line-length 120
pytest -W error -q src/text/wrap_test.py::test_wrap_text > output/testing/wrap_test.log 2>&1
./run check
```

---

## Prohibited Patterns (Zero-Tolerance)

**Forbidden words in code/tests/comments:** `simulate`, `mock`, `fake`, `pretend`, `placeholder`, `stub`, `dummy`, `sleep`, `todo`

**Forbidden method-name fragments:** `_simulate_`, `_mock_`, `_fake_`, `_stub_`, `_dummy_`, `_sleep_`

**Forbidden comments:** `# TODO: replace with real implementation`

Any appearance of the above indicates failure of the task. If a dependency is missing or a system is unavailable, **stop and report a blocker** with exact requirements to proceed:
```
Cannot proceed: Postgres unreachable at $DATABASE_URL
Tried 3 retries with 1/2/4s backoff
Required next step: provide credentials and network access
```

---

## Architecture Heuristics

- **Separation of concerns**: business rules are thin adapters over generic utilities. Cluster by domain (`text/`, `net/`, `io/`, `llm/`).
- **Stateless first**: prefer pure, parameter-defined functions. Memoize when appropriate.
- **Size signals**: refactor if a file exceeds ~200–300 lines or a function exceeds ~25 lines.
- **Interfaces before conditionals**: if you foresee two modes (physical/memory, online/offline), define an interface and two real implementations rather than branching inside one class.

```python
# text/wrap.py
def wrap_text(text: str, width: int) -> str: ...
# loan_system/term_loan.py
def wrap_description(description: str) -> str: return wrap_text(description, 80)

# net/http_client.py
def get_json(url: str, timeout: int) -> dict: ...
# loan_system/catalog.py
def fetch_catalog() -> dict: return get_json(BASE_URL + "/catalog", DEFAULT_TIMEOUT_SEC)
```

**Loop + helper**
```python
# ##################################################################
# normalise all
# go through and normalise each item in the list
def normalize_all(rows: list[str]) -> list[str]:
    return [normalize(r) for r in rows]

# ##################################################################
# normalise
# make sure there's exactly one space between words
def normalize(s: str) -> str:
    return " ".join(s.strip().split())
```

---

## Workflow Integration (n8n)

When Python must execute an n8n workflow:
```python
import requests

def execute_n8n_workflow(workflow_id: str, data: dict = None) -> dict:
    payload = {
        "authKey": "c9543cecb08a4f84644110bedf91b4b04493d95e21b528508d94107c99178b28",
        "workflowId": workflow_id
    }
    if data:
        payload["data"] = data
    response = requests.post(
        "https://BFC910259E30AA1A89A40802CF16112CE.asuscomm.com:11133/webhook/run/workflow",
        json=payload
    )
    response.raise_for_status()
    return response.json()

# execute_n8n_workflow("ppfQrpbtEExtanYe")            # Without data
# execute_n8n_workflow("WORKFLOW_ID", {"key": "value"})  # With data
```

---

## Development Workflow

**For each file change:**
1. Write/modify code following the standards above.
2. Lint: `ruff check --line-length 120`.
3. Run that file's tests: `pytest src/path/file_test.py`, output to `output/testing/`.
4. Fix and repeat 2–3 until clean.

**At task completion:**
1. `./run check` (runs `dazpycheck`).
2. All tests pass; no warnings or lint errors.
3. Clean the repository and re-run until zero errors.
4. Only then commit — `dazpycheck` must be green.
