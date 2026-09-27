# SOP — Container Monitoring Stack (Netdata + Prometheus + Grafana + Email Alerts)

**Purpose:** Deploy a full container + host monitoring stack on any Linux server running Docker, with a Grafana dashboard and email alerts for process/PID leaks.
**Audience:** DevOps / sysadmins.
**Last updated:** 2026-09-27.
**Reference deployment:** any Linux host running Docker; Grafana optionally behind HTTPS (e.g. `https://grafana.example.com`).

> **Repo deployment:** this repository replaces `setup.sh` / `netdata-setup.sh` with a single `docker-compose.yml`. Clone the repo, run `cp .env.example .env`, edit `.env`, then `docker compose up -d` (see `README.md`). The file paths below map to the repo root instead of `~/monitoring/`. Everything else in this SOP (SMTP, recipients, alert tuning, troubleshooting) still applies.

---

## 1. What this stack is

| Component | Container | Role | Exposed |
|---|---|---|---|
| **Netdata** | `netdata` | Collects per-container + host metrics. Works with modern Docker (containerd snapshotter). | Host port **19999** |
| **Prometheus** | `mon_prometheus` | Scrapes Netdata + node-exporter, stores 7d/1GB. | Internal only |
| **node-exporter** | `mon_node_exporter` | Host CPU/RAM/swap/disk metrics. | Internal only |
| **Grafana** | `mon_grafana` | Dashboard UI + alerting engine + email. | Host port **3001** (public) |

**Data flow:**
```
containers ─┐
host       ─┴─► Netdata(:19999) ──► Prometheus ──► Grafana ──► Dashboard (browser)
                                        │                └────► Alert rule ──► SMTP ──► email
                node-exporter ─────────►┘
```

**Why Netdata and not cAdvisor:** cAdvisor cannot read Docker's **containerd snapshotter** layout (`Storage Driver: overlayfs`, `io.containerd.snapshotter.v1`) used by Docker 25+/29 — it fails to register containers and emits no per-container metrics. Netdata reads cgroups + the Docker socket directly and works. Check your server's driver with `docker info | grep -i 'storage driver'`; if it says `overlayfs`/containerd snapshotter, **do not** use cAdvisor.

---

## 2. Prerequisites

On the target server:

1. **Docker Engine** + **Docker Compose v2** (`docker compose version`). Install: https://docs.docker.com/engine/install/
2. **Root/sudo** (for firewall + reading host paths).
3. **Free ports:** `19999` (Netdata) and `3001` (Grafana). Change them if taken — see §4.4 / §9.
4. **Resources:** ~1–1.3 GiB RAM total for the stack. Check headroom with `free -h`.
5. **Outbound network** to pull images (Docker Hub, `gcr.io` not needed anymore, `quay`/`ghcr` no).
6. **(Optional) A domain + reverse proxy** if you want HTTPS like `grafana.example.com`.
7. **(Optional) SMTP mailbox** for alert emails (host, port, user, password) — see §6.

**Files you need (copy both to the server or run via SSH pipe):**
- `setup.sh` — Prometheus + Grafana + node-exporter + dashboard + alert rule.
- `netdata-setup.sh` — Netdata collector.

Copy them up, e.g.:
```bash
scp -P <ssh_port> setup.sh netdata-setup.sh user@SERVER:~/
```
…or run directly from your machine without copying:
```bash
ssh -p <ssh_port> user@SERVER 'bash -s' < netdata-setup.sh
```

---

## 3. Deployment — quick version

Run these on (or against) the target server, in order:

```bash
# 1) Netdata (the metrics collector)
ssh -p <PORT> <USER>@<SERVER> 'bash -s' < netdata-setup.sh

# 2) Prometheus + Grafana + node-exporter + dashboard + email alert
ssh -p <PORT> <USER>@<SERVER> \
  'GF_PORT=3001 \
   GF_ADMIN_PASSWORD="<choose-strong-pass>" \
   GF_SMTP_HOST="smtp.example.com:465" \
   GF_SMTP_USER="alerts@example.com" \
   GF_SMTP_FROM_ADDRESS="alerts@example.com" \
   GF_SMTP_PASSWORD="<smtp-password>" \
   GF_ALERT_TO="ops@example.com" \
   bash -s' < setup.sh

# 3) Open the firewall for Grafana (only if ufw is active)
ssh -p <PORT> <USER>@<SERVER> "sudo ufw allow 3001/tcp"
```

