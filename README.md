<div align="center">

# PROMETHEUS CAPITAL
### Risk Platform — Technical Guide
**QuantConnect / LEAN Implementation**

![Version](https://img.shields.io/badge/version-1.0-gold)
![Platform](https://img.shields.io/badge/platform-QuantConnect-blue)
![Language](https://img.shields.io/badge/language-Python-yellow)
![Status](https://img.shields.io/badge/status-Internal%20Confidential-red)

*For: Quantitative Researchers & Quantitative Analysts*

</div>

---

## Table of Contents

- [Overview](#overview)
- [Architecture — The Seven Multiplicative Layers](#architecture--the-seven-multiplicative-layers)
  - [Layer 1 — Alpha Signal](#layer-1--alpha-signal)
  - [Layer 2 — Equal Risk Contribution Weights](#layer-2--equal-risk-contribution-weights)
  - [Layer 3 — Volatility Targeting](#layer-3--volatility-targeting)
  - [Layer 4 — VIX Risk Profile](#layer-4--vix-risk-profile)
  - [Layer 5 — Correlation Limiter](#layer-5--correlation-limiter)
  - [Layer 6 — Position Cap](#layer-6--position-cap)
  - [Layer 7 — Drawdown Kill Switch](#layer-7--drawdown-kill-switch)
  - [Combined Formula](#combined-position-sizing-formula)
- [Asset Universe](#asset-universe)
- [Guide for Quantitative Researchers](#guide-for-quantitative-researchers)
  - [Pre-Deployment Checklist](#pre-deployment-checklist)
  - [Standard Risk Metrics Template](#standard-risk-metrics-template)
  - [Stress Testing Protocol](#stress-testing-protocol)
  - [Modifying Risk Parameters](#modifying-risk-parameters)
  - [Adding New Assets](#adding-new-assets)
- [Guide for Quantitative Analysts](#guide-for-quantitative-analysts)
  - [Daily Monitoring Checklist](#daily-monitoring-checklist)
  - [Dashboard Reference](#dashboard-reference)
  - [Interpreting Log Messages](#interpreting-log-messages)
  - [Escalation Protocol](#escalation-protocol)
- [Governance](#governance)
  - [Capital Tier System](#capital-tier-system)
  - [Risk Review Committee](#risk-review-committee)
  - [Prohibited Actions](#prohibited-actions)
- [QuantConnect Workflow](#quantconnect-workflow)

---

## Overview

The Prometheus Risk Platform is a systematic, institutional-grade portfolio management framework built entirely within QuantConnect (LEAN). It governs every strategy deployed by Prometheus Capital through a series of **multiplicative risk layers** applied before any order reaches the market.

### Design Philosophy

| Principle | Description |
|-----------|-------------|
| **Risk first, alpha second** | No signal reaches the market without passing through all risk layers |
| **Multiplicative scaling** | Each layer compounds the previous — a position survives only if all layers permit it |
| **No discretion at execution** | Kill switches and position caps are fully systematic — no human override permitted post-deployment without a governance review |

---

## Architecture — The Seven Multiplicative Layers

Every order passes through the following stack in sequence. Each layer produces a scalar that multiplies the previous output.

```
┌─────────────────────────────────────────────────┐
│       Layer 1 — Alpha Signal (SMA Crossover)    │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│    Layer 2 — Equal Risk Contribution Weights    │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│          Layer 3 — Volatility Targeting         │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│       Layer 4 — VIX Risk Profile                │
│         (Regime Classification + Beta)          │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│          Layer 5 — Correlation Limiter          │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│       Layer 6 — Position Cap (Hard 20%)         │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│  ⛔ Layer 7 — Drawdown Kill Switch (Hard 20%)   │
└─────────────────────────────────────────────────┘
```

---

### Layer 1 — Alpha Signal

The entry gate. An asset is only allocated to if its **50-day SMA is above its 200-day SMA**. Assets failing this condition are fully liquidated.

```
signal(s) = 1   if SMA_50(s) > SMA_200(s)
            0   otherwise → asset liquidated
```

---

### Layer 2 — Equal Risk Contribution Weights

Capital is allocated **inversely proportional to each asset's realised volatility** so that every active asset contributes equal risk to the portfolio.

```
w_i = (1 / σ_i) / Σ(1 / σ_j)   for all j in active universe
```

Where `σ_i` is the 30-day rolling realised standard deviation of daily returns.

---

### Layer 3 — Volatility Targeting

Each position is scaled so its expected daily volatility contribution matches a fixed target of **2%**. Assets running hotter than the target are scaled down.

```
vol_scale_i = σ_target / σ_i     where σ_target = 0.02
```

---

### Layer 4 — VIX Risk Profile

The most dynamic layer — operates in two sub-components.

#### 4a — Regime Classification

| Regime | VIX Range | Portfolio Scalar | Behaviour |
|--------|-----------|-----------------|-----------|
| `LOW` | < 15 | `1.00` | Full risk-on |
| `NORMAL` | 15–25 | `0.80` | Mild reduction |
| `ELEVATED` | 25–35 | `0.50` | Significant compression |
| `CRISIS` | > 35 | `0.25` | Capital preservation mode |

#### 4b — Per-Asset VIX Beta

Each asset's **60-day rolling beta to VIX changes** is computed:

```
β_vix_i = Cov(r_i, ΔVIX) / Var(ΔVIX)
```

Exposure scalars are then derived:

```
beta_scalar_i = max(0.10, 1 - β_vix_i / β_clip)     if β_vix_i >= 0  (risky asset, cut exposure)
                min(1.00, 1 + 0.25 * |β_vix_i|)      if β_vix_i <  0  (safe haven, preserve/boost)

vix_scale_i = clip(beta_scalar_i × regime_scalar, 0.05, 1.0)
```

> **Positive beta** (e.g. XLE, EEM) → asset rises with vol spikes → exposure cut
> **Negative beta** (e.g. TLT, GLD) → safe haven behaviour → exposure preserved or gently boosted

---

### Layer 5 — Correlation Limiter

Active asset pairs with a **30-day rolling return correlation above 0.85** are flagged. The secondary asset receives a **50% haircut** on its combined scale factor, preventing factor crowding.

```
corr_scale_i = 0.5   if ρ(r_i, r_j) > 0.85 for any j
               1.0   otherwise
```

---

### Layer 6 — Position Cap

No single position may exceed **20% of total portfolio value**. Hard ceiling, unconditional.

```
Q_final_i = sign(Q_raw_i) × min(|Q_raw_i|, 0.20 × NAV / P_i)
```

---

### Layer 7 — Drawdown Kill Switch

> ⛔ **If portfolio drawdown from peak exceeds 20%, all positions are immediately liquidated. Unconditional. Automated. No override.**

```python
if (NAV - Peak_NAV) / Peak_NAV < -0.20:
    algorithm.Liquidate()
    return []
```

---

### Combined Position Sizing Formula

The final quantity for each asset is the product of all applicable scalars:

```
Q_i = clip(
    w_i × vol_scale_i × vix_scale_i × corr_scale_i × (NAV / P_i),
    0,
    0.20 × NAV / P_i
)
```

---

## Asset Universe

| Ticker | Type | Role |
|--------|------|------|
| `SPY` | US Equity ETF | S&P 500 broad market |
| `QQQ` | US Equity ETF | Nasdaq-100 growth |
| `IWM` | US Equity ETF | Russell 2000 small-cap |
| `TLT` | Bond ETF | Long-duration US Treasuries (safe haven) |
| `GLD` | Commodity ETF | Gold (safe haven, negative VIX beta) |
| `EEM` | EM ETF | Global emerging market equity |
| `XLF` | Sector ETF | US Financials |
| `XLE` | Sector ETF | US Energy |

> **Note on Forex:** EURUSD and GBPUSD are currently excluded. Interactive Brokers enforces minimum notional sizes of EUR 20,000 and GBP 5,000,000. After VIX and vol scalars compress forex positions in elevated regimes, orders fall below these minimums and are rejected. Re-integration requires a dedicated forex account or custom brokerage model.

---

## Guide for Quantitative Researchers

### Pre-Deployment Checklist

Every strategy must satisfy **all** conditions below before deployment. Strategies failing any single criterion are returned to research without exception.

| Metric | Requirement | Pass |
|--------|-------------|------|
| Sharpe Ratio | ≥ 1.0 annualised over full backtest | ☐ |
| Sortino Ratio | ≥ 1.2 | ☐ |
| Max Drawdown | < 20% in backtest | ☐ |
| Calmar Ratio | ≥ 0.5 | ☐ |
| Annual Return | Positive over full period | ☐ |
| Return Skew | > −1.5 | ☐ |
| Worst 5-day Return | Documented and explained | ☐ |
| COVID Stress Test | Feb–Apr 2020 performance reviewed | ☐ |
| 2022 Stress Test | Jan–Dec 2022 performance reviewed | ☐ |
| VIX Regime Test | Performance across all four regimes reviewed | ☐ |

---

### Standard Risk Metrics Template

Paste into a QuantConnect Research notebook. Run on your strategy's daily returns before any submission.

```python
import numpy as np
import pandas as pd

def prometheus_risk_report(returns: pd.Series) -> dict:
    r = returns.dropna()
    cum = (1 + r).cumprod()
    peak = cum.cummax()
    dd = (cum - peak) / peak

    annual_ret = (1 + r.mean()) ** 252 - 1
    annual_vol = r.std() * np.sqrt(252)
    sharpe     = annual_ret / annual_vol
    downside   = r[r < 0].std() * np.sqrt(252)
    sortino    = annual_ret / downside if downside > 0 else np.nan
    max_dd     = dd.min()
    calmar     = annual_ret / abs(max_dd) if max_dd != 0 else np.nan
    var_95     = np.percentile(r, 5)
    cvar_95    = r[r <= var_95].mean()
    worst_5d   = r.rolling(5).sum().min()

    return {
        "annual_return":   round(annual_ret, 4),
        "annual_vol":      round(annual_vol, 4),
        "sharpe":          round(sharpe, 3),
        "sortino":         round(sortino, 3),
        "max_drawdown":    round(max_dd, 4),
        "calmar":          round(calmar, 3),
        "skew":            round(float(r.skew()), 3),
        "kurtosis":        round(float(r.kurt()), 3),
        "var_95":          round(var_95, 4),
        "cvar_95":         round(cvar_95, 4),
        "worst_5d_return": round(worst_5d, 4),
    }

report = prometheus_risk_report(strategy_returns)
for k, v in report.items():
    print(f"{k:25s}: {v}")
```

---

### Stress Testing Protocol

Slice the strategy returns across each window and run `prometheus_risk_report` on every sub-period independently.

| Period | Date Range | What to Look For |
|--------|------------|-----------------|
| COVID Crash | Feb 20 – Apr 3, 2020 | Max drawdown, recovery speed |
| Post-COVID Rally | Apr 2020 – Dec 2021 | Sharpe in sustained uptrend |
| 2022 Tightening | Jan – Dec 2022 | Performance in rising rate environment |
| VIX LOW Regime | Days where VIX < 15 | Alpha capture in complacent markets |
| VIX CRISIS Regime | Days where VIX > 35 | Drawdown containment under kill switch pressure |

---

### Modifying Risk Parameters

> ⚠️ Any change to the parameters below requires a full backtest and written Risk Review Committee approval before going to production.

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| `max_drawdown` | `0.20` | 0.10–0.30 | Portfolio kill switch threshold |
| `max_position_pct` | `0.20` | 0.05–0.25 | Hard position ceiling per asset |
| `vol_target` | `0.02` | 0.01–0.04 | Target daily vol per asset |
| `correlation_limit` | `0.85` | 0.70–0.95 | Crowding detection threshold |
| `vol_lookback` | `30` | 20–60 | Days for rolling vol computation |
| `vix_lookback` | `60` | 30–90 | Days for VIX beta computation |
| `beta_clip` | `2.0` | 1.0–3.0 | Max absolute VIX beta clamp |

> ⛔ Reducing `max_drawdown` below `0.10` will cause frequent liquidations in normal volatility. Increasing above `0.25` materially weakens capital protection. Both require CIO sign-off.

---

### Adding New Assets

1. Add the ticker to `EQUITY_UNIVERSE` in `PrometheusFundAllocator`
2. Add corresponding `SMA` indicators in `Initialize()`
3. Run a full backtest from 2018 to present
4. Confirm the asset has at least **60 trading days** of return history for VIX beta computation to be valid
5. Submit backtest results and stress test report to the Risk Review Committee

---

## Guide for Quantitative Analysts

### Daily Monitoring Checklist

| Dashboard Panel | What to Check |
|-----------------|--------------|
| `Prometheus \| Fund / Portfolio Value` | NAV tracking expected trajectory |
| `Prometheus \| Risk / Drawdown %` | Has not approached −15% |
| `Prometheus \| VIX / VIX Level` | Current VIX vs prior day |
| `Prometheus \| VIX / Regime (1–4)` | Consistent with market conditions |
| `Prometheus \| Risk / Avg Asset Vol` | No anomalous vol spikes |
| Algorithm Logs | No `[TAIL ALERT]` or `[PROMETHEUS] DRAWDOWN` messages |

---

### Dashboard Reference

| Chart | Description | Escalate If |
|-------|-------------|-------------|
| Portfolio Value | Absolute NAV over time | Sharp decline > 5% in one week |
| Drawdown % | Drawdown from all-time peak | Approaches −15% |
| VIX Level | Raw VIX index close | Spike > 10 points overnight |
| Regime (1–4) | 1=Low, 2=Normal, 3=Elevated, 4=Crisis | Moves to 3 or 4 |
| Avg Asset Vol | Mean 30-day vol across universe | Doubles from prior month |

---

### Interpreting Log Messages

**Per-asset VIX log** — emitted every rebalance day:
```
[VIX] SPY beta=0.421 scalar=0.589 regime=ELEVATED vix=28.3
```
SPY has a positive VIX beta of 0.421. In the current ELEVATED regime its exposure scalar is 0.589 — it is trading at ~59% of its base-weighted allocation. Expected and correct behaviour.

---

**Soft alert** — escalate immediately:
```
[TAIL ALERT] Drawdown -15.42% | VIX=38.1 | Regime=CRISIS
```
Portfolio has breached the −15% soft alert. System still running (kill switch at −20%) but all regime scalars are already compressed to 0.25. **Immediate escalation to Risk Review Committee required.**

---

**Kill switch** — full post-mortem required before any redeployment:
```
[PROMETHEUS] DRAWDOWN -20.03% --- LIQUIDATING
```
All positions liquidated. No further orders will be placed. Quant Res must conduct a full post-mortem before any strategy is redeployed.

---

### Escalation Protocol

| Trigger | Immediate Action | Escalate To |
|---------|-----------------|-------------|
| Drawdown −10% to −15% | Increase monitoring frequency, log observation | Quant Res lead |
| Regime moves to ELEVATED | Review all position scalars in logs | Quant Res lead (awareness) |
| `[TAIL ALERT]` fires | Emergency review within 24 hours | Risk Review Committee |
| Regime moves to CRISIS | Full portfolio review, confirm kill switch proximity | CIO + Committee |
| `[KILL SWITCH]` fires | Halt all deployment activity | CIO + full post-mortem required |

---

## Governance

### Capital Tier System

| Tier | Mode | Criteria to Advance | Capital |
|------|------|---------------------|---------|
| **1** | Paper trading only | Pre-deployment checklist fully passed | $0 |
| **2** | Small live allocation | 3 months paper, Sharpe > 1.0 on live data | < 5% of fund |
| **3** | Core allocation | 6 months Tier 2, all metrics stable | Up to 20% of fund |

---

### Risk Review Committee

Before any strategy advances tiers, the researcher must present all four of the following:

1. Full `prometheus_risk_report` output across all stress test windows
2. A backtest with the full `PrometheusRiskModel` attached and no custom brokerage overrides
3. VIX regime performance breakdown — Sharpe and max drawdown across each of the four regimes
4. A written explanation of the alpha source and why it is expected to persist out-of-sample

---

### Prohibited Actions

> ⛔ The following are **strictly prohibited** without written Risk Review Committee approval:

- Modifying `max_drawdown`, `max_position_pct`, or `vol_target` in any live strategy
- Bypassing `PrometheusRiskModel` or removing the `SetRiskManagement` call
- Deploying any strategy that has not passed the full pre-deployment checklist
- Manually overriding a liquidation triggered by the kill switch
- Setting asset leverage above `1.0` without committee sign-off

---

## QuantConnect Workflow

### Deploying or Updating the Platform

1. Open `Prometheus_FundAllocator` in **Algorithm Lab**
2. Select `main.py` in the left file tree
3. Press **Ctrl+A** to select all existing code, then delete it
4. Paste the current platform code in full
5. Press **Ctrl+S** to save
6. Click **Backtest**
7. Wait for warm-up confirmation in logs: `Algorithm finished warming up`
8. Review the `Prometheus | Fund`, `Prometheus | Risk`, and `Prometheus | VIX` chart panels

---

### Quick Reference — Key Parameters

| Parameter | Value | Effect |
|-----------|-------|--------|
| `SetStartDate` | 2018-01-01 | Backtest start |
| `SetEndDate` | 2023-12-31 | Backtest end |
| `SetCash` | 1,000,000 | Starting capital |
| `FAST_PERIOD` | 50 | Fast SMA period |
| `SLOW_PERIOD` | 200 | Slow SMA period |
| `REBALANCE_DAYS` | 7 | Rebalance every 7 days |
| `max_drawdown` | 0.20 | Kill switch at −20% portfolio drawdown |
| `max_position_pct` | 0.20 | Hard 20% ceiling per asset |
| `vol_target` | 0.02 | 2% daily volatility target per asset |
| `correlation_limit` | 0.85 | Crowding detection threshold |
| `vol_lookback` | 30 | 30-day rolling vol window |
| `vix_lookback` | 60 | 60-day VIX beta computation window |
| `ALERT_THRESHOLD` | 0.15 | Soft tail alert at −15% drawdown |

---

<div align="center">

**PROMETHEUS CAPITAL — Internal Confidential**

*Unauthorised distribution is strictly prohibited*

</div>
