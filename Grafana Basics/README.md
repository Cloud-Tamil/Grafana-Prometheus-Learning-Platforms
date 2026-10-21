Grafana Basics — Advanced Edition
A deeper, production-oriented rewrite of the course. This version keeps the original learning path but adds architecture context, hardening, verification, troubleshooting, and best practices that a real operator would need. All commands remain faithful to the source material; new sections are clearly marked as added.

Grafana Basics — Advanced Course
https://grafana_logo.png/

1. Introduction
1.1 What Is Grafana?
Grafana is a leading open-source data visualization and monitoring platform. It lets you build interactive dashboards and graphs over many data sources — relational databases, cloud services, and time-series databases such as Prometheus and InfluxDB.

https://0.jpg/

1.2 What You Will Learn
Install and run Grafana in a virtual environment.

Configure Grafana and connect it to Prometheus.

Build a basic dashboard to visualize system metrics.

(Advanced) Understand the metrics pipeline end-to-end, harden the services, verify each layer, and troubleshoot common failures.

1.3 Reference Architecture
text
┌──────────────┐     scrape /metrics      ┌──────────────┐     query      ┌──────────────┐
│ Node Exporter│ ───────────────────────► │  Prometheus  │ ◄───────────── │   Grafana    │
│  :9100       │   (pull, every 5s)       │    :9090     │   (PromQL)     │    :3000     │
└──────────────┘                          └──────────────┘                └──────────────┘
       ▲                                          ▲
       │ exposes host/OS metrics                  │ stores time series (TSDB)
       │                                          │
   Linux kernel                              /var/lib/prometheus
Data flow: the kernel exposes metrics → Node Exporter translates them to Prometheus format on :9100 → Prometheus pulls (scrapes) them every 5s and stores them in its TSDB → Grafana queries Prometheus via PromQL and renders panels.

Key concept: Prometheus is a pull-based system. Targets are discovered from scrape_configs; nothing is pushed from Node Exporter.

Note: When asked to go to a specific port, click the link provided in the terminal. This redirects you to the correct URL for this training instance.

2. Step 1 — Collecting Metrics
Before visualizing data, we must collect it. We use Node Exporter (metric producer) and Prometheus (metric store).

2.1 Node Exporter
Node Exporter is a Prometheus exporter for hardware and OS metrics exposed by *nix kernels. It is written in Go with pluggable metric collectors.

Download:

bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
Extract and install the binary:

bash
tar -xvf node_exporter-1.7.0.linux-amd64.tar.gz && cd node_exporter-1.7.0.linux-amd64 && sudo cp node_exporter /usr/local/bin
Create a dedicated service user and systemd unit:

bash
sudo useradd -rs /bin/false node_exporter && echo -e "[Unit]\nDescription=Node Exporter\nAfter=network.target\n\n[Service]\nUser=node_exporter\nGroup=node_exporter\nType=simple\nExecStart=/usr/local/bin/node_exporter\n\n[Install]\nWantedBy=multi-user.target" | sudo tee /etc/systemd/system/node_exporter.service > /dev/null && sudo systemctl daemon-reload && sudo systemctl start node_exporter
Verify: open http://localhost:9100. A page full of metrics means Node Exporter is running.

Advanced — Verify from the command line
bash
# Is the service active?
sudo systemctl status node_exporter --no-pager

# Is the port listening?
ss -tlnp | grep 9100

# Are metrics actually being served? (first lines)
curl -s http://localhost:9100/metrics | head

# Sanity-check a key metric used later in the dashboard
curl -s http://localhost:9100/metrics | grep '^node_cpu_seconds_total'
Advanced — Why useradd -rs /bin/false?
-r creates a system account (no aging, low UID range) — appropriate for a daemon.

-s /bin/false sets a non-interactive shell, so the account cannot be used to log in. This is least-privilege hardening for a service that only needs to read kernel counters.

2.2 Prometheus
Prometheus is a monitoring and alerting toolkit designed for reliability, scalability, and maintainability. It collects and queries metrics and stores those produced by Node Exporter.

Download:

bash
wget https://github.com/prometheus/prometheus/releases/download/v2.50.1/prometheus-2.50.1.linux-amd64.tar.gz
Extract and install the binary:

bash
tar -xvf prometheus-2.50.1.linux-amd64.tar.gz && cd prometheus-2.50.1.linux-amd64 && sudo cp prometheus /usr/local/bin
Create the service user, directories, config, and systemd unit:

