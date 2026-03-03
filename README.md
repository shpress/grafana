# Grafana Monitoring Stack

Centralized monitoring infrastructure for a 5-server homelab using Grafana, Prometheus, Loki, and various exporters.

## Architecture

```
                    ┌─────────────────────────────────────────┐
                    │          ZimaCube .222 (Hub)             │
                    │                                         │
                    │  ┌──────────┐  ┌────────────┐          │
                    │  │ Grafana  │  │ Prometheus  │          │
                    │  │  :3333   │  │   :9090     │          │
                    │  └──────────┘  └─────┬──────┘          │
                    │  ┌──────────┐        │ scrapes          │
                    │  │   Loki   │◄───────┼─── Promtail     │
                    │  │  :3100   │        │    (all 5)       │
                    │  └──────────┘        │                  │
                    └──────────────────────┼──────────────────┘
                                           │
              ┌────────────┬───────────────┼──────────┬──────────────┐
              ▼            ▼               ▼          ▼              ▼
         ┌─────────┐ ┌─────────┐    ┌──────────┐ ┌────────┐  ┌──────────┐
         │  .222   │ │  .224   │    │  .228    │ │  .46   │  │  Oracle  │
         │node-exp │ │node-exp │    │node-exp  │ │node-exp│  │node-exp  │
         │cAdvisor │ │cAdvisor │    │cAdvisor  │ │cAdvisor│  │cAdvisor  │
         │nvidia   │ │         │    │          │ │        │  │          │
         │speedtest│ │         │    │          │ │        │  │          │
         │pihole-  │ │         │    │          │ │        │  │          │
         │exporter │ │         │    │          │ │        │  │          │
         └─────────┘ └─────────┘    └──────────┘ └────────┘  └──────────┘
```

## Servers Monitored

| Server | Role | IP | Exporters |
|--------|------|-----|-----------|
| ZimaCube .222 | GPU compute, Plex, Immich | 192.168.2.222 | node-exporter, cAdvisor, nvidia-gpu, speedtest, pihole-exporter |
| ZimaCube .224 | Download automation (*arr) | 192.168.2.224 | node-exporter, cAdvisor |
| ZimaCube .228 | DNS (Pi-hole) | 192.168.2.228 | node-exporter, cAdvisor (128MB limit) |
| Synology .46 | NAS, media storage | 192.168.2.46 | node-exporter, cAdvisor |
| Oracle Cloud | Public gateway, reverse proxy | 100.99.150.34 (Tailscale) | node-exporter, cAdvisor |

## Components

### Prometheus (port 9090)
- 90-day data retention
- Scrapes 13 targets across 5 servers
- Config: `prometheus/prometheus.yml`

### Grafana (port 3333)
- Public URL: https://dashboard.spressman.me
- Provisioned datasources: Prometheus + Loki
- 10 dashboards (provisioned from JSON)

### Loki (port 3100)
- Log aggregation from all 5 servers via Promtail
- TSDB storage with filesystem backend
- 30-day retention with compaction

### Dashboards

| Dashboard | Description |
|-----------|-------------|
| Infrastructure Command Center | Main overview — server stats, CPU/memory/network, Docker containers |
| Pi-hole DNS | DNS query stats, ads blocked, filter rate |
| Log Explorer | Centralized log viewer with server/container filters |
| Docker Containers | Per-container resource usage |
| GPU — RTX 3050 | NVIDIA GPU metrics (utilization, VRAM, temp, power) |
| Internet Speed | Speedtest results over time |
| Network | Network traffic by server/interface |
| Server Overview | Per-server detailed stats |
| Storage | Disk usage across all servers |
| Electricity Cost | Power consumption estimates |

## Network Connectivity

- Local servers (.222, .224, .228, .46) communicate via LAN (192.168.2.0/24)
- Oracle Cloud connects via Tailscale mesh VPN
- .222 Tailscale runs in **kernel TUN mode** (`TS_USERSPACE=false`) for proper routing
- Promtail on Oracle pushes to Loki via Tailscale IP (100.127.112.49:3100)

## Deployment Notes

### .222 (Hub)
The main Grafana stack runs via Komodo at `/etc/komodo/stacks/grafana_222/compose.yaml`. Loki, Promtail, and pihole-exporter run as standalone Docker containers connected to `grafana_222_default` network.

### .228 (Lightweight)
cAdvisor has a 128MB memory limit. Promtail has a 64MB memory limit. No docker compose — containers deployed via `docker run`.

### Synology .46
Docker requires `sudo` with full path (`/usr/local/bin/docker`). Docker root is at `/volume3/@docker` (not `/var/lib/docker`). No `/dev/kmsg` device. cAdvisor uses `/volume3/@docker` for container data.

### Oracle Cloud (ARM64)
All images must be multi-arch (amd64+arm64). Ports 9100/9080 opened in iptables for Prometheus scraping. Node-exporter uses `network_mode: host`.

## File Structure

```
grafana/
├── prometheus/
│   └── prometheus.yml          # Prometheus scrape config
├── grafana/
│   └── provisioning/
│       ├── datasources/
│       │   ├── prometheus.yml  # Prometheus datasource
│       │   └── loki.yaml       # Loki datasource
│       └── dashboards/
│           ├── dashboards.yml  # Dashboard provisioning config
│           ├── infrastructure-beautiful.json
│           ├── pihole.json
│           ├── logs.json
│           ├── docker-containers.json
│           ├── gpu.json
│           ├── internet-speed.json
│           ├── network.json
│           ├── server-overview.json
│           ├── storage.json
│           └── electricity-cost.json
├── loki/
│   └── loki-config.yaml        # Loki server config
├── promtail/
│   └── configs/
│       ├── promtail-222.yaml
│       ├── promtail-224.yaml
│       ├── promtail-228.yaml
│       ├── promtail-46.yaml
│       └── promtail-oracle.yaml
├── stacks/
│   └── grafana-222.yaml        # Main Grafana compose (Komodo)
└── oracle-monitoring/
    └── compose.yaml            # Oracle node-exporter + cAdvisor
```
