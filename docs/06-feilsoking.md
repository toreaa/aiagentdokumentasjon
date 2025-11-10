# Feilsøkingsguide

## Problemer vi har løst

Denne guiden dokumenterer faktiske problemer opplevd under deployment og hvordan de ble løst.

## Server 1 - Høy Load (LØST)

### Problem
Server load på 46+ (normalt er 1-2), server ekstemt treg.

**Symptomer:**
```bash
$ uptime
08:01:40 up  9:41,  4 users,  load average: 46.16, 44.37, 40.26
```

### Diagnose

```bash
# Sjekk prosesser som bruker mest CPU
ps aux --sort=-%cpu | head -20

# Sjekk minne
free -h

# Sjekk swap usage
cat /proc/swaps
```

### Rot-årsak funnet

1. **Django runserver kjørte i produksjon** (2 instanser)
   - PID 5255 og 6613
   - `manage.py runserver` er kun for development!

2. **Promtail brukte 18% CPU**
   - 79 minutter CPU-tid
   - Feilkonfigurert eller for aggressive log collection

3. **Kritisk lavt minne**
   - Kun 295Mi tilgjengelig av 1.9Gi
   - `kswapd0` swapped konstant (22% CPU)

### Løsning

```bash
# 1. Drep runserver prosesser
kill 5253 5255 6613

# 2. Stopp Promtail midlertidig
sudo systemctl stop promtail

# 3. Verifiser at Gunicorn kjører korrekt
systemctl status netbox

# Resultat: Load falt fra 46.16 → 1.14
```

**Lærdommer:**
- ✅ ALDRI bruk `manage.py runserver` i produksjon
- ✅ Bruk systemd services (netbox.service)
- ✅ Overvåk ressursbruk kontinuerlig

## Prometheus Targets Down

### Problem
Flere monitoring targets vises som "down" i Prometheus, men tjenestene kjører.

**Symptomer:**
```json
{
  "job": "redis",
  "instance": "server1-netbox",
  "health": "down",
  "lastError": "Get \"http://146.190.148.117:9121/metrics\": context deadline exceeded"
}
```

### Diagnose

```bash
# På Server 3 (Prometheus), test connectivity
curl -s --connect-timeout 3 http://146.190.148.117:9100/metrics

# På Server 1, sjekk at exporter kjører
systemctl status redis_exporter

# Sjekk hva exporter lytter på
ss -tulpn | grep 9121
```

### Rot-årsak

Exporters konfigurert til å kun lytte på `localhost`, ikke tilgjengelig eksternt.

**Eksempel - feil konfigurasjon:**
```
ExecStart=/usr/local/bin/redis_exporter -redis.addr=localhost:6379
# Mangler: -web.listen-address=0.0.0.0:9121
```

### Løsning

Oppdater exporter service til å lytte på alle interfaces:

```bash
# Rediger service fil
sudo nano /etc/systemd/system/redis_exporter.service

# Endre ExecStart til:
ExecStart=/usr/local/bin/redis_exporter -redis.addr=localhost:6379 -web.listen-address=0.0.0.0:9121

# Reload og restart
sudo systemctl daemon-reload
sudo systemctl restart redis_exporter

# Verifiser
curl http://146.190.148.117:9121/metrics | head -20
```

**Samme løsning for andre exporters:**
- node_exporter: Ingen endring nødvendig (default 0.0.0.0)
- nginx_exporter: `--web.listen-address=:9113`
- postgres_exporter: Standard lytter på alle interfaces

## Grafana Dashboard Viser Ingen Data

### Problem
Dashboard lastet, men alle panels viser "No data".

### Diagnose

```bash
# 1. Sjekk at Prometheus har data
curl -s "http://localhost:9090/api/v1/query?query=up" | jq

# 2. Sjekk datasource i dashboard
# I Grafana UI -> Dashboard Settings -> JSON Model
# Se etter "datasource" feltet
```

### Rot-årsak

Dashboard bruker feil datasource navn (f.eks. `${DS_SIGNCL-PROMETHEUS}` i stedet for `Prometheus`).

### Løsning

```bash
# Finn og erstatt datasource i dashboard JSON
sudo sed -i 's/${DS_SIGNCL-PROMETHEUS}/Prometheus/g' /var/lib/grafana/dashboards/blackbox.json

# Restart Grafana
sudo systemctl restart grafana-server
```

## PostgreSQL Connection Refused

### Problem
Netbox kan ikke koble til PostgreSQL database.

**Symptom:**
```
django.db.utils.OperationalError: could not connect to server: Connection refused
```

### Diagnose

