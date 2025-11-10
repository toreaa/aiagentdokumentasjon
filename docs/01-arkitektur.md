# Systemarkitektur

## Oversikt

Denne løsningen bruker en 3-tier arkitektur med separasjon av concerns for sikkerhet, skalerbarhet og vedlikehold.

```
┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│   Server 1          │     │   Server 2          │     │   Server 3          │
│   Netbox App        │────▶│   PostgreSQL        │     │   Monitoring        │
│   146.190.148.117   │     │   104.248.172.65    │     │   165.232.66.71     │
│                     │     │                     │     │                     │
│  - Nginx (80)       │     │  - PostgreSQL (5432)│     │  - Prometheus (9090)│
│  - Gunicorn (8001)  │     │  - PG Exporter      │     │  - Grafana (3000)   │
│  - Redis (6379)     │◀────┼─────────────────────┼─────│  - Node Exporter    │
│  - Node Exporter    │     │  - Node Exporter    │     │  - Blackbox Export  │
│  - Redis Exporter   │     │                     │     │                     │
│  - Nginx Exporter   │     │                     │     │                     │
└─────────────────────┘     └─────────────────────┘     └─────────────────────┘
         │                                                        │
         └────────────────────────────────────────────────────────┘
                      Metrics scraping (15s interval)
```

## Komponenter

### Server 1 - Netbox Applikasjon (146.190.148.117)

**Rolle**: Frontend applikasjonsserver

**Kjernefunksjonalitet:**
- Netbox 4.4.5 Django-applikasjon
- Nginx reverse proxy for HTTP requests
- Gunicorn WSGI server (7 workers)
- Redis for caching og task queue

**Monitoring komponenter:**
- Node Exporter (9100) - System metrics
- Nginx Exporter (9113) - Web server metrics
- Redis Exporter (9121) - Cache metrics

**Avhengigheter:**
- PostgreSQL på Server 2 (TCP 5432)
- Redis lokalt (TCP 6379)

### Server 2 - Database (104.248.172.65)

**Rolle**: Dedikert databaseserver

**Kjernefunksjonalitet:**
- PostgreSQL 17 database
- Netbox database (121 migrasjoner)

**Monitoring komponenter:**
- Node Exporter (9100) - System metrics
- PostgreSQL Exporter (9187) - Database metrics

**Sikkerhet:**
- Aksepterer kun connections fra Server 1 IP
- pg_hba.conf konfigurert for ekstern tilgang

### Server 3 - Monitoring (165.232.66.71)

**Rolle**: Sentralisert monitoring og visualisering

**Kjernefunksjonalitet:**
- Prometheus 3.7.3 - Metrics collection
- Grafana - Dashboards og visualisering
- Blackbox Exporter - Endpoint monitoring
- Loki 3.5.8 - Log aggregation (stoppet pga ressurser)

**Scrape konfigurasjon:**
- 9 targets monitored
- 15 sekunders scrape interval
- 15 dagers retention

## Dataflyt

### HTTP Requests

```
User → Nginx (80) → Gunicorn (8001) → Django App → PostgreSQL (Server 2:5432)
                                              ↓
                                          Redis (localhost:6379)
```

### Monitoring Dataflyt

```
Prometheus (Server 3) ─┬─▶ Node Exporter (Server 1:9100)
                       ├─▶ Nginx Exporter (Server 1:9113)
                       ├─▶ Redis Exporter (Server 1:9121)
                       ├─▶ Node Exporter (Server 2:9100)
                       ├─▶ PostgreSQL Exporter (Server 2:9187)
                       ├─▶ Blackbox HTTP checks
                       └─▶ Self (localhost:9090)

Grafana (Server 3:3000) ─▶ Prometheus ─▶ Visualisering
```

## Sikkerhet

### Nettverksisolasjon

- Server 1 og 2: Kun nødvendige porter åpne
- Server 3: Kan scrape metrics fra Server 1 og 2
- PostgreSQL: Kun tilgjengelig fra Server 1 IP

### Brukertilgang

- Non-root user: `opsuser` med sudo-tilgang
- Tjenester kjører som `opsuser` (ikke root)
- Separate service accounts hvor nødvendig

## Skalerbarhet

### Gunicorn Workers

Konfigurert med `2 * CPU_COUNT + 1` workers:
```python
workers = multiprocessing.cpu_count() * 2 + 1
threads = 3
timeout = 120
```

### Database Connection Pooling

Django CONN_MAX_AGE satt til 300 sekunder for persistent connections.

### Redis Databases

- Database 0: Task queue (RQ)
- Database 1: Caching

## Ressursbruk

### Server 1 (Netbox)
- RAM: 1.9GB total
- Load: Normal ~1-2 (problemer ved høy load 40+)
- Disk: Standard SSD

### Server 2 (PostgreSQL)
- RAM: Dedikert for database
- Status: **Midlertidig utilgjengelig**

### Server 3 (Monitoring)
- RAM: ~2GB
- Disk: 15 dagers metrics retention
- Load: Minimal (~0.1-0.2)

## High Availability Betraktninger

**Single Point of Failure:**
- ❌ Database (Server 2) - Ingen replication
- ❌ Applikasjon (Server 1) - Ikke load balanced
- ✅ Monitoring (Server 3) - Kan gjenopprettes uten datatap

**Forbedringer for produksjon:**
- PostgreSQL replication (streaming/logical)
- Load balancer foran flere Netbox servere
- Alertmanager for notification (planlagt)
- Backup strategi for alle komponenter
