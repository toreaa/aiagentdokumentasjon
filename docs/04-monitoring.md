# Monitoring Stack

## Oversikt

Monitoring-løsningen bruker Prometheus for metrics-innsamling og Grafana for visualisering.

## Arkitektur

```
Prometheus (Server 3) ─┬─▶ 9 Targets monitored
                       │
                       ├─▶ Server 1: Node, Nginx, Redis exporters
                       ├─▶ Server 2: Node, PostgreSQL exporters
                       ├─▶ Server 3: Node exporter
                       └─▶ Blackbox: HTTP endpoint checks

Grafana (Server 3) ────▶ Prometheus ─▶ 6 Dashboards
```

## Komponenter

### Prometheus

**Versjon:** 3.7.3
**Plassering:** Server 3 (165.232.66.71:9090)
**Formål:** Metrics collection og time-series database

**Konfigurasjon:**
- Scrape interval: 15 sekunder
- Retention: 15 dager
- 9 aktive targets
- Ekstern lagring: Lokal disk

**Tilgang:**
```bash
# Web UI
http://165.232.66.71:9090

# API
curl "http://165.232.66.71:9090/api/v1/query?query=up"
```

### Grafana

**Versjon:** 11.x
**Plassering:** Server 3 (165.232.66.71:3000)
**Formål:** Visualisering og dashboards

