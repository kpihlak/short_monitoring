# Prometheus Labor 2: Lisaülesanded

**Kestus:** Vabatahtlik (kodus)  
**Eesmärk:** Pushgateway, Service Discovery, API automatiseerimine, edasijõudnud teemad

> Need ülesanded on vabatahtlikud! Tee neid kodus, kui tahad süvendada teadmisi.

---

## Lisaülesanne 1: Pushgateway

### Eesmärk

Õpid saatma mõõdikuid lühiajalistest töödest (batch jobs, cron jobs).

**Probleem pull mudeliga:**

Batch job töötab 30 sekundit ja lõpeb. Prometheus scrape'ib iga 15 sekundi järel. Kui Prometheus küsib just siis, kui job ei tööta, ei saa mõõdikuid!

**Lahendus: Pushgateway**

Job saadab mõõdikud Pushgateway'sse. Prometheus scrape'ib Pushgateway'd regulaarselt.

---

### Paigaldamine

**Lisa Docker Compose'i:**

```bash
cd ~/prometheus-lab/docker-compose/prometheus
cat >> docker-compose.yml << 'EOF'

  pushgateway:
    image: prom/pushgateway:latest
    container_name: pushgateway
    ports:
      - "9091:9091"
    restart: unless-stopped
EOF
```

**Käivita:**
```bash
docker-compose up -d
```

**Kontrolli UI:**
```
http://192.168.1.150:9091
```

Peaksid nägema tühja lehte (veel mõõdikuid pole).

---

### Lisa Prometheus'e

**Uuenda `config/prometheus.yml`:**

```yaml
  - job_name: 'pushgateway'
    honor_labels: true
    static_configs:
      - targets: ['pushgateway:9091']
```

**Oluline:** `honor_labels: true` säilitab originaalsed label'id pushed mõõdikutest.

**Lae uuesti:**
```bash
curl -X POST http://localhost:9090/-/reload
```

---

### Python Batch Job

**Loo skript:**

```bash
cd ~/prometheus-lab/instrumentation
mkdir -p push-gw
cat > push-gw/batch_job.py << 'EOF'
from prometheus_client import CollectorRegistry, Gauge, push_to_gateway
import time
import random

registry = CollectorRegistry()

# Mõõdikud
job_duration = Gauge('batch_job_duration_seconds', 
                     'Batch job kestus sekundites', 
                     registry=registry)
job_last_success = Gauge('batch_job_last_success_timestamp',
                         'Viimase õnnestunud töö timestamp',
                         registry=registry)
records_processed = Gauge('batch_job_records_processed',
                          'Töödeldud kirjete arv',
                          registry=registry)

# Simuleerime batch job'i
print("Batch job algas...")
start_time = time.time()

# Töö simulatsioon (5 sekundit)
time.sleep(5)
processed = random.randint(1000, 2000)

# Salvesta mõõdikud
duration = time.time() - start_time
job_duration.set(duration)
job_last_success.set_to_current_time()
records_processed.set(processed)

# Saada Pushgateway'sse
push_to_gateway('localhost:9091', job='batch_job', registry=registry)

print(f"Valmis! Töödeldud {processed} kirjet {duration:.2f}s jooksul")
EOF
```

---

### Käivita Batch Job

```bash
cd ~/prometheus-lab/instrumentation
source venv/bin/activate
pip install prometheus-client  # Kui pole juba
python3 push-gw/batch_job.py
```

**Väljund:**
```
Batch job algas...
Valmis! Töödeldud 1234 kirjet 5.01s jooksul
```

---

### Vaata Prometheus'es

**Päringud:**

```promql
batch_job_last_success_timestamp
batch_job_duration_seconds
batch_job_records_processed
```

**Oluline:** Need mõõdikud jäävad Pushgateway'sse kuni:
- Uus push samale job'ile
- Manuaalne kustutamine

---

### Cron Job Setup

**Lisa cron job:**

```bash
crontab -e
```

Lisa rida (iga 5 minuti järel):
```
*/5 * * * * cd /home/ubuntu/prometheus-lab/instrumentation && /home/ubuntu/prometheus-lab/instrumentation/venv/bin/python3 push-gw/batch_job.py >> /tmp/batch_job.log 2>&1
```

**Kontrolli:**
```bash
tail -f /tmp/batch_job.log
```

---

### Ülesanne: Error Tracking

**Täienda skripti error tracking'uga:**

```python
job_errors = Gauge('batch_job_errors_total',
                   'Vigade arv viimases töös',
                   registry=registry)

try:
    # Töö
    if random.random() < 0.1:  # 10% tõenäosus viga
        raise Exception("Simuleeritud viga!")
    
    # Õnnestus
    job_errors.set(0)
    job_last_success.set_to_current_time()
    records_processed.set(processed)
except Exception as e:
    job_errors.set(1)
    print(f"Viga: {e}")
finally:
    # Saada alati
    push_to_gateway('localhost:9091', job='batch_job', registry=registry)
```

