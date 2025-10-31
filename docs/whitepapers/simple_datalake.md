### A lightweight “quant-data lake” you can run on a laptop

| Layer                              | Tech / format                                                                                                  | Why it fits the use-case                                                          | Rust crates                                  |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- | -------------------------------------------- |
| **Raw capture**                    | Append **JSON** files per vendor-API pull (ticks, option greeks, headlines).  Compress with `zstd`.            | Vendor schemas change—keeping the raw payload guarantees replay / re-parse later. | `serde_json`, `zstd`                         |
| **Canonical tables**               | **Apache Parquet** partitioned *by date / ticker* (e.g. `price/minute/symbol=NVDA/date=2025-10-31/*.parquet`). | Column-store, compresses 10×, random-access by predicate push-down.               | `parquet`, `polars`                          |
| **Query engine (local)**           | **DuckDB** on top of the Parquet folder.                                                                       | Zero server, SQL over billions of rows at RAM speed.                              | DuckDB’s C API via `duckdb-rs`, or call CLI. |
| **Hot cache**                      | In-memory `HashMap` / `DashMap` keyed by `(symbol, minute)` for today’s session.                               | Millisecond read path for live strategy.                                          | `dashmap`                                    |
| **Metadata / heuristics registry** | **Postgres** table `heuristic_defs(id, json_spec, added_at)` + `matches(id, ts, symbol, score)`.               | ACID log of every signal version and back-test result.                            | `sqlx`                                       |

```
S3 / local SSD
└── parquet lake  <— DuckDB query
        ▲
 nightly ETL  (Rust)
        ▲
raw_zstd/
```

---

### Daily ETL recipe (Rust & Polars)

```rust
use polars::prelude::*;
fn etl_day(date: NaiveDate) -> Result<()> {
    // 1. read raw json lines
    let df = JsonReader::new(File::open(raw_path(date)?)).infer_schema(None).finish()?;
    // 2. normalise schema
    let norm = df.select([
        col("symbol"),
        col("ts").cast(DataType::Datetime(TimeUnit::Milliseconds, None)),
        col("open"), col("high"), col("low"), col("close"), col("volume")
    ]);
    // 3. write parquet partitioned by symbol
    for (sym, sub) in norm.partition_by("symbol")? {
        let out = format!("parquet/price/minute/symbol={sym}/date={date}.parquet");
        ParquetWriter::new(File::create(out)?)
            .with_compression(ParquetCompression::Zstd(4))
            .finish(&sub)?;
    }
    Ok(())
}
```

Run via `cargo run --release -- date=2025-10-31`.

---

### Building & versioning **heuristics**

| Concept            | Implementation idea                                                                           |
| ------------------ | --------------------------------------------------------------------------------------------- |
| **Heuristic spec** | JSON blob, e.g.:<br>`{ "name": "saw_tooth_v1", "window":5, "amp_factor":1.2, "min_count":3 }` |
| **Back-test**      | DuckDB SQL filters Parquet:                                                                   |

```sql
SELECT symbol, ts
FROM parquet_scan('parquet/price/minute/**/*.parquet')
WHERE symbol='NVDA'
ORDER BY ts
```

stream into your Rust detector; write matches into `matches` table. |
| **Leaderboard** | SQL view joining `matches` ↔ `option_pnl` to rank f1, sharpe. |
| **Version control** | Bump `heuristic_defs.version`; re-run offline tests. |

Use Git to version both spec files and ETL code; CI job replays last 90 days and posts metrics.

---

### “Local playback” for strategy debugging

```bash
duckdb -cmd "
  COPY (
    SELECT *
    FROM parquet_scan('parquet/price/minute/symbol=NVDA/*.parquet')
    ORDER BY ts
  ) TO 'nvda_minute.csv' WITH (HEADER);
"

# then pipe into your live engine with a 60-sec sleep between rows
```

Or in Rust:

```rust
for row in reader.records() {
    process_tick(row?);
    std::thread::sleep(Duration::from_secs(1));
}
```

---

### Where Redis *could* still help

* **Pub-sub** replay: push each historical bar onto `REDIS_STREAM price_feed` so the exact same consumer code handles live and back-test.
* **Distributed lock** for multi-node back-test workers.

If you stay single-process, skip Redis.

---

#### Toolchain checklist

* ☐ `reqwest` + vendor REST → raw JSON in `/raw_zstd/`
* ☐ nightly ETL (`cargo cronet`) → Parquet lake
* ☐ `duckdb` CLI for ad-hoc SQL
* ☐ `polars + parquet` in Rust for heavy transforms
* ☐ `sqlx` for Postgres heuristics registry
* ☐ Grafana → Postgres for match counts / PnL curves

With this setup you can **store, query, replay and evolve heuristics entirely offline** before letting the live saw-tooth bot touch real money.