**Konfigurasjon:**
- Datasource: Prometheus (http://localhost:9090)
- Dashboard provisioning: `/var/lib/grafana/dashboards/`
- Admin bruker: admin
- Passord: [REDACTED]

**Tilgang:**
```bash
# Web UI
http://165.232.66.71:3000

# Logg inn med admin/[REDACTED]
```

## Exporters

### Node Exporter

**Versjon:** 1.8.2
**Port:** 9100
**Installert på:** Alle 3 servere

**Metrics samlet:**
- CPU usage og load average
- Memory usage (RAM, swap)
- Disk usage og I/O
- Network traffic
- System uptime

**Verifisering:**
```bash
curl http://146.190.148.117:9100/metrics | grep node_load1
curl http://104.248.172.65:9100/metrics | grep node_memory_MemAvailable_bytes
curl http://165.232.66.71:9100/metrics | grep node_filesystem_avail_bytes
```

### Redis Exporter

**Versjon:** 1.62.0
**Port:** 9121
**Installert på:** Server 1

**Metrics samlet:**
- Connected clients
- Memory usage
- Commands processed
- Keys per database
- Replication status
- Uptime

**Verifisering:**
```bash
curl http://146.190.148.117:9121/metrics | grep redis_connected_clients
```

**Viktig konfigurasjon:**
```ini
ExecStart=/usr/local/bin/redis_exporter -redis.addr=localhost:6379 -web.listen-address=0.0.0.0:9121
```

### Nginx Exporter

**Versjon:** 1.4.1
**Port:** 9113
**Installert på:** Server 1

**Metrics samlet:**
- Active connections
- Requests per second
- Reading/writing/waiting connections
- Request status codes

**Krav:**
Nginx må ha stub_status aktivert:
```nginx
location /nginx_status {
    stub_status on;
    allow 127.0.0.1;
    allow 165.232.66.71;
    deny all;
}
```

**Verifisering:**
```bash
curl http://146.190.148.117:9113/metrics | grep nginx_connections_active
```

### PostgreSQL Exporter

**Versjon:** 0.16.0
**Port:** 9187
**Installert på:** Server 2

**Metrics samlet:**
- Database size
- Connection count
- Transaction throughput
- Query performance
- Table statistics
- Replication lag

**Konfigurasjon:**
```ini
Environment="DATA_SOURCE_NAME=postgresql://netbox:[REDACTED]@localhost:5432/netbox?sslmode=disable"
```

**Verifisering:**
```bash
curl http://104.248.172.65:9187/metrics | grep pg_database_size_bytes
```

### Blackbox Exporter

**Versjon:** 0.25.0
**Port:** 9115
**Installert på:** Server 3

**Formål:** HTTP endpoint monitoring

**Monitored endpoints:**
- http://146.190.148.117 (Netbox)
- http://165.232.66.71:3000 (Grafana)

**Metrics samlet:**
- probe_success (1 = up, 0 = down)
- probe_duration_seconds
- probe_http_status_code
- probe_ssl_earliest_cert_expiry

**Verifisering:**
```bash
curl "http://165.232.66.71:9115/probe?module=http_2xx&target=http://146.190.148.117"
```

## Prometheus Targets

### Target oversikt

| Job | Instance | Endpoint | Status |
|-----|----------|----------|--------|
| prometheus | server3-monitoring | localhost:9090 | UP |
| server1-node | server1-netbox | 146.190.148.117:9100 | UP |
| server2-node | server2-postgres | 104.248.172.65:9100 | DOWN* |
| server3-node | server3-monitoring | localhost:9100 | UP |
| nginx | server1-netbox | 146.190.148.117:9113 | UP |
| redis | server1-netbox | 146.190.148.117:9121 | UP |
| postgres | server2-postgres | 104.248.172.65:9187 | DOWN* |
| blackbox | netbox | via localhost:9115 | UP |
| blackbox | grafana | via localhost:9115 | UP |

*Server 2 midlertidig utilgjengelig (SSH timeout)

### Sjekke target status

**Via Prometheus UI:**
```
http://165.232.66.71:9090/targets
```

**Via API:**
```bash
curl -s "http://165.232.66.71:9090/api/v1/targets" | jq '.data.activeTargets[] | {job: .labels.job, health: .health, lastError: .lastError}'
```

**Via PromQL:**
```promql
# Se alle targets som er oppe
up

# Targets som er nede
up == 0
```

## Nyttige PromQL Queries

### System Health

```promql
# CPU usage percentage
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage percentage
100 - ((node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100)

# Disk usage percentage
100 - ((node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100)

# Load average
node_load1
node_load5
node_load15
```

### Application Metrics

```promql
# Redis connected clients
redis_connected_clients

# Redis memory usage (bytes)
redis_memory_used_bytes

# Nginx active connections
nginx_connections_active

# PostgreSQL database size (bytes)
pg_database_size_bytes{datname="netbox"}

# PostgreSQL active connections
pg_stat_activity_count{datname="netbox",state="active"}
```

### Endpoint Monitoring

```promql
# Endpoint availability (1 = up, 0 = down)
probe_success

# HTTP response time
probe_http_duration_seconds

# HTTP status code
probe_http_status_code
```

### Alerting Queries

```promql
# High CPU usage (>80%)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80

# Low memory (<20% available)
(node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 < 20

# High load average (>4)
node_load5 > 4

# Endpoint down
probe_success == 0

# Service down
up{job!="server2-node"} == 0
```

## Log Aggregation (Loki)

**Status:** Installert men STOPPET pga ressursproblemer

**Versjon:** 3.5.8
**Plassering:** Server 3 (port 3100)

### Promtail

**Status:** STOPPET pga høy CPU-bruk (18%)

**Årsak til stopping:**
- Server 1 har kun 1.9GB RAM
- Promtail brukte for mye CPU
- Forårsaket høy load (40+)

**Løsning:**
Promtail kan re-aktiveres etter:
1. Minne-oppgradering på Server 1
2. Optimalisering av log scraping intervall
3. Filtrering av mindre viktige logs

## Grafana Datasource Konfigurasjon

### Legge til Prometheus datasource

**Via UI:**
1. Logg inn på Grafana (http://165.232.66.71:3000)
2. Gå til Configuration → Data Sources
3. Klikk "Add data source"
4. Velg "Prometheus"
5. Konfigurer:
   - Name: Prometheus
   - URL: http://localhost:9090
   - Access: Server (default)
   - Scrape interval: 15s
6. Klikk "Save & Test"

**Via provisioning:**

Fil: `/etc/grafana/provisioning/datasources/prometheus.yml`

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

## Alerting (Planlagt)

**Status:** Ikke implementert ennå

**Planlagte alerts:**
- Server down (up == 0)
- High CPU usage (>80% i 5 min)
- Low memory (<20% available)
- High disk usage (>90%)
- Endpoint down (probe_success == 0)
- PostgreSQL down
- Redis down
- Nginx down

**Implementasjon krever:**
1. Alertmanager installasjon
2. Prometheus alert rules
3. Notification channels (email, Slack, etc.)

Se [Vedlikehold](08-vedlikehold.md) for implementasjonsplan.

## Monitoring Best Practices

### Scrape Intervals

- Standard: 15 sekunder
- For høy-frekvens metrics: 5 sekunder
- For mindre kritiske metrics: 60 sekunder

### Retention

- Prometheus: 15 dager
- Loki (når aktiv): 7 dager

### Target Labeling

Konsistent labeling for enkel filtrering:
```yaml
labels:
  instance: 'server1-netbox'
  server: 'netbox'
  environment: 'production'
```

### Dashboard Organization

1. **SLA Dashboard** - Overordnet status
2. **System Dashboards** - Per-server metrics
3. **Application Dashboards** - Per-tjeneste metrics
4. **Troubleshooting Dashboards** - Detaljert analyse

## Troubleshooting

### Prometheus target down

```bash
# Sjekk exporter status
systemctl status node_exporter

# Sjekk at exporter lytter eksternt
ss -tulpn | grep 9100

# Test connectivity fra Prometheus server
curl http://146.190.148.117:9100/metrics
```

### Grafana shows "No data"

1. Sjekk datasource:
   - Grafana → Configuration → Data Sources → Prometheus
   - Klikk "Test" - skal være grønn
2. Sjekk at Prometheus har data:
   ```bash
   curl "http://165.232.66.71:9090/api/v1/query?query=up"
   ```
3. Sjekk dashboard datasource:
   - Dashboard settings → JSON Model
   - Se etter `"datasource"` felt

### Høy Prometheus disk usage

```bash
# Sjekk disk usage
du -sh /opt/prometheus/data

# Reduser retention
# Rediger prometheus.service:
--storage.tsdb.retention.time=7d

# Restart
sudo systemctl restart prometheus
```

## Ressursbruk

### Server 3 (Monitoring)

**Prometheus:**
- RAM: ~500MB
- CPU: <5%
- Disk: ~2GB (15 dagers data)

**Grafana:**
- RAM: ~200MB
- CPU: <2%

**Exporters:**
- Node Exporter: ~10MB RAM, <1% CPU
- Blackbox Exporter: ~15MB RAM, <1% CPU

### Server 1 (Netbox)

**Exporters:**
- Node Exporter: ~10MB RAM, <1% CPU
- Redis Exporter: ~15MB RAM, <1% CPU
- Nginx Exporter: ~10MB RAM, <1% CPU

### Server 2 (PostgreSQL)

**Exporters:**
- Node Exporter: ~10MB RAM, <1% CPU
- PostgreSQL Exporter: ~20MB RAM, <2% CPU

## Neste Steg

1. Implementere Alertmanager (se [Vedlikehold](08-vedlikehold.md))
2. Opprette flere dashboards (se [Dashboards](05-dashboards.md))
3. Re-aktivere Loki/Promtail etter ressursoppgradering
4. Konfigurere notification channels
5. Lage custom metrics for Netbox-spesifikk monitoring