**Loo alert:**

```yaml
- alert: BatchJobFailed
  expr: batch_job_errors_total > 0
  for: 1m
  labels:
    severity: warning
  annotations:
    summary: "Batch job ebaõnnestus"
    description: "Viimane batch job sai vea"
```

---

### Mõõdikute Kustutamine

**Kustuta üks job:**
```bash
curl -X DELETE http://localhost:9091/metrics/job/batch_job
```

**Kustuta kõik:**
```bash
curl -X PUT http://localhost:9091/api/v1/admin/wipe
```

---

## Lisaülesanne 2: Service Discovery

### Eesmärk

Õpid dünaamiliselt target'eid avastama (file-based service discovery).

**Päris maailmas:** Kubernetes SD, Consul SD. Siin õpime file_sd põhimõtet.

---

### File-Based Service Discovery

**Loo kataloog:**

```bash
cd ~/prometheus-lab/docker-compose/prometheus
mkdir -p config/file-sd
```

---

### Target Fail

**Loo fail `config/file-sd/targets.json`:**

```bash
cat > config/file-sd/targets.json << 'EOF'
[
  {
    "targets": ["nodeexporter:9100"],
    "labels": {
      "job": "node",
      "env": "production",
      "datacenter": "dc1"
    }
  },
  {
    "targets": ["cadvisor:8080"],
    "labels": {
      "job": "cadvisor",
      "env": "production",
      "datacenter": "dc1"
    }
  }
]
EOF
```

**Selgitus:**

Iga objekt on target grupp.

`labels` - Lisab ühised label'id kõigile target'idele grupis.

---

### Uuenda Prometheus Konfiguratsiooni

**Lisa `config/prometheus.yml` faili:**

```yaml
  - job_name: 'file-sd'
    file_sd_configs:
      - files:
        - 'file-sd/*.json'
        refresh_interval: 30s
```

**Selgitus:**

`files` - Glob pattern. Võib olla mitu faili.

`refresh_interval: 30s` - Kontrollib faili iga 30 sekundi järel.

**Lae uuesti:**
```bash
curl -X POST http://localhost:9090/-/reload
```

---

### Kontrolli

**Status → Targets:**

Peaksid nägema `file-sd` job'i target'eid koos custom label'itega!

**Tee päring:**
```promql
up{env="production"}
```

---

### Dünaamiline Uuendamine

**Muuda `targets.json` faili:**

```bash
cat > config/file-sd/targets.json << 'EOF'
[
  {
    "targets": ["nodeexporter:9100"],
    "labels": {
      "job": "node",
      "env": "staging",
      "datacenter": "dc2"
    }
  }
]
EOF
```

**Oota 30 sekundit** (või vähem) ja vaata Prometheus Targets lehte.

Label'id peaksid uuenema!

---

### Ülesanne: Multi-Environment Setup

**Loo eraldi failid:**

**Production:**
```bash
cat > config/file-sd/prod.json << 'EOF'
[
  {
    "targets": ["192.168.1.150:9100"],
    "labels": {
      "env": "production",
      "tier": "infrastructure"
    }
  }
]
EOF
```

**Staging:**
```bash
cat > config/file-sd/staging.json << 'EOF'
[
  {
    "targets": ["192.168.1.150:9101"],
    "labels": {
      "env": "staging",
      "tier": "infrastructure"
    }
  }
]
EOF
```

**Päringud per environment:**
```promql
up{env="production"}
up{env="staging"}
```

---

## Lisaülesanne 3: Prometheus HTTP API

### Eesmärk

Õpid kasutama Prometheus API'd skriptides ja automatiseerimises.

---

### Instant Query

**Lihtne päring:**
```bash
curl -G http://localhost:9090/api/v1/query \
  --data-urlencode 'query=up'
```

**Vastus (JSON):**
```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {"instance": "localhost:9090", "job": "prometheus"},
        "value": [1731312345, "1"]
      }
    ]
  }
}
```

---

### Range Query

**Ajavahemiku päring:**
```bash
curl -G http://localhost:9090/api/v1/query_range \
  --data-urlencode 'query=rate(node_cpu_seconds_total[5m])' \
  --data-urlencode 'start=2024-11-11T10:00:00Z' \
  --data-urlencode 'end=2024-11-11T11:00:00Z' \
  --data-urlencode 'step=15s'
```

**Selgitus:**

`start` ja `end` - Ajavahemik ISO 8601 formaadis

`step` - Samm (resolution)

---

### Skript: Health Check

**Loo skript `instrumentation/check_targets.sh`:**