Then browse to `http://<SERVER>:3001` → login `admin` / `<chosen pass>` → dashboard **"Server Containers - Live"**.

> The scripts are **idempotent** — re-run them any time to change config; only changed containers are recreated. **Always pass the same `GF_PORT`, `GF_ADMIN_PASSWORD`, and `GF_SMTP_*` on every re-run**, or Grafana may try a different port / the printed password may drift (the actual admin password only changes on first init or via reset — see §11).

---

## 4. Deployment — detailed

### 4.1 Deploy Netdata
```bash
ssh -p <PORT> <USER>@<SERVER> 'bash -s' < netdata-setup.sh
```
Netdata runs with `network_mode: host`, so it listens on **`0.0.0.0:19999`** and can see all cgroups. It auto-discovers Docker containers via `/var/run/docker.sock`. Verify:
```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:19999/api/v1/info      # expect 200
curl -s 'http://localhost:19999/api/v1/charts' | grep -oE 'cgroup_[a-z0-9_.-]+' | sort -u | head
```

### 4.2 Deploy Prometheus + Grafana + node-exporter
Run `setup.sh` with the environment variables from §5. It writes everything under `~/monitoring/` on the server:
```
~/monitoring/
├── docker-compose.yml
├── .env                       # secrets (chmod 600)
├── prometheus.yml             # scrape config (netdata + node + prometheus)
└── grafana/
    ├── provisioning/
    │   ├── datasources/ds.yml            # Prometheus datasource (uid: PROM)
    │   ├── dashboards/provider.yml       # dashboard loader
    │   └── alerting/
    │       ├── contactpoints.yaml        # WHO gets emailed  <-- edit to add recipients
    │       └── rules.yaml                # the PID-trend alert rule
    └── dashboards/containers.json        # the dashboard
```

### 4.3 How Prometheus reaches Netdata
Netdata is on the **host** network; Prometheus is on a bridge. `setup.sh` adds
`extra_hosts: ["host.docker.internal:host-gateway"]` to Prometheus and the scrape target is
`host.docker.internal:19999` with `/api/v1/allmetrics?format=prometheus&filter=cgroup_*`,
then keeps only 3 metric families via `metric_relabel_configs` (keeps cardinality tiny).

### 4.4 Ports / conflicts
If `3001` or `19999` is taken:
- Grafana: run `setup.sh` with a different `GF_PORT=<n>` and open that port instead.
- Netdata: edit `netdata-setup.sh` (or Netdata config) — but `19999` is standard.
Find what's using a port: `ss -ltnp 'sport = :3001'` or `docker ps --format '{{.Names}} {{.Ports}}' | grep 3001`.

### 4.5 (Optional) HTTPS via reverse proxy
Point a subdomain (e.g. `grafana.example.com`) at the server and proxy it to `127.0.0.1:3001`.
Minimal nginx:
```nginx
server {
    server_name grafana.example.com;
    location / { proxy_pass http://127.0.0.1:3001; proxy_set_header Host $host; }
    listen 443 ssl;   # add certbot/Let's Encrypt cert
}
```
When behind HTTPS, also set `GF_SERVER_ROOT_URL=https://grafana.example.com` (add to the Grafana `environment:` in `docker-compose.yml`) if you hit login/redirect issues.

---

## 5. Configuration reference (environment variables)

Passed to `setup.sh` at run time; stored in `~/monitoring/.env`.

