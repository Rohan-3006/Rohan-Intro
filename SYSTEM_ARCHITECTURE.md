# Rohan's Trading System Architecture

**Author:** Rohan K. Jadhav | **Updated:** 2026-06-07 | **Version:** 2.0

---

## System Overview

A **production-grade algo trading infrastructure** for NSE NIFTY/BANKNIFTY options and derivatives.

Data flows from live market feeds → intelligent scoring → AI analysis → manual trade execution → automated tracking → performance analytics.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        TRADING SYSTEM ARCHITECTURE                       │
└─────────────────────────────────────────────────────────────────────────┘

                          🕐 09:15 IST (Market Open)
                                  │
                                  ▼
                    ┌─────────────────────────────┐
                    │   MARKET DATA FEED          │
                    │  (Dhan API / NSE / Yahoo)   │
                    └──────────────┬──────────────┘
                                  │
                    ┌─────────────────────────────┐
                    │      OptionsFlow            │  [Repository: OptionsFlow]
                    │  ─────────────────────────  │
                    │  • IV Percentile Calc       │
                    │  • OI Buildup Detection     │
                    │  • Strike Scoring (1-10)    │
                    │  • High-conviction ranking  │
                    └──────────────┬──────────────┘
                                  │
                    ┌─────────────────────────────┐
                    │      AI_Base (Claude)       │  [Repository: AI_Base]
                    │  ─────────────────────────  │
                    │  • Pre-market briefing      │
                    │  • Options interpretation   │
                    │  • Setup validation         │
                    │  • Strategy recommendation  │
                    └──────────────┬──────────────┘
                                  │
                    ┌─────────────────────────────┐
                    │    HUMAN DECISION LAYER     │
                    │  ─────────────────────────  │
                    │  • Accept/Reject signal     │
                    │  • Manual entry on chart    │
                    │  • TradingView execution    │
                    │  • Telegram notifications   │
                    └──────────────┬──────────────┘
                                  │
                    ┌─────────────────────────────┐
                    │     EdgeTracker             │  [Repository: EdgeTracker]
                    │  ─────────────────────────  │
                    │  • CSV trade log capture    │
                    │  • P&L calculation          │
                    │  • Win rate tracking        │
                    │  • Equity curve generation  │
                    └──────────────┬──────────────┘
                                  │
                    ┌─────────────────────────────┐
                    │    SQL_Ledger               │  [Repository: SQL_Ledger]
                    │  ─────────────────────────  │
                    │  • SQLite persistence       │
                    │  • Monthly summaries        │
                    │  • Strategy breakdown       │
                    │  • Portfolio analytics      │
                    └──────────────┬──────────────┘
                                  │
                                  ▼
                          🕖 15:30 IST (Market Close)

```

---

## Module Specifications

### 1. **OptionsFlow** — Strike Scoring Engine

**Purpose:** Generate high-conviction options chain scores.

**Inputs:**
- Live NSE options chain (symbol, expiry, strike, CE/PE, IV, OI, volume, bid-ask)
- IV percentile (calculated or input)

**Processing:**
```
Composite Score = (OI_Score × 0.35) + (IV_Score × 0.30) + (Delta_Score × 0.20) + (Liq_Score × 0.15)

Signal Classification:
  ≥ 7.5 → 🟢 HIGH CONVICTION
  ≥ 6.0 → 🟡 MODERATE
  ≥ 4.5 → 🟠 WATCH
  < 4.5 → 🔴 AVOID
