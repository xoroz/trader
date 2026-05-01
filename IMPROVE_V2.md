# IMPROVE_V2: Strategy Overhaul & Implementation Guide

This document outlines the architectural changes required to upgrade the `overnight` trading strategy from a single-pass quant filter to a robust, ensemble-based system with LLM validation and dynamic risk management.

## 1. Ensemble Architecture (The "Double Checker")
Instead of relying on a single scoring formula, split the logic into isolated sub-strategies. 

### Implementation Steps:
* **Refactor `bot/strategies/overnight.py`**: Break the current monolithic score function into three distinct scoring functions:
    1.  `score_momentum(data)`: Heavily weights the 21-day trend.
    2.  `score_mean_reversion(data)`: Targets intraday dips on structurally sound stocks.
    3.  `score_vwap_consolidation(data)`: Prefers low volatility and tight VWAP adherence.
* **Create Aggregator**: Introduce an `aggregator` function that ranks the top 10 from each sub-strategy.
* **Consensus Logic**: Only allocate slots (25-29) to tickers that appear in the top 10 of *at least two* sub-strategies.

## 2. LLM Validation Layer
Integrate the existing LLM infrastructure to act as a qualitative filter over the quantitative consensus.

### Implementation Steps:
* **Modify Schedule**: Run the quant aggregator at 15:40 ET to produce the top 15 consensus candidates.
* **LLM Call**: At 15:45 ET, pass the 15 tickers to `bot/llm.py` via a new prompt template (e.g., `prompt_overnight_catalyst.j2`).
* **Prompt Directives**: Ask the LLM to filter out tickers with pending after-hours earnings, imminent FDA PDUFA dates, or macro-sensitive exposure.
* **Final Selection**: The LLM returns the final 5 safest tickers for execution at 15:55 ET MOC.

## 3. Regime-Adaptive Weights
Make the scoring penalties dynamic based on broader market volatility.

### Implementation Steps:
* **Fetch VIX/VIX9D**: During the 15:45 ET scan, query the broker for the current VIX value.
* **Dynamic Penalty**: Replace static weights (e.g., `W_VOL = 0.5`). 
    * *Example*: `W_VOL = 0.5 * (current_vix / baseline_vix)`
    * If the VIX spikes > 25, the volatility penalty automatically doubles, naturally pushing the bot toward safer, lower-beta stocks.

## 4. Pre-Market Emergency Exits
Replace the blind MOO (Market-on-Open) exit with an active pre-market monitoring system to mitigate gap-down risks.

### Implementation Steps:
* **Add Pre-Market Loop**: Create `monitor_premarket_fills(pool, ib)` running continuously from 04:00 ET to 09:30 ET.
* **Drawdown Threshold**: Monitor the mark price of open positions. If a stock drops > 3% below the MOC entry price during pre-market hours:
    1.  Cancel the pending MOO order.
    2.  Immediately submit an aggressive Limit Sell order (e.g., Mid-price) to cut losses before the 09:30 ET retail open.
