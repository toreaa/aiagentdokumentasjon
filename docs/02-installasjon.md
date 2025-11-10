# Installasjonsveiledning

## Forutsetninger

Denne guiden forutsetter at du har:
- 3 Ubuntu 25.10 servere (DigitalOcean droplets eller tilsvarende)
- Root eller sudo tilgang på alle servere
- Grunnleggende Linux-kommandolinjekunnskap
- SSH-tilgang til alle servere

## Server Oppsett

### Opprette non-root bruker (alle servere)

```bash
# Som root på hver server
useradd -m -s /bin/bash opsuser
echo "opsuser:[REDACTED]" | chpasswd
usermod -aG sudo opsuser
echo "opsuser ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/opsuser
chmod 440 /etc/sudoers.d/opsuser
```

### Grunnleggende systemoppdateringer (alle servere)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl wget vim htop net-tools
```

## Server 1 - Netbox Applikasjon (146.190.148.117)

### 1. Installer PostgreSQL Client

```bash
sudo apt install -y postgresql-client
```

### 2. Installer Redis

```bash
sudo apt install -y redis-server
sudo systemctl enable redis-server
sudo systemctl start redis-server
```

### 3. Installer Python og avhengigheter

```bash
sudo apt install -y python3 python3-pip python3-venv python3-dev
sudo apt install -y build-essential libxml2-dev libxslt1-dev libffi-dev libpq-dev libssl-dev zlib1g-dev
```

### 4. Installer Netbox

```bash
# Last ned Netbox
cd /opt
sudo git clone -b master --depth 1 https://github.com/netbox-community/netbox.git

# Opprett virtual environment
cd /opt/netbox
sudo python3 -m venv venv
source venv/bin/activate

# Installer Python packages
pip install --upgrade pip
pip install -r requirements.txt

# Kopier konfigurasjon
cd netbox/netbox
sudo cp configuration_example.py configuration.py
```

### 5. Konfigurer Netbox

Rediger `/opt/netbox/netbox/netbox/configuration.py`:

```python
ALLOWED_HOSTS = ['*']

DATABASE = {
    'NAME': 'netbox',
    'USER': 'netbox',
    'PASSWORD': '[REDACTED]',
    'HOST': '104.248.172.65',
    'PORT': '',
}

REDIS = {
    'tasks': {
        'HOST': 'localhost',
        'PORT': 6379,
        'DATABASE': 0,
    },
    'caching': {
        'HOST': 'localhost',
        'PORT': 6379,
        'DATABASE': 1,
    }
}

SECRET_KEY = '[REDACTED]'
```

### 6. Kjør migrasjoner og opprett superuser

```bash
cd /opt/netbox/netbox
source ../venv/bin/activate
python3 manage.py migrate
python3 manage.py createsuperuser
python3 manage.py collectstatic --no-input
```

### 7. Installer og konfigurer Gunicorn

```bash
sudo cp /opt/netbox/contrib/gunicorn.py /opt/netbox/gunicorn.py
```

Opprett `/opt/netbox/gunicorn.py`:

```python
import multiprocessing

bind = '127.0.0.1:8001'
workers = multiprocessing.cpu_count() * 2 + 1
threads = 3
timeout = 120
max_requests = 5000
max_requests_jitter = 500
```

### 8. Opprett systemd service for Netbox

```bash
sudo cp /opt/netbox/contrib/netbox.service /etc/systemd/system/
sudo cp /opt/netbox/contrib/netbox-rq.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable netbox netbox-rq
sudo systemctl start netbox netbox-rq
```

### 9. Installer og konfigurer Nginx

```bash
sudo apt install -y nginx
sudo cp /opt/netbox/contrib/nginx.conf /etc/nginx/sites-available/netbox
sudo ln -s /etc/nginx/sites-available/netbox /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

### 10. Installer Exporters

**Node Exporter:**
```bash
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar xvfz node_exporter-1.8.2.linux-amd64.tar.gz
sudo mv node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/
sudo useradd -rs /bin/false node_exporter
```

Opprett `/etc/systemd/system/node_exporter.service`:
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

**Redis Exporter:**
```bash
cd /tmp
wget https://github.com/oliver006/redis_exporter/releases/download/v1.62.0/redis_exporter-v1.62.0.linux-amd64.tar.gz
tar xvfz redis_exporter-v1.62.0.linux-amd64.tar.gz
sudo mv redis_exporter-v1.62.0.linux-amd64/redis_exporter /usr/local/bin/
```

Opprett `/etc/systemd/system/redis_exporter.service`:
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

**Nginx Exporter:**
```bash
cd /tmp
wget https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v1.4.1/nginx-prometheus-exporter_1.4.1_linux_amd64.tar.gz
tar xvfz nginx-prometheus-exporter_1.4.1_linux_amd64.tar.gz
sudo mv nginx-prometheus-exporter /usr/local/bin/
```

Opprett `/etc/systemd/system/nginx_exporter.service`:
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

Start alle exporters:
```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter redis_exporter nginx_exporter
sudo systemctl start node_exporter redis_exporter nginx_exporter
```

## Server 2 - PostgreSQL Database (104.248.172.65)

### 1. Installer PostgreSQL 17

```bash
sudo apt install -y postgresql-17 postgresql-contrib-17
sudo systemctl enable postgresql
sudo systemctl start postgresql
```

### 2. Konfigurer PostgreSQL

```bash
sudo -u postgres psql
```

