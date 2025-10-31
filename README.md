# PriceWhisperer

*A real-time options-trading engine that listens for “whispers” in minute-bar price action and headlines, then monetises the noise with the right option strategy.*

---

<p align="center">
  <img alt="Project logo" src="docs/images/logo.png" width="400"/>
</p>

---

## ✨ Key features at Milestone #1

| Component          | Status                                                                                                   | Highlights                                     |
| ------------------ | -------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| **Data ingestion** | ✅ Live minute bars & option greeks (Polygon / IBKR) streamed into in-memory cache & archived as Parquet. | `tungstenite`, `polars`, `zstd` compression    |
| **Pattern engine** | ✅ Rust saw-tooth detector + gap/regime switch & FinBERT sentiment fusion.                                | windowed peak/trough; ATR filter; news overlay |
| **Trade logic**    | ✅ Condor, strangle & gamma-scalp playbook wired to signals.                                              | dynamic sizing, auto-hedge rules               |
| **Execution**      | ✅ End-to-end on Interactive Brokers **paper** gateway (identical API to live).                           | `ibkr-rust`, 50 msg/s limit-aware              |
| **Risk & logging** | ✅ Real-time theta/delta caps; orders & fills persisted in Postgres, prices in DuckDB lake.               | Prometheus metrics, Grafana dashboard          |

> **Definition of Done:** 24-hour unattended paper run with zero runtime exceptions and a complete trade ledger.

---

## 🏗️  Architecture at a glance

```mermaid
flowchart LR
    subgraph Live Loop
        M[Market-Data WS] -->|1 min bars| Cache
        N[News Feed] --> Sent
        Cache --> ST[\"Saw-tooth\"\]
        Sent --> Fusion
        ST --> Fusion
        Fusion --> RM[Risk Manager]
        RM --> OB[Order Builder]
        OB --> IB[IB Gateway]
    end
    IB --> Ledger[(Postgres)]
    Cache --> Lake[(Parquet lake)]
    Sent --> Ledger
```

*One code-path drives both live trading and historical back-tests.*

---

## 🚀  Quick start

```bash
# 1 clone & build
$ git clone https://github.com/your-org/pricewhisperer.git
$ cd pricewhisperer && cargo build --release

# 2 start the IB Gateway in paper mode (port 7497)
# 3 run the bot
$ cargo run --release -- \  
      --config examples/ibkr_paper.toml
```

> **Requirements:** Rust 1.73+, PostgreSQL 14+, DuckDB 0.10+, an IBKR account with market-data subscriptions.

---

## 🗄️  Repo structure

```
pricewhisperer/
 ├── crates/
 │   ├── core/           # saw-tooth & sentiment fusion
 │   ├── broker_ib/      # IBKR socket + REST bindings
 │   ├── strategies/     # condor, strangle, gamma-scalp modules
 │   └── storage/        # Parquet + Postgres helpers
 ├── config/             # .toml samples
 ├── docs/               # design notes & logo
 └── README.md
```

---

## 🛣️  Roadmap

1. **Milestone #2 – Live-money pilot**
   • Switch to small funded account, confirm real-fill slippage < 0.15 × premium.
   • Add dynamic IV filtering and skew checks.
2. **Milestone #3 – Multi-asset scaling**
   • Extend to SPX 0-DTE and crypto options (Deribit).
   • Introduce Redis pub-sub for cluster deployment.
3. **Milestone #4 – Reinforcement learner overlay**
   • Live policy gradient agent to weight strategy selection.

---

## 📜  License

Apache 2.0 — free for personal & commercial use with attribution.

---

### 👥  Contributors

*Core dev:* `@casibbald`
*Quant research:* `@quant-friends`

*We welcome PRs & issues!*