```

**Outputs:**
- Ranked DataFrame (top strikes by composite score)
- JSON export for AI_Base consumption
- CSV for manual reference

**Tech Stack:** Python, pandas, dataclasses, type hints

**Data Lifecycle:** Live during market hours (09:15–15:30 IST)

**GitHub:** [OptionsFlow](https://github.com/Rohan-3006/OptionsFlow)

---

### 2. **AI_Base** — Market Intelligence Layer

**Purpose:** Automate analysis and generate trade recommendations.

**Workflows:**

| Workflow | Input Source | Output | Timing |
|----------|--------------|--------|--------|
| Pre-Market Analysis | SGX Nifty, VIX, events | Bias, levels, risk factors | 08:45 IST |
| Options Chain Read | OptionsFlow top 5 strikes | Range prediction, strategy | 09:20 IST |
| Trade Review | EdgeTracker trade log | Objective critique, lessons | EOD 15:45 IST |
| Pine Script Debug | Error + code snippet | Root cause, fix, prevention | On-demand |
| Backtest Interpretation | Test metrics JSON | Verdict: prod-ready/optimize/discard | Weekly |

**LLM Config:**
- **Model:** Claude 3.5 Sonnet (Anthropic API)
- **Max tokens:** 1024 (configurable)
- **Temperature:** 0.7 (balanced creativity + accuracy)
- **Rate limit:** 1 req/sec (built-in backoff)

**Tech Stack:** Python, requests, anthropic SDK, python-dotenv

**Data Lifecycle:** Per-workflow (multiple times daily)

**GitHub:** [AI_Base](https://github.com/Rohan-3006/AI_Base)

---

### 3. **Pine_5** — Execution Strategy Library

**Purpose:** Provide battle-tested Pine Script strategies for live execution.

**Strategies:**

| Strategy | Platform | Timeframe | Instrument | Entry Signal | Exit Signal |
|----------|----------|-----------|------------|--------------|-------------|
| VWAP + EMA Trend | TradingView | 5-min | NIFTY, BANKNIFTY | Price > VWAP + EMA(10)>EMA(20) | EMA crossover flip or ATR SL |
| RSI + BB Mean Reversion | TradingView | 15-min | Nifty 50 stocks | RSI < 30 or > 70 + BB touch | RSI midline or opposite BB |
| Supertrend Momentum | TradingView | 15-min | NIFTY, BANKNIFTY | Supertrend flip + ADX > 25 | Supertrend flip or ATR SL |

**Risk Management (All Strategies):**
- ATR(14) × 1.5 = stop loss
- Session filter: 09:15–14:45 IST only
- Volume filter: > 20-SMA × 1.2x
- Commission: 0.03% per trade

**Tech Stack:** Pine Script v5 (TradingView)

**Data Lifecycle:** Intraday (live signals during market hours)

**GitHub:** [Pine_5](https://github.com/Rohan-3006/Pine_5)

---

### 4. **EdgeTracker** — Trade Capture & Analytics

**Purpose:** Import, normalize, and analyze trade logs.

**Input:** CSV from manual entry or broker export
```csv
trade_id, symbol, direction, entry_date, entry_time, exit_date, exit_time,
entry_price, exit_price, quantity, pnl, strategy
```

**Metrics Calculated:**
- Win Rate: `(Wins / Total) × 100`
- Profit Factor: `Gross Profit / Gross Loss`
- Expectancy: `Net P&L / Total Trades`
- Max Drawdown: `(Lowest Equity - Peak) / Peak × 100`
- Sharpe Ratio: `(Mean Return - RF) / Std Dev`

**Outputs:**
- Console report (formatted table)
- JSON: `output/report_YYYYMMDD.json`
- CSV: Equity curve, monthly summary, strategy breakdown

**Tech Stack:** Python, pandas, numpy, python-dateutil

**Data Lifecycle:** Post-market (EOD analysis)

**GitHub:** [EdgeTracker](https://github.com/Rohan-3006/EdgeTracker)

---

### 5. **SQL_Ledger** — Portfolio Analytics Database

**Purpose:** Persistent storage and historical portfolio reporting.

**Database Schema:**
```sql
CREATE TABLE trades (
    trade_id INTEGER PRIMARY KEY AUTOINCREMENT,
    symbol TEXT NOT NULL,
    direction TEXT NOT NULL,
    entry_date DATE NOT NULL,
    entry_time TIME NOT NULL,
    exit_date DATE NOT NULL,
    exit_time TIME NOT NULL,
    entry_price REAL NOT NULL,
    exit_price REAL NOT NULL,
    quantity INTEGER NOT NULL,
    pnl REAL NOT NULL,
    strategy TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Query Layer:**
- Monthly aggregations (P&L by month)
- Strategy performance breakdown
- Portfolio-wide metrics (Sharpe, MDD, PF)
- Year-to-date cumulative P&L

**Outputs:**
- JSON exports for reporting
- CSV for Excel/Sheets import
- HTML dashboard (future)

**Tech Stack:** Python, SQLite, pandas, sqlite3

**Data Lifecycle:** Batch (daily/weekly aggregation)

**GitHub:** [SQL_Ledger](https://github.com/Rohan-3006/SQL_Ledger)

---

## Data Flow & Contracts

### Flow 1: Real-time Scoring → Analysis → Decision

```
09:15 IST  Market Open
  │
  ├─ Fetch NSE chain (Dhan API)
  │
  ├─ OptionsFlow.scan_chain(strikes, iv_pct=35)
  │  Output: {
  │    "top_5": [
  │      {"strike": 22500, "type": "CE", "composite": 8.06, "signal": "🟢"}
  │    ]
  │  }
  │
  ├─ AI_Base.options_analysis(top_5, pcr=1.2, iv_pct=35)
  │  Output: {
  │    "analysis": {
  │      "bias": "BULLISH",
  │      "recommended_strategy": "Buy 22500 CE",
  │      "risk_levels": {...}
  │    }
  │  }
  │
  └─ Human: Accept/Reject → Manual Entry on TradingView
      ✓ Accepted → Telegram notification
      ✗ Rejected → Next signal
```

**Data Contract (OptionsFlow → AI_Base):**
```json
{
  "top_strikes": [
    {
      "strike": number,
      "type": "CE" | "PE",
      "composite": number (0-10),
      "signal": string ("🟢 HIGH CONVICTION" | ...),
      "iv": number,
      "oi": integer,
      "delta": number
    }
  ],
  "pcr": number,
  "iv_percentile": number
}
```

### Flow 2: Post-Trade Analysis → Database → Reporting

```
15:35 IST  Manual trade log entry (CSV)
  │
  ├─ EdgeTracker.analyze(trades.csv)
  │  Output: {
  │    "win_rate": 0.667,
  │    "profit_factor": 2.134,
  │    "expectancy": 483.33,
  │    "equity_curve": [...],
  │    "monthly_summary": [...]
  │  }
  │
  ├─ AI_Base.trade_review(trade_data)
  │  Output: {
  │    "verdict": "Well-executed setup",
  │    "lessons": ["Entry timing perfect", "Exit too early"]
  │  }
  │
  └─ SQL_Ledger.persist(trades) → SQLite
      │
      └─ SQL_Ledger.monthly_report()
         Output: PDF/JSON to Google Drive
```

**Data Contract (EdgeTracker → SQL_Ledger):**
```json
{
  "trades": [
    {
      "trade_id": integer,
      "symbol": string,
      "direction": "BUY" | "SELL",
      "entry_price": number,
      "exit_price": number,
      "pnl": number,
      "strategy": string,
      "entry_date": "YYYY-MM-DD",
      "exit_date": "YYYY-MM-DD"
    }
  ]
}
```

---

## NSE Market Context

### Trading Hours
- **Pre-open:** 09:00–09:15 IST
- **Main session:** 09:15–15:30 IST
- **Settlement:** T+1 (next business day)

### Instruments
- **NIFTY:** 50-stock index
  - Lot size: 25 contracts
  - Multiplier: ₹1 per point
- **BANKNIFTY:** Bank sector index
  - Lot size: 15 contracts
  - Multiplier: ₹1 per point

### Options
- **Expiry:** Every Thursday (weekly)
- **Types:** CE (Call) / PE (Put)
- **Greeks:** Delta, Gamma, Theta, Vega supported
- **Brokerage:** ~0.03% (Dhan rates)

---

## Repository Dependencies

```
OptionsFlow (source: NSE API)
   ├── OUTPUT: Ranked strikes JSON
   └── CONSUMED BY: AI_Base

AI_Base (source: OptionsFlow + manual input)
   ├── OUTPUT: Trade recommendations
   └── CONSUMED BY: Human Decision + Trade Review workflow

Pine_5 (source: TradingView)
   ├── OUTPUT: Live trade signals on chart
   └── CONSUMED BY: Manual entry into EdgeTracker

EdgeTracker (source: CSV trade log)
   ├── OUTPUT: Performance metrics + trade data
   └── CONSUMED BY: AI_Base (review) + SQL_Ledger (persistence)

SQL_Ledger (source: EdgeTracker)
   ├── OUTPUT: Monthly reports, portfolio P&L
   └── CONSUMED BY: Performance reporting, strategy backtesting
```

---

## Error Handling & Resilience

### Failure Scenarios

| Scenario | Consequence | Mitigation |
|----------|-------------|------------|
| Dhan API timeout | OptionsFlow can't score | Fall back to Yahoo Finance + cache yesterday's data |
| Claude API rate limit | AI_Base requests fail | Queue with exponential backoff + Telegram alert |
| CSV malformed | EdgeTracker crashes | Validate headers, log schema errors, suggest fix |
| SQLite locked | Can't write to DB | Use `data/live/backup.db` as temp cache |
| Market circuit break | Sudden data gap | Pause strategies, manual review required |

### Logging

```python
import logging

log = logging.getLogger(__name__)
log.info(f"Scored {len(strikes)} strikes | Top: {top.name} (score={top.composite})")
log.warning(f"API timeout, retrying in {backoff}s")
log.error(f"Critical: {e}", exc_info=True)
```

**Log destination:** `logs/trading_system_YYYYMMDD.log`

---

## Performance Benchmarks

| Component | Metric | Target | Current |
|-----------|--------|--------|---------|
| OptionsFlow | Chain scoring (full) | <2s | ✅ <1s |
| AI_Base | Claude response | <10s | ✅ ~5s |
| EdgeTracker | CSV analysis (100 trades) | <5s | ✅ ~2s |
| SQL_Ledger | Monthly aggregation | <1s | ✅ <500ms |
| E2E (scoring → decision) | Time to recommendation | <30s | ✅ ~20s |

---

## Security & Privacy

### Credentials
- **Storage:** `.env` (local, gitignored)
- **Secrets:** `ANTHROPIC_API_KEY`, `DHAN_API_KEY`, `TELEGRAM_BOT_TOKEN`
- **Never commit:** `.env`, `config_local.py`, `secrets.py`

### Data
- **Live trades:** Stored in `data/live/` (gitignored)
- **Personal data:** Never committed to public repos
- **Backups:** `data/live/trades_backup.csv` (local only)

---

## Roadmap

### Q2 2026
- [ ] Live Telegram alerts for high-conviction setups
- [ ] Google Sheets API integration (paste OI → get analysis)
- [ ] Backtester harness (shared across repos)
- [ ] Strategy comparison dashboard

### Q3 2026
- [ ] Position sizing calculator (Kelly Criterion)
- [ ] Multi-account portfolio tracking
- [ ] Web dashboard (Flask + Plotly)
- [ ] Automated broker imports (Dhan, Zerodha)

### Q4 2026
- [ ] Options Greeks live monitor
- [ ] Risk analytics (VaR, Sortino, Calmar)
- [ ] Machine learning signal booster (ensemble)
- [ ] Trading performance audit trail (compliance)

---

## Getting Started

### Clone All Repos

```bash
cd ~/projects
git clone https://github.com/Rohan-3006/OptionsFlow
git clone https://github.com/Rohan-3006/EdgeTracker
git clone https://github.com/Rohan-3006/AI_Base
git clone https://github.com/Rohan-3006/SQL_Ledger
git clone https://github.com/Rohan-3006/Pine_5
```

### Setup Environment

```bash
cd OptionsFlow
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

cd ../AI_Base
cp .env.example .env
# Edit .env: add ANTHROPIC_API_KEY
pip install -r requirements.txt
```

### Run Daily Workflow

```bash
# 08:45 IST: Pre-market briefing
python AI_Base/scripts/ai_workflow.py --workflow pre_market

# 09:20 IST: Options chain analysis
python OptionsFlow/src/options_scanner.py

# 15:45 IST: Trade analysis & review
python EdgeTracker/src/analytics.py
python AI_Base/scripts/ai_workflow.py --workflow trade_review

# Weekly: Portfolio summary
python SQL_Ledger/src/analytics.py --report monthly
```

---

## Author & Contact

**Rohan K. Jadhav**  
Technical Analyst | Algo Trading Developer | NISM-VIII Certified

📧 rohan.jadhav3006@gmail.com  
🔗 [GitHub](https://github.com/Rohan-3006) | [TradingView](https://www.tradingview.com/)

---

**Last Updated:** 2026-06-07  
**System Status:** ✅ Production (v2.0)