| Variable | Default | Meaning |
|---|---|---|
| `GF_PORT` | `3000` | Public port Grafana is published on. Use `3001` if 3000 is taken. |
| `GF_ADMIN_USER` | `admin` | Grafana admin username. |
| `GF_ADMIN_PASSWORD` | **required** | Grafana admin password (set on **first** DB init only; later use §11 to change). |
| `GF_SMTP_HOST` | (empty) | SMTP server `host:port`. See §6. |
| `GF_SMTP_USER` | (empty) | SMTP login (usually a full email). |
| `GF_SMTP_FROM_ADDRESS` | (empty) | "From" address on alert emails. |
| `GF_SMTP_PASSWORD` | (empty) | SMTP password / app-password. **Pass at run time, never commit.** |
| `GF_ALERT_TO` | (empty) | Recipient(s) for alerts. Semicolon-separate for many — see §7. |

> **Set these per server** in `.env` (see `.env.example`). Never commit `.env`.

---

## 6. Mail (SMTP) configuration — full detail

Grafana sends alert emails itself via SMTP. It's configured through the `GF_SMTP_*` variables, which `setup.sh` turns into these container env vars:

```yaml
GF_SMTP_ENABLED=true
GF_SMTP_HOST=<host:port>
GF_SMTP_USER=<login>
GF_SMTP_PASSWORD=<password>
GF_SMTP_FROM_ADDRESS=<from>
GF_SMTP_FROM_NAME=Grafana Alerts
GF_SMTP_SKIP_VERIFY=true        # accept self-signed/internal certs; set false for public CAs
```

### 6.1 Port / TLS rules
- **Port 465** = implicit SSL/TLS (Grafana 11 supports it).
- **Port 587** = STARTTLS — the most broadly compatible; prefer it if your server offers both.
- **Port 25** = usually blocked by cloud providers; avoid.
- `GF_SMTP_SKIP_VERIFY=true` disables certificate verification — needed for internal/self-signed mail servers. Set it to `false` (edit `docker-compose.yml`) if your provider has a valid public certificate.

### 6.2 Provider examples
| Provider | `GF_SMTP_HOST` | `GF_SMTP_USER` | Password |
|---|---|---|---|
| Gmail / Google Workspace | `smtp.gmail.com:587` | your full address | **App Password** (16 chars; 2FA required) |
| Microsoft 365 | `smtp.office365.com:587` | your full address | account or app password |
| Zoho / zmail | `smtp.zoho.com:465` or your `zmail.<domain>:465` | full mailbox address | mailbox / app password |
| Amazon SES | `email-smtp.<region>.amazonaws.com:587` | SES SMTP user | SES SMTP password |

> Gmail/O365 with 2FA **require an app-password**, not the normal login password.

### 6.3 Apply SMTP settings / change them later
Re-run `setup.sh` with the new `GF_SMTP_*` values (recommended), **or** edit `~/monitoring/.env` and restart Grafana:
```bash
cd ~/monitoring && nano .env          # edit GF_SMTP_* lines
docker compose up -d                  # picks up .env changes
```

### 6.4 Test email delivery
```bash
curl -s -o /dev/null -w 'HTTP %{http_code}\n' -u admin:<GF_ADMIN_PASSWORD> \
  -H 'Content-Type: application/json' \
  -X POST http://localhost:3001/api/alertmanager/grafana/config/api/v1/receivers/test \
  -d '{"receivers":[{"name":"email-ops","grafana_managed_receiver_configs":[{"name":"email-ops","type":"email","settings":{"addresses":"you@example.com","singleEmail":true}}]}],"alert":{"annotations":{"summary":"Test"},"labels":{"alertname":"TEST"}}}'
```
`HTTP 200` = SMTP works. On failure, read the reason:
```bash
docker logs --tail 40 mon_grafana 2>&1 | grep -iE 'smtp|mail|error'
```

---

## 7. Adding / changing alert recipients

The recipient list lives in **`~/monitoring/grafana/provisioning/alerting/contactpoints.yaml`**:

```yaml
apiVersion: 1
contactPoints:
  - orgId: 1
    name: email-ops
    receivers:
      - uid: email_ops_recv
        type: email
        settings:
          addresses: person1@example.com;person2@example.com;team@example.com   # <-- semicolon-separated
          singleEmail: true
policies:
  - orgId: 1
    receiver: email-ops
    group_by: ['alertname', 'cgroup_name']
```

**Three ways to add recipients:**

**A. Easiest — at deploy time:** pass a semicolon list to `setup.sh`:
```bash
GF_ALERT_TO="a@example.com;b@example.com;oncall@example.com" ... bash -s < setup.sh
```

