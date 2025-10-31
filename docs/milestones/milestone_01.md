# PriceWhisperer

## **Milestone #1 – “PriceWhisperer α-loop”**

| Area                  | What’s finished                                                                                                                                                                                                                             | Why it matters                                                                           |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| **Data plumbing**     | • Live minute-bars + option greeks streamed from Polygon/IBKR into an **in-memory cache**.<br>• Raw JSON archived → nightly **Parquet data-lake** queried with DuckDB.<br>• News feeds ingested; sentiment scored via FinBERT ONNX in Rust. | Gives us deterministic historical replay **and** real-time fire-hose on a single laptop. |
| **Pattern engine**    | • Rust saw-tooth detector (peak/trough, regime switch, gap override).<br>• Signal fusion with news sentiment and ATR filters.<br>• Outputs JSON alerts to Redis queue.                                                                      | Converts noisy price series into **actionable, rate-limited trade triggers**.            |
| **Trade logic**       | • Options playbook: iron-condor for oscillations; long strangle for gaps; gamma-scalp module.<br>• Risk-manager throttles delta/theta and message rate.                                                                                     | First **rules-based strategy set** tied directly to detected regimes.                    |
| **Execution**         | • Full Interactive Brokers paper-account integration (socket + REST).<br>• Order ack-log, PnL, fills persisted to Postgres.                                                                                                                 | We can run end-to-end with **live market micro-fills**, zero capital at risk.            |
| **Back-test harness** | • DuckDB → Rust stream → identical signal & order code.<br>• Monte-Carlo / option-P&L analytics in Polars.                                                                                                                                  | One code-path covers **simulation and production**.                                      |
| **Observability**     | • Prometheus metrics on latency, hit-rate, theta exposure.<br>• Grafana dashboard.                                                                                                                                                          | Makes tuning & compliance evidence painless.                                             |

**Definition of done:** 24-hour unattended run on IBKR paper trading **with automatic restart**, no runtime exceptions, and full trade ledger persisted.

---

## Elevator pitches

### 15-second teaser

> “PriceWhisperer spots micro-oscillations and headline shocks in real time, then auto-fires the option spread that best monetises that volatility, all tested in IBKR’s paper market.”

---

### 30-second corridor version

> “We’re building *PriceWhisperer*, an engine that listens to every listed stock, tags the ‘saw-tooth’ reversals you see on intraday charts, overlays live sentiment, and instantly decides whether to sell premium, buy gamma or fade a gap.
> The α-milestone is live: data lake, signal logic, and automated options orders already run end-to-end in Interactive Brokers’ paper account. Next step—turn it loose on small real money and start compounding the edge.”

---

### 2-minute investor pitch

> “Equity markets have splintered into two extremes: ultra-high-frequency firms clipping micro-spreads and retail momentum chasers pushing price into predictable saw-tooth patterns.
> **PriceWhisperer** lives in the middle.
>
> * How it works:
    >
    >   1. **Stream & store** minute-level trades, option chains and news headlines into a Parquet lake—cheap and query-ready.
    >   2. **Detect** micro-patterns—oscillations, regime breaks, gaps—via an engine that flags only moves bigger than 1.2 × ATR but smaller than classic trend systems even notice.
>   3. **Fuse** that with real-time sentiment so we know if a spike is data-driven or pure order-flow.
>   4. **Exploit** via the option Greeks best suited to the pattern: sell defined-risk condors when range-bound, buy strangles when IV is mis-priced, or gamma-scalp straddles when realised vol outruns implied.
> * Why it’s credible: same codebase already back-tests off months of Parquet history **and** executes live—today—in Interactive Brokers’ sandbox, with latency <1 ms inside one laptop.
> * Path to money: flip the switch to a funded IBKR account, size trades conservatively, and iterate; then scale capital once live Sharpe > 1.5 is demonstrated over 90 days.
    >   **Bottom line:** we’re not predicting the future—we’re arbitraging micro-behaviour everyone can see but few can automate at the options-level. The infrastructure is live; we’re ready for real capital.”

