# Internet Monitoring Stack

Self-hosted network monitoring with Prometheus, Blackbox Exporter, and Grafana. Probes ICMP, HTTP, and DNS targets every 15 seconds and ships alerts to Telegram when connectivity drops.

## Architecture

```
Prometheus ──scrape──▶ Blackbox Exporter ──probe──▶ Targets
     │                       (ICMP/HTTP/DNS)
     │
     ◀──query──────── Grafana ──alert──▶ Telegram
```

| Service | Port | Purpose |
|---------|------|---------|
| Prometheus | `9090` | Metrics collection and storage |
| Blackbox Exporter | `9115` | Active probing (ping, HTTP, DNS) |
| Grafana | `3000` | Dashboards and alerting |

## Quick Start

```bash
# 1. Copy the environment template and fill in your Telegram credentials
cp .env.example .env

# 2. Edit .env with your bot token and chat ID
#    TELEGRAM_BOT_TOKEN=123456789:ABCdefGHIjklMNOpqrsTUVwxyz
#    TELEGRAM_CHAT_ID=123456789

# 3. Start the stack
docker compose up -d

# 4. Open the dashboard
open http://localhost:3000
```

Default login: `admin` / `admin`

## Monitored Targets

| Job | Method | Targets |
|-----|--------|---------|
| `blackbox_icmp` | ICMP ping | `1.1.1.1`, `8.8.8.8` |
| `blackbox_http` | HTTP GET | `google.com`, `youtube.com`, `facebook.com`, `instagram.com`, `github.com` |
| `blackbox_dns` | DNS resolve | `1.1.1.1`, `8.8.8.8` |

All probes run at a 15-second interval. DNS probes resolve `example.com` against each target server.

## Dashboard

The **Internet Connectivity** dashboard is auto-provisioned and includes:

- **Target Status** — up/down indicator for all probed endpoints
- **Ping Latency** — ICMP round-trip time over time
- **HTTP Latency** — HTTP response time over time
- **DNS Resolution Time** — DNS query duration over time
- **Uptime Percentage** — 7-day rolling average of probe success

Refreshes every 15 seconds, default view is last 6 hours.

## Telegram Alerts

Alerts are routed to a Telegram contact point with these settings:

| Setting | Value |
|---------|-------|
| Group wait | 30s |
| Group interval | 5m |
| Repeat interval | 1h |
| Grouping | by `alertname` and `instance` |

To set up the Telegram bot:

1. Create a bot via [@BotFather](https://t.me/BotFather) and copy the token
2. Get your chat ID by messaging [@userinfobot](https://t.me/userinfobot) or using the API:  
   `curl "https://api.telegram.org/bot<TOKEN>/getUpdates?offset=-1"`
3. Add both values to `.env`

> **Note:** Alert rules are not provisioned by default. Create them in Grafana under **Alerting → Alert rules** after the stack is running.

## Project Structure

```
.
├── docker-compose.yml          # Service definitions
├── .env.example                # Template for Telegram credentials
├── prometheus/
│   └── prometheus.yml          # Scrape config (15s interval, 999d retention)
├── blackbox/
│   └── blackbox.yml            # Probe modules (icmp, http, dns)
└── grafana/
    ├── grafana.ini             # Server config (anonymous access off, sign-up off)
    └── provisioning/
        ├── alerting/
        │   └── alerting.yml    # Telegram contact point + notification policy
        ├── dashboards/
        │   ├── dashboard.yml   # Dashboard provisioning config
        │   └── internet-check.json
        ├── datasources/        # (auto-detected: Prometheus on same network)
        └── plugins/
```

## Configuration

### Add or Remove Targets

Edit `prometheus/prometheus.yml` and restart:

```bash
docker compose restart prometheus
```

### Change Probe Behavior

Edit `blackbox/blackbox.yml` (timeouts, DNS query names, HTTP status codes) and restart:

```bash
docker compose restart blackbox
```

### Change Alert Routing

Edit `grafana/provisioning/alerting/alerting.yml` and restart:

```bash
docker compose restart grafana
```

### Data Retention

Prometheus is configured for **999 days** of retention via the `--storage.tsdb.retention.time` flag. Data persists in the `prometheus_data` Docker volume across restarts.

## Stopping

```bash
docker compose down        # Stop containers (data preserved in volumes)
docker compose down -v     # Stop and delete all stored metrics
```