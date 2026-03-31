# Python — Gotchas & Known Pitfalls

**Read these before debugging mysterious failures.**

## FastAPI
- `{path:path}` and `/{persona_id}` routes are greedy — they swallow sub-paths. Register specific sub-routes (e.g., `/budget-mode/new`, `/history`) BEFORE catch-all routes, or they'll never match.
- `TestClient` streaming (SSE): `iter_bytes()`/`iter_lines()` block indefinitely on infinite SSE generators. Use a real uvicorn server in a `threading.Thread` with `server.should_exit = True` for cleanup. Don't use multiprocessing for test servers on macOS (spawn context can't pickle local functions).
- `async def` endpoints with synchronous DB calls block the event loop — stalling ALL concurrent SSE clients. Use plain `def` instead; FastAPI auto-runs them in a threadpool.
- `StaticFiles` mount doesn't set cache headers. For immutable content-hash caching, replace with a custom route that sets `Cache-Control: public, max-age=31536000, immutable` and use `resolve()` + prefix check for path traversal.
- httpx (used by `TestClient`) deprecated per-request `cookies=` parameter. With `-W error`, tests setting `cookies={"key": "val"}` on individual `client.get()` calls fail as `DeprecationWarning`. Fix: `client.cookies.set("key", "val")` before making requests.
- `@app.on_event("startup")` is deprecated. Use `lifespan` context manager: `@asynccontextmanager async def lifespan(app): ... yield ...` passed to `FastAPI(lifespan=lifespan)`.
- Jinja2 template dict fields: avoid naming dict keys `values`, `keys`, `items`, `get`, `update`, or `pop` — these collide with Python dict methods. In Jinja2 `metric.values` resolves to `dict.values()`, not a field. Rename to `scenario_values`, `data_values`, etc.

## SQLAlchemy
- SQLAlchemy + Lambda + PostgreSQL: always use `NullPool` (`create_engine(url, poolclass=NullPool)`). Lambda concurrency exhausts `max_connections`. NullPool opens/closes per request.
- Dual-DB pattern (SQLite for tests, PostgreSQL for prod): guard with `if engine.url.drivername.startswith("sqlite")` (not `"postgresql"`) for robustness to driver changes.

## SQLite
- Multiple connection factories: ensure ALL of them set critical PRAGMAs (`busy_timeout`, `journal_mode`). Missing `busy_timeout` causes immediate "database is locked" errors.
- Per-request `connect()`/`close()` in web servers adds ~500ms overhead. Use thread-local connections (`threading.local()`).
- `CREATE TABLE IF NOT EXISTS` does NOT modify existing tables — won't add new columns. Use `ALTER TABLE ADD COLUMN` in a migration function.
- Computed/normalized columns: when the normalization function changes, recompute ALL rows (not just NULLs). Migration that only fills `WHERE normalized_col IS NULL` silently leaves stale values.
- `UPDATE ... SET x = ?` on a table with `UNIQUE(name, x)` will fail if the target already has that name. When merging parent entities, find-or-merge children first, then reassign remaining, then delete the source parent.

## Playwright / Testing
- `page.goto()` defaults to `wait_until="load"` which waits for ALL resources. Use `wait_until="domcontentloaded"` for heavy image grids.
- `expect()` has its own timeout separate from `page.set_default_timeout()`. Use `expect.set_options(timeout=N)` to configure assertion timeouts globally.
- E2E: when a click triggers an API fetch that updates DOM, use `page.expect_response(lambda r: ...)` as context manager around the click.
- `Path.rglob()` on network volumes (NFS/SMB) is extremely slow. Iterate top-level dirs and rglob within each with an early-exit limit.
- Tests spawning tmux sessions: set `HOME` to a temp dir and `SHELL=/bin/bash` in TestMain to skip shell init files that can block.
- Tests for fire-and-forget launchers (`subprocess.Popen(["open", url])`) must NEVER pass valid URLs — they actually open in the browser. Only test rejection of invalid inputs.
- Pytest `scope="module"` fixtures that bind ports stay alive for the entire module. Standalone tests in the same file that start their own servers MUST use a different port.

