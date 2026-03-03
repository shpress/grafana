# CLAUDE.md — Grafana Monitoring Stack

## What This Is
Configuration backup for a Grafana/Prometheus/Loki monitoring stack spanning 5 servers. The hub runs on ZimaCube .222 (192.168.2.222). This repo contains only configs and dashboards — no application code.

## Key Architecture Decisions

- **Prometheus** scrapes all metrics (pull model) from local network + Tailscale
- **Loki** receives logs via push from Promtail agents on each server
- **Grafana** is provisioned entirely from files (datasources + dashboards as JSON)
- **Oracle Cloud** connects via Tailscale (kernel TUN mode on .222, `TS_USERSPACE=false`)
- **Synology** has special Docker paths (`/volume3/@docker`, requires `sudo /usr/local/bin/docker`)

## Server Connectivity from .222

| Server | How Prometheus reaches it |
|--------|--------------------------|
| .224 | LAN 192.168.2.224:9100/9080 |
| .228 | LAN 192.168.2.228:9100/9080 |
| .46 | LAN 192.168.2.46:9100/9080 |
| Oracle | Tailscale 100.99.150.34:9100/9080 |

## Prometheus Targets (13 total)

- `node-222`, `node-224`, `node-228`, `node-46`, `node-oracle` — system metrics
- `cadvisor-222`, `cadvisor-224`, `cadvisor-228`, `cadvisor-46`, `cadvisor-oracle` — container metrics
- `nvidia-222` — GPU metrics (RTX 3050)
- `speedtest` — internet speed (30min interval)
- `pihole` — Pi-hole DNS metrics (192.168.2.220)

## Ports

| Service | Port | Server |
|---------|------|--------|
| Grafana | 3333 | .222 |
| Prometheus | 9090 | .222 |
| Loki | 3100 | .222 |
| node-exporter | 9100 | all |
| cAdvisor | 9080 | all |
| nvidia-gpu-exporter | 9835 | .222 |
| speedtest-exporter | 9798 | .222 |
| pihole-exporter | 9617 | .222 |
| Grafana renderer | 8081 | .222 |

## Dashboard Color Scheme

| Server | Color |
|--------|-------|
| .222 | #4ecdc4 (teal) |
| .224 | #ff6b6b (coral) |
| .228 | #ffd93d (yellow) |
| .46 | #6bcf7f (green) |
| Oracle | #9b59b6 (purple) |

## Working with Dashboards

Dashboard JSON files are in `grafana/provisioning/dashboards/`. They are provisioned automatically by Grafana on startup. To update:

1. Edit the JSON file
2. Restart Grafana or wait for file watcher to pick up changes
3. Validate JSON: `python3 -c 'import json; json.load(open("file.json"))'`

## Deployment

The main stack on .222 is managed by Komodo (`/etc/komodo/stacks/grafana_222/`). Additional containers (Loki, Promtail, pihole-exporter) run standalone but connect to `grafana_222_default` Docker network.

To sync configs back to .222:
```bash
scp prometheus/prometheus.yml zima222:/media/sda/docker/prometheus/prometheus.yml
scp grafana/provisioning/dashboards/*.json zima222:/media/sda/docker/grafana/provisioning/dashboards/
ssh zima222 "curl -X POST http://localhost:9090/-/reload"  # reload prometheus
ssh zima222 "docker restart grafana"  # reload dashboards
```

## Sensitive Values (NOT in repo)

- Grafana admin password (set via env var `GF_SECURITY_ADMIN_PASSWORD`)
- Pi-hole password (set via env var `PIHOLE_PASSWORD` on pihole-exporter container)
- Synology SSH password
