# Orange3 for Pyodide (headless)

This branch builds a **headless** Orange3 wheel for the
[Pyodide](https://pyodide.org) / WebAssembly runtime. The wheel contains the
complete data-mining core (data, preprocess, classification, regression,
clustering, distance, projection, evaluation, modelling) but **no GUI**:
`Orange.widgets` and `Orange.canvas` are excluded from the build.

Target: **Pyodide 0.29.4** (Python 3.13, Emscripten 4.0.9). This version ships
numpy 2.2.5 / pandas 2.3.3 / scikit-learn 1.7.0 / scipy 1.14.1 /
xgboost 2.1.4 / bottleneck 1.6.0, all inside Orange's requirement ranges.

## Building the wheel

Requires a native Python 3.13 environment (matching Pyodide 0.29.4's
interpreter) and Node.js (to run the wasm interpreter for tests).

```bash
# one-time setup
uv venv .venv --python 3.13
source .venv/bin/activate
uv pip install "pyodide-build==0.39.0"
pyodide xbuildenv install 0.29.4

# build (produces dist/orange3-*-wasm32.whl)
pyodide build
```

The headless mode is selected automatically when cross-compiling for
Emscripten; it can also be forced for a native build with `ORANGE_NO_GUI=1`.

## Running tests in the Pyodide runtime

```bash
pyodide venv .venv-pyodide          # creates a wasm venv running under Node
.venv-pyodide/bin/pip install dist/orange3-*wasm32*.whl
# run outside the repo root so the source tree does not shadow the wheel
cd /tmp && /path/to/.venv-pyodide/bin/python -m unittest Orange.tests
```

The full core suite (`Orange.tests`, which also discovers
`Orange/*/tests` except widgets — same as upstream CI) passes in the wasm
runtime: 1840 tests, 0 failures, 0 errors (246 skips, see below).
Note: upstream runs the suite with `unittest`; running it with `pytest`
additionally collects a few helper functions/classes whose names start with
`test` (e.g. `Orange.tests.test_filename`, `TestTree` in
`test_orangetree.py`) and reports spurious errors for them — that happens on
any platform and is unrelated to this port.

## Using Orange in the browser

```html
<script src="https://cdn.jsdelivr.net/pyodide/v0.29.4/full/pyodide.js"></script>
<script type="module">
  const pyodide = await loadPyodide();
  await pyodide.loadPackage(["numpy", "scipy", "scikit-learn", "pandas",
                             "xgboost", "bottleneck", "micropip"]);
  await pyodide.runPythonAsync(`
      import micropip
      await micropip.install("<URL-of-your>/orange3-...-wasm32.whl")

      from Orange.data import Table
      from Orange.classification import LogisticRegressionLearner
      from Orange.evaluation import CrossValidation, CA

      data = Table("iris")                      # bundled dataset, no network
      res = CrossValidation(k=5)(data, [LogisticRegressionLearner()])
      print("CA:", CA(res)[0])
  `);
</script>
```

## Differences from the native build

| Area | Status in the wasm wheel |
|---|---|
| `Orange.widgets`, `Orange.canvas` | not included (headless build) |
| GUI entry points (`orange-canvas`) | not included |
| openTSNE (`Orange.projection.manifold.TSNE`) | not installed (no wasm wheel); module imports fine, using TSNE raises `ImportError` |
| python-louvain (`Orange.clustering.louvain.Louvain`) | not installed (sdist-only on PyPI); module imports fine, fitting raises `ImportError` |
| serverfiles / baycomp | not installed (sdist-only on PyPI); unused by the core library |
| multiprocessing / threads | unavailable in the wasm runtime; joblib and sklearn run sequentially (`n_jobs=1`); `Table` concurrency tests are skipped |
| network access | not available from the wasm venv; URL readers and dataset downloading are skipped/unusable, bundled datasets (`iris`, `titanic`, …) work |
| xgboost sparse predict | crashes the wasm runtime (fatal Emscripten ffi error); the two `test_predict_sparse` tests are skipped on emscripten — dense xgboost works |
| trubar localization | the multilingual install step is skipped in headless builds (it would inject `orangecanvas` imports) |

## Skipped tests, grouped by reason

Exact breakdown of the 246 skips in the green wasm run
(`Ran 1840 tests ... OK (skipped=246, expected failures=1)`):

| count | reason | kind |
|---|---|---|
| 122 | No connection provided for postgres | pre-existing, also skips natively |
| 52 | No connection provided for mssql | pre-existing, also skips natively |
| 13 | python-louvain is not installed | skip on ImportError (sdist-only on PyPI) |
| 8 | openTSNE is not installed | skip on ImportError (no wasm wheel; auto-enables if one appears) |
| 8 | Base class | pre-existing |
| 6 | no network in wasm runtime | platform skip (`test_url_reader.py`) |
| 2 | xgboost sparse predict fatally crashes the wasm runtime | platform skip |
| 1 | wasm runtime cannot spawn threads | platform skip (`ConcurrencyTests`) |
| 1 | Orange.widgets is not available | skip on ImportError (`test_discretize.py`) |
| 1 | Orange.canvas is not available (headless build) | skip on ImportError (`test_util.py`) |
| 4 | misc pre-existing skips (slow tests, deprecation-flag tests, …) | pre-existing |

Additionally `Orange/misc/tests/test_embedder_utils.py` and
`test_server_embedder.py` skip at module level when their Qt-dependent
imports fail (they also skip natively in a Qt-less environment; they still
run in upstream CI, which installs PyQt).
