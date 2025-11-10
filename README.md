# Netbox Infrastructure Demo - Dokumentasjon

Komplett dokumentasjon for 3-server Netbox deployment med full monitoring stack.

## Oversikt

Dette er en produksjonsklar Netbox-deployment med separert arkitektur for applikasjon, database og monitoring.

**Oppsettet består av:**
- **Server 1**: Netbox applikasjon (146.190.148.117)
- **Server 2**: PostgreSQL database (104.248.172.65)
- **Server 3**: Prometheus + Grafana monitoring (165.232.66.71)

## Dokumentasjon

- [Arkitektur](docs/01-arkitektur.md) - Systemarkitektur og design
- [Installasjon](docs/02-installasjon.md) - Komplett installasjonsveiledning
- [Konfigurasjoner](docs/03-konfigurasjoner.md) - Alle konfigurasjonsfiler
- [Monitoring](docs/04-monitoring.md) - Prometheus, Grafana og exporters
- [Dashboards](docs/05-dashboards.md) - Grafana dashboards og visualisering
- [Feilsøking](docs/06-feilsoking.md) - Vanlige problemer og løsninger
- [Tilgang](docs/07-tilgang.md) - Passord, URLer og tilgangsinformasjon
- [Vedlikehold](docs/08-vedlikehold.md) - Daglig drift og vedlikehold

## Rask Start

### Tilgang til systemer

**Netbox:**
```
URL: http://146.190.148.117
Bruker: admin
Passord: [REDACTED]
```

**Grafana:**
```
URL: http://165.232.66.71:3000
Bruker: admin
Passord: [REDACTED]
```

**SSH til servere:**
```bash
# Server 1 - Netbox
ssh opsuser@146.190.148.117
# Passord: [REDACTED]

# Server 2 - PostgreSQL
ssh opsuser@104.248.172.65
# Passord: [REDACTED]

# Server 3 - Monitoring
ssh opsuser@165.232.66.71
# Passord: [REDACTED]
```

## Systemstatus

For å sjekke systemstatus, gå til Grafana og åpne:
1. **SLA Monitoring Dashboard** - Overordnet status
2. **Node Exporter Full** - Server health
3. **PostgreSQL Database** - Database metrics
4. **Redis Dashboard** - Cache metrics
5. **NGINX Exporter** - Web server metrics
6. **Prometheus Blackbox Exporter** - Endpoint monitoring

## Teknologier

- **OS**: Ubuntu 25.10 (Questing Quokka)
- **Applikasjon**: Netbox 4.4.5
- **Database**: PostgreSQL 17
- **Cache**: Redis 8.0.2
- **Web Server**: Nginx 1.28.0
- **WSGI Server**: Gunicorn
- **Monitoring**: Prometheus 3.7.3 + Grafana
- **Log Aggregering**: Loki 3.5.8 + Promtail

## Support

For problemer, se [Feilsøkingguiden](docs/06-feilsoking.md) eller kontakt systemadministrator.

## Lisens

Intern dokumentasjon for demo-formål.
