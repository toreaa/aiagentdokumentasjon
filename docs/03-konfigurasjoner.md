# Konfigurasjonsfiler

Dette dokumentet inneholder alle viktige konfigurasjonsfiler for systemet.

## Server 1 - Netbox Applikasjon

### Netbox Configuration

**Fil:** `/opt/netbox/netbox/netbox/configuration.py`

```python
ALLOWED_HOSTS = ['*']

DATABASE = {
    'NAME': 'netbox',
    'USER': 'netbox',
    'PASSWORD': '[REDACTED]',
    'HOST': '104.248.172.65',
    'PORT': '',
    'CONN_MAX_AGE': 300,
}

REDIS = {
    'tasks': {
        'HOST': 'localhost',
        'PORT': 6379,
        'PASSWORD': '',
        'DATABASE': 0,
        'SSL': False,
    },
    'caching': {
        'HOST': 'localhost',
        'PORT': 6379,
        'PASSWORD': '',
        'DATABASE': 1,
        'SSL': False,
    }
}

SECRET_KEY = '[REDACTED]'

# IMPORTANT: SECRET_KEY må holdes hemmelig og må ikke committes til git
```

### Gunicorn Configuration

**Fil:** `/opt/netbox/gunicorn.py`

```python
import multiprocessing

# Bind til localhost kun (nginx reverse proxy)
bind = '127.0.0.1:8001'

# Workers basert på CPU count
workers = multiprocessing.cpu_count() * 2 + 1

# Threads per worker
threads = 3

# Timeout for requests
timeout = 120

# Graceful worker restart
max_requests = 5000
max_requests_jitter = 500

# Logging
accesslog = '-'
errorlog = '-'
loglevel = 'info'
```

### Nginx Configuration

**Fil:** `/etc/nginx/sites-available/netbox`

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;
    client_max_body_size 25m;

    location /static/ {
        alias /opt/netbox/netbox/static/;
    }

    location / {
        proxy_pass http://127.0.0.1:8001;
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Nginx status endpoint for monitoring
    location /nginx_status {
        stub_status on;
        access_log off;
        allow 127.0.0.1;
        allow 165.232.66.71;
        deny all;
    }
}
```

### Systemd Services

**Fil:** `/etc/systemd/system/netbox.service`

```ini
[Unit]
Description=NetBox WSGI Service
Documentation=https://docs.netbox.dev/
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=opsuser
Group=opsuser
PIDFile=/var/tmp/netbox.pid
WorkingDirectory=/opt/netbox/netbox
ExecStart=/opt/netbox/venv/bin/gunicorn --config /opt/netbox/gunicorn.py netbox.wsgi
Restart=on-failure
RestartSec=30
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

**Fil:** `/etc/systemd/system/netbox-rq.service`

```ini
[Unit]
Description=NetBox Request Queue Worker
Documentation=https://docs.netbox.dev/
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=opsuser
Group=opsuser
WorkingDirectory=/opt/netbox/netbox
ExecStart=/opt/netbox/venv/bin/python3 /opt/netbox/netbox/manage.py rqworker
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

**Fil:** `/etc/systemd/system/redis_exporter.service`

```ini
[Unit]
Description=Redis Exporter
After=network.target

[Service]
User=opsuser
Group=opsuser
Type=simple
ExecStart=/usr/local/bin/redis_exporter -redis.addr=localhost:6379 -web.listen-address=0.0.0.0:9121

[Install]
WantedBy=multi-user.target
```

**Fil:** `/etc/systemd/system/nginx_exporter.service`

```ini
[Unit]
Description=NGINX Prometheus Exporter
After=network.target

[Service]
User=opsuser
Group=opsuser
Type=simple
ExecStart=/usr/local/bin/nginx-prometheus-exporter -nginx.scrape-uri=http://localhost/nginx_status -web.listen-address=:9113

[Install]
WantedBy=multi-user.target
```

**Fil:** `/etc/systemd/system/node_exporter.service`

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

## Server 2 - PostgreSQL Database

### PostgreSQL Configuration

**Fil:** `/etc/postgresql/17/main/postgresql.conf`

Viktige innstillinger:

```conf
# Nettverk
listen_addresses = '*'
port = 5432
max_connections = 100

