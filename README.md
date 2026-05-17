DNSE STREAMING
# DNSE OHLCV Data Fetcher

Real-time OHLCV (Open, High, Low, Close, Volume) data fetcher for Vietnamese stock market via DNSE WebSocket API.

## Overview

This service connects to DNSE's MQTT WebSocket feed, subscribes to stock quotes, and stores OHLCV candles in PostgreSQL while publishing real-time updates to Redis for downstream consumers.

## Architecture

```
DNSE WebSocket (MQTT)
        │
        ▼
┌───────────────┐
│  MQTT Client │  ← Authenticates with DNSE API
└───────────────┘
        │
        ▼
┌───────────────┐
│  OHLCV Parser │  ← Parses incoming quotes, converts timestamps
└───────────────┘
        │
   ┌────┴────┐
   ▼         ▼
┌──────┐  ┌──────┐
│Redis │  │Postgres│
│Publish│  │ Upsert │
└──────┘  └──────┘
```

## Prerequisites

- Python 3.9+
- PostgreSQL
- Redis
- DNSE account with API access

## Configuration

Edit the configuration section in the script:

```python
# Symbol list to subscribe (from List.exchange module)
SYMBOL_MODULE = "List.exchange"
SYMBOL_NAME = "HOSE3"  # Options: HNX1, HOSE3, UPCOM4, etc.

# DNSE credentials
USERNAME = "YOUR_DNSE_USERNAME"
PASSWORD = "YOUR_DNSE_PASSWORD"

# Database
DB_URL = "postgresql://user:password@host:port/database"
DB_SCHEMA = "ohlcv"

# Redis
REDIS_URL = "redis://default:password@host:port/0"

# Resolution (see supported resolutions below)
RESOLUTION = "1D"
```

## Supported Resolutions

| Resolution | Description | Table Suffix | Redis Function |
|------------|-------------|--------------|----------------|
| `1` | 1 minute | `_1M` | `chart_1m` |
| `3` | 3 minutes | `_3M` | `chart_3m` |
| `15` | 15 minutes | `_15M` | `chart_15m` |
| `30` | 30 minutes | `_30M` | `chart_30m` |
| `45` | 45 minutes | `_45M` | `chart_45m` |
| `1H` | 1 hour | `_1H` | `chart_1h` |
| `1D` | 1 day | `_1D` | `chart_1d` |

Redis channel and table names are auto-generated from resolution (e.g., `RESOLUTION = "15"` → channel `ohlcv_15`, table suffix `_15M`).

## Symbol Lists

Symbol lists are defined in the `List.exchange` module:

- `HNX1` - HNX board symbols
- `HOSE3` - HOSE board symbols (3xxx)
- `UPCOM4` - UPCOM board symbols (4xxx)

## Data Storage

### PostgreSQL Schema

Each symbol gets its own table: `{schema}.{symbol}_{resolution_suffix}`

Table schema:
```sql
CREATE TABLE "{schema}"."{symbol}_1D" (
    symbol TEXT,
    time TIMESTAMP WITH TIME ZONE,
    open DOUBLE PRECISION,
    close DOUBLE PRECISION,
    high DOUBLE PRECISION,
    low DOUBLE PRECISION,
    volume BIGINT,
    PRIMARY KEY (symbol, time)
);
```

- Schema is auto-created if not exists
- Table is auto-created on first data insert
- Upsert on `(symbol, time)` to handle late updates

### Redis Pub/Sub

Published message format:
```json
{
    "function": "chart_1d",
    "symbol": "VND",
    "time": "2026-05-15 15:00:00",
    "open": 45.5,
    "close": 46.2,
    "high": 46.5,
    "low": 45.0,
    "volume": 1250000,
    "exchange": "HOSE"
}
```

## Trading Hours

The service only processes data during Vietnam trading hours:

- Morning session: 09:00 - 11:30
- Afternoon session: 13:00 - 14:45

Data outside these hours is ignored.

## Installation

```bash
pip install paho-mqtt redis sqlalchemy psycopg2-binary requests
```

## Usage

```bash
python dnse_ohlcv_1d.py
```

Expected output:
```
Loaded 300 symbols from List.exchange.HOSE3
Connected Redis
Connected MQTT
[1D] VND @ 2026-05-15 15:00:00 VN
Redis: {"function": "chart_1d", "symbol": "VND", ...}
```

## Docker

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

COPY dnse_ohlcv_1d.py .
COPY List/ ./List/

CMD ["python", "dnse_ohlcv_1d.py"]
```

## Error Handling

- MQTT connection failure: retries automatically
- Database errors: logged and skipped (non-fatal)
- Redis errors: logged and skipped (non-fatal)
- Trading hours check: data outside hours is ignored

## Monitoring

The service outputs:

- `Loaded X symbols from ...` - on startup
- `Connected Redis` - Redis connection status
- `Connected MQTT` - MQTT connection status
- `[{resolution}] {symbol} @ {time} VN` - per candle insert
- `on_message error: ...` - any processing errors

## Dependencies

- `paho-mqtt` - MQTT WebSocket client
- `redis` - Redis client
- `sqlalchemy` - PostgreSQL ORM
- `psycopg2-binary` - PostgreSQL driver
- `requests` - HTTP client for DNSE auth
- `zoneinfo` - Timezone handling (Python 3.9+)
