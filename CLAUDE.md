# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repo overview

Qlib (pip name `pyqlib`) is Microsoft's AI-oriented quantitative investment platform. The Python package is `qlib/`. The library is published on PyPI; supported Python is 3.8–3.12.

There is a Cython prerequisite: `qlib/data/_libs/rolling.pyx` and `expanding.pyx` are compiled in-tree (not via `pyproject.toml`). A clean source checkout will not work until those `.so` files exist.

## Setup & common commands

All workflows go through the top-level `Makefile`:

- `make install` — `prerequisite` (build Cython libs) + editable install. Required first step on a fresh clone.
- `make dev` — `prerequisite` + install everything in `[pywinpty,dev,lint,docs,package,test,analysis,rl]`. Use when you need lint/test/docs tooling.
- `make develop` / `make lint` / `make test` / `make rl` — install just one optional extra in editable mode.
- `make clean` — removes built `.so`/`.cpp`, caches, `dist/`, etc. After running this, re-run `make prerequisite` before importing qlib.

Lint (run individually or all via `make lint`):
- `make black` — `black . -l 120 --check --diff` (line length is 120, not 88).
- `make pylint` — runs against `qlib/` and `scripts/`. The Makefile carries a long `--disable=` list; respect those suppressions when adding code rather than re-enabling.
- `make flake8` — uses a custom ignore set (`E501,F541,E266,E402,W503,E731,E203`).
- `make mypy` — `mypy qlib`. Note `mypy<1.5.0` is pinned in `[lint]` extras.
- `make nbqa` — runs black + pylint over the notebooks.

Tests (pytest, configured in `tests/pytest.ini`):
- `cd tests && python -m pytest . -m "not slow"` — what CI runs. The `slow` marker exists; the slow workflow is in `.github/workflows/test_qlib_from_source_slow.yml`.
- `cd tests && python -m pytest test_pit.py` — run one file. Tests depend on data being downloaded — see `scripts/get_data.py qlib_data ...` in the CI workflow for the exact invocation.
- `tests/conftest.py` skips `tests/rl/` on non-Linux platforms.

Running a workflow end-to-end:
- CLI entry point is `qrun` (defined in `qlib/cli/run.py` via the `[project.scripts]` table). It takes a YAML config: `cd examples && qrun benchmarks/LightGBM/workflow_config_lightgbm_Alpha158.yaml`.
- The README warns: do not run `qrun` from a directory that contains a `qlib/` subdir (it will shadow the installed package). The convention in the repo is to `cd examples` first.
- Data must be prepared first. The community drop-in is `wget https://github.com/chenditc/investment_data/releases/latest/download/qlib_bin.tar.gz` extracted to `~/.qlib/qlib_data/cn_data` (the official `python scripts/get_data.py qlib_data ...` route is temporarily disabled per the README).

## Architecture

Qlib is built as loosely-coupled layers. Each can be used standalone but they compose through `qlib.init(...)` + the `qrun` config-driven workflow.

