# Plan: Build Orange3 (headless, no-GUI) for Pyodide

**Goal:** `pyodide build` produces a working Orange3 wheel for the Pyodide/WASM runtime,
stripped of all UI components (no `Orange.widgets`, no `Orange.canvas`, no Qt), and the
Orange test suite passes **inside the Pyodide runtime**.

**Branch:** `pyodide` (based on `master`). UI code stays in the repo — it is only
**excluded from the build**, so the branch remains cheaply rebaseable on upstream.

---

## Verified environment (do NOT re-derive; already confirmed 2026-08-05)

**Deliberate choice:** we target **Pyodide 0.29.4**, not the newest 314.x line,
because 0.29.4 ships numpy 2.2.5 and pandas 2.3.3, which fit Orange's existing pins
(`numpy<2.4`, `pandas<3`). Targeting 314.x would force a numpy-2.4/pandas-3
migration of Orange, which is explicitly out of scope. Do not "upgrade" the target.

| Component | State |
|---|---|
| venv | `.venv` (uv), Python **3.13.14** (matches Pyodide 0.29.4's Python 3.13), activate with `source .venv/bin/activate` |
| pyodide-build | 0.39.0 (`pyodide` CLI 0.5.0) |
| xbuildenv | **0.29.4** installed — Python 3.13.2, Emscripten 4.0.9. Do not switch versions. |
| Node | v22.23.2 at `~/.local/bin/node` (needed to run Pyodide for tests) |
| Pyodide lockfile | cached at `/tmp/lock0294.json` (from `https://cdn.jsdelivr.net/pyodide/v0.29.4/full/pyodide-lock.json`) |

### Dependency availability in Pyodide 0.29.4 (verified against the lockfile)

**Packaged in Pyodide** (versions): numpy 2.2.5, scipy 1.14.1, scikit-learn 1.7.0,
pandas 2.3.3, **xgboost 2.1.4**, **bottleneck 1.6.0**, plus joblib, networkx,
httpx, pyyaml, requests, xlrd. All of these satisfy Orange's requirement ranges in
`requirements-core.txt` — no version-cap conflicts.

**Not packaged but pure Python** → installable via micropip/pip from PyPI, no work
needed: python-louvain, keyring, keyrings.alt, baycomp, serverfiles, openpyxl,
xlsxwriter, chardet, catboost is not a dep. (Verify each installs; if any has a
sneaky binary dep, make it optional like openTSNE below.)

**Not packaged, binary** → must become optional: **openTSNE** (used only in
`Orange/projection/manifold.py` in core). Stretch goal: cross-compile it later.

### Version notes

- Orange pins `xgboost<2.1` only on darwin; emscripten matches the un-capped line,
  so the packaged xgboost 2.1.4 is acceptable per the requirements — but verify
  Orange's xgboost tests pass with 2.1.4 (skip-with-reason if it misbehaves in WASM).
- numpy 2.2.5 and pandas 2.3.3 fit Orange's pins; no cap relaxing is needed.

### Known code facts (verified by grep; trust these)

- Core Orange is almost Qt-free. The **only** Qt import outside
  `Orange/widgets`/`Orange/canvas` is `Orange/misc/server_embedder.py` (AnyQt).
- Only two core test files import `Orange.widgets`:
  `Orange/tests/test_discretize.py`, `Orange/tests/sql/test_misc.py`.
- C/Cython extensions built by `setup.py`: `_simple_tree.c` plus `.pyx` modules
  (`classification/_tree_scorers`, `distance/_distance`, `preprocess/_discretize`,
  `preprocess/_relieff`, `data/_valuecount`, `data/_io`, …) via the
  `cythonize(Extension("*", ["Orange/*/*.pyx"]))` glob at `setup.py:473`.
- `setup.py` key sites: `PACKAGES = find_packages(include=("Orange*",))` (line ~190),
  `PACKAGE_DATA` (~194), `ENTRY_POINTS` (~98), `INSTALL_REQUIRES` (~88).
- **First (current) build error:** `Orange/classification/_simple_tree.c` — the
  `QSORT_R_FUNC`/`SORT_CMP_FUNC` macro ladder (lines 59–79) covers
  BSD/Apple, glibc/linux, and MSVC but not `__EMSCRIPTEN__`, so compilation fails
  with "undeclared identifier" errors. Emscripten uses musl ≥1.2.3, which provides
  **GNU-style** `qsort_r`. Fix: add `|| defined __EMSCRIPTEN__` to the GNU branch
  (both the `QSORT_R_FUNC` ladder and any `QSORT_R_STYLE_GNU` selection above it).

---

## Feedback loops (use throughout; never batch many changes between checks)

1. **Build loop** (~minutes): `source .venv/bin/activate && pyodide build 2>&1 | tail -80`.
   Run after every native-code / setup.py change. Wheel lands in `dist/*_wasm32.whl`.
2. **Import smoke loop**: once a wheel exists, create a Pyodide virtualenv **once**:
   `pyodide venv .venv-pyodide` — then after each new wheel:
   `.venv-pyodide/bin/pip install --force-reinstall dist/Orange3-*wasm32*.whl && .venv-pyodide/bin/python -c "import Orange; import Orange.classification, Orange.data, Orange.evaluation; print(Orange.version.version)"`
   (This runs the real WASM interpreter under Node.)
3. **In-Pyodide test loop** (the acceptance gate):
   `.venv-pyodide/bin/pip install pytest && .venv-pyodide/bin/python -m pytest --pyargs Orange.tests -x -q`
   Start with single modules (`Orange.tests.test_table`) before the full suite; the
   WASM runtime is slow, so use `-x` while iterating and the full run as a milestone.
4. **Native regression loop**: the same source must still build/test natively.
   Periodically (at least at each milestone): `python -m pytest Orange/tests -q` in a
   normal (non-pyodide) environment. The pyodide branch must not break native use.

Commit at every green milestone with a message describing the milestone.

---

## Phase 1 — make `pyodide build` succeed

1. Fix `_simple_tree.c` qsort_r ifdefs for `__EMSCRIPTEN__` (see Known facts).
2. Re-run the build loop; fix each next compile error the same way. Expect possible
   issues in other C sources with platform ifdefs; prefer adding an Emscripten branch
   over rewriting. Cython modules should compile as-is once C sources are fixed.
3. **Exclude UI from the wheel.** In `setup.py`, when building for Emscripten (detect
   via env var `ORANGE_NO_GUI=1`, and set it automatically when
   `os.environ.get("_PYTHON_HOST_PLATFORM", "").startswith("emscripten")` or
   `sysconfig.get_platform().startswith("emscripten")`):
   - `find_packages(include=("Orange*",), exclude=("Orange.widgets*", "Orange.canvas*"))`
   - drop widget/canvas entries from `PACKAGE_DATA` and `DATA_FILES`
   - drop GUI entry points (`orange-canvas` / `Orange.canvas` gui_scripts)
   - the wheel must NOT contain `Orange/widgets` or `Orange/canvas` (verify with
     `unzip -l dist/*.whl | grep -c widgets` → 0).
4. **Requirements for emscripten.** Adjust `INSTALL_REQUIRES` (sourced from
   `requirements-core.txt`) with environment markers rather than deletions, e.g.:
   - `openTSNE ... ; platform_system != "Emscripten"` (or `sys_platform != "emscripten"` — check which marker the Pyodide resolver honors; Pyodide sets `sys_platform == "emscripten"`)
   - `keyring`, `keyrings.alt`, `serverfiles`, `baycomp`: leave — pure Python, they
     install fine; only mark them optional if they turn out to pull binary deps.
5. Milestone P1: `pyodide build` exits 0 and produces a wasm32 wheel without
   widgets/canvas. Commit.

## Phase 2 — Orange imports and runs in the Pyodide runtime

1. Create `.venv-pyodide` via `pyodide venv`; install the wheel (loop 2). pip inside a
   Pyodide venv resolves Pyodide-packaged deps from the lockfile and pure wheels from
   PyPI automatically.
2. Fix import-time failures:
   - `Orange/projection/manifold.py`: make the openTSNE import lazy — import inside
     the TSNE class/function and raise a clear `ImportError("openTSNE is not
     available in this environment")` on use. Keep module import safe.
   - `Orange/misc/server_embedder.py`: must not be imported at package import time
     (verify); it stays in the wheel but importing it may fail — that is acceptable,
     it is not part of the headless surface.
   - Anything touching `multiprocessing`: Pyodide has no processes and (by default)
     no threads for compute. joblib works in sequential mode; force/verify
     `n_jobs=1` paths. Fix failures as they appear, guarded by
     `sys.platform == "emscripten"` where behavior must differ.
3. Smoke script must pass: load `iris` via `Orange.data.Table("iris")` (bundled
   dataset — no network), fit `LogisticRegressionLearner`, run `CrossValidation`.
4. Milestone P2: smoke script green inside `.venv-pyodide`. Commit.

## Phase 3 — test suite passes inside Pyodide

1. Run the suite per loop 3, module by module first, then full
   `pytest --pyargs Orange.tests`.
2. Expected classes of failures and the intended handling:
   - **Widgets-importing tests** (`test_discretize.py`, `sql/test_misc.py`): make the
     widget-dependent parts import-guarded/skipped when `Orange.widgets` is absent
     (try/except ImportError + skip), NOT deleted.
   - **Network-dependent tests** (serverfiles/datasets/urls): skip on emscripten with
     `@unittest.skipIf(sys.platform == "emscripten", "no network in wasm CI")`.
   - **Multiprocessing/threading tests**: same skip pattern.
   - **openTSNE tests**: skip when openTSNE is unavailable (skip on ImportError, not
     on platform — so they'd auto-enable if openTSNE is ever ported).
   - Genuine WASM behavior differences (float rounding, missing OS APIs): fix
     code where reasonable, otherwise platform-skip with a comment explaining why.
3. Rule: every skip must carry a reason string; keep a running count. The final
   report must list all skipped tests grouped by reason.
4. Milestone P3: full `Orange.tests` suite green in Pyodide (failures = 0,
   errors = 0, documented skips allowed). Commit.

## Phase 4 — wrap-up

1. Re-run the native regression loop (build + full native test suite) — must be green.
2. Write `README-pyodide.md`: how to build the wheel, create the venv, run tests, and
   a minimal "use Orange in the browser via micropip" snippet; list the feature
   deltas (no widgets/canvas, no openTSNE, no multiprocessing, network-dependent
   dataset fetching limited).
3. Final commit; summarize: what was changed, what is skipped and why, wheel size,
   test counts.

---

## Guardrails

- Never modify code under `Orange/widgets/` or `Orange/canvas/` (they're excluded,
  not maintained here). Test files are the only exception (skip guards).
- Keep every platform-specific change behind `__EMSCRIPTEN__` (C) or
  `sys.platform == "emscripten"` / env-var (Python) so native builds are unaffected.
- Prefer environment markers over deleting requirements.
- Don't upgrade/downgrade the toolchain (pyodide-build 0.39, xbuildenv 0.29.4,
  Python 3.13 venv) without strong evidence the toolchain itself is at fault. In
  particular, do NOT move to the 314.x Pyodide line — that pulls in numpy 2.4 and
  pandas 3, which is out of scope.
- If a single test module hangs in WASM, note it, skip it at the runner level, and
  move on — do not burn hours on one hang before the rest of the suite is green.
- Commit at milestones; small fix commits in between are fine. Do not push.
