# Orange3 → Pyodide: what was changed and why

Everything lives on the `pyodide` branch (4 commits on top of `master`,
~425 lines added / 37 removed across 19 files). The result is a 1.2 MB
headless `orange3-*-wasm32.whl` whose full core test suite passes inside the
Pyodide runtime: **1840 tests, 0 failures, 0 errors, 246 documented skips**.

Design rule followed throughout: every platform-specific change is behind
`__EMSCRIPTEN__` (C), `sys_platform == "emscripten"` markers, or the
`NO_GUI` build flag — native builds and native test runs are unaffected.
`Orange/widgets` and `Orange/canvas` were never modified; they are only
excluded from the wasm build.

---

## 1. Making the wheel build (`pyodide build`)

**`Orange/classification/_simple_tree.c`** — the only true C-code fix.
The `qsort_r` ifdef ladder covered BSD/Apple, glibc/Linux, and MSVC but not
Emscripten, so compilation failed. Emscripten's musl (≥ 1.2.3) provides
GNU-style `qsort_r`, so `__EMSCRIPTEN__` was added to the GNU branch. One line.

**`setup.py`** — a new headless mode, `NO_GUI`. It turns on automatically
when cross-compiling for Emscripten (detected via `_PYTHON_HOST_PLATFORM` /
`sysconfig.get_platform()`) or explicitly with `ORANGE_NO_GUI=1`. When active:

- `find_packages` excludes `Orange.widgets*` and `Orange.canvas*`
- widget/canvas entries are dropped from `PACKAGE_DATA`
- GUI entry points are dropped (`orange-canvas`, widget/drop-handler registrations)
- the `Orange.widgets.utils._grid_density` C++ extension is not built
- `requirements-gui.txt` is not merged into `install_requires`
- the **trubar localization install step is skipped**. This was a hidden
  trap: `bdist_wheel` runs the `install` cmdclass, and Orange's
  `InstallMultilingualCommand` rewrites every module to start with
  `from orangecanvas.localization import ...` — silently re-introducing a
  GUI dependency into a supposedly headless wheel.

**`requirements-core.txt`** — four dependencies got
`; sys_platform != "emscripten"` markers instead of being removed:

| package | why it can't install in Pyodide |
|---|---|
| openTSNE | binary package, no wasm wheel exists |
| python-louvain | only an sdist on PyPI; Pyodide's pip installs wheels only |
| serverfiles | same (sdist-only) |
| baycomp | same (sdist-only) |

serverfiles and baycomp are not used anywhere in core Orange, so markers
sufficed. The other two needed code changes (next section).

## 2. Making Orange import and run in the wasm runtime

**`Orange/projection/manifold.py`** — openTSNE already had a lazy-import
proxy, but three *type annotations* (`embedding: openTSNE.TSNEEmbedding`, …)
were evaluated at class-creation time and triggered the import anyway.
Adding `from __future__ import annotations` makes annotations lazy; using
TSNE without openTSNE now raises a clear
`ImportError("openTSNE is not available in this environment")`.

**`Orange/clustering/louvain.py`** — `from community import best_partition`
moved into a try/except; the module stays importable and only
`Louvain(...).fit_graph()` raises `ImportError` when python-louvain is absent.

**`Orange/__init__.py`** — `'tree'` and `'utils'` added to the lazy-module
list. `test_orange_has_modules` asserts every Orange submodule is reachable
as an attribute; natively that held only through import side effects of
*other* modules, which don't occur the same way in wasm. Now it's
deterministic on every platform.

## 3. Making the test suite green in wasm

Runner note: upstream CI runs `python -m unittest Orange.tests` — that was
used as the acceptance gate. (pytest additionally collects helper names like
`Orange.tests.test_filename` and the `TestTree` mixin and reports ~40
spurious errors on any platform, wasm or not.)

Skips added, each with a reason string (11 files touched, all test files):

- **No threads in wasm** — `test_table.py::ConcurrencyTests`
- **No network in wasm** — `test_url_reader.py` (whole module)
- **openTSNE missing** — `test_manifold.py::TestTSNE` skips on *ImportError*,
  not platform, so it auto-re-enables if openTSNE ever gets a wasm wheel
- **python-louvain missing** — `test_clustering_louvain.py`, same pattern
- **Qt/GUI packages missing** — `test_discretize.py` (fallback
  `table_dense_sparse` decorator), `test_util.py::test_get_entry_point`,
  `misc/tests/test_embedder_utils.py`, `misc/tests/test_server_embedder.py`
  (module-level `SkipTest`); these also skip gracefully in any Qt-less
  native environment where they previously *errored*
- **xgboost sparse predict** — `test_predict_sparse` in
  `classification/tests/test_xgb_cls.py` and
  `regression/tests/test_xgb_reg.py`: platform-skipped because sparse
  `DMatrix` prediction **fatally crashes** the wasm runtime
  (`Cannot convert 4 to a BigInt` in Emscripten's ffi) — dense xgboost works

Two genuine test-robustness fixes (not skips):

- `test_knn.py::test_random` drew from the *global* numpy RNG, so the sample
  depended on which tests ran before it; on wasm it landed at chance level
  (CA 0.086 vs the asserted > 0.1). It now seeds explicitly.
- `test_manifold.py::test_mds_pca_init` compared components that are exactly
  0 with `rtol` only; wasm produces ~1e-11 rounding noise there, which is an
  infinite relative error. Added `atol=1e-8`.

## 4. Not a code change, but worth knowing

- **Stale Cython output poisons the wasm build.** The gitignored generated
  `.c`/`.cpp` files in-tree get regenerated by whatever native Cython you
  last ran; output from Cython 3.2.9 doesn't compile against Pyodide 0.29.4's
  numpy 2.2.5 headers. Deleting the generated files lets the isolated build
  env regenerate them consistently.
- **Run the wasm Python outside the repo root**, otherwise the source tree
  shadows the installed wheel and imports fail on the missing compiled
  extensions.
- **Toolchain pinned deliberately**: Pyodide 0.29.4 (Python 3.13,
  Emscripten 4.0.9) because its numpy 2.2.5 / pandas 2.3.3 fit Orange's pins;
  the newer 314.x line would force a numpy 2.4 / pandas 3 migration.

## Commits

| commit | milestone |
|---|---|
| `3590123ab` | P1 — `pyodide build` produces a wasm32 wheel with no widgets/canvas |
| `6bf97ef44` | P2 — Orange imports and runs in Pyodide (iris + LR + 5-fold CV smoke test) |
| `858b0e0a2` | P3 — full core suite green in Pyodide (1840 tests, 0 fail / 0 err) |
| `2d439c65a` | P4 — `README-pyodide.md` (build/test/browser instructions, skip report) |

`demo.html` (repo root, uncommitted) is a self-contained browser demo:
serve the repo with any static server, open it, click — it loads Pyodide
from the CDN, installs the wheel via micropip, and cross-validates on iris.