```bash
cat > ~/prometheus-lab/instrumentation/check_targets.sh << 'EOF'
#!/bin/bash

API="http://localhost:9090/api/v1"

echo "=== Prometheus Targets Health ==="
echo ""

# Küsi kõiki target'eid
RESULT=$(curl -s "$API/query?query=up")

# Paigalda jq, kui puudub
if ! command -v jq &> /dev/null; then
    echo "Paigaldan jq..."
    sudo apt install -y jq
fi

# Parsi JSON'i
echo "$RESULT" | jq -r '.data.result[] | "\(.metric.job)/\(.metric.instance): \(if .value[1] == "1" then "UP" else "DOWN" end)"'

echo ""
echo "=== Kokkuvõte ==="
UP_COUNT=$(echo "$RESULT" | jq -r '.data.result[] | select(.value[1] == "1")' | jq -s 'length')
TOTAL=$(echo "$RESULT" | jq -r '.data.result | length')
echo "UP: $UP_COUNT / $TOTAL"
EOF

chmod +x ~/prometheus-lab/instrumentation/check_targets.sh
```

**Käivita:**
```bash
~/prometheus-lab/instrumentation/check_targets.sh
```

**Väljund:**
```
=== Prometheus Targets Health ===

prometheus/localhost:9090: UP
node/localhost:9100: UP
cadvisor/cadvisor:8080: UP

=== Kokkuvõte ===
UP: 3 / 3
```

---

### Monitoring Report

**Loo raport skript:**

```bash
cat > ~/prometheus-lab/instrumentation/generate_report.sh << 'EOF'
#!/bin/bash

API="http://localhost:9090/api/v1"

echo "========================================="
echo "   PROMETHEUS MONITORING REPORT"
echo "========================================="
echo "Generated: $(date)"
echo ""

# Targets UP
UP=$(curl -s "$API/query?query=count(up==1)" | jq -r '.data.result[0].value[1]')
echo "Targets UP: $UP"

# CPU Usage
CPU=$(curl -s "$API/query?query=100-(avg(rate(node_cpu_seconds_total{mode=\"idle\"}[5m]))*100)" | \
      jq -r '.data.result[0].value[1]')
CPU_ROUNDED=$(printf "%.1f" $CPU)
echo "CPU Usage: ${CPU_ROUNDED}%"

# Memory Usage
MEM=$(curl -s "$API/query?query=(1-(node_memory_MemAvailable_bytes/node_memory_MemTotal_bytes))*100" | \
      jq -r '.data.result[0].value[1]')
MEM_ROUNDED=$(printf "%.1f" $MEM)
echo "Memory Usage: ${MEM_ROUNDED}%"

# Active Alerts
ALERTS=$(curl -s "$API/alerts" | jq -r '.data.alerts | length')
echo "Active Alerts: $ALERTS"

echo ""
echo "========================================="
EOF

chmod +x ~/prometheus-lab/instrumentation/generate_report.sh
```

**Käivita:**
```bash
~/prometheus-lab/instrumentation/generate_report.sh
```

---

### Slack Report Bot

**Saada raport Slack'i:**

```bash
cat > ~/prometheus-lab/instrumentation/slack_report.sh << 'EOF'
#!/bin/bash

WEBHOOK="YOUR_SLACK_WEBHOOK_URL"  # Asenda!
REPORT=$(~/prometheus-lab/instrumentation/generate_report.sh)

curl -X POST $WEBHOOK \
  -H 'Content-type: application/json' \
  --data "{\"text\":\"$REPORT\"}"
EOF

chmod +x ~/prometheus-lab/instrumentation/slack_report.sh
```

**Cron job (iga päev kell 9:00):**
```bash
crontab -e
```

Lisa:
```
0 9 * * * /home/ubuntu/prometheus-lab/instrumentation/slack_report.sh
```

---

## Lisaülesanne 4: Advanced PromQL

### Eesmärk

Õpid keerukamaid PromQL tehnikaid.

---

### Subqueries

**Viimase tunni max CPU kasutus:**
```promql
max_over_time(
  (100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100))
  [1h:5m]
)
```

**Selgitus:**

`[1h:5m]` - Viimane tund, 5-minutilise sammuga (subquery)

---

### Binary Operators

**Liitmine:**
```promql
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
```

**Jagamine:**
```promql
rate(node_network_receive_bytes_total[5m]) / 1024 / 1024
```

**Võrdlus:**
```promql
up == 1
```

---

### Vector Matching

**Many-to-one:**
```promql
rate(node_cpu_seconds_total[5m])
  * on(instance) group_left(hostname)
  node_uname_info
```

**Selgitus:**

`group_left` võtab label'id vasakult vektorilt (`hostname` label tuleb `node_uname_info`-st).

---

### Keerukad Ülesanded

**1. Prognoos: Millal ketas täis?**
```promql
predict_linear(node_filesystem_free_bytes{mountpoint="/"}[1h], 24*3600)
```

