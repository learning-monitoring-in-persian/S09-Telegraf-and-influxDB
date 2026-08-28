[English](README.md) | [فارسی](README-persian.md)

# Set up Telegraf and InfluxDB

**Telegraf** is a powerful, plugin-driven server agent for collecting and sending metrics and events. Out of the box, it collects detailed system metrics from your machines (like CPU, Memory, Disk, Network IO, Processes, etc. - very similar to Node Exporter). However, it also has hundreds of plugins to collect data from databases, message queues, and external APIs.

**InfluxDB** is a highly popular, purpose-built Time-Series Database (TSDB) designed to store the metrics collected by Telegraf. 

> [!NOTE]
> **Why are we learning this?**
> We already know about **Grafana Alloy** as a modern telemetry collector. However, the Telegraf + InfluxDB stack (often referred to as the "TICK stack") is extremely common and widely used in older, existing infrastructures. As an engineer, you need to recognize and understand this stack when you encounter it in the wild.

> [!TIP]
> **The Future:** In modern observability stacks, InfluxDB is increasingly being replaced by **OpenTelemetry** and highly scalable backends like Prometheus/Mimir. We will explore OpenTelemetry in detail in a future repository.

---

## Install Telegraf as a System Service (Binary)

> [!TIP]
> **Monitoring Multiple Servers:** If you have multiple machines to monitor, the best approach is to install the Telegraf binary on **all** of them (it's lightweight and avoids Docker overhead on every machine) and point them all to a single, central InfluxDB server!

If you just want to install the Telegraf agent on a Linux server to monitor its resources, you can run it as a standalone systemd service.

### 1. Download and Install the Binary

First, download the pre-compiled binary for Linux:

```bash
# Important Note
# If your system architecture is not amd64 the command below will not work for you.
# For example if it is arm64, replace all `amd64` with `arm64` in the commands below:

VERSION=$(curl -s https://api.github.com/repos/influxdata/telegraf/releases/latest | grep '"tag_name"' | cut -d'"' -f4 | sed 's/v//')
wget -O telegraf.tar.gz https://dl.influxdata.com/telegraf/releases/telegraf-${VERSION}_linux_amd64.tar.gz
tar xvfz telegraf.tar.gz
```

Move the extracted binary to the system's bin path:
```bash
sudo mv telegraf-${VERSION}/usr/bin/telegraf /usr/local/bin/telegraf
rm -rf telegraf-${VERSION} telegraf.tar.gz
```

For security, create a dedicated system user and directories:
```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin telegraf
sudo mkdir -p /etc/telegraf
```

### 2. Create a configuration file

Create `/etc/telegraf/telegraf.conf`:
```toml
[agent]
  interval = "10s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = "0s"

# OUTPUT: Where to send the data (InfluxDB v2)
[[outputs.influxdb_v2]]
  urls = ["http://localhost:8086"]
  token = "${INFLUX_TOKEN}"
  organization = "my-org"
  bucket = "my-bucket"

# INPUTS: What data to collect from the machine
[[inputs.cpu]]
  percpu = true
  totalcpu = true
  collect_cpu_time = false
  report_active = false
[[inputs.disk]]
  ignore_fs = ["tmpfs", "devtmpfs", "devfs", "iso9660", "overlay", "aufs", "squashfs"]
[[inputs.mem]]
[[inputs.net]]
[[inputs.system]]
```

Set the permissions:
```bash
sudo chown -R telegraf:telegraf /etc/telegraf
```

### 3. Create a systemd service

Create the environment file `/etc/telegraf/telegraf.env` for your tokens:
```env
INFLUX_TOKEN=your-super-secret-token
```

Create `/etc/systemd/system/telegraf.service`:
```ini
[Unit]
Description=Telegraf
Wants=network-online.target
After=network-online.target

[Service]
User=telegraf
Group=telegraf
EnvironmentFile=/etc/telegraf/telegraf.env
ExecStart=/usr/local/bin/telegraf --config /etc/telegraf/telegraf.conf

Restart=always

[Install]
WantedBy=multi-user.target
```

Start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable telegraf
sudo systemctl start telegraf
```

---

## Set up InfluxDB and Telegraf with Docker Compose

Usually, you run the database (InfluxDB) using Docker. Here is a `docker-compose.yml` that spins up both InfluxDB (v2) and Telegraf.

```yaml
services:
  influxdb:
    image: influxdb:2.7
    container_name: influxdb
    restart: unless-stopped
    ports:
      - "8086:8086"
    volumes:
      - influxdb-data:/var/lib/influxdb2
      - influxdb-config:/etc/influxdb2
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=secure_password
      - DOCKER_INFLUXDB_INIT_ORG=my-org
      - DOCKER_INFLUXDB_INIT_BUCKET=my-bucket
      - DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=my-super-secret-auth-token

  telegraf:
    image: telegraf:latest
    container_name: telegraf
    restart: unless-stopped
    volumes:
      - ./telegraf.conf:/etc/telegraf/telegraf.conf:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro # Allow collecting Docker metrics
    environment:
      - INFLUX_TOKEN=my-super-secret-auth-token

volumes:
  influxdb-data:
  influxdb-config:
```

Run it:
```bash
docker compose up -d
```

---

## What can we do inside InfluxDB?

InfluxDB is more than just a place to store data. If you visit `http://{IP_ADDRESS}:8086` in your browser and log in with your credentials, you get access to the **InfluxDB Data Explorer (Web UI)**.

Inside the UI you can:
1. **Explore Data**: Visually browse through buckets and metrics.
2. **Build Dashboards**: InfluxDB has its own built-in dashboarding system! (Though Grafana is still usually preferred).
3. **Use Flux Language**: Write powerful queries using `Flux` or `InfluxQL` to aggregate data.
4. **Create Tasks**: You can run background jobs that automatically downsample old data (to save disk space) or trigger alerts.

---

## Connect InfluxDB to Grafana

To visualize your Telegraf metrics in Grafana, you need to connect InfluxDB as a Data Source via the Grafana UI:

1. Open Grafana in your browser (`http://{IP_ADDRESS}:3000`).
2. Go to **Connections** > **Data Sources** (or **Configuration** > **Data Sources** in older versions).
3. Click **Add data source** and search for **InfluxDB**.
4. In the settings:
   - **Query Language**: Choose **Flux**.
   - **URL**: Enter `http://influxdb:8086` (or the IP if running externally).
5. Scroll down to **InfluxDB Details**:
   - **Organization**: `my-org`
   - **Token**: `my-super-secret-auth-token`
   - **Default Bucket**: `my-bucket`
6. Click **Save & Test**. Grafana is now connected to InfluxDB! You can import community dashboards (like [Telegraf System Dashboard](https://grafana.com/grafana/dashboards/928-telegraf-system-dashboard/)) to instantly see your server metrics.