**B. Edit the file on the server, then reload:**
```bash
nano ~/monitoring/grafana/provisioning/alerting/contactpoints.yaml   # edit addresses:
cd ~/monitoring && docker compose restart grafana                    # re-reads provisioning
```

**C. Multiple teams / routing (advanced):** add more contact points and route by label. Example — send `severity=critical` to on-call, everything else to ops:
```yaml
contactPoints:
  - orgId: 1
    name: email-ops
    receivers: [{uid: r_ops, type: email, settings: {addresses: "ops@example.com", singleEmail: true}}]
  - orgId: 1
    name: email-oncall
    receivers: [{uid: r_onc, type: email, settings: {addresses: "oncall@example.com", singleEmail: true}}]
policies:
  - orgId: 1
    receiver: email-ops
    routes:
      - receiver: email-oncall
        matchers: ['severity = critical']
```
`singleEmail: true` = one email listing all firing alerts; `false` = one email per alert.

> Provisioned contact points/policies are **read-only in the Grafana UI** (they're managed by files). To edit, change the YAML and restart Grafana.

---

## 8. The alert rule

File: `~/monitoring/grafana/provisioning/alerting/rules.yaml`. Rule **`pid-trend-up`**:

- **Query A:** `netdata_cgroup_pids_current_pids_average - (netdata_cgroup_pids_current_pids_average offset 30m)`
  → for each container, how many PIDs it gained in the last 30 minutes.
- **Condition C:** `A > 50` (gained more than 50 processes).
- **`for: 10m`** → must stay elevated 10 minutes (ignores brief spikes).
- **Result:** emails the contact point with `{{ $labels.cgroup_name }}`.
- Needs ~30 min of history before its first evaluation; `noDataState: OK` so it won't false-fire.

**Tune it:** edit `rules.yaml`, then `docker compose restart grafana`.
- Change sensitivity: the `params: [50]` under the threshold (lower = more sensitive).
- Change window: the `offset 30m` in the expr and `relativeTimeRange.from`.
- Change debounce: `for: 10m`.

**Add a hard-ceiling rule** (fire if any container exceeds 500 PIDs) — add another entry under `rules:` with expr `netdata_cgroup_pids_current_pids_average` and threshold `gt 500`.

**Other useful alerts** (same pattern, different expr):
- Host memory > 90%: `(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 90`
- Container mem > 2 GiB: `netdata_cgroup_mem_usage_MiB_average{dimension="ram"} > 2048`
- A container disappeared / down: `absent(...)` patterns.

---

## 9. The dashboard

File: `~/monitoring/grafana/dashboards/containers.json` (uid `server-containers`). Panels:
| Panel | Query |
|---|---|
| Running containers | `count(count by (cgroup_name)(netdata_cgroup_pids_current_pids_average))` |
| Host memory used % | `(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100` |
| Host swap used % | `(1 - node_memory_SwapFree_bytes / node_memory_SwapTotal_bytes) * 100` |
| CPU % per container | `sum by (cgroup_name)(netdata_cgroup_cpu_percentage_average)` |
| Memory per container | `netdata_cgroup_mem_usage_MiB_average{dimension="ram"} * 1048576` |
| PIDs per container (leak detector) | `netdata_cgroup_pids_current_pids_average` (red line at 500) |

Metric label to group/filter by is **`cgroup_name`** (the container name). Netdata also provides an `image` label. Set the dashboard time range to "Last 15–30 minutes" right after deploy (data is new).

Add a panel: edit `containers.json` (or build in the UI, then export JSON back into that file) and `docker compose restart grafana`.

---

## 10. Verification checklist

```bash
# all monitoring containers up
cd ~/monitoring && docker compose ps
docker ps --format '{{.Names}} {{.Status}}' | grep -E 'netdata|mon_'

# Grafana reachable + login
curl -s -o /dev/null -w '%{http_code}\n' -u admin:<pass> http://localhost:3001/api/health   # 200

# Prometheus scraping all targets (expect value "3": netdata, node, prometheus)
docker exec mon_prometheus wget -qO- 'http://localhost:9090/api/v1/query?query=sum(up)'

# per-container metrics present (expect ~ number of containers)
docker exec mon_prometheus wget -qO- 'http://localhost:9090/api/v1/query?query=count(netdata_cgroup_pids_current_pids_average)'

# alert rule + contact point loaded
curl -s -u admin:<pass> http://localhost:3001/api/v1/provisioning/alert-rules | grep pid-trend-up
curl -s -u admin:<pass> http://localhost:3001/api/v1/provisioning/contact-points | grep email-ops
```

---

## 11. Common operations

```bash
cd ~/monitoring     # (or ~/netdata for netdata)

docker compose ps                    # status
docker compose logs -f grafana       # follow logs (or prometheus / netdata)
docker compose pull && docker compose up -d   # update images
docker compose restart grafana       # reload provisioning after editing YAML/JSON
docker compose down                  # stop (keeps volumes/data)
docker compose down -v               # stop AND delete data volumes (wipes history)

# Reset Grafana admin password (works regardless of current value):
docker exec mon_grafana grafana-cli --homepath=/usr/share/grafana admin reset-admin-password 'NEW_PASS'
```

---

## 12. Troubleshooting

| Symptom | Cause / Fix |
|---|---|
| `Bind for 0.0.0.0:3000 failed: port is already allocated` | Port taken. Re-run `setup.sh` with a different `GF_PORT`. |
| Dashboard panels empty (containers) | Netdata not scraped. Check `up{job="netdata"}` = 1; confirm Prometheus can reach `host.docker.internal:19999` (needs the `extra_hosts` line); confirm Netdata is up. |
| Grafana login "invalid password" | Admin password only set on first init. Reset with `grafana-cli` (§11). |
| Test email fails / SMTP error in log | Wrong password / port / TLS. Try port `587`; for internal certs set `GF_SMTP_SKIP_VERIFY=true`; Gmail/O365 need an **app password**. |
| Netdata sees 0 containers | Ensure `/var/run/docker.sock` is mounted (it is in `netdata-setup.sh`); wait ~60s after start. |
| cAdvisor "failed to identify the read-write layer ID" | Docker uses containerd snapshotter — cAdvisor is incompatible. This stack uses Netdata instead; don't add cAdvisor. |
| Alert never fires | Needs 30 min of history first; check the rule state in Grafana → Alerting → Alert rules. |
| Prometheus using too much RAM | Retention is capped 7d/1GB; the `filter=cgroup_*` + `metric_relabel_configs keep` limit series. Don't remove those. |

---

## 13. Security hardening (do before exposing publicly)

1. **Strong Grafana admin password** (`GF_ADMIN_PASSWORD`); change from any default. Rotate if it ever leaks.
2. **Do NOT expose Netdata `:19999` publicly** — it has no login and reveals system internals. Firewall it to trusted IPs; Prometheus reaches it internally regardless:
   ```bash
   sudo ufw deny 19999/tcp
   sudo ufw allow from <your.office.ip> to any port 19999 proto tcp
   ```
3. **Put Grafana behind HTTPS** (reverse proxy + Let's Encrypt) rather than plain `:3001`. Restrict the port by IP if possible.
4. **Never commit `.env`** or the SMTP password to git. `.env` is chmod 600 on the server.
5. Keep `GF_USERS_ALLOW_SIGN_UP=false` and `GF_AUTH_ANONYMOUS_ENABLED=false` (already set).
6. Prometheus/node-exporter are internal-only — keep them off published ports.

---

## 14. Uninstall

```bash
cd ~/monitoring && docker compose down -v      # remove stack + data
cd ~/netdata    && docker compose down -v      # remove netdata + data
sudo ufw delete allow 3001/tcp                 # close the port
rm -rf ~/monitoring ~/netdata
```

---

## Appendix — the two scripts

The source of truth is:
- `setup.sh` — Prometheus + Grafana + node-exporter + dashboard + alert rule + SMTP.
- `netdata-setup.sh` — Netdata collector.

Keep both under version control (minus secrets). To replicate on a new server: copy both scripts, then follow §3 with that server's values.
