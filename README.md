# 📊 toolkit-monitoring

Monitoring stack for [toolkit](https://github.com/nulldeploy/toolkit). Metrics, logs, and Telegram alerts — all in one `docker compose up`.

---

## Architecture

```
VPS (application)                   Monitoring server / laptop
                                    
┌─────────────────┐                ┌──────────────────────────┐
│  toolkit app    │                │                          │
│                 │                │  Prometheus :9090        │
│  Node Exporter  │ ←── scrape ────│  (collects metrics)      │
│  :9100          │                │           │              │
└─────────────────┘                │           ▼              │
                                   │  Alertmanager :9093      │
                                   │  (alerts → Telegram)     │
                                   │                          │
                                   │  Loki :3100              │
                                   │  (stores logs)           │
                                   │           ▲              │
                                   │           │ push         │
                                   │  Alloy :12345            │
                                   │  (collects Docker logs)  │
                                   │                          │
                                   │  Grafana :3000           │
                                   │  (metrics + logs UI)     │
                                   └──────────────────────────┘
```

---

## Stack

| Service | Purpose | Port |
|---------|---------|------|
| Prometheus | Metrics collection and storage (pull) | 9090 |
| Node Exporter | Host system metrics | 9100 (on VPS) |
| Alertmanager | Alert routing → Telegram | 9093 |
| Loki | Log storage (push) | 3100 |
| Grafana Alloy | Docker log collection agent | 12345 |
| Grafana | Metrics and logs visualization | 3000 |

---

## Structure

```
toolkit-monitoring/
├── prometheus/
│   ├── prometheus.yml      # scrape config + alerting
│   └── alerts.yml          # alert rules (PromQL)
├── alertmanager/
│   └── alertmanager.yml    # routing → Telegram (not in git)
├── loki/
│   └── config.yml          # log storage config
├── alloy/
│   └── config.alloy        # agent config (River syntax)
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── datasources.yml  # auto-connect Prometheus + Loki
└── docker-compose.yml
```

---

## Quick Start

### 1. Secrets

```bash
cp alertmanager/alertmanager.example.yml alertmanager/alertmanager.yml
# fill in bot_token and chat_id
```

### 2. Run

```bash
docker compose up -d
docker compose ps
```

### 3. Verify

| Service | URL |
|---------|-----|
| Prometheus | http://localhost:9090 |
| Prometheus Alerts | http://localhost:9090/alerts |
| Alertmanager | http://localhost:9093 |
| Loki | http://localhost:3100/ready |
| Alloy UI | http://localhost:12345 |
| Grafana | http://localhost:3000 |

Grafana login: `devops` / password from `.env`

---

## Alerts

Three rules configured in `prometheus/alerts.yml`:

| Alert | Condition | Severity |
|-------|-----------|----------|
| `ServiceDown` | Target unreachable > 1 minute | critical |
| `HighCPU` | CPU > 80% > 2 minutes | warning |
| `LowDiskSpace` | Disk < 15% free > 5 minutes | warning |

On trigger — Telegram message via Alertmanager.

---

## Grafana Datasources

Connected automatically on startup via provisioning:

- **Prometheus** — metrics, query language: PromQL
- **Loki** — Docker container logs, query language: LogQL

---

## Configuration

`.env` for Grafana:

```bash
cp .env.example .env
```

```env
GF_SECURITY_ADMIN_USER=devops
GF_SECURITY_ADMIN_PASSWORD=your_password
```

---

## Useful Commands

```bash
# Status of all services
docker compose ps

# Recreate a specific service after config change
docker compose up -d --force-recreate prometheus

# Verify Prometheus sees Alertmanager
curl http://localhost:9090/api/v1/alertmanagers

# Check loaded rules
curl http://localhost:9090/api/v1/rules

# Service logs
docker logs toolkit-prometheus -f
docker logs toolkit-loki -f
docker logs toolkit-alloy -f
```

---

## Related Repositories

- [toolkit](https://github.com/nulldeploy/toolkit) — main application
- [toolkit-infra](https://github.com/nulldeploy/toolkit-infrastructure) — Ansible deployment automation
