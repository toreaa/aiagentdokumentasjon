# Vedlikeh old og Drift

## Daglig Drift

### Morgensj ekk (Daily Health Check)

**Utfør hver morgen eller ved arbeidsstart:**

```bash
# 1. Sjekk alle servere er tilgjengelige
ping -c 3 146.190.148.117  # Server 1
ping -c 3 104.248.172.65   # Server 2
ping -c 3 165.232.66.71    # Server 3

# 2. Sjekk Grafana dashboards
# Åpne: http://165.232.66.71:3000
# Verifiser at alle dashboards viser data

# 3. Sjekk Prometheus targets
# Åpne: http://165.232.66.71:9090/targets
# Verifiser at alle targets er UP (unntatt Server 2 midlertidig)

# 4. Test Netbox tilgjengelighet
curl -I http://146.190.148.117
# Forventet: HTTP/1.1 200 OK
```

### Ukentlig Vedlikehold

**Hver fredag:**

1. **Gjennomgå Grafana dashboards for trender:**
   - CPU usage trending oppover?
   - Minne usage økende?
   - Disk space reduseres?

2. **Sjekk disk space:**
```bash
# På alle servere
df -h

# Spesielt viktig:
# Server 1: / (root)
# Server 2: /var/lib/postgresql
# Server 3: /opt/prometheus/data
```

3. **Review logs for errors:**
```bash
# Server 1
sudo journalctl -u netbox -p err --since "7 days ago"
sudo journalctl -u nginx -p err --since "7 days ago"

# Server 3
sudo journalctl -u prometheus -p err --since "7 days ago"
sudo journalctl -u grafana-server -p err --since "7 days ago"
```

4. **Backup check:**
   - Verifiser at backups kjører
   - Test restore prosedyre månedlig

### Månedlig Vedlikehold

**Første mandag i måneden:**

1. **System oppdateringer:**
```bash
# På alle servere (VIKTIG: Gjør i maintenance window)
sudo apt update
sudo apt list --upgradable
# Review liste før oppgradering
sudo apt upgrade -y
sudo reboot  # Hvis kernel ble oppgradert
```

2. **Sikkerhetsaudit:**
   - Gjennomgå brukertilganger
   - Sjekk for unødvendige åpne porter
   - Review firewall rules

3. **Kapasitetsplanlegging:**
   - Analyse av ressursbruk siste måned
   - Forecast for neste 3 måneder
   - Bestill oppgraderinger hvis nødvendig

4. **Dokumentasjonsgjennomgang:**
   - Oppdater dokumentasjon med endringer
   - Verifiser at passord dokumentasjon er oppdatert
   - Review troubleshooting guide

## Backup og Restore

### Backup Strategi

**Hva skal backes up:**

1. **Netbox Database (Kritisk - Daglig)**
```bash
# På Server 2 eller Server 1
pg_dump -h 104.248.172.65 -U netbox -d netbox -F c -f netbox_backup_$(date +%Y%m%d).dump
# Password: [REDACTED]

# Kopier til sikker lokasjon
scp netbox_backup_*.dump backup-server:/backups/netbox/
```

2. **Netbox konfigurasjoner (Ukentlig)**
```bash
# På Server 1
sudo tar -czf netbox-config_$(date +%Y%m%d).tar.gz \
  /opt/netbox/netbox/netbox/configuration.py \
  /opt/netbox/gunicorn.py \
  /etc/nginx/sites-available/netbox
```

3. **Grafana dashboards (Ukentlig)**
```bash
# På Server 3
sudo tar -czf grafana-dashboards_$(date +%Y%m%d).tar.gz \
  /var/lib/grafana/dashboards/ \
  /etc/grafana/grafana.ini \
  /etc/grafana/provisioning/
```

4. **Prometheus konfigurasjon (Ved endring)**
```bash
# På Server 3
sudo cp /opt/prometheus/prometheus.yml \
  /opt/prometheus/prometheus.yml.backup_$(date +%Y%m%d)
```

### Restore Prosedyrer

**Restore Netbox Database:**
```bash
# 1. Stopp Netbox
sudo systemctl stop netbox netbox-rq

# 2. Drop og recrea te database
sudo -u postgres psql << EOF
DROP DATABASE netbox;
CREATE DATABASE netbox;
GRANT ALL PRIVILEGES ON DATABASE netbox TO netbox;
\q
EOF

# 3. Restore from backup
pg_restore -h 104.248.172.65 -U netbox -d netbox netbox_backup_20240110.dump

# 4. Start Netbox
sudo systemctl start netbox netbox-rq
```

**Restore Grafana Dashboards:**
```bash
# 1. Stop Grafana
sudo systemctl stop grafana-server

# 2. Restore files
sudo tar -xzf grafana-dashboards_20240110.tar.gz -C /

# 3. Fix permissions
sudo chown -R grafana:grafana /var/lib/grafana/dashboards/

# 4. Start Grafana
sudo systemctl start grafana-server
```

## Oppgraderinger

### Netbox Oppgradering

**Før oppgradering:**
1. Les release notes: https://github.com/netbox-community/netbox/releases
2. Ta full backup av database og konfigurasjon
3. Test i staging miljø hvis mulig

**Oppgraderingsprosedyre:**
```bash
# 1. Backup
pg_dump -h 104.248.172.65 -U netbox -d netbox -F c -f netbox_pre_upgrade.dump

# 2. Stopp Netbox
sudo systemctl stop netbox netbox-rq

# 3. Pull ny versjon
cd /opt/netbox
sudo git fetch
sudo git checkout v4.4.6  # Ny versjon

# 4. Oppgrader Python packages
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt

# 5. Kjør migrasjoner
cd netbox
python3 manage.py migrate

# 6. Collect static files
python3 manage.py collectstatic --no-input

# 7. Start Netbox
sudo systemctl start netbox netbox-rq

# 8. Verifiser
curl -I http://146.190.148.117
# Sjekk at versjon er oppdatert i UI
```

### Prometheus Oppgradering

```bash
# 1. Last ned ny versjon
cd /opt
sudo wget https://github.com/prometheus/prometheus/releases/download/v3.8.0/prometheus-3.8.0.linux-amd64.tar.gz
sudo tar xvfz prometheus-3.8.0.linux-amd64.tar.gz

# 2. Backup config
sudo cp /opt/prometheus/prometheus.yml /opt/prometheus/prometheus.yml.backup

# 3. Stopp Prometheus
sudo systemctl stop prometheus

# 4. Oppdater symlink
sudo rm /opt/prometheus
sudo mv prometheus-3.8.0.linux-amd64 prometheus
sudo ln -s /opt/prometheus-3.8.0.linux-amd64 /opt/prometheus

# 5. Restore config
sudo cp /opt/prometheus/prometheus.yml.backup /opt/prometheus/prometheus.yml

# 6. Fix permissions
sudo chown -R prometheus:prometheus /opt/prometheus

# 7. Start Prometheus
sudo systemctl start prometheus

# 8. Verifiser
curl http://localhost:9090/-/healthy
```

### Grafana Oppgradering

```bash
# 1. Backup
sudo tar -czf grafana-backup.tar.gz /var/lib/grafana /etc/grafana

# 2. Oppgrader via apt
sudo apt update
sudo apt install grafana

# 3. Restart
sudo systemctl restart grafana-server

# 4. Verifiser
curl -I http://localhost:3000
```

## Sikkerhet

### Passord Rotasjon

**Hver 90. dag:**

1. **Netbox admin passord:**
```bash
cd /opt/netbox/netbox
source ../venv/bin/activate
python3 manage.py changepassword admin
```

2. **Grafana admin passord:**
```bash
# Via Grafana UI: Profile → Change Password
# Eller via grafana-cli:
sudo grafana-cli admin reset-admin-password NewPassword123!
```

3. **PostgreSQL passord:**
```bash
sudo -u postgres psql
ALTER USER netbox WITH PASSWORD 'NewDatabasePassword123';
\q

# Oppdater configuration.py på Server 1
sudo nano /opt/netbox/netbox/netbox/configuration.py
# Endre DATABASE['PASSWORD']

# Restart Netbox
sudo systemctl restart netbox netbox-rq
```

4. **SSH bruker passord:**
```bash
# På alle servere
sudo passwd opsuser
```

### Firewall Konfigurasjon

**Installer UFW (hvis ikke installert):**
```bash
sudo apt install ufw
```

**Server 1 (Netbox):**
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow from 165.232.66.71 to any port 9100  # Node Exporter
sudo ufw allow from 165.232.66.71 to any port 9113  # Nginx Exporter
sudo ufw allow from 165.232.66.71 to any port 9121  # Redis Exporter
sudo ufw enable
```

**Server 2 (PostgreSQL):**
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp    # SSH
sudo ufw allow from 146.190.148.117 to any port 5432  # PostgreSQL
sudo ufw allow from 165.232.66.71 to any port 9100  # Node Exporter
sudo ufw allow from 165.232.66.71 to any port 9187  # PostgreSQL Exporter
sudo ufw enable
```

**Server 3 (Monitoring):**
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 3000/tcp  # Grafana
sudo ufw allow 9090/tcp  # Prometheus
sudo ufw enable
```

### SSL/TLS Sertifikater

**Installer Let's Encrypt:**
```bash
sudo apt install certbot python3-certbot-nginx
```

**Opprett sertifikat for Netbox:**
```bash
# På Server 1
sudo certbot --nginx -d netbox.example.com

# Auto-renewal
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer
```

## Monitoring og Alerting

### Alertmanager Installasjon

**Status:** Ikke implementert ennå

**Installasjonssteg:**

1. **Last ned Alertmanager:**
```bash
# På Server 3
cd /opt
sudo wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
sudo tar xvfz alertmanager-0.27.0.linux-amd64.tar.gz
sudo mv alertmanager-0.27.0.linux-amd64 alertmanager
```

2. **Konfigurer Alertmanager:**
```yaml
# /opt/alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@example.com'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password: 'password'

route:
  group_by: ['alertname', 'instance']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'email-notifications'

receivers:
  - name: 'email-notifications'
    email_configs:
      - to: 'admin@example.com'
        headers:
          Subject: "Alert: {{ .GroupLabels.alertname }}"
```

3. **Opprett systemd service:**
```ini
# /etc/systemd/system/alertmanager.service
[Unit]
Description=Alertmanager
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/opt/alertmanager/alertmanager --config.file=/opt/alertmanager/alertmanager.yml --storage.path=/opt/alertmanager/data

[Install]
WantedBy=multi-user.target
```

4. **Konfigurer Prometheus alerts:**
```yaml
# /opt/prometheus/alert.rules.yml
groups:
  - name: system_alerts
    interval: 30s
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} down"
          description: "{{ $labels.instance }} has been down for more than 5 minutes."

      - alert: HighCPUUsage
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is above 80% for 5 minutes."

      - alert: LowMemory
        expr: (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 < 20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low memory on {{ $labels.instance }}"
          description: "Available memory is below 20%."

      - alert: DiskSpaceL ow
        expr: 100 - ((node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100) > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk space low on {{ $labels.instance }}"
          description: "Disk usage is above 90%."

      - alert: EndpointDown
        expr: probe_success == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Endpoint {{ $labels.instance }} is down"
          description: "{{ $labels.instance }} has been unreachable for 2 minutes."
```

5. **Oppdater Prometheus config:**
```yaml
# Legg til i /opt/prometheus/prometheus.yml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - localhost:9093

rule_files:
  - "alert.rules.yml"
```

## Performance Tuning

### Netbox Optimalisering

**Gunicorn workers:**
```python
# /opt/netbox/gunicorn.py
workers = (CPU_COUNT * 2) + 1
threads = 3
worker_class = 'gthread'
max_requests = 5000
max_requests_jitter = 500
```

**Database connection pooling:**
```python
# /opt/netbox/netbox/netbox/configuration.py
DATABASE = {
    ...
    'CONN_MAX_AGE': 300,
}
```

### PostgreSQL Tuning

**Konfigurasjon for 2GB RAM server:**
```conf
# /etc/postgresql/17/main/postgresql.conf
shared_buffers = 512MB
effective_cache_size = 1536MB
maintenance_work_mem = 128MB
work_mem = 5MB
max_connections = 100
```

**Vacuum og analyze:**
```bash
# Kjør ukentlig
sudo -u postgres psql -d netbox -c "VACUUM ANALYZE;"
```

### Prometheus Optimalisering

**Reduser retention hvis disk er fullt:**
```bash
# Rediger /etc/systemd/system/prometheus.service
--storage.tsdb.retention.time=7d

sudo systemctl daemon-reload
sudo systemctl restart prometheus
```

## Troubleshooting Playbook

### Problem: Netbox returnerer 502 Bad Gateway

**Diagnose:**
```bash
sudo systemctl status netbox
sudo journalctl -u netbox -n 50 --no-pager
```

**Løsning:**
```bash
sudo systemctl restart netbox netbox-rq
```

### Problem: Høy load på Server 1

**Diagnose:**
```bash
uptime
top -o %CPU
ps aux --sort=-%cpu | head -20
```

**Løsning:**
- Sjekk for runserver prosesser: `ps aux | grep runserver` → KILL
- Sjekk Promtail CPU: Stopp hvis nødvendig
- Restart tjenester hvis memory leak

### Problem: PostgreSQL connection refused

**Diagnose:**
```bash
# Fra Server 1
psql -h 104.248.172.65 -U netbox -d netbox

# På Server 2
sudo systemctl status postgresql
sudo ss -tulpn | grep 5432
```

**Løsning:**
```bash
# Start PostgreSQL
sudo systemctl start postgresql

# Sjekk pg_hba.conf
sudo cat /etc/postgresql/17/main/pg_hba.conf | grep netbox
```

### Problem: Prometheus target down

**Diagnose:**
```bash
# Test fra Server 3
curl http://146.190.148.117:9100/metrics

# På target server
systemctl status node_exporter
ss -tulpn | grep 9100
```

**Løsning:**
```bash
# Restart exporter
sudo systemctl restart node_exporter

# Verifiser lytter på 0.0.0.0
ss -tulpn | grep 9100
```

## Kontaktinformasjon

**Systemadministrator:**
- Navn: [NAVN]
- Epost: [EPOST]
- Telefon: [TELEFON]

**Eskalering ved kritiske issues:**
1. Kontakt systemadministrator
2. Hvis ikke tilgjengelig: [BACKUP KONTAKT]
3. Kritiske alerts → SMS + Email

**Vendor Support:**
- Netbox Community: https://github.com/netbox-community/netbox/discussions
- Prometheus: https://prometheus.io/community/
- Grafana: https://community.grafana.com/

## Vedlikeholdslogg

Dokumenter alle vedlikeholdsaktiviteter:

```
Dato: 2024-01-10
Utført av: Admin
Aktivitet: Oppgradert Netbox fra v4.4.4 til v4.4.5
Resultat: Vellykket
Nedtid: 5 minutter
Notater: Ingen problemer
```

Bruk git repository eller spreadsheet for logging.