## Python stdlib / Runtime
- `os.walk()` on NFS can hang indefinitely. Replace with manual BFS + `os.listdir()` in a daemon thread with `queue.get(timeout=N)`.
- `datetime.fromisoformat()` on RFC3339 returns tz-aware; `datetime.now()` returns naive. Fix at ingestion: `.astimezone().replace(tzinfo=None)`.
- `run` scripts using `os.execv`: MUST compare `Path(sys.executable).resolve()` to the venv python path before re-exec, or infinite loop.
- Optional-module try/except import pattern causes Pyright `union-attr` errors. Fix: `# type: ignore[union-attr]` on each usage site.
- Python 3.14 + Homebrew: `cffi`/`pycparser` may have broken RECORD files. Use `pip install --ignore-installed <pkg>`.
- Wrapper scripts importing from deep internal package paths break on version upgrades. Prefer stable public APIs.
- `\uD83D\uDE80` (UTF-16 surrogate pairs) are invalid in Python UTF-8 strings. Use `\U0001F680`.
- `dict.get("key", "")` returns `None` when key IS present but value is `None`. Use `(d.get("key") or "")`.
- Python 3.14 `sqlite3.Cursor` does NOT support context manager protocol. Use `cur = conn.cursor(); try: ... finally: cur.close()`.
- `asyncio.run()` in sync pytest test with pytest-asyncio active: run in `ThreadPoolExecutor` thread instead.
- Bulk module rename via `.replace("old_name.", "new_name.")` double-hits existing `new_name.old_name.`. Use exact `from old_name`/`import old_name` patterns in a single pass.

## Pyright / Type Checking
- ABC method returning `AsyncIterator[str]` must be `def stream(...)` (NOT `async def`). Subclasses implement as `async def` with `yield`.
- `reportMissingImports` for installed packages: use `# pyright: ignore[reportMissingImports]`. Don't touch `site-packages/`.

## macOS / PyObjC
- `pyobjc-framework-Quartz` is NOT installed by default. Needed for `CGWindowListCopyWindowInfo`.
- PyObjC bridge calls are expensive in bulk. Filter/sort on raw ObjC objects first, convert only the needed subset to Python dataclasses.
- macOS screen sleep/wake: use `NSWorkspace.sharedWorkspace().notificationCenter().addObserverForName_object_queue_usingBlock_("NSWorkspaceScreensDidSleepNotification", ...)`. Requires `pyobjc-framework-Cocoa`.
- EventKit EKEventStore must be a process-level singleton. New instances per poll cycle causes throttling after ~2 minutes.
- PySide6 blocking in QTimer callbacks freezes UI. Use `threading.Thread(daemon=True)` + `Signal(object)`.
- `keyring` in macOS venvs requires `keyrings.alt` backend — otherwise `get_password()` returns None silently.
- Homebrew Python code-signing breakage: `brew upgrade` can leave invalid Team IDs. Fix: `brew reinstall python@3.XX`.

## ML / AI Models
- SadTalker/basicsr on Python 3.12+: `basicsr.data.degradations` imports `torchvision.transforms.functional_tensor` (removed in torchvision 0.25). Patch to `torchvision.transforms.functional`. Also `np.float` removed in numpy 2.x — patch `my_awing_arch.py` to use `float`. And `np.array()` with inhomogeneous shapes in `preprocess.py` — use `.item()` on numpy scalars.
- Neural network integration: always check original training code's input normalization. Mismatched ranges cause brightness/contrast corruption.
- Long-running ML generation: use subprocess-per-chunk for memory isolation. MLX/PyTorch leak memory across iterations. Save `.npy` intermediates for resume.
- HuggingFace MLX Community Whisper repos use `-mlx` suffix. Exception: `whisper-large-v3-turbo` has no suffix.
- fastembed first call per process downloads/loads model (~30s uncached, ~3s cached). Expect stall on first chunk in daemons.
- Chaining large ML model calls: `gc.collect()` between them. Wrap second in retry loop (3 attempts) for transient OOM kills.

## Web / APIs
- Browser `getUserMedia()` requires user gesture context. Web Audio nodes must be stored at module/global scope to avoid GC.
- OAuth token refresh: proactive refresh (10min before expiry), background check (every 5min), minimum refresh interval (30s).
- Wikidata SPARQL: for 100K+ results, request CSV (`Accept: text/csv`) instead of JSON.
- YouTube playlists: yt-dlp `--flat-playlist --dump-json` returns titles — pre-filter before fetching full metadata.
- Open Library author search can return wrong authors. Validate returned name matches searched name.

## Services / Processes
- uvicorn `--reload` on macOS: worker subprocess can hang after hours/days. Never use `--reload` for daemon-managed services.
- JSONL watcher daemons: cap lines processed per file per poll (e.g., 150 max). Store byte offset for resume.
- SSE snapshot-then-live: on reconnect, clear stale data before new snapshot via `reconnecting` callback.
- Process group killing: `os.killpg(os.getpgid(pid), signal)` when `start_new_session=True`. Critical for services with workers.
- Port pre-flight: check availability with socket bind attempt. Include `lsof -i :<port>` in error messages.

## Media (ffmpeg)
- Concatenating audio with `-c copy`: ALL inputs MUST have identical sample rates. Use re-encoding (`-ar <rate>`) if mismatched.
- Floating-point `-ss`/`-t` accumulates drift. Snap to frame boundaries (`round(t * fps) / fps`) and use `-frames:v N`.
- Playwright screenshots of WebAssembly pages: fresh browser per capture, `wait_until="domcontentloaded"` not `"networkidle"`.
