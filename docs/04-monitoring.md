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

## Prometheus Konfiguras