bash
sudo useradd -rs /bin/false prometheus && sudo mkdir /etc/prometheus /var/lib/prometheus && sudo cp prometheus.yml /etc/prometheus/prometheus.yml && sudo cp -r consoles/ console_libraries/ /etc/prometheus/ && sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus && echo -e "[Unit]\nDescription=Prometheus\nAfter=network.target\n\n[Service]\nUser=prometheus\nGroup=prometheus\nType=simple\nExecStart=/usr/local/bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/var/lib/prometheus --web.console.templates=/etc/prometheus/consoles --web.console.libraries=/etc/prometheus/console_libraries\n\n[Install]\nWantedBy=multi-user.target" | sudo tee /etc/systemd/system/prometheus.service > /dev/null && sudo systemctl daemon-reload && sudo systemctl start prometheus
Add Node Exporter as a scrape target. The prometheus.yml should look like this:

yaml
# my global config
global:
  scrape_interval: 5s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 5s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "Node Exporter"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["0.0.0.0:9100"]
Apply the config and restart:

bash
sudo cp /education/prometheus.yml /etc/prometheus/prometheus.yml && sudo systemctl restart prometheus
Verify: open http://localhost:9090. A graph page means Prometheus is running.

Advanced — Verify the scrape pipeline
bash
# Service and port checks
sudo systemctl status prometheus --no-pager
ss -tlnp | grep 9090

# Is the config syntactically valid? (run the bundled checker)
promtool check config /etc/prometheus/prometheus.yml

# Query the Prometheus HTTP API directly — is the target UP?
curl -s 'http://localhost:9090/api/v1/targets' | grep -o '"health":"[^"]*"'

# Run the exact PromQL the dashboard will use, via the API
curl -s --data-urlencode 'query=100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)' \
  http://localhost:9090/api/v1/query
Advanced — Config field notes
Field	Value here	Meaning / trade-off
scrape_interval	5s	How often Prometheus pulls. Lower = fresher data, more CPU/disk. Default 1m.
evaluation_interval	5s	How often alerting/recording rules are evaluated. Default 1m.
scrape_timeout	10s (default)	Per-scrape deadline; must be ≤ scrape_interval.
job_name	Node Exporter	Becomes the job label on every series — used for filtering in PromQL.
targets	["0.0.0.0:9100"]	The scrape endpoint. In production, prefer the host's real IP or a DNS name.
Advanced tip: 0.0.0.0 as a scrape target works here because Node Exporter listens on all interfaces, but it is ambiguous in multi-homed hosts. Prefer 127.0.0.1:9100 (same host) or a concrete address.

Advanced — Persistent storage & retention
The unit sets --storage.tsdb.path=/var/lib/prometheus. By default Prometheus retains 15 days of data. To change it, add to ExecStart:

text
--storage.tsdb.retention.time=30d
Also consider --storage.tsdb.retention.size=10GB to cap disk usage. Ensure /var/lib/prometheus is owned by prometheus:prometheus (the install command above already does this).

3. Step 2 — Installing Grafana
Grafana is an open-source platform for monitoring and observability that lets you create, explore, and share dashboards and data visualizations.

3.1 Install via apt
bash
sudo apt-get install -y apt-transport-https software-properties-common wget &&
sudo mkdir -p /etc/apt/keyrings/ &&
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null &&
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list &&
sudo apt-get update && sudo apt-get install -y grafana
3.2 Start the service
bash
sudo systemctl start grafana-server && sudo systemctl status grafana-server
Verify: open http://localhost:3000. A login prompt means Grafana is running.

Advanced — What each install step does
Step	Purpose
apt-transport-https	Lets apt fetch over HTTPS.
software-properties-common	Provides add-apt-repository-style tooling.
mkdir -p /etc/apt/keyrings/	Standard location for third-party signing keys (apt ≥ 2.4).
gpg --dearmor	Converts the ASCII-armored key to the binary .gpg format apt expects.
signed-by=...	Pins the repo to that key — prevents key-substitution attacks.
Advanced — Verify and enable at boot
bash
# Is Grafana listening?
ss -tlnp | grep 3000

# Health endpoint (returns JSON with version + database status)
curl -s http://localhost:3000/api/health

# Enable on boot (the course only starts it; production should enable it)
sudo systemctl enable grafana-server
Advanced — Key paths and config
Path	Contents
/etc/grafana/grafana.ini	Main config (ports, auth, SMTP, etc.).
/etc/grafana/provisioning/datasources/	Auto-provisioned data sources (used in Step 3, Method 2).
/etc/grafana/provisioning/dashboards/	Auto-provisioned dashboards.
/var/lib/grafana/	Grafana's SQLite DB, plugins, sessions.
/var/log/grafana/	Logs (or journalctl -u grafana-server).
Advanced security note: the default admin/admin credentials force a password change on first login in recent versions. In production, set [security] admin_password in grafana.ini or provision via SSO, and change the default HTTP port if 3000 is exposed.

