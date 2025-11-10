# Grafana Dashboards

## Oversikt

Systemet har 6 Grafana dashboards for overvåking av infrastrukturen.

## Tilgjengelige Dashboards

### 1. Node Exporter Full

**Formål:** Detaljert system metrics for alle servere

**Metrics vist:**
- CPU usage (per core og totalt)
- Load average (1m, 5m, 15m)
- Memory usage (total, used, available, cached)
- Swap usage
- Disk usage per mountpoint
- Disk I/O (read/write bytes)
- Network traffic (RX/TX bytes)
- System uptime
- Open file descriptors

**Bruksområde:**
- Kapasitetsplanlegging
- Identifisere ressursflaskehalser
- Historisk analyse av ressursbruk

**Panels:**
- CPU Usage %
- Load Average
- Memory Usage
- Disk Usage
- Network Traffic
- System Uptime

### 2. PostgreSQL Database

**Formål:** Database health og performance

**Metrics vist:**
- Database size (bytes)
- Number of connections (total, active, idle)
- Transaction rate (commits/rollbacks per second)
- Tuple statistics (inserted, updated, deleted)
- Cache hit ratio
- Deadlocks
- Locks by type
- Table sizes
- Index usage

**Bruksområde:**
- Database performance tuning
- Connection pool sizing
- Query optimization
- Capacity planning

**Viktige alerts å overvåke:**
- Connection count nær max_connections
- Lav cache hit ratio (<90%)
- Høy antall deadlocks
- Database størrelse nær disk kapasitet

### 3. Redis Dashboard

**Formål:** Cache metrics og performance

**Metrics vist:**
- Connected clients
- Memory usage (used, peak, fragmentation)
- Keys per database (DB 0 og DB 1)
- Commands processed per second
- Keyspace hits/misses
- Evicted keys
- Expired keys
- Network I/O
- CPU usage

**Bruksområde:**
- Cache effectiveness monitoring
- Memory optimization
- Connection monitoring

**Viktige metrics:**
- `redis_connected_clients` - Antall aktive connections
- `redis_memory_used_bytes` - Minne i bruk
- `redis_keyspace_hits_total` / `redis_keyspace_misses_total` - Cache hit ratio

### 4. NGINX Exporter

**Formål:** Web server performance

**Metrics vist:**
- Active connections
- Requests per second
- Reading/Writing/Waiting connections
- Accepts/Handled/Requests counts

**Bruksområde:**
- Web traffic monitoring
- Connection pool sizing
- Load balancer beslutninger

**Viktige metrics:**
- `nginx_connections_active` - Aktive connections
- `nginx_connections_reading` - Connections som leser request
- `nginx_connections_writing` - Connections som skriver response
- `nginx_connections_waiting` - Idle keepalive connections

### 5. Prometheus Blackbox Exporter

**Formål:** Endpoint availability monitoring

**Metrics vist:**
- Probe success (up/down status)
- HTTP status codes
- Response time (probe duration)
- SSL certificate expiry
- DNS lookup time
- TCP connection time
- TLS handshake time

**Monitored endpoints:**
- http://146.190.148.117 (Netbox)
- http://165.232.66.71:3000 (Grafana)

**Bruksområde:**
- Uptime monitoring
- SLA tracking
- Response time trends
- SSL certificate monitoring

**Panels:**
- Probe Success/Failure
- HTTP Status Codes
- Response Time
- SSL Certificate Days Remaining

### 6. SLA Monitoring Dashboard (Planlagt)

**Formål:** High-level oversikt for management

**Planlagt innhold:**
- Overall system status (alle komponenter)
- Uptime percentage (99.9% target)
- Critical alerts overview
- Service availability timeline
- Response time percentiles (p50, p95, p99)

**Status:** Ikke opprettet ennå - se implementasjonsplan under.

## Dashboard Organisation

### Folder struktur

```
Grafana
├── Home (default dashboard)
├── System Metrics
│   ├── Node Exporter Full
│   └── (future: per-server dashboards)
├── Application Metrics
│   ├── PostgreSQL Database
│   ├── Redis Dashboard
│   └── NGINX Exporter
├── Endpoint Monitoring
│   └── Prometheus Blackbox Exporter
└── SLA & Alerting
    └── (future: SLA Dashboard, Alertmanager)
```

## Importere Dashboards

### Fra Grafana.com

Mange dashboards er tilgjengelige på https://grafana.com/grafana/dashboards/

**Eksempel - Node Exporter Full:**

1. Gå til https://grafana.com/grafana/dashboards/1860
2. Kopier Dashboard ID: `1860`
3. I Grafana UI:
   - Klikk "+" → "Import"
   - Lim inn ID: 1860
   - Velg "Prometheus" datasource
   - Klikk "Import"

**Populære dashboard IDs:**
- Node Exporter Full: `1860`
- PostgreSQL Database: `9628`
- Redis Dashboard: `11835`
- Nginx: `12708`
- Blackbox Exporter: `7587`

### Via JSON fil

**Eksempel - legge til custom dashboard:**

1. Eksporter dashboard fra en annen Grafana:
   - Dashboard Settings → JSON Model
   - Kopier JSON
2. I target Grafana:
   - "+" → "Import"
   - Lim inn JSON
   - Klikk "Load"

