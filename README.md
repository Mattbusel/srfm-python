# srfm-python

Pure-Python SDK for **Special Relativity in Financial Modeling (SRFM)**: compute price velocity (β), Lorentz factors (γ), spacetime intervals and a geodesic-deviation signal on OHLCV bars, straight from a pandas or Polars DataFrame.

SRFM treats each price bar as an event in a Minkowski-like spacetime and asks whether relativistic quantities are useful features for market data. The reference implementation is C++20; this package ports the core pipeline to NumPy so you can experiment in a notebook without building anything. Only NumPy and pandas are required.

> Research code. SRFM is an experimental modeling idea, the outputs are features to study, not trading signals with demonstrated value, and nothing here is financial advice.

## Install

The package is not on PyPI (the PyPI name `srfm` belongs to an unrelated physics project). Install from GitHub:

```bash
pip install "git+https://github.com/Mattbusel/srfm-python"
```

For development:

```bash
git clone https://github.com/Mattbusel/srfm-python
cd srfm-python
pip install -e ".[dev]"          # pytest, pytest-cov, polars
pip install -e ".[polars]"       # optional Polars support only
```

Requires Python 3.10+.

## Quickstart

```python
import pandas as pd
import srfm  # importing registers the df.srfm accessor

df = pd.read_csv("ohlcv.csv")   # needs open, high, low, close, volume (any case)

beta     = df.srfm.beta()                 # pd.Series of β, clamped to (-0.9999, 0.9999)
gamma    = df.srfm.gamma()                # pd.Series of γ = 1/sqrt(1 - β²), always >= 1
geodesic = df.srfm.geodesic(window=20)    # geodesic deviation signal
interval = df.srfm.spacetime_interval()   # scalar ds² over the series

enriched = df.srfm.run()                  # full pipeline
print(enriched[["beta", "gamma", "relativistic_return", "geodesic_signal"]].tail())
```

`run()` adds these columns: `log_return`, `beta`, `gamma`, `relativistic_return`, `volume_norm`, `hl_range_norm`, `spacetime_interval`, `geodesic_signal`, `geodesic_deviation`.

The same pipeline without the accessor, with its parameters exposed:

```python
from srfm import SRFMEngine, BetaVelocity, LorentzFactor

engine = SRFMEngine(max_velocity=1.0, effective_mass=1.0, c_market=1.0, rolling_window=20)
result = engine.run(df)

LorentzFactor.from_beta(BetaVelocity(0.6))   # LorentzFactor(value=1.25)
```

### Polars

```python
import polars as pl
from srfm.polars_ext import SRFMPolars

enriched = SRFMPolars(pl.read_csv("ohlcv.csv")).run()
```

## Pipeline

1. OHLCV to log returns: `r = ln(close_t / close_{t-1})`
2. Returns to β: `β = r / (max_velocity · rolling_max|r|)`, clamped to `BETA_MAX` (0.9999)
3. β to γ: `γ = 1/sqrt(1 - β²)`
4. Relativistic return: `γ · m_eff · r`
5. Spacetime interval: `ds² = -(c·dt)² + dr₁² + dr₂² + dr₃²` (return, normalized volume, normalized high-low range)
6. Geodesic signal: `(Δγ/γ) · sign(β)`

## Modules

| Module | Contents |
|---|---|
| `srfm/core.py` | `BetaVelocity`, `LorentzFactor`, `RelativisticSignal`, vectorized `compute_beta_array` / `compute_gamma_array` |
| `srfm/engine.py` | `SRFMEngine`: batch pipeline over a DataFrame, plus `run_bar` for one bar at a time |
| `srfm/manifold.py` | `SpacetimeManifold`: 4x4 metric tensor, spacetime interval, Christoffel symbols |
| `srfm/geodesic.py` | `GeodesicSignal`: rolling geodesic deviation |
| `srfm/pandas_ext.py` | the `df.srfm` accessor |
| `srfm/polars_ext.py` | `SRFMPolars` wrapper |

## Tests

```bash
python -m pytest tests/ -v    # 131 tests
```

## The SRFM project family

SRFM (Special Relativity in Financial Modeling) is split across four repositories:

| Repository | What it is |
|---|---|
| [Special-Relativity-in-Financial-Modeling](https://github.com/Mattbusel/Special-Relativity-in-Financial-Modeling) | C++20 core implementation: price velocity (beta), Lorentz factor (gamma), spacetime interval classification, Christoffel symbols and geodesic deviation on OHLCV bars, plus Python validation scripts |
| [srfm-paper-impl](https://github.com/Mattbusel/srfm-paper-impl) | The paper (PDF), scripts and a notebook that regenerate its figures, and a small dependency-free Rust reference implementation of the core formulas |
| **srfm-python** (this repo) | Pure-Python SDK: a pandas `df.srfm` accessor and a Polars wrapper for the Lorentz-factor pipeline |
| [srfm-lab](https://github.com/Mattbusel/srfm-lab) | Large multi-language research lab that builds trading research on the idea: the black-hole (BH) physics signal, an idea automation engine, backtesting and paper trading |

The Rust crate [fin-stream](https://github.com/Mattbusel/fin-stream) also ships a streaming `lorentz` module built on the same transform.

The formulas here follow the C++ implementation and the paper in srfm-paper-impl.
