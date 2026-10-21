# Grafana Basics — Advanced Production Edition

[![Grafana Version](https://img.shields.io/badge/Grafana-v10.x-F46800?logo=grafana&logoColor=white)](https://grafana.com/)
[![Prometheus Version](https://img.shields.io/badge/Prometheus-v2.50.1-E6522C?logo=prometheus&logoColor=white)](https://prometheus.io/)
[![Node Exporter](https://img.shields.io/badge/Node_Exporter-v1.7.0-blue?logo=linux&logoColor=white)](https://github.com/prometheus/node_exporter)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](https://opensource.org/licenses/MIT)

A production-oriented operational guide for deploying Grafana, aggregating host metrics via Prometheus and Node Exporter, authoring system utilization dashboards, validating every pipeline layer, and troubleshooting common infrastructure failures.

---

## Table of Contents

- [1. Architectural Overview](#1-architectural-overview)
- [2. Prerequisites & Assumptions](#2-prerequisites--assumptions)
- [3. Step 1 — Metrics Collection Layer](#3-step-1--metrics-collection-layer)
  - [3.1 Node Exporter Setup](#31-node-exporter-setup)
  - [3.2 Prometheus Setup](#32-prometheus-setup)
- [4. Step 2 — Installing Grafana](#4-step-2--installing-grafana)
- [5. Step 3 — Provisioning Prometheus in Grafana](#5-step-3--provisioning-prometheus-in-grafana)
  - [Method A: Grafana Web UI](#method-a-grafana-web-ui)
  - [Method B: Declarative File-Based Provisioning](#method-b-declarative-file-based-provisioning)
- [6. Step 4 — Building Dashboards & PromQL Deep Dive](#6-step-4--building-dashboards--promql-deep-dive)
  - [6.1 Constructing the CPU Panel](#61-constructing-the-cpu-panel)
  - [6.2 PromQL Query Breakdown](#62-promql-query-breakdown)
  - [6.3 Key Baseline Host Metrics](#63-key-baseline-host-metrics)
- [7. End-to-End Verification Checklist](#7-end-to-end-verification-checklist)
- [8. Troubleshooting Matrix](#8-troubleshooting-matrix)
- [9. Production Hardening & Operational Runbook](#9-production-hardening--operational-runbook)
- [10. Change Log: Basic vs. Advanced Edition](#10-change-log-basic-vs-advanced-edition)

---

## 1. Architectural Overview

Prometheus uses a **pull-based telemetry architecture**. Node Exporter translates kernel and hardware statistics into Prometheus-compatible metrics. Prometheus periodically pulls (`scrapes`) these metrics and indexes them into its Time Series Database (TSDB). Grafana queries Prometheus via PromQL and presents the visualizations.

```
┌─────────────────────┐       HTTP GET /metrics       ┌─────────────────────┐         PromQL Query        ┌─────────────────────┐
│    Node Exporter    │ ◄───────────────────────────  │     Prometheus      │ ◄────────────────────────── │       Grafana       │
│     Port :9100      │   (Scrape: 5s poll loop)      │     Port :9090      │     (HTTP API / Port 3000)  │     Port :3000      │
└─────────────────────┘                               └─────────────────────┘                             └─────────────────────┘
           ▲                                                     ▲
           │ Gathers metrics                                     │ Writes TSDB blocks
           │                                                     │
   [ Linux Kernel ]                                    [/var/lib/prometheus]
   /proc, /sys filesystem                              WAL & Block Storage
```

> **Core Concept:** Exporters are passive metric servers. No data is pushed from Node Exporter to Prometheus. Target availability is evaluated via scrape success states.

---

## 2. Prerequisites & Assumptions

- **Environment:** Dedicated Linux VM / bare-metal server (Debian/Ubuntu-derived distributions assumed).
- **Privileges:** `sudo` root escalation access with `systemd` supervisor management.
- **Pinned Versions:**
  - Node Exporter: `v1.7.0`
  - Prometheus: `v2.50.1`
- **Networking:** Local loopback interface access. For remote hosts, use SSH tunnels or private subnets—**never expose raw metrics ports to the public internet**.

```bash
# Create local secure tunnels if using a remote VM
ssh -L 9100:localhost:9100 -L 9090:localhost:9090 -L 3000:localhost:3000 user@remote-vm-ip
```

---

## 3. Step 1 — Metrics Collection Layer

### 3.1 Node Exporter Setup

Node Exporter exposes hardware and OS-level counters (`/proc`, `/sys`) over an HTTP endpoint.

#### 1. Download and Extract Binary
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar -xvf node_exporter-1.7.0.linux-amd64.tar.gz
sudo cp node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter-1.7.0.linux-amd64*
```

#### 2. Create System User and Systemd Service
```bash
# Create non-interactive system account (least-privilege)
sudo useradd -rs /bin/false node_exporter

# Provision systemd unit
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<'EOF'
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

#### 3. Verification Commands
```bash
# Verify process status
sudo systemctl status node_exporter --no-pager

# Check socket state
ss -tlnp | grep 9100

# Inspect metric output
curl -s http://localhost:9100/metrics | head -n 15

# Confirm target metric is exposed
curl -s http://localhost:9100/metrics | grep '^node_cpu_seconds_total' | head -n 5
```

---

### 3.2 Prometheus Setup

Prometheus pulls metrics from target endpoints and persists them into local time-series chunks.

#### 1. Download and Deploy Binary
```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.50.1/prometheus-2.50.1.linux-amd64.tar.gz
tar -xvf prometheus-2.50.1.linux-amd64.tar.gz
cd prometheus-2.50.1.linux-amd64

sudo cp prometheus promtool /usr/local/bin/
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo cp -r consoles/ console_libraries/ /etc/prometheus/
cd .. && rm -rf prometheus-2.50.1.linux-amd64*
```

#### 2. Configure Scrape Specifications
Create `/etc/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 5s       # Collect targets every 5 seconds
  evaluation_interval: 5s   # Evaluate alerting rules every 5 seconds
  scrape_timeout: 4s        # Must be strictly less than scrape_interval

alerting:
  alertmanagers:
    - static_configs:
        - targets: []

rule_files: []

scrape_configs:
  - job_name: "node_exporter"
    metrics_path: "/metrics"
    scheme: "http"
    static_configs:
      - targets: ["127.0.0.1:9100"]
        labels:
          environment: "production"
          host: "primary-node"
```

#### 3. Configure Permissions and Service Unit
```bash
sudo useradd -rs /bin/false prometheus
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus

sudo tee /etc/systemd/system/prometheus.service > /dev/null <<'EOF'
[Unit]
Description=Prometheus Monitoring Engine
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --storage.tsdb.retention.time=30d \
  --storage.tsdb.retention.size=10GB \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=127.0.0.1:9090
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
```

#### 4. Scrape Pipeline Verification
```bash
# Validate configuration syntax
promtool check config /etc/prometheus/prometheus.yml

# Verify target scraping health status via API
curl -s http://127.0.0.1:9090/api/v1/targets | grep -o '"health":"[^"]*"'

# Run a sample PromQL query via HTTP endpoint
curl -s --data-urlencode 'query=100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)' \
  http://127.0.0.1:9090/api/v1/query | jq .
```

---

## 4. Step 2 — Installing Grafana

Install Grafana via the official APT repository using cryptographically pinned keys.

```bash
# Install system utilities
sudo apt-get install -y apt-transport-https software-properties-common wget gpg

# Add and verify Grafana GPG keyring
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

# Register the stable repository mirror
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list

# Synchronize package indices and install
sudo apt-get update && sudo apt-get install -y grafana

# Start and enable daemon
sudo systemctl daemon-reload
sudo systemctl enable --now grafana-server

# Verify service health status
curl -sf http://localhost:3000/api/health | jq .
```

---

## 5. Step 3 — Provisioning Prometheus in Grafana

Grafana requires an upstream connection to query Prometheus data.

### Method A: Grafana Web UI
1. Navigate to `http://localhost:3000`.
2. Sign in (`admin` / `admin`). Set a new administrative password when prompted.
3. Go to **Connections** &rarr; **Data Sources** &rarr; **Add data source**.
4. Select **Prometheus**.
5. Set **Prometheus server URL**: `http://127.0.0.1:9090`.
6. Click **Save & Test**. Confirm the green checkmark: *"Data source is working"*.

---

### Method B: Declarative File-Based Provisioning (Recommended)

File-based provisioning ensures dashboards and data sources survive machine restarts and allows tracking changes in Git.

Create `/etc/grafana/provisioning/datasources/prometheus.yaml`:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://127.0.0.1:9090
    isDefault: true
    jsonData:
      httpMethod: POST
      timeInterval: 5s
    editable: false
```

Apply the file:
```bash
sudo chown -R root:grafana /etc/grafana/provisioning/datasources/prometheus.yaml
sudo systemctl restart grafana-server

# Verify datasource ingestion via API
curl -s -u admin:admin http://localhost:3000/api/datasources | jq .
```

---

## 6. Step 4 — Building Dashboards & PromQL Deep Dive

### 6.1 Constructing the CPU Panel

1. In Grafana, select **Dashboards** &rarr; **New** &rarr; **New Dashboard**.
2. Click **+ Add visualization**, selecting **Prometheus** as the data source.
3. Switch the query editor from **Builder** to **Code**.
4. Input the following PromQL expression:

```promql
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

5. Configure Panel Presentation (Right Side):
   - **Title:** `Host CPU Utilization`
   - **Unit:** `Misc` &rarr; `Percent (0-100)`
   - **Min:** `0`
   - **Max:** `100`
   - **Thresholds:** `80` (Orange / Warning), `90` (Red / Critical)
6. Click **Apply**, then **Save Dashboard**.

---

### 6.2 PromQL Query Breakdown

$$\text{Active CPU \%} = 100 - \left( \text{avg}_{\text{instance}} \left( \Delta_{\text{instant}}(\text{node\_cpu\_seconds\_total}[\text{mode}=\text{"idle"}]) \right) \times 100 \right)$$

| Query Component | Functionality & Mechanics |
| :--- | :--- |
| `node_cpu_seconds_total{mode="idle"}` | Counter metric tracking idle CPU core seconds. |
| `[5m]` | Defines the evaluation range window across the preceding 5 minutes. |
| `irate(...)` | Calculates the per-second instant rate of increase using the last two data points. Highly responsive to quick spikes. |
| `avg by (instance) (...)` | Aggregates individual CPU cores into a unified per-machine average. |
| `* 100` | Normalizes values into standard percentage increments ($[0.0, 1.0] \to [0, 100]$). |
| `100 - (...)` | Subtracts idle time from total time to calculate busy/active execution time. |

#### Useful Query Variations:
```promql
# Long-term smoothed baseline (better for broad trends than irate)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Per-core breakdown (useful for identifying single-thread bottlenecks)
100 - (irate(node_cpu_seconds_total{mode="idle"}[5m]) * 100)
```

---

### 6.3 Key Baseline Host Metrics

| Metric Objective | PromQL Expression | Unit |
| :--- | :--- | :--- |
| **Memory Utilization %** | `100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))` | `Percent (0-100)` |
| **Disk Space Consumed (Root)** | `100 * (1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/" herit}))` | `Percent (0-100)` |
| **System Load (1m avg)** | `node_load1` | `Scalar` |
| **Network RX Bandwidth** | `irate(node_network_receive_bytes_total{device!="lo"}[5m])` | `bytes/sec` |
| **Network TX Bandwidth** | `irate(node_network_transmit_bytes_total{device!="lo"}[5m])` | `bytes/sec` |

---

## 7. End-to-End Verification Checklist

Run these commands in order to verify each layer of your setup:

```bash
# 1. Verify Node Exporter is serving metrics
systemctl is-active --quiet node_exporter && \
curl -sf http://127.0.0.1:9100/metrics > /dev/null && \
echo "✓ Node Exporter: Operational" || echo "✗ Node Exporter: Failed"

# 2. Check Prometheus syntax and scrape targets
systemctl is-active --quiet prometheus && \
promtool check config /etc/prometheus/prometheus.yml > /dev/null && \
curl -s http://127.0.0.1:9090/api/v1/targets | grep -q '"health":"up"' && \
echo "✓ Prometheus Pipeline: Operational" || echo "✗ Prometheus Pipeline: Failed"

# 3. Check Grafana service status
systemctl is-active --quiet grafana-server && \
curl -sf http://127.0.0.1:3000/api/health > /dev/null && \
echo "✓ Grafana Server: Operational" || echo "✗ Grafana Server: Failed"

# 4. Confirm Grafana can query the Prometheus datasource
curl -sf -u admin:admin http://localhost:3000/api/datasources/name/Prometheus > /dev/null && \
echo "✓ Datasource Link: Operational" || echo "✗ Datasource Link: Failed"
```

---

## 8. Troubleshooting Matrix

| Problem | Root Cause | Diagnosis & Resolution |
| :--- | :--- | :--- |
| **Port 9100 Connection Refused** | Service failed to start or firewall blocked port. | Run `sudo systemctl status node_exporter` to view error logs. Run `ss -tlnp \| grep 9100` to verify port binding. |
| **Target status shows `DOWN` in Prometheus** | Target IP/port is unreachable or firewall is dropping packets. | Open `http://localhost:9090/targets`. Test connection manually: `curl -v http://<TARGET_IP>:9100/metrics`. |
| **`promtool check config` returns errors** | YAML indentation or syntax errors. | Check indentation (use spaces, not tabs). Rerun `promtool check config /etc/prometheus/prometheus.yml`. |
| **Grafana `Save & Test` reports HTTP Bad Gateway** | Prometheus URL is incorrect or Prometheus is stopped. | Verify Prometheus is running (`curl http://127.0.0.1:9090/-/healthy`). Ensure the URL uses `http://`, not `https://`. |
| **Empty dashboard panel ("No Data")** | Query uses wrong labels, or the time range has no data. | Check the time picker in the top right. Run the metric directly in Prometheus (`http://localhost:9090`) to verify data collection. |
| **DataSource provisioning changes are ignored** | YAML is in the wrong path or contains invalid keys. | Ensure the file is in `/etc/grafana/provisioning/datasources/`. Check Grafana logs: `sudo journalctl -u grafana-server -e`. |
| **Prometheus runs out of disk space** | Retention period is too long for the storage volume. | Limit storage size in systemd: add `--storage.tsdb.retention.size=10GB` and `--storage.tsdb.retention.time=15d`. |

---

## 9. Production Hardening & Operational Runbook

1. **Network Isolation & Interface Binding:**
   Avoid binding to `0.0.0.0`. Restrict services to loopback (`127.0.0.1`) or private networks:
   - Prometheus: `--web.listen-address=127.0.0.1:9090`
   - Node Exporter: `--web.listen-address=127.0.0.1:9100`

2. **Reverse Proxy with TLS Termination:**
   Never expose Grafana's HTTP port directly. Place it behind Nginx or Caddy with TLS certificates and standard rate limiting:
   ```nginx
   server {
       listen 443 ssl http2;
       server_name monitoring.internal.net;
       ssl_certificate /etc/ssl/certs/bundle.crt;
       ssl_certificate_key /etc/ssl/private/key.pem;

       location / {
           proxy_pass http://127.0.0.1:3000;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```

3. **Systemd Resource Constraints:**
   Prevent memory leaks or query spikes from taking down the host. Add limits to `/etc/systemd/system/prometheus.service`:
   ```ini
   [Service]
   MemoryHigh=4G
   MemoryMax=6G
   CPUQuota=200%
   TasksMax=4096
   ```

4. **Automated Backup Strategy:**
   Back up these critical state files on a schedule:
   - **Grafana State:** `/var/lib/grafana/grafana.db` (SQLite) or your external PostgreSQL instance.
   - **Grafana Configuration:** `/etc/grafana/grafana.ini` and `/etc/grafana/provisioning/`.
   - **Prometheus State:** `/etc/prometheus/` and snapshots of `/var/lib/prometheus`.

---

## 10. Change Log: Basic vs. Advanced Edition

| Focus Area | Basic Edition | Advanced Edition (This Guide) |
| :--- | :--- | :--- |
| **System Architecture** | High-level overview | ASCII connection flow, component boundaries, and scrape mechanics |
| **Execution Security** | Default root / standard execution | Low-privilege system users (`/bin/false`), systemd isolation, and keyring pinning |
| **PromQL Analysis** | Single raw string | Mathematical breakdown, sub-component role table, and variation guide |
| **Grafana Provisioning** | Manual UI workflow | UI + declarative GitOps configuration via YAML |
| **Operations** | Process start instructions | Storage caps, systemd limits, TLS reverse proxy, and backup procedures |
| **Validation** | Browser page checks | Automated four-stage shell test script |
| **Diagnostics** | Basic troubleshooting notes | Symptom &rarr; Cause &rarr; Solution lookup matrix |