**Selgitus:**

Ennustab 24h (24*3600 sekundit) tulevikku, kasutades viimase tunni trendi.

**2. Anomaalia: Tavapärasest kõrvalekalle:**
```promql
abs(
  rate(http_requests_total[5m])
  - avg_over_time(rate(http_requests_total[5m])[1h:5m])
) > 10
```

**Selgitus:**

Võrdleb praegust rate'i tunni keskmisega. Kui erinevus > 10, on anomaalia.

**3. Percentile üle instance'te:**
```promql
histogram_quantile(0.95,
  sum by (le) (rate(request_duration_seconds_bucket[5m]))
)
```

---

### Recording Rules

**Loo recording rule:**

```bash
cat > config/rules/recording-rules.yml << 'EOF'
groups:
  - name: recording.rules
    interval: 30s
    rules:
      - record: job:cpu_usage:rate5m
        expr: 100 - (avg by (job) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

      - record: job:memory_usage:percentage
        expr: (1 - (avg by (job) (node_memory_MemAvailable_bytes) / avg by (job) (node_memory_MemTotal_bytes))) * 100

      - record: instance:http_requests:rate5m
        expr: sum by (instance) (rate(http_requests_total[5m]))
EOF
```

**Selgitus:**

Recording rules arvutavad ette tihti kasutatavad päringud. Tulemused salvestatakse uute mõõdikutena.

**Lae uuesti:**
```bash
curl -X POST http://localhost:9090/-/reload
```

**Kasuta:**
```promql
job:cpu_usage:rate5m
job:memory_usage:percentage
```

Palju kiirem kui originaalpäring!

---

## Challenge: Production-Ready Setup

### Eesmärk

Loo täielik, production-ready monitoring stack.

---

### Nõuded

**1. Infrastructure (10 punkti)**
- 3x Node Exporter (simuleerib 3 serverit)
- Docker Engine metrics
- cAdvisor
- 2x Python rakendus (erinevad portid)

**2. Service Discovery (5 punkti)**
- File-based SD erinevate keskkondade jaoks
- Automaatne target'ite uuendamine (skript)

**3. Pushgateway (5 punkti)**
- 3 batch job'i (backup, log analysis, cleanup)
- Cron schedule
- Error tracking ja alerting

**4. Alerts (15 punkti)**
- Infrastructure alerts (CPU, memory, disk, network)
- Application alerts (slow requests, high error rate)
- Batch job alerts (failure, long duration)
- 3 severity level'it (critical, warning, info)

**5. Grafana (10 punkti)**
- Infrastructure dashboard (system overview)
- Application dashboard (app metrics)
- Batch jobs dashboard (job status, duration)
- Alert overview dashboard

**6. Automation (5 punkti)**
- Health check skript (API)
- Daily report Slack'i
- Auto-remediation (restart service kui DOWN)

**7. Documentation (10 punkti)**
- README.md (setup guide)
- Architecture diagram
- Alert runbooks (mida teha iga alert'i korral)
- Troubleshooting guide

---

### Hindamine

| Komponent | Punktid | Kriteeriumid |
|-----------|---------|--------------|
| Infrastructure | 10 | Kõik komponendid töötavad |
| Service Discovery | 5 | Dünaamiline, automaatne |
| Pushgateway | 5 | Batch job'id töötavad |
| Alerts | 15 | Mõistlikud reeglid, testitud |
| Grafana | 10 | Professionaalsed dashboardid |
| Automation | 5 | Skriptid töötavad |
| Documentation | 10 | Selge, täielik |
| **Kokku** | **60** | |

---

## Lisalugemist

**Prometheus:**
- [Federation](https://prometheus.io/docs/prometheus/latest/federation/)
- [Remote Storage](https://prometheus.io/docs/prometheus/latest/storage/)
- [Security Best Practices](https://prometheus.io/docs/operating/security/)

**PromQL:**
- [Functions Reference](https://prometheus.io/docs/prometheus/latest/querying/functions/)
- [Operators](https://prometheus.io/docs/prometheus/latest/querying/operators/)
- [HTTP API](https://prometheus.io/docs/prometheus/latest/querying/api/)

**Best Practices:**
- [Naming Conventions](https://prometheus.io/docs/practices/naming/)
- [Instrumentation](https://prometheus.io/docs/practices/instrumentation/)
- [Alerting](https://prometheus.io/docs/practices/alerting/)
- [Recording Rules](https://prometheus.io/docs/practices/rules/)

**Integrations:**
- [Exporters List](https://prometheus.io/docs/instrumenting/exporters/)
- [Client Libraries](https://prometheus.io/docs/instrumenting/clientlibs/)
- [Grafana Dashboards](https://grafana.com/grafana/dashboards/)