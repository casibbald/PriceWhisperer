```mermaid
sequenceDiagram
    autonumber
    actor MD as Market-Data WebSocket
    actor NS as News/Sentiment API
    participant ST as Saw-tooth Detector (Rust task)
    participant SA as Sentiment Analyzer (Rust task + ONNX FinBERT)
    participant FM as Signal Fusion / Scorer
    participant RM as Risk Manager
    participant OS as Option Selector & Order Builder
    participant BR as Broker API (IBKR / Tradier)
    participant DB as Postgres / Parquet Log
    
    MD->>ST: Stream 1-min OHLC (JSON)\n(every 1 000 ms)
    ST->>DB: Append price bar
    ST-->>FM: Saw-tooth event {“ticker”: NVDA, “type”: trough, t=10:31}
    
    Note over NS: every 30 s async poll
    NS->>SA: Push headline JSON
    SA->>DB: Cache headline + score
    SA-->>FM: Sentiment flip {“ticker”: NVDA, “score”: +0.82}
    
    FM->>RM: Combined signal \n(NVDA trough + bullish score)
    RM-->>OS: OK if theta_limit not breached
    OS->>BR: Place limit order\n Buy 1× NVDA 0-DTE $450 Call @ $1.25
    BR-->>OS: FIX/REST “FILLED” 1 × $1.24
    OS->>DB: Record fill, greeks
    
    Note over RM, BR: continuous
    BR-->>RM: Position feed (WebSocket)
    RM->>BR: If risk breach -> send CLOSE order
    
    alt Order rejected
        BR-->>OS: “REJECT” (liquidity)
        OS->>DB: Log reject
    end

```