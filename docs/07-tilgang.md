# Tilgang og Påloggingsinformasjon

## Web-grensesnitt

### Netbox

```
URL: http://146.190.148.117
Brukernavn: admin
Passord: [REDACTED]
```

**Funksjonalitet:**
- IPAM (IP Address Management)
- DCIM (Data Center Infrastructure Management)
- Device tracking
- Circuit management

### Grafana Dashboards

```
URL: http://165.232.66.71:3000
Brukernavn: admin
Passord: [REDACTED]
```

**Tilgjengelige dashboards:**
1. Node Exporter Full - System metrics
2. PostgreSQL Database - Database health
3. Redis Dashboard - Cache metrics
4. NGINX Exporter - Web server metrics
5. Prometheus Blackbox Exporter - Endpoint monitoring
6. SLA Monitoring Dashboard - Overall status (hvis opprettet)
7. Endpoint Monitoring - Blackbox (custom)

### Prometheus

```
URL: http://165.232.66.71:9090
Ingen autentisering påkrevd
```

**Nyttige queries:**
- `up` - Se alle targets som er oppe
- `node_load1` - System load
- `redis_connected_clients` - Redis connections
- `probe_success` - Endpoint availability

## SSH Tilgang

### Generell pålogging

Alle servere bruker samme bruker og passord:

```bash
Bruker: opsuser
Passord: [REDACTED]
```

### Server 1 - Netbox (146.190.148.117)

```bash
ssh opsuser@146.190.148.117
# Passord: [REDACTED]
```

**Viktige stier:**
- Netbox: `/opt/netbox/`
- Virtual env: `/opt/netbox/venv/`
- Configuration: `/opt/netbox/netbox/netbox/configuration.py`
- Logs: `/var/log/nginx/` og `journalctl -u netbox`

### Server 2 - PostgreSQL (104.248.172.65)

```bash
ssh opsuser@104.248.172.65
# Passord: [REDACTED]
```

**STATUS: MIDLERTIDIG UTILGJENGELIG - SSH timeout**

**Viktige stier:**
- PostgreSQL config: `/etc/postgresql/17/main/`
- Data: `/var/lib/postgresql/17/main/`

### Server 3 - Monitoring (165.232.66.71)

```bash
ssh opsuser@165.232.66.71
# Passord: [REDACTED]
```

**Viktige stier:**
- Prometheus: `/opt/prometheus/`
- Prometheus config: `/opt/prometheus/prometheus.yml`
- Grafana dashboards: `/var/lib/grafana/dashboards/`
- Grafana config: `/etc/grafana/`

## Database Tilgang

### PostgreSQL på Server 2

**Fra Server 1 (Netbox):**
```bash
psql -h 104.248.172.65 -U netbox -d netbox
# Passord: [REDACTED]
```

**Direkte på Server 2 (som postgres user):**
```bash
sudo -u postgres psql
```

**Database detaljer:**
- Database navn: `netbox`
- Database bruker: `netbox`
- Database passord: `[REDACTED]`
- Port: 5432

### Redis på Server 1

**Lokal tilgang:**
```bash
redis-cli
# Ingen passord konfigurert
```

**Databases:**
- DB 0: Task queue (RQ worker)
- DB 1: Application caching

## Systemd Tjenester

### Server 1 - Netbox

```bash
# Netbox hovedtjeneste
sudo systemctl status netbox
sudo systemctl restart netbox

# Netbox RQ worker (background tasks)
sudo systemctl status netbox-rq
sudo systemctl restart netbox-rq

# Nginx
sudo systemctl status nginx
sudo systemctl restart nginx

# Redis
sudo systemctl status redis-server

# Exporters
sudo systemctl status node_exporter
sudo systemctl status redis_exporter
sudo systemctl status nginx_exporter

# Promtail (STOPPET - ressursproblemer)
sudo systemctl status promtail
```

### Server 2 - PostgreSQL

```bash
# PostgreSQL
sudo systemctl status postgresql

# Exporters
sudo systemctl status node_exporter
sudo systemctl status postgres_exporter
```

### Server 3 - Monitoring

```bash
# Prometheus
sudo systemctl status prometheus
sudo systemctl restart prometheus

# Grafana
sudo systemctl status grafana-server
sudo systemctl restart grafana-server

# Exporters
sudo systemctl status node_exporter
sudo systemctl status blackbox_exporter

# Loki (hvis installert)
sudo systemctl status loki
```

## Netbox Internals

### Django Admin

Tilgang via command line:
```bash
cd /opt/netbox/netbox
source ../venv/bin/activate
python3 manage.py shell
```

### Netbox SECRET_KEY

**KRITISK - Må holdes hemmelig!**
```
SECRET_KEY = '[REDACTED]'
```

Brukes for:
- Session encryption
- Password hashing
- CSRF tokens

## API Tilgang

### Netbox API

```
Endpoint: http://146.190.148.117/api/
Dokumentasjon: http://146.190.148.117/api/docs/
```

**Opprette API token:**
1. Logg inn på Netbox web-grensesnitt
2. Gå til din profil (øverst til høyre)
3. Velg "API Tokens"
4. Klikk "Add a token"

### Prometheus API

```
Endpoint: http://165.232.66.71:9090/api/v1/
Eksempel: http://165.232.66.71:9090/api/v1/query?query=up
```

### Grafana API

```
Endpoint: http://165.232.66.71:3000/api/
Auth: Basic auth med admin:[REDACTED]
```

## Sikkerhetsnotater

**VIKTIG:**
- ⚠️ Alle passord i dette dokumentet er for demo-formål
- ⚠️ Bytt alle passord før produksjon!
- ⚠️ Bruk SSH nøkler i stedet for passord i produksjon
- ⚠️ SECRET_KEY må aldri committes til git
- ⚠️ Database passord må roteres regelmessig

## Backup av Credentials

For katastrofe-recovery, husk å ta backup av:
1. `/opt/netbox/netbox/netbox/configuration.py` (SECRET_KEY)
2. PostgreSQL dump med credentials
3. Grafana dashboards (`/var/lib/grafana/dashboards/`)
4. Prometheus config (`/opt/prometheus/prometheus.yml`)