4. Step 3 — Provisioning the Prometheus Data Source
Grafana must be told where Prometheus lives. Two methods: UI and provisioning file.

Note: The provisioning file method simply moves a pre-created config file and restarts Grafana. It is the fastest path to visualization.

4.1 Method 1 — Grafana UI
Open http://localhost:3000.

Sign in:

Username: admin

Password: admin

Left menu → Connections → Data Sources.

https://data_sources.png/

Click Add data source.

Select Prometheus.

https://data_sources_list.png/

In Connection, set:

Prometheus server URL: http://localhost:9090

Click Save & Test.

https://save_and_test.png/

You have now configured Grafana to use Prometheus as a data source.

4.2 Method 2 — Provisioning File
bash
sudo cp /education/prometheus_datasource.yml /etc/grafana/provisioning/datasources/ && sudo systemctl restart grafana-server
Then open http://localhost:3000 and sign in with admin / admin.

Advanced — What a provisioning file looks like
A typical prometheus_datasource.yml has this shape (verify against the file in /education):

yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://localhost:9090
    isDefault: true
    editable: true
Field	Meaning
apiVersion: 1	Provisioning schema version (currently 1).
access: proxy	Grafana server proxies queries to Prometheus (recommended; avoids browser CORS).
isDefault: true	New panels default to this source.
editable	Whether the UI may modify it (provisioned sources are read-only from disk on restart).
Advanced — Verify provisioning took effect
bash
# Did Grafana load the datasource? (requires auth)
curl -s -u admin:admin http://localhost:3000/api/datasources | head

# Check Grafana logs for provisioning errors
sudo journalctl -u grafana-server --since "5 min ago" | grep -i -E "provision|datasource|error"
Advanced tip: Provisioned data sources are declarative — if you edit them in the UI and Grafana restarts, the file wins. For drift-free environments, keep the YAML in version control and treat the UI as read-only.

5. Step 4 — Creating a Basic Dashboard
We will add a dashboard and build a CPU-usage time-series panel.

5.1 Add a New Dashboard
Open http://localhost:3000.

Sign in with admin / admin.

Note: If already logged in, skip this step.

Left menu → Dashboards → + Create Dashboard.

5.2 Create the Graph Panel
Click Add visualization.

Select Prometheus as the data source.

Select the Time Series visualization (right panel).

In Query, set:

Data Source: Prometheus

Toggle the query editor from Builder to Code and enter:

promql
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
https://query.png/

What this query does: it computes the average CPU utilization (non-idle) per instance over the last 5 minutes. It measures idle time per CPU via irate(...[5m]), averages it per instance, converts to a percentage, and subtracts from 100 to yield active usage.

Click Run.

Configure the panel (right-hand configuration panel):

Title: CPU Usage

Unit: percent(0-100)

Min: 0

Max: 100

https://search.png/

Top Tip: Use the search bar in the configuration panel to find these fields quickly.

Click Apply.

Click Save dashboard, enter a name, and click Save.

https://graph.png/

Advanced — Deconstructing the PromQL
promql
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
Part	Role
node_cpu_seconds_total{mode="idle"}	Counter of idle CPU-seconds, labeled by cpu, mode, instance.
[5m]	Range selector: the last 5 minutes of samples.
irate(...)	Instant rate over the last two samples in the range — responsive to spikes. (rate() would average over the whole window and be smoother.)
avg by (instance)	Collapses per-cpu series into one series per host.
* 100	Converts a 0–1 fraction to a percentage.
100 - (...)	Inverts idle → busy.
Variants worth knowing:

promql
# Smoother, less spike-sensitive (rate over full window)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Per-CPU breakdown (drop the avg)
100 - (irate(node_cpu_seconds_total{mode="idle"}[5m]) * 100)

# Filter to one host
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle", instance="localhost:9100"}[5m])) * 100)
Advanced — Other useful panels to add
Panel	PromQL
Memory used %	100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
Disk used % (root)	100 * (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})
Load average (1m)	node_load1
Network RX (bytes/s)	irate(node_network_receive_bytes_total{device!="lo"}[5m])
Advanced — Thresholds, alerts, and reuse
Thresholds: in the panel's Thresholds section, add a warning at e.g. 80 and critical at 90 to color the series.

Alerting: Grafana 10+ supports unified alerting on panels; alternatively define recording/alerting rules in Prometheus (rule_files) and let Alertmanager handle routing.