I PostgreSQL shell:
```sql
CREATE DATABASE netbox;
CREATE USER netbox WITH PASSWORD '[REDACTED]';
ALTER DATABASE netbox OWNER TO netbox;
GRANT ALL PRIVILEGES ON DATABASE netbox TO netbox;
\q
```

### 3. Konfigurer ekstern tilgang

Rediger `/etc/postgresql/17/main/postgresql.conf`:
```
listen_addresses = '*'
```

Rediger `/etc/postgresql/17/main/pg_hba.conf`:
```
host    netbox          netbox          146.190.148.117/32      scram-sha-256
```

Restart PostgreSQL:
```bash
sudo systemctl restart postgresql
```

### 4. Installer Node Exporter

Samme fremgangsmåte som for Server 1.

### 5. Installer PostgreSQL Exporter

```bash
cd /tmp
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v0.16.0/postgres_exporter-0.16.0.linux-amd64.tar.gz
tar xvfz postgres_exporter-0.16.0.linux-amd64.tar.gz
sudo mv postgres_exporter-0.16.0.linux-amd64/postgres_exporter /usr/local/bin/
```

Opprett `/etc/systemd/system/postgres_exporter.service`:
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

```bash
sudo systemctl daemon-reload
sudo systemctl enable postgres_exporter
sudo systemctl start postgres_exporter
```

## Server 3 - Monitoring (165.232.66.71)

### 1. Installer Prometheus

```bash
cd /opt
sudo wget https://github.com/prometheus/prometheus/releases/download/v3.7.3/prometheus-3.7.3.linux-amd64.tar.gz
sudo tar xvfz prometheus-3.7.3.linux-amd64.tar.gz
sudo mv prometheus-3.7.3.linux-amd64 prometheus
sudo useradd -rs /bin/false prometheus
sudo chown -R prometheus:prometheus /opt/prometheus
```

Opprett `/opt/prometheus/prometheus.yml`:
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'server1-node'
    static_configs:
      - targets: ['146.190.148.117:9100']
        labels:
          instance: 'server1-netbox'

  - job_name: 'server2-node'
    static_configs:
      - targets: ['104.248.172.65:9100']
        labels:
          instance: 'server2-postgres'

  - job_name: 'server3-node'
    static_configs:
      - targets: ['localhost:9100']
        labels:
          instance: 'server3-monitoring'

  - job_name: 'nginx'
    static_configs:
      - targets: ['146.190.148.117:9113']
        labels:
          instance: 'server1-netbox'

  - job_name: 'redis'
    static_configs:
      - targets: ['146.190.148.117:9121']
        labels:
          instance: 'server1-netbox'

  - job_name: 'postgres'
    static_configs:
      - targets: ['104.248.172.65:9187']
        labels:
          instance: 'server2-postgres'

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

Opprett `/etc/systemd/system/prometheus.service`:
```ini
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/opt/prometheus/prometheus --config.file=/opt/prometheus/prometheus.yml --storage.tsdb.path=/opt/prometheus/data --storage.tsdb.retention.time=15d

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

### 2. Installer Grafana

```bash
sudo apt-get install -y software-properties-common
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo apt update
sudo apt install -y grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

### 3. Installer Blackbox Exporter

```bash
cd /tmp
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz
tar xvfz blackbox_exporter-0.25.0.linux-amd64.tar.gz
sudo mv blackbox_exporter-0.25.0.linux-amd64/blackbox_exporter /usr/local/bin/
sudo mkdir /etc/blackbox_exporter
```

Opprett `/etc/blackbox_exporter/blackbox.yml`:
```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: []
      method: GET
      preferred_ip_protocol: "ip4"
```

Opprett `/etc/systemd/system/blackbox_exporter.service`:
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

```bash
sudo systemctl daemon-reload
sudo systemctl enable blackbox_exporter
sudo systemctl start blackbox_exporter
```

### 4. Installer Node Exporter

Samme fremgangsmåte som for Server 1.

## Verifisering

### Sjekk Netbox

```bash
curl http://146.190.148.117
# Skal returnere Netbox login-side
```

### Sjekk Prometheus

```bash
curl http://165.232.66.71:9090/api/v1/targets
# Skal vise alle targets
```

### Sjekk Grafana

```bash
curl http://165.232.66.71:3000
# Skal returnere Grafana login-side
```

### Sjekk alle exporters

```bash
# Fra Server 3
curl http://146.190.148.117:9100/metrics | head
curl http://146.190.148.117:9113/metrics | head
curl http://146.190.148.117:9121/metrics | head
curl http://104.248.172.65:9100/metrics | head
curl http://104.248.172.65:9187/metrics | head
```

## Vanlige installasjons-problemer

### Problem: PostgreSQL connection refused

Sjekk at PostgreSQL lytter eksternt:
```bash
sudo ss -tulpn | grep 5432
```

### Problem: Exporter targets down i Prometheus

Sjekk at exporters lytter på 0.0.0.0, ikke kun localhost:
```bash
ss -tulpn | grep 9100
```

### Problem: Netbox returnerer 502 Bad Gateway

Sjekk Gunicorn status:
```bash
sudo systemctl status netbox
journalctl -u netbox -n 50
```

## Neste Steg

Etter installasjon:
1. Konfigurer Grafana datasource (se [Monitoring](04-monitoring.md))
2. Importer dashboards (se [Dashboards](05-dashboards.md))
3. Konfigurer alerting (se [Vedlikehold](08-vedlikehold.md))