### Via Provisioning

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

**Legg dashboard JSON filer i:**
```bash
/var/lib/grafana/dashboards/
```

Grafana laster dashboards automatisk ved oppstart.

## Opprette Custom Dashboards

### SLA Dashboard Implementasjon

**Mål:** Single-pane-of-glass for system health

**Steg 1: Opprett dashboard**
```bash
# I Grafana UI
+ → Create Dashboard → Add new panel
```

**Steg 2: Legg til panels**

**Panel 1: System Status**
```promql
# Visualization: Stat
# Query:
up{job!="server2-node"}

# Options:
- Value mappings: 1 = UP (green), 0 = DOWN (red)
- Repeat by: instance
```

**Panel 2: Uptime Percentage**
```promql
# Visualization: Gauge
# Query:
avg_over_time(up{job!="server2-node"}[24h]) * 100

# Options:
- Thresholds: 0-95 (red), 95-99 (yellow), 99-100 (green)
- Unit: Percent (0-100)
```

**Panel 3: Response Times**
```promql
# Visualization: Time series
# Query:
probe_http_duration_seconds{job="blackbox"}

# Legend: {{instance}}
```

**Panel 4: Active Alerts**
```promql
# Visualization: Stat
# Query:
ALERTS{alertstate="firing"}

# Options:
- Color: Red if > 0
```

**Panel 5: Service Health Timeline**
```promql
# Visualization: Status history
# Query:
up{job!="server2-node"}

# Options:
- Show legend
- Group by: instance
```

## Dashboard Variables

### Bruke variabler for dynamiske dashboards

**Eksempel: Server filter**

1. Dashboard Settings → Variables → Add variable
2. Konfigurer:
   ```
   Name: server
   Type: Query
   Query: label_values(up, instance)
   Multi-value: yes
   Include All option: yes
   ```
3. Bruk i queries:
   ```promql
   up{instance=~"$server"}
   node_load1{instance=~"$server"}
   ```

## Dashboard Annotations

### Legge til deployment markers

**Automatisk via API:**
```bash
curl -X POST http://165.232.66.71:3000/api/annotations \
  -H "Content-Type: application/json" \
  -u admin:[REDACTED] \
  -d '{
    "text": "Netbox deployment v4.4.6",
    "tags": ["deployment", "netbox"],
    "time": '$(date +%s000)'
  }'
```

**Manuelt:**
1. Ctrl+Click på dashboard
2. "Add annotation"
3. Skriv beskrivelse
4. Lagre

## Dashboard Alerting

### Opprette alert på dashboard panel

1. Edit panel
2. Alert tab
3. Create alert rule:
   ```
   Name: High CPU Usage
   Condition: WHEN avg() OF query(A, 5m) IS ABOVE 80
   Evaluate every: 1m
   For: 5m
   ```
4. Add notification channel (email, Slack, etc.)

## Dashboard Sharing

### Offentlig link (uten login)

1. Dashboard Settings → Links
2. Enable "Public dashboard"
3. Kopier link

### Eksport til PDF/PNG

1. Dashboard → Share → Export
2. Velg format (PDF/PNG)
3. Velg time range
4. Download

### Snapshot

1. Dashboard → Share → Snapshot
2. Publish til Grafana.com eller lokal snapshot
3. Del link

## Dashboard Best Practices

### Design Guidelines

1. **Start bred, drill ned:**
   - Overordnet status først
   - Detaljert troubleshooting på nederste nivå

2. **Konsistent layout:**
   - Viktigste metrics øverst
   - Time series i midten
   - Logs/details nederst

3. **Bruk colors effektivt:**
   - Grønn: OK/good
   - Gul: Warning
   - Rød: Critical/error

4. **Optimiser for skjermstørrelse:**
   - Test på både laptop og stor skjerm
   - Bruk row collapse for mange panels

### Query Optimization

1. **Bruk rate() for counters:**
   ```promql
   rate(node_cpu_seconds_total[5m])
   ```

2. **Unngå large time ranges:**
   - Use $__interval variable
   - Step: Auto

3. **Filter tidlig:**
   ```promql
   # God
   up{job="nginx"}

   # Dårlig
   up and job="nginx"
   ```

## Troubleshooting Dashboards

### Dashboard viser "No data"

1. **Sjekk datasource:**
   - Test connection i Data Sources
   - Verifiser URL er korrekt

2. **Sjekk query:**
   - Kjør query i Prometheus UI
   - Sjekk at metrics eksisterer

3. **Sjekk time range:**
   - Utvid time range
   - Sjekk at data eksisterer for perioden

### Dashboard laster treg

1. **Reduser antall queries:**
   - Kombiner relaterte metrics
   - Øk query interval

2. **Optimaliser queries:**
   - Bruk recording rules
   - Reduser time range

3. **Sjekk Prometheus performance:**
   ```bash
   curl http://165.232.66.71:9090/metrics | grep prometheus_query_duration
   ```

## Neste Steg

1. Implementere SLA dashboard
2. Sette opp alerting på kritiske panels
3. Opprette per-server dashboards for detaljert analyse
4. Legge til custom Netbox-spesifikke metrics
5. Konfigurere automatic screenshots for rapporter

For alerting konfigurasjon, se [Vedlikehold](08-vedlikehold.md).