```bash
# Fra Server 1 (Netbox)
psql -h 104.248.172.65 -U netbox -d netbox

# Sjekk PostgreSQL lytter eksternt
sudo ss -tulpn | grep 5432

# Sjekk pg_hba.conf
sudo cat /etc/postgresql/17/main/pg_hba.conf | grep netbox
```

### Løsning

1. **Konfigurer PostgreSQL for ekstern tilgang:**

```bash
# /etc/postgresql/17/main/postgresql.conf
listen_addresses = '*'

# /etc/postgresql/17/main/pg_hba.conf
host    netbox          netbox          146.190.148.117/32      scram-sha-256

# Restart PostgreSQL
sudo systemctl restart postgresql
```

2. **Åpne firewall (hvis aktiv):**
```bash
sudo ufw allow from 146.190.148.117 to any port 5432
```

## Server 2 Utilgjengelig (PÅGÅENDE)

### Problem
Kan ikke SSH inn til Server 2 (104.248.172.65).

**Symptom:**
```
Connection timed out during banner exchange
Connection to 104.248.172.65 port 22 timed out
```

### Diagnose

Krever tilgang til DigitalOcean console:
1. Logg inn på DigitalOcean
2. Gå til Droplets
3. Velg Server 2
4. Klikk "Access" → "Launch Console"

### Mulige årsaker

1. **Server reboot**
2. **SSH daemon crashed**
3. **Firewall blokkering**
4. **Nettverksproblem**
5. **Server ut av minne (OOM killer)**

### Midlertidig workaround

Hvis server har console-tilgang:

```bash
# Start SSH daemon
sudo systemctl start sshd

# Sjekk status
systemctl status sshd

# Sjekk firewall
sudo ufw status
```

## Netbox UserConfig Error

### Problem
```
ValueError: Cannot assign Config object: User.config must be a UserConfig instance
```

### Løsning

```bash
cd /opt/netbox/netbox
source ../venv/bin/activate

python3 manage.py shell
>>> from users.models import User, UserConfig
>>> admin = User.objects.get(username='admin')
>>> user_config = UserConfig.objects.create(user=admin)
>>> exit()
```

## Generelle Feilsøkings-kommandoer

### Sjekk Tjenestestatus

```bash
# Alle viktige tjenester på Server 1
systemctl is-active netbox netbox-rq nginx redis-server \
  redis_exporter nginx_exporter node_exporter

# Se failed services
systemctl list-units --state=failed

# Se journald logs
journalctl -u netbox -n 100 --no-pager
journalctl -u nginx -n 100 --no-pager
```

### Sjekk Ressursbruk

```bash
# Load average
uptime

# Top prosesser
top -o %CPU

# Minne
free -h

# Disk
df -h

# Nettverksstatus
ss -tulpn
```

### Sjekk Prometheus Targets

```bash
# Fra Server 3
curl -s "http://localhost:9090/api/v1/targets" | \
  jq '.data.activeTargets[] | {job: .labels.job, health: .health, error: .lastError}'
```

### Test Ekstern Tilgang

```bash
# Test HTTP endpoints
curl -I http://146.190.148.117
curl http://146.190.148.117:9100/metrics | head -5

# Test database connection
psql -h 104.248.172.65 -U netbox -d netbox -c "SELECT version();"
```

## Vanlige Feil og Quick Fixes

| Problem | Quick Fix |
|---------|-----------|
| Netbox returnerer 502 | `sudo systemctl restart netbox gunicorn` |
| Grafana ikke tilgjengelig | `sudo systemctl restart grafana-server` |
| Prometheus targets down | Sjekk exporter lytter på 0.0.0.0 ikke localhost |
| Høy load | Drep runserver, sjekk Promtail |
| Database connection error | Sjekk pg_hba.conf og postgresql.conf |
| Redis connection error | Sjekk at Redis kjører: `systemctl status redis-server` |

## Debugging Tips

1. **Alltid sjekk logs først:**
   ```bash
   journalctl -xe
   ```

2. **Sjekk nettverkstilkobling:**
   ```bash
   ping -c 3 <ip>
   telnet <ip> <port>
   ```

3. **Verifiser konfigurasjon:**
   ```bash
   nginx -t  # Test nginx config
   sudo -u postgres psql -c "SHOW config_file;"  # PostgreSQL config
   ```

4. **Sjekk prosesser:**
   ```bash
   ps aux | grep <service>
   pgrep -a <service>
   ```

## Eskaler Problem

Hvis problemet ikke løses:

1. Samle informasjon:
   ```bash
   uptime
   free -h
   df -h
   systemctl status <failing-service>
   journalctl -u <failing-service> -n 200
   ```

2. Ta screenshots av feilmeldinger
3. Noter tidspunkt når problemet startet
4. Sjekk om endringer ble gjort rett før problemet
5. Kontakt systemadministrator med all info