Reuse: export the dashboard JSON (Dashboard settings → JSON Model) and store it under /etc/grafana/provisioning/dashboards/ for reproducible deployments.

6. Verification Checklist (End-to-End)
Run these in order; each should succeed before moving on.

bash
# 1. Node Exporter up and serving metrics
systemctl is-active node_exporter && curl -sf http://localhost:9100/metrics >/dev/null && echo "node_exporter OK"

# 2. Prometheus up, config valid, target healthy
systemctl is-active prometheus && promtool check config /etc/prometheus/prometheus.yml && \
  curl -s http://localhost:9090/api/v1/targets | grep -q '"health":"up"' && echo "prometheus OK"

# 3. Grafana up and healthy
systemctl is-active grafana-server && curl -sf http://localhost:3000/api/health && echo "grafana OK"

# 4. Data source reachable from Grafana (after login)
curl -s -u admin:admin http://localhost:3000/api/datasources/1/health
7. Troubleshooting Matrix
Symptom	Likely cause	Fix
:9100 unreachable	Service not started / port blocked	systemctl status node_exporter; ss -tlnp | grep 9100
Prometheus target shows DOWN	Wrong target address or Node Exporter down	Check Status → Targets in Prometheus UI; verify prometheus.yml
promtool check config fails	YAML indentation / typo	Fix indentation; re-run promtool check config
Grafana login fails	Wrong creds / DB not initialized	Check journalctl -u grafana-server; reset admin password via grafana-cli admin reset-admin-password
Save & Test fails in Grafana	Prometheus URL wrong or not reachable from Grafana host	Use http://localhost:9090; test curl from the Grafana host
Dashboard panel is empty	Query error, wrong time range, or no data	Click Run; check the query inspector; widen the time range
Provisioned datasource ignored	File in wrong dir or invalid YAML	Ensure it's in /etc/grafana/provisioning/datasources/; check logs
Prometheus disk filling	Retention too long	Add --storage.tsdb.retention.time / --storage.tsdb.retention.size
8. Production Hardening Notes (Added)
These go beyond the lab but are the natural next step:

Bind to loopback / firewall. Node Exporter, Prometheus, and Grafana all listen broadly in this lab. In production, bind to 127.0.0.1 or a private interface and restrict with ufw/nftables.

TLS + auth. Put Grafana behind a reverse proxy (nginx/Caddy) with TLS; enable Grafana's [auth] and SSO. Prometheus supports --web.config.file for basic auth/TLS.

Enable services at boot. systemctl enable node_exporter prometheus grafana-server.

Resource limits. Add MemoryMax=/CPUQuota= to the systemd units, and set GOMAXPROCS if needed.

Backups. Back up /etc/prometheus, /var/lib/prometheus (or use snapshots), /etc/grafana, and /var/lib/grafana/grafana.db.

Version pinning. The lab pins Node Exporter 1.7.0 and Prometheus 2.50.1; keep pins in production and track CVEs.

Log aggregation. Ship journalctl output for all three services to your log stack.

Alerting. Add Alertmanager (alertmanager:9093 is already stubbed in prometheus.yml) and define rules under rule_files.

9. Course Complete
https://grafana_logo.png/

You have completed the Grafana Basics course — advanced edition. You can now install and run Grafana, connect it to Prometheus, build a CPU dashboard, verify each layer of the pipeline, and troubleshoot common failures.

What's Next?
Grafana Plugins — extend visualizations and data sources.

Grafana Dashboards — templating, variables, and dashboard-as-code.

Grafana API — automate dashboards and data sources.

PromQL deep dive — recording rules, histogram_quantile, subqueries.

Alerting — Prometheus rules + Alertmanager routing.

Change Log vs. the Basic Edition
Area	Basic Edition	Advanced Edition
Structure	4 steps + intro/outro	9 numbered sections with a reference architecture diagram
Verification	"visit the URL"	systemctl, ss, curl, promtool, API checks + a consolidated checklist
PromQL	Query given	Query deconstructed part-by-part + variants + extra panels
Provisioning	Copy file	Explained schema, fields, drift behavior, log verification
Security	Not covered	Least-privilege users, signed-by, loopback binding, TLS, SSO notes
Operations	Not covered	Retention, backups, resource limits, boot-enable, log shipping
Troubleshooting	Not covered	Symptom → cause → fix matrix
Formatting	Mixed	Consistent H1/H2/H3, fenced code blocks, tables, blockquote notes
If you'd like, I can also produce a lab worksheet (fill-in-the-blank commands), a quiz with answers, or a single provisioning bundle (prometheus.yml + prometheus_datasource.yml + dashboard JSON) ready to drop into /education.
