# Container Monitoring Stack

[![validate](https://github.com/Satyamkan10/container-monitoring/actions/workflows/validate.yml/badge.svg)](https://github.com/Satyamkan10/container-monitoring/actions/workflows/validate.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

Netdata + Prometheus + node-exporter + Grafana, with a ready-made dashboard and an email alert for container process/PID leaks. One `docker compose up -d` on any Linux server running Docker.

```
containers ─┐
host       ─┴─► Netdata(:19999) ──► Prometheus ──► Grafana(:GF_PORT) ──► dashboard
                                        ▲                 └──► alert rule ──► SMTP ──► email
                node-exporter ──────────┘
```

## Quick start

```bash
git clone https://github.com/Satyamkan10/container-monitoring.git monitoring
cd monitoring
cp .env.example .env
nano .env                     # set GF_ADMIN_PASSWORD, SMTP settings, GF_ALERT_TO, GF_PORT
docker compose up -d
sudo ufw allow 3001/tcp       # only if ufw is active; use your GF_PORT
```

Open `http://<server>:<GF_PORT>` and log in as `admin` with your password. The dashboard is **Server Containers - Live**.

**Requirements:** Docker Engine + Compose v2, a free `19999` and `GF_PORT` port, and about 1.3 GiB RAM.

## What's in the repo

| Path | Purpose |
|---|---|
| `docker-compose.yml` | All four services |
| `.env.example` | Config template. Copy to `.env`, which is git-ignored. |
| `prometheus/prometheus.yml` | Scrapes Netdata (cgroup metrics only), node-exporter, and Prometheus itself |
| `grafana/provisioning/datasources/ds.yml` | Prometheus datasource (uid `PROM`) |
| `grafana/provisioning/alerting/contactpoints.yaml` | Who gets emailed (from `GF_ALERT_TO`) |
| `grafana/provisioning/alerting/rules.yaml` | `pid-trend-up` rule: a container gained more than 50 PIDs in 30m, sustained for 10m |
| `grafana/dashboards/containers.json` | The dashboard |
| `docs/SOP.md` | Full operating procedure: SMTP, recipients, tuning, troubleshooting, security |

## Common operations

```bash
docker compose ps                              # status
docker compose logs -f grafana                 # logs
docker compose restart grafana                 # reload after editing provisioning YAML/JSON
docker compose up -d                           # apply .env changes
git pull && docker compose pull && docker compose up -d   # update
docker compose down                            # stop (keeps data)
docker compose down -v                         # stop + wipe data

# reset Grafana admin password (GF_ADMIN_PASSWORD only applies on first start)
docker exec mon_grafana grafana-cli --homepath=/usr/share/grafana admin reset-admin-password 'NEW_PASS'
```

## Notes

- **Netdata instead of cAdvisor:** cAdvisor can't read Docker's containerd snapshotter (Docker 25+), so it produces no per-container metrics there.
- **Don't expose `:19999` publicly.** Netdata has no login. Prometheus reaches it internally.
- For HTTPS, put Grafana behind a reverse proxy and set `GF_SERVER_ROOT_URL` in `docker-compose.yml`.
- The alert needs about 30 minutes of history before it can fire.

See [`docs/SOP.md`](docs/SOP.md) for the full guide.

## License

[MIT](LICENSE)