# Minne
shared_buffers = 256MB
effective_cache_size = 1GB
work_mem = 4MB
maintenance_work_mem = 64MB

# Logging
log_destination = 'stderr'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_line_prefix = '%m [%p] %q%u@%d '
log_timezone = 'UTC'
```

**Fil:** `/etc/postgresql/17/main/pg_hba.conf`

```conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local connections
local   all             postgres                                peer
local   all             all                                     peer

# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256

# Allow netbox from Server 1
host    netbox          netbox          146.190.148.117/32      scram-sha-256

# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
```

### PostgreSQL Exporter

**Fil:** `/etc/systemd/system/postgres_exporter.service`

```ini
[Unit]
Description=PostgreSQL Exporter
After=network.target

[Service]
User=postgres
Group=postgres
Type=simple
Environment="DATA_SOURCE_NAME=postgresql://netbox:[REDACTED]@localhost:5432/netbox?sslmode=disable"
ExecStart=/usr/local/bin/postgres_exporter

[Install]
WantedBy=multi-user.target
```

## Server 3 - Monitoring

### Prometheus Configuration

**Fil:** `/opt/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'netbox-demo'
    environment: 'production'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance: 'server3-monitoring'

  - job_name: 'server1-node'
    static_configs:
      - targets: ['146.190.148.117:9100']
        labels:
          instance: 'server1-netbox'
          server: 'netbox'

  - job_name: 'server2-node'
    static_configs:
      - targets: ['104.248.172.65:9100']
        labels:
          instance: 'server2-postgres'
          server: 'database'

  - job_name: 'server3-node'
    static_configs:
      - targets: ['localhost:9100']
        labels:
          instance: 'server3-monitoring'
          server: 'monitoring'

  - job_name: 'nginx'
    static_configs:
      - targets: ['146.190.148.117:9113']
        labels:
          instance: 'server1-netbox'
          service: 'nginx'

  - job_name: 'redis'
    static_configs:
      - targets: ['146.190.148.117:9121']
        labels:
          instance: 'server1-netbox'
          service: 'redis'

  - job_name: 'postgres'
    static_configs:
      - targets: ['104.248.172.65:9187']
        labels:
          instance: 'server2-postgres'
          service: 'postgresql'

  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - http://146.190.148.117
        - http://165.232.66.71:3000
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115
```

**Fil:** `/etc/systemd/system/prometheus.service`

```ini
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target
Wants=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/opt/prometheus/prometheus \
  --config.file=/opt/prometheus/prometheus.yml \
  --storage.tsdb.path=/opt/prometheus/data \
  --storage.tsdb.retention.time=15d \
  --web.console.templates=/opt/prometheus/consoles \
  --web.console.libraries=/opt/prometheus/console_libraries
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Grafana Configuration

**Fil:** `/etc/grafana/grafana.ini`

Viktige innstillinger:

```ini
[server]
protocol = http
http_addr = 0.0.0.0
http_port = 3000
domain = localhost
root_url = http://165.232.66.71:3000

[security]
admin_user = admin
admin_password = [REDACTED]

[auth.anonymous]
enabled = false

[dashboards]
default_home_dashboard_path = /var/lib/grafana/dashboards/sla.json
```

### Grafana Datasource

**Fil:** `/etc/grafana/provisioning/datasources/prometheus.yml`

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://localhost:9090
    isDefault: true
    editable: true
    jsonData:
      timeInterval: 15s
      queryTimeout: 60s
```

### Grafana Dashboard Provisioning

**Fil:** `/etc/grafana/provisioning/dashboards/dashboards.yml`

```yaml
apiVersion: 1

providers:
  - name: 'Default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true
```

### Blackbox Exporter Configuration

**Fil:** `/etc/blackbox_exporter/blackbox.yml`

```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: []  # Defaults to 2xx
      method: GET
      preferred_ip_protocol: "ip4"
      follow_redirects: true
      fail_if_ssl: false
      fail_if_not_ssl: false

  http_post_2xx:
    prober: http
    timeout: 5s
    http:
      method: POST
      valid_status_codes: []

  tcp_connect:
    prober: tcp
    timeout: 5s

  icmp:
    prober: icmp
    timeout: 5s