**Data layer (`qlib/data/`)** — the performance-critical core.
- `data.py` defines the `D` provider singleton and `Cal`/`Inst`/`FeatureProvider` interfaces; user code typically calls `D.features(...)`, `D.calendar(...)`, `D.instruments(...)`.
- Storage is pluggable via `storage/` (default `FileCalendarStorage`/`FileInstrumentStorage`/`FileFeatureStorage` over binary files under `~/.qlib/qlib_data/<region>`). The Cython modules in `_libs/` accelerate rolling/expanding windows; cache layers in `cache.py` provide `ExpressionCache` and `DatasetCache` (the `+E +D` columns in the README's perf table).
- `ops.py` is the registry of expression-engine operators (`Ref`, `Mean`, `Std`, etc.) that handlers compose into features.
- `dataset/` contains the higher-level ML-facing API: `DataHandler` and `DataHandlerLP` (the "learn/predict" handler with separate raw/infer/learn DataFrames), `DatasetH`, `loader.py`, and `processor.py` (normalize, fillna, drop-na, etc.). `pit.py` is the point-in-time financial-statement data path.

**Model layer (`qlib/model/`, `qlib/contrib/model/`)**
- `model/base.py` defines `BaseModel` / `Model` (with `fit(dataset)` / `predict(dataset)`). `model/trainer.py` orchestrates training; `task_train` is the unit `qrun` calls.
- All ~20 reference models (LightGBM, XGBoost, LSTM/GRU/ALSTM/GATs/TFT/Transformer/TRA/...) live under `qlib/contrib/model/`. Pytorch variants come in two flavors: plain (`pytorch_gru.py`) and time-series (`pytorch_gru_ts.py`). New models inherit from `Model` and are referenced from YAML configs by import path.

**Workflow & experiment tracking (`qlib/workflow/`)**
- `R` (the `QlibRecorder` singleton in `__init__.py`) is a wrapper over MLflow that exposes a context-manager API (`with R.start(...)`) plus `log_object`/`load_object` for Python objects. See the docstring at the top of `qlib/workflow/__init__.py` for the design rationale vs. raw MLflow.
- `record_temp.py` defines `SignalRecord`, `SigAnaRecord`, `PortAnaRecord` — these are the "record templates" referenced from YAML.
- `workflow/online/` implements online model serving / rolling retraining; `workflow/task/` handles task generation, management, and collection for batch experiment runs.

**Strategy & backtest (`qlib/strategy/`, `qlib/backtest/`)**
- `strategy/base.py` defines `BaseStrategy`, `RLStrategy`, `RLIntStrategy`. Concrete strategies live in `qlib/contrib/strategy/` (`signal_strategy.py`, `rule_strategy.py`, `cost_control.py`).
- `backtest/` provides the `Exchange`, `Executor` (including `NestedExecutor` for multi-level strategies — see `examples/nested_decision_execution/`), `Account`, `Position`, `Report`, and `decision.py` (the `TradeDecision` / `Order` objects strategies emit).

**RL framework (`qlib/rl/`)** — built on tianshou (pinned `<=0.4.10`).
- `simulator.py`, `interpreter.py` (`StateInterpreter`/`ActionInterpreter`), `reward.py`, `trainer/`, plus scenario code in `order_execution/`. Most users get to this via `examples/rl_order_execution/`.

**Contrib (`qlib/contrib/`)** is a deliberate dumping ground for scenario-specific code (models, strategies, rolling/meta-learning, evaluation, reports). It is allowed to depend on the core but core must not depend on it.

## YAML-driven workflow

`qrun` is the most common entry point and the config schema is the de-facto interface to the library. A workflow YAML has top-level `qlib_init`, `market`, `data_handler_config`, then `task:` with `model:`, `dataset:`, `record:`. Everything inside uses `init_instance_by_config` (`qlib/utils/__init__.py`) — `class: ...`, `module_path: ...`, `kwargs: ...`. To add a model/handler/strategy, the typical change is: write the class, then reference it by `module_path` in an example YAML under `examples/benchmarks/<Name>/workflow_config_*.yaml`.

The Jinja2 templating in `qlib/cli/run.py:render_template` lets configs interpolate environment variables — useful for parameterizing market/region in CI runs.

## CI matrix

`.github/workflows/test_qlib_from_source.yml` is the source of truth for "does it work." It tests `{windows-latest, ubuntu-24.04, ubuntu-22.04, macos-14, macos-15} × py{3.8..3.12}`, runs `make dev` → `make black/pylint/flake8/mypy/nbqa` → executes the LightGBM Alpha158 workflow → runs the pytest non-slow set with up to 3 retries. If a CI step fails locally, replicating the exact step from this file is usually faster than guessing.

## Conventions worth noting

- Line length is 120 everywhere (black, pylint, nbqa). Don't reformat to 88.
- Docstrings use NumPy style (see `docs/developer/code_standard_and_dev_guide.rst`).
- `qlib.init(...)` must be called before any `D.*` / `R.*` access; calling it twice with `skip_if_reg=True` is the supported re-entry pattern.
- Pandas `groupby(..., group_key=False)` is set deliberately for pandas 1.5→2.0 compat (see the README "Break change" section). Don't remove `group_key=False`.
- The `pylint` invocation in the Makefile boosts `astroid` recursion to 500 / `sys.setrecursionlimit(2000)`. If you see unexpected pylint recursion errors locally, replicate the `--init-hook` from the Makefile.
