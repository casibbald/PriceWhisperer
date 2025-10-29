# Polygon Analytics Platform — Architecture Diagrams (v2: Price↔News Correlation)

This version adds end‑to‑end sequencing for: price change detection → news collection → trust & sentiment triage → correlation → notification. It also expands the component graph to include scraping/APIs, NLP, and alerting.

---

## 1) End‑to‑End Sequence (Daily/Hourly Trigger → User Review)

```mermaid
sequenceDiagram
  autonumber
  participant SCHED as Scheduler (Cron/Worker)
  participant POLY as Polygon Data (S3/REST/WSS)
  participant ETL as Ingest Service (Rust)
  participant DB as PostgreSQL/TimescaleDB
  participant ALERT as Signal Engine (Thresholds/Anomaly)
  participant QUEUE as Events Bus (optional)
  participant NEWS as News Aggregator (Scraper/API)
  participant NLP as NLP Service (Sentiment + Summarize)
  participant TRUST as Trust Scorer
  participant API as Web API Rust
  participant UI as SolidJS UI
  participant USER as User

  Note over POLY,ETL: Nightly or hourly data refresh
  SCHED->>ETL: Run backfill/incremental job
  ETL->>POLY: Fetch bars/trades (S3/REST)
  ETL->>DB: COPY/UPSERT OHLCV (+ corporate actions)
  ETL-->>SCHED: Metrics/logs

  Note over ALERT,DB: Signal detection loop
  SCHED->>ALERT: Evaluate rules (hourly/daily)
  ALERT->>DB: Query latest OHLCV / indicators
  DB-->>ALERT: Series (per ticker)
  ALERT-->>QUEUE: Emit PriceEvent(ticker, Δ%, window)

  Note over NEWS,NLP: Correlate news to the event
  QUEUE->>NEWS: PriceEvent(ticker, ts)
  NEWS->>NEWS: Build sources list (RSS/HTML/APIs)
  NEWS->>NEWS: Fetch headlines/summaries/links
  NEWS->>TRUST: Source→trust score
  TRUST-->>NEWS: trust_score per item
  NEWS->>NLP: Headlines/summaries
  NLP-->>NEWS: sentiment(label, score) + short summary
  NEWS->>DB: UPSERT News(ticker, ts, src, sentiment, trust)

  Note over API,UI: Notify + drill‑down
  ALERT->>API: Create CorrelationRecord(event_id)
  API->>DB: Join PriceEvent↔News (time window)
  API-->>UI: Webhook/WS Notify(event + top news)
#  USER->>UI: Open alert; review items
  UI->>API: Fetch full event context & links
  API-->>DB: Read correlated news & price slice
  DB-->>API: Data
  API-->>UI: JSON
  UI-->>USER: Charts + ranked news (sentiment/trust) + links
```



---

## 2) System Component Graph (with News Correlation)
```mermaid
flowchart LR
  subgraph Client
    UI[SolidJS Frontend]
    Notif[Push/Email/WebSocket Notifications]
  end

  subgraph Core[Core Services Rust]
    API[Web API / GraphQL]
    Ingest[ETL Ingestion Service]
    Signals[Signal/Anomaly Engine]
    Corr[Correlation Orchestrator]
  end

  subgraph Data[Storage]
    PG[(PostgreSQL / TimescaleDB)]
    Cache[(Redis optional)]
    Obj[(Object Store backups)]
  end

  subgraph MarketData[Market Data]
    PolyS3[(Polygon S3 CSVs)]
    PolyAPI[Polygon REST/WSS]
  end

  subgraph NewsLayer[News Intake]
    Scrapers[Scraper Workers RSS/HTML]
    NewsAPIs[News APIs AlphaVantage/Finnhub/NewsAPI]
    Trust[Trust Scorer]
    NLP[Sentiment & Summarizer]
  end

  subgraph Ops[Platform]
    K8s[Kubernetes]
    Obs[Prometheus/Grafana/Loki]
    Sec[K8s Secrets API keys]
    MQ[NATS/Kafka optional]
  end

  UI -->|REST/WS| API
  Notif --> UI

  Ingest -->|download| PolyS3
  Ingest -->|query| PolyAPI
  Ingest --> PG

  Signals --> PG
  Signals --> MQ
  MQ --> Corr

  Corr --> Scrapers
  Corr --> NewsAPIs
  Scrapers --> Trust
  NewsAPIs --> Trust
  Trust --> NLP
  NLP --> PG
  Scrapers --> PG
  NewsAPIs --> PG

  API --> PG
  API --> Cache
  API --> MQ

  K8s --- Core
  K8s --- NewsLayer
  K8s --- Data
  K8s --- Ops
  Obs -.metrics/logs.-> Core
  Obs -.metrics/logs.-> NewsLayer
  Sec -.secrets.-> Core
  Sec -.secrets.-> NewsLayer
  Obj -.backups.-> Data
```

---

## 3) Data Model (Concise ER Sketch)
```mermaid
erDiagram
  TICKER ||--o{ OHLCV : has
  TICKER {
    string symbol PK
    string name
    string exchange
  }
  OHLCV {
    string symbol FK
    timestamptz ts PK
    float open
    float high
    float low
    float close
    bigint volume
    jsonb meta
  }
  PRICE_EVENT ||--o{ CORRELATION : produces
  PRICE_EVENT {
    uuid id PK
    string symbol FK
    timestamptz window_start
    timestamptz window_end
    float pct_change
    jsonb features
  }
  NEWS ||--o{ CORRELATION : explains
  NEWS {
    uuid id PK
    string symbol FK
    timestamptz published_at
    text headline
    text summary
    text url
    text source
    float sentiment_score
    text sentiment_label
    float trust_score
    jsonb raw
  }
  CORRELATION {
    uuid event_id FK
    uuid news_id FK
    float rank_score
  }

```

---

### Notes & Options
- **Trust Scoring:** simple whitelist → weighted score; enrich later with domain reputation signals.
- **Sentiment:** start with headline‑only; upgrade to FinBERT or vendor‑provided scores when using APIs.
- **Scheduling:** K8s CronJobs for EOD/hourly; MQ (NATS/Kafka) if you want decoupled fan‑out.
- **Performance:** index `(symbol, ts)` on OHLCV/NEWS; consider Timescale continuous aggregates; batch UPSERTs.
- **Security:** respect robots.txt for scrapers; rotate API keys via K8s Secrets.

```