```

**Fil:** `/etc/systemd/system/blackbox_exporter.service`

```ini
[Unit]
Description=Blackbox Exporter
After=network.target

[Service]
User=opsuser
Group=opsuser
Type=simple
ExecStart=/usr/local/bin/blackbox_exporter --config.file=/etc/blackbox_exporter/blackbox.yml

[Install]
WantedBy=multi-user.target
```

## Loki Configuration (Optional - Stoppet)

**Fil:** `/opt/loki/loki-config.yml`

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /opt/loki
  storage:
    filesystem:
      chunks_directory: /opt/loki/chunks
      rules_directory: /opt/loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 168h
```

## Promtail Configuration (Optional - Stoppet)

**Fil:** `/etc/promtail/promtail-config.yml`

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://165.232.66.71:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          host: server1-netbox
          __path__: /var/log/*log

  - job_name: netbox
    static_configs:
      - targets:
          - localhost
        labels:
          job: netbox
          host: server1-netbox
          __path__: /opt/netbox/netbox/*.log

  - job_name: nginx
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx
          host: server1-netbox
          __path__: /var/log/nginx/*log
```

## Redis Configuration

**Fil:** `/etc/redis/redis.conf`

Viktige innstillinger:

```conf
bind 127.0.0.1 ::1
port 6379
protected-mode yes
databases 16

# Persistence
save 900 1
save 300 10
save 60 10000

# Logging
loglevel notice
logfile /var/log/redis/redis-server.log

# Memory management
maxmemory-policy allkeys-lru
```

## Konfigurasjonshåndtering

### Backup av konfigurasjoner

For å ta backup av alle konfigurasjonsfiler:

```bash
# Server 1
sudo tar -czf netbox-config-backup.tar.gz \
  /opt/netbox/netbox/netbox/configuration.py \
  /opt/netbox/gunicorn.py \
  /etc/nginx/sites-available/netbox \
  /etc/systemd/system/netbox*.service \
  /etc/systemd/system/*_exporter.service

# Server 2
sudo tar -czf postgres-config-backup.tar.gz \
  /etc/postgresql/17/main/postgresql.conf \
  /etc/postgresql/17/main/pg_hba.conf

# Server 3
sudo tar -czf monitoring-config-backup.tar.gz \
  /opt/prometheus/prometheus.yml \
  /etc/grafana/grafana.ini \
  /etc/grafana/provisioning/ \
  /etc/blackbox_exporter/
```

### Verifisere konfigurasjoner

```bash
# Nginx
sudo nginx -t

# PostgreSQL
sudo -u postgres psql -c "SHOW config_file;"

# Prometheus
/opt/prometheus/promtool check config /opt/prometheus/prometheus.yml
```

### Endre konfigurasjoner

Ved endring av konfigurasjonsfiler:

```bash
# Reload eller restart relevante tjenester
sudo systemctl reload nginx          # For Nginx
sudo systemctl restart netbox        # For Netbox
sudo systemctl restart prometheus    # For Prometheus
sudo systemctl reload postgresql     # For PostgreSQL
```

## Sikkerhet

### Kritiske filer som må beskyttes

```bash
# Begrens tilgang til sensitive filer
sudo chmod 600 /opt/netbox/netbox/netbox/configuration.py
sudo chown opsuser:opsuser /opt/netbox/netbox/netbox/configuration.py

sudo chmod 600 /etc/postgresql/17/main/pg_hba.conf
sudo chown postgres:postgres /etc/postgresql/17/main/pg_hba.conf
```

### Konfigurasjonsfiler i Git

**ALDRI commit følgende filer til git:**
- configuration.py (Netbox SECRET_KEY og database passord)
- prometheus.yml hvis den inneholder API keys
- grafana.ini (admin passord)
- Enhver fil som inneholder credentials

Bruk environment variables eller separate credential stores i produksjon.
