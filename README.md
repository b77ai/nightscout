# MCP Nightscout by [B77.ai](https://b77.ai)

An [MCP (Model Context Protocol)](https://modelcontextprotocol.io/) server that connects AI assistants to [Nightscout](http://www.nightscout.info/) — the open-source CGM remote monitoring platform for Type 1 Diabetes management.

Built with [FastMCP](https://github.com/jlowin/fastmcp), this server exposes Nightscout data as MCP tools, enabling AI assistants like Claude to read glucose data, analyze patterns, log treatments, and more.

## Features

### Glucose Monitoring

- **get_current_glucose** — latest CGM reading with value, trend arrow, and device info
- **get_recent_glucose** — recent readings (default: last 3 hours)
- **get_glucose_by_date_range** — raw readings for a specific short date range

### Analytics

- **calculate_time_in_range** — TIR stats with breakdown (in range, below, above, very low, very high), estimated HbA1c, and coefficient of variation
- **get_daily_glucose_stats** — daily stats with hourly averages
- **analyze_glucose_patterns** — multi-day pattern analysis to identify recurring highs/lows by time of day
- **get_glucose_stats_batch** — aggregated glucose stats for long date ranges using a single range query

### Treatments

- **get_recent_treatments** — recent insulin boluses, carb entries, temp basals, notes
- **get_treatments_by_date** — treatments for a specific date range
- **log_treatment** — log a new treatment (bolus, carbs, exercise, note, etc.)
- **remove_treatment** — delete a treatment by ID

### Device & Profile

- **get_insulin_on_board** — current IOB from loop/pump device status
- **get_latest_device_status** — pump status, loop decisions, CGM status
- **server_status** — Nightscout server version, plugins, thresholds, units
- **get_current_profile** — basal rates, ISF, IC ratios, target BG, DIA
- **update_nightscout_profile** — update profile settings

## Quick Start

### 1. Clone and configure

```bash
git clone https://github.com/easyweek/mcp-nightscout.git
cd mcp-nightscout
cp .env.example .env
```

Edit `.env` with your Nightscout instance details:

```env
NIGHTSCOUT_URL=https://your-nightscout.example.com
NIGHTSCOUT_TOKEN=your-readable-token
MCP_AUTH_TOKEN=your-secret-mcp-token
```

### 2. Run with Docker

```bash
docker compose up --build
```

The server starts on `http://localhost:8000`.

### 3. Connect to Claude

#### Claude Code (CLI)

Add to your MCP settings:

```json
{
  "mcpServers": {
    "nightscout": {
      "type": "http",
      "url": "http://localhost:8000/mcp",
      "headers": {
        "Authorization": "Bearer your-secret-mcp-token"
      }
    }
  }
}
```

#### Claude.ai (remote)

When deployed to a public URL, connect using the token-in-path format:

```
https://your-server.example.com/mcp/your-secret-mcp-token
```

## Authentication

The server supports two auth modes for MCP clients:

| Method        | Format                          | Use case             |
| ------------- | ------------------------------- | -------------------- |
| Bearer header | `Authorization: Bearer <token>` | Claude Code CLI      |
| Token in path | `POST /mcp/<token>`             | Claude.ai connectors |

The `/health` endpoint is always open (no auth required).

Set `MCP_AUTH_TOKEN` to enable authentication. If unset, the server runs without auth (suitable for local development only).

### Nightscout Authentication

Connect to your Nightscout instance using **either**:

- **API secret** (`NIGHTSCOUT_API_SECRET`) — SHA1 hash of your plaintext secret:
  ```bash
  echo -n "your-secret" | shasum -a 1 | awk '{print $1}'
  ```
- **Readable token** (`NIGHTSCOUT_TOKEN`) — created in Nightscout Admin Tools

## Development

### Run locally (without Docker)

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn src.server:app --host 0.0.0.0 --port 8000 --log-config src/log_config.yaml
```

### Run tests

```bash
pip install pytest pytest-asyncio
pytest tests/
```

### Project structure

```
src/
  server.py             # MCP tools, auth middleware, health endpoint
  nightscout_client.py  # HTTP client for Nightscout REST API v1
  log_filter.py         # Filters /health from access logs
  log_config.yaml       # JSON logging configuration
tests/
  test_auth.py          # Auth middleware tests
```

### Adding a new tool

1. Add the API call to `src/nightscout_client.py` if needed
2. Add a `@mcp.tool()` function in `src/server.py` with a clear docstring
3. Test with `docker compose up --build`

## Environment Variables

| Variable                | Required | Default | Description                  |
| ----------------------- | -------- | ------- | ---------------------------- |
| `NIGHTSCOUT_URL`        | Yes      | —       | Nightscout instance base URL |
| `NIGHTSCOUT_API_SECRET` | No\*     | —       | API secret (SHA1 hash)       |
| `NIGHTSCOUT_TOKEN`      | No\*     | —       | Readable token               |
| `NIGHTSCOUT_TIMEOUT`    | No       | `30`    | Request timeout in seconds   |
| `MCP_AUTH_TOKEN`        | No       | —       | MCP server auth token        |

\* One of `NIGHTSCOUT_API_SECRET` or `NIGHTSCOUT_TOKEN` is required.

## License

[MIT](LICENSE) — Copyright (c) – 2026 B77.ai (EasyWeek GmbH)
