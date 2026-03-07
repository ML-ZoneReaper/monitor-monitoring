# Monitor Monitoring ![Stars](https://img.shields.io/github/stars/ML-ZoneReaper/monitor-monitoring?style=flat) [![Go Report Card](https://goreportcard.com/badge/github.com/ML-ZoneReaper/monitor-monitoring)](https://goreportcard.com/report/github.com/ML-ZoneReaper/monitor-monitoring) ![GitHub release (latest SemVer)](https://img.shields.io/github/v/release/ML-ZoneReaper/monitor-monitoring?sort=semver) ![GitHub go.mod Go version](https://img.shields.io/github/go-mod/go-version/ML-ZoneReaper/monitor-monitoring) ![License](https://img.shields.io/github/license/ML-ZoneReaper/monitor-monitoring)

Lightweight monitoring tool (~6MB binary). Checks HTTP/HTTPS endpoints, DNS records, and TCP ports. Sends alerts via Telegram, Slack, Discord, or Mattermost.

## Features

- **Multi-protocol** — HTTP/HTTPS, DNS (A/AAAA/CNAME), TCP
- **Smart alerting** — configurable failure threshold + automatic retry before alerting
- **Batch notifications** — anti-spam window groups events before sending
- **Fallback notifier** — secondary channel if primary fails
- **Graceful shutdown** — SIGTERM flushes pending notifications and sends shutdown alert
- **12-factor config** — all settings via environment variables

## Quick Start

```bash
git clone https://github.com/ML-ZoneReaper/monitor-monitoring.git
cd monitor-monitoring
CGO_ENABLED=0 go build -ldflags="-s -w" -o monitor-monitoring .
```

Create `config.yaml` (see [Configuration](#yaml-configuration)), then run:

```bash
export TELEGRAM_BOT_TOKEN="..."
export TELEGRAM_CHAT_ID="..."
./monitor-monitoring
```

## Notification Channels

Set `PRIMARY_NOTIFIER` (default: `telegram`) and optionally `FALLBACK_NOTIFIER`.

| Channel | Required env vars |
|---|---|
| `telegram` | `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID` |
| `slack` | `SLACK_WEBHOOK_URL` |
| `discord` | `DISCORD_WEBHOOK_URL` |
| `mattermost` | `MATTERMOST_WEBHOOK_URL` |

**Get Telegram credentials:**
- Bot token: message [@BotFather](https://t.me/BotFather), send `/newbot`
- Chat ID: add [@userinfobot](https://t.me/userinfobot) to your group, or call `https://api.telegram.org/bot<TOKEN>/getUpdates`

**Example with fallback:**
```bash
export PRIMARY_NOTIFIER="telegram"
export FALLBACK_NOTIFIER="slack"
export TELEGRAM_BOT_TOKEN="..."
export TELEGRAM_CHAT_ID="..."
export SLACK_WEBHOOK_URL="..."
```

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `PRIMARY_NOTIFIER` | `telegram` | Primary channel: `telegram`, `slack`, `discord`, `mattermost` |
| `FALLBACK_NOTIFIER` | — | Fallback channel if primary fails |
| `TELEGRAM_BOT_TOKEN` | — | Telegram bot token |
| `TELEGRAM_CHAT_ID` | — | Telegram chat/group/channel ID |
| `SLACK_WEBHOOK_URL` | — | Slack incoming webhook URL |
| `DISCORD_WEBHOOK_URL` | — | Discord webhook URL |
| `MATTERMOST_WEBHOOK_URL` | — | Mattermost incoming webhook URL |
| `CHECK_INTERVAL` | `60s` | Interval between check cycles |
| `REQUEST_TIMEOUT` | `10s` | HTTP request timeout |
| `RETRY_DELAY` | `3s` | Delay before retry on failed check |
| `FAILURE_THRESHOLD` | `2` | Consecutive failures before alert |
| `NOTIFY_BATCH_WINDOW` | `40s` | Max wait before sending a batch |
| `MAX_BATCH_SIZE` | `50` | Max events per notification |
| `MAX_CONCURRENT_CHECKS` | `20` | Max parallel checks |
| `MAX_RESPONSE_BODY_SIZE` | `524288` | Max HTTP response body to drain (bytes) |
| `DNS_TIMEOUT` | `5s` | DNS query timeout |
| `TCP_TIMEOUT` | `8s` | TCP connection timeout |
| `CONFIG_PATH` | `config.yaml` | Path to YAML config file |
| `MONITOR_HOSTNAME` | _(auto)_ | Hostname shown in notifications |

## YAML Configuration

### HTTP/HTTPS

```yaml
endpoints:
  - url: https://example.com              # required
    type: http                            # optional, default: http
    method: GET                           # optional, default: GET
    expected_status: 200                  # optional, default: 200
    headers:                              # optional
      Authorization: Bearer token
```

Any valid HTTP method is accepted (`GET`, `POST`, `PUT`, `DELETE`, `HEAD`, `PATCH`, `OPTIONS`, etc.).

### DNS

```yaml
endpoints:
  - type: dns
    host: example.com
    record_type: A                        # required: A, AAAA, or CNAME
    expected: 192.0.2.1                   # optional: validate specific value
```

Supported record types: `A` (IPv4), `AAAA` (IPv6), `CNAME`.

### TCP

```yaml
endpoints:
  - type: tcp
    host: db.example.com
    port: 5432

  # or using address directly
  - type: tcp
    address: "redis.example.com:6379"
```

## Docker

```dockerfile
FROM golang:1.26-alpine AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o monitor-monitoring .

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/monitor-monitoring .
COPY config.yaml .
CMD ["./monitor-monitoring"]
```

```yaml
# docker-compose.yml
services:
  monitor:
    build: .
    env_file: .env
    volumes:
      - ./config.yaml:/root/config.yaml
    restart: unless-stopped
```

## Systemd

```ini
# /etc/systemd/system/monitor-monitoring.service
[Unit]
Description=Monitor Monitoring
After=network.target

[Service]
Type=simple
User=monitor
WorkingDirectory=/opt/monitor-monitoring
ExecStart=/opt/monitor-monitoring/monitor-monitoring
Restart=always
RestartSec=10
EnvironmentFile=/opt/monitor-monitoring/.env
MemoryMax=256M
CPUQuota=10%

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now monitor-monitoring
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| No notifications | Check token/webhook URL; verify bot is admin in Telegram group |
| False positives | Increase `FAILURE_THRESHOLD` or `REQUEST_TIMEOUT` |
| Delayed alerts | Decrease `NOTIFY_BATCH_WINDOW` |
| High resource usage | Lower `MAX_CONCURRENT_CHECKS` and `MAX_RESPONSE_BODY_SIZE` |
| DNS checks failing | Ensure host's DNS resolver is reachable from monitoring machine |
