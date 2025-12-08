# Prometheus Labor 2: Infrastructure, Alerting ja Visualiseerimine

**Kestus:** 3 × 45 minutit  
**Eeldused:** Labor 1 tehtud  
**Eesmärk:** Jälgid Docker konteinereid, seadistad hoiatusi Slack'iga, lood Grafana dashboarde

> **Loengust:** Vaata üle peatükid 7-8 (Alerting, Hoiatuste süsteemid)

---

## Tunni 1: Infrastructure Monitoring (45 min)

### Mida õpid selles tunnis?

- Jälgid Docker Engine'i sisemisi mõõdikuid
- Lisad cAdvisor'i konteinerite detailsete mõõdikute jaoks
- Teed infrastructure päringuid

---

### Docker Engine Metrics

Docker Engine saab pakkuda oma sisemisi mõõdikuid (konteinerite arv, image'id, sündmused).

**Seadista Docker Daemon:**

```bash
cat > daemon.json << 'EOF'
{
  "metrics-addr": "0.0.0.0:9323",
  "experimental": true
}
EOF
```

**Liiguta õigesse kohta:**
```bash
sudo mv daemon.json /etc/docker/daemon.json
sudo systemctl restart docker
```

**Oluline:** Docker restart peatab KÕIK konteinerid!

**Taaskäivita oma teenused:**
```bash
cd ~/prometheus-lab/docker-compose/prometheus
docker-compose up -d
```

**Testi Docker metrics:**
```bash
curl http://localhost:9323/metrics | head -20
```

Peaksid nägema Docker'i sisemisi mõõdikuid.

---

### Lisa Docker Engine Prometheus'e

**Uuenda `config/prometheus.yml` (asenda oma VM IP-ga!):**

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'python-app'
    static_configs:
      - targets: ['192.168.1.150:5001']

  - job_name: 'docker-engine'
    static_configs:
      - targets: ['192.168.1.150:9323']
```

**Lae uuesti:**
```bash
curl -X POST http://localhost:9090/-/reload
```

**Kontrolli:** Status → Targets → `docker-engine` peaks olema UP

---

### Ülesanne 1: Docker Engine Mõõdikud

**Tee päringuid:**

**1. Konteinerite arv olekute kaupa:**
```promql
engine_daemon_container_states_containers
```

Näed:
```
engine_daemon_container_states_containers{state="running"} 3
engine_daemon_container_states_containers{state="paused"} 0
engine_daemon_container_states_containers{state="stopped"} 1
```

**2. Ainult töötavaid konteinereid:**
```promql
engine_daemon_container_states_containers{state="running"}
```

**3. Docker engine info:**
```promql
engine_daemon_engine_info
```

**Mida õppisid?**
- Docker Engine sisemiste mõõdikute lugemine
- Konteineri olekute jälgimine

---

### cAdvisor Paigaldamine

> **Loengust:** cAdvisor (Container Advisor) annab detailseid mõõdikuid iga konteineri kohta: CPU, RAM, võrk, disk.

**Lisa Docker Compose'i:**

```bash
cd ~/prometheus-lab/docker-compose/prometheus
cat >> docker-compose.yml << 'EOF'

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /var/run/docker.sock:/var/run/docker.sock:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    ports:
      - "8080:8080"
    restart: unless-stopped
EOF
```

**Selgitus:**

cAdvisor vajab ligipääsu Docker socket'ile (`/var/run/docker.sock`)

Mount'ib süsteemi katalooge read-only (`ro`)

**Käivita:**
```bash
docker-compose up -d
```

**Kontrolli:**
```bash
docker ps | grep cadvisor
docker logs cadvisor
```

---

### cAdvisor UI

**Ava brauseris:**
```
http://192.168.1.150:8080
```

**Mida näed:**
- Kõik töötavad konteinerid
- CPU/RAM graafikud reaalajas
- Disk IO statistika
- Network traffic

**Uurimine:** Kliki ükskõik millise konteineri peale ja vaata detaile.

---

### Lisa cAdvisor Prometheus'e

**Uuenda `config/prometheus.yml`:**

```yaml
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

**Selgitus:**

Kasutame `cadvisor:8080` mitte `localhost:8080`, sest Prometheus ja cAdvisor on samas Docker võrgus.

**Lae uuesti:**
```bash
curl -X POST http://localhost:9090/-/reload
```

**Kontrolli:** Status → Targets → `cadvisor` peaks olema UP

---

### Ülesanne 2: cAdvisor Mõõdikud

**Tee päringuid:**

**1. Konteinerite CPU kasutus:**
```promql
container_cpu_usage_seconds_total
```

**2. Ainult Docker konteinerid (mitte süsteem):**
```promql
container_cpu_usage_seconds_total{id=~"/docker/.*"}
```

`id=~"/docker/.*"` on regex - ID algab "/docker/" prefiksiga

**3. Konteineri mälu kasutus (MB):**
```promql
container_memory_usage_bytes{name=~"prometheus|nodeexporter"} / 1024 / 1024
```

**4. Konteineri võrgu traffic:**
```promql
rate(container_network_receive_bytes_total[5m])
```

**5. Konteinerid, mis kasutavad kõige rohkem mälu:**
```promql
topk(3, container_memory_usage_bytes{id=~"/docker/.*"})
```

---

### Ülesanne 3: Infrastructure Ülevaade

**Võrdle süsteemi ja konteinerite ressursse:**

**Süsteemi CPU kokku:**
```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Docker konteinerite CPU kokku:**
```promql
sum(rate(container_cpu_usage_seconds_total{id=~"/docker/.*"}[5m]))
```

**Küsimus:** Kui suur osa süsteemi CPU-st kulub Docker konteineritele?

**Mälu võrdlus:**
```promql
# Süsteemi kokku mälu
node_memory_MemTotal_bytes

# Docker konteinerite mälu kokku
sum(container_memory_usage_bytes{id=~"/docker/.*"})
```

---

### Portide Ülevaade

| Teenus | Port | URL |
|--------|------|-----|
| Prometheus | 9090 | `http://192.168.1.150:9090` |
| Node Exporter | 9100 | `http://192.168.1.150:9100/metrics` |
| Docker Engine | 9323 | `http://192.168.1.150:9323/metrics` |
| cAdvisor | 8080 | `http://192.168.1.150:8080` |
| Python app | 5000-5001 | `http://192.168.1.150:5000` |

---

### Kokkuvõte: Tunni 1

Õppisid:
- Docker Engine metrics'i kogumist
- cAdvisor paigaldamist ja kasutamist
- Konteinerite detailset jälgimist
- Süsteemi vs konteinerite ressursside võrdlust

**Järgmine tund:** Hoiatuste seadistamine AlertManager'iga ja Slack integratsioon.

---

## Tunni 2: Alerting Setup (45 min)

### Mida õpid selles tunnis?

- Lood hoiatuste reeglid
- Paigaldad AlertManager'i
- Seadistadjalad Slack teavitused
- Testid hoiatusi praktikas

> **Loengust:** Hoiatused reageerivad automaatselt probleemidele. Prometheus hindab reegleid, AlertManager saadab teavitusi.

---

### Alert Rules Loomine

**Loo rules kataloog:**
```bash
cd ~/prometheus-lab/docker-compose/prometheus
mkdir -p config/rules
```

**Loo fail `config/rules/basic-alerts.yml`:**

```bash
cat > config/rules/basic-alerts.yml << 'EOF'
groups:
  - name: basic.rules
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} on maas"
          description: "{{ $labels.job }}/{{ $labels.instance }} ei vasta üle 1 minuti."

      - alert: HighCPUUsage
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Kõrge CPU kasutus"
          description: "CPU kasutus on {{ $value | humanize }}% üle 2 minuti."

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Kõrge mälu kasutus"
          description: "Mälu kasutus on {{ $value | humanize }}%"
EOF
```

**Selgitus:**

`expr` - PromQL päring. Kui tõene, hoiatus käivitub.

`for: 1m` - Tingimus peab kehtima 1 minuti, enne kui hoiatus "fires". Vältib false positive'e.

`labels` - Lisainfo (severity level, meeskond, jne)

`annotations` - Inimloetav tekst hoiatuses

`{{ $value }}` - Asendatakse päring tulemusega

---

### Uuenda Prometheus Konfiguratsiooni

**Lisa `config/prometheus.yml` faili algusesse:**

```yaml
rule_files:
  - "rules/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

**Täielik näide:**

```yaml
global:
  scrape_interval: 15s

rule_files:
  - "rules/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'python-app'
    static_configs:
      - targets: ['192.168.1.150:5001']

  - job_name: 'docker-engine'
    static_configs:
      - targets: ['192.168.1.150:9323']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

**Lae uuesti:**
```bash
curl -X POST http://localhost:9090/-/reload
```

---

### Ülesanne 4: Kontrolli Alert Rules

**Ava Prometheus UI:**
```
http://192.168.1.150:9090/alerts
```

Peaksid nägema:

| Alert | State |
|-------|-------|
| InstanceDown | green (OK) |
| HighCPUUsage | green (OK) |
| HighMemoryUsage | green (OK) |

**Olekud:**
- **Green (Inactive)** - Ei käivitu
- **Orange (Pending)** - Tingimus on tõene, aga `for:` aeg ei ole veel möödas
- **Red (Firing)** - Hoiatus käivitunud!

---

### Slack Setup

**1. Loo Slack channel:**

Ava oma Slack workspace ja loo uus channel:
```
#prometheus-alerts
```

**2. Loo Incoming Webhook:**

1. Mine: [api.slack.com/apps](https://api.slack.com/apps)
2. **Create New App** → **From scratch**
3. Vali oma workspace
4. **Incoming Webhooks** → **Activate Incoming Webhooks** (ON)
5. **Add New Webhook to Workspace**
6. Vali `#prometheus-alerts` channel
7. **Kopeeri Webhook URL** (näiteks: `https://hooks.slack.com/services/T00/B00/XXX`)

**Oluline:** Hoia see URL saladuses! Ära jaga avalikult.

---

### AlertManager Konfiguratsioon

**Loo kataloog:**
```bash
cd ~/prometheus-lab
mkdir -p alertmanager
```

**Loo fail `alertmanager/alertmanager.yml`:**

```bash
cat > alertmanager/alertmanager.yml << 'EOF'
global:
  resolve_timeout: 5m

route:
  receiver: 'slack-alerts'
  group_by: ['alertname', 'severity']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h

receivers:
  - name: 'slack-alerts'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/T00/B00/XXX'  # ASENDA oma URL-ga!
        channel: '#prometheus-alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: "{{ range .Alerts }}{{ .Annotations.summary }}\n{{ .Annotations.description }}\n{{ end }}"
        send_resolved: true
EOF
```

**Selgitus:**

`group_by` - Grupeerib sarnased hoiatused (nt 10 serverit DOWN → 1 teade)

`group_wait: 10s` - Ootab 10s enne saatmist (võimaldab grupeerimist)

`repeat_interval: 12h` - Saadab uuesti hoiatuse iga 12 tunni järel, kui probleem püsib

`send_resolved: true` - Saadab ka "probleem lahendatud" teavituse

---

### Lisa AlertManager Docker Compose'i

**Lisa `docker-compose.yml` faili:**

```bash
cat >> docker-compose.yml << 'EOF'

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    volumes:
      - ../../alertmanager:/etc/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    ports:
      - "9093:9093"
    restart: unless-stopped
EOF
```

**Käivita:**
```bash
docker-compose up -d
```

**Kontrolli:**
```bash
docker ps | grep alertmanager
docker logs alertmanager
```

Ei tohiks näha ERROR'eid.

---

### Ülesanne 5: Testi Hoiatusi

**Variant 1: Peata Node Exporter (simuleerib InstanceDown)**

```bash
docker stop nodeexporter
```

**Mida juhtub:**

1. **30-60 sekundit:** Prometheus scrape ebaõnnestub
2. **1 minut:** Alert läheb PENDING (orange)
3. **Veel 1 minut:** Alert läheb FIRING (red)
4. **10 sekundit:** AlertManager saadab Slack'i

**Kontrolli:**
- Prometheus UI: `/alerts` - Peaksid nägema InstanceDown FIRING
- Slack: `#prometheus-alerts` - Peaksid saama teavituse

**Taasta:**
```bash
docker start nodeexporter
```

Mõne minuti pärast saad "resolved" teavituse Slack'is!

---

**Variant 2: CPU Koormuse Genereerimine**

**Paigalda stress:**
```bash
sudo apt install stress
```

**Genereeri koormus:**
```bash
stress --cpu 4 --timeout 180s
```

**Jälgi:**
```
http://192.168.1.150:9090/alerts
```

`HighCPUUsage` peaks minema:
1. Inactive → Pending (orange)
2. Pending → Firing (red) pärast 2 minutit
3. Slack'i teavitus

---

### Ülesanne 6: Loo Oma Alert

**Lisa `config/rules/basic-alerts.yml` faili uus reegel:**

```yaml
      - alert: LowDiskSpace
        expr: (node_filesystem_free_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Vaba kettaruum väike"
          description: "Ainult {{ $value | humanize }}% ruumi jäänud"
```

**Lae uuesti:**
```bash
curl -X POST http://localhost:9090/-/reload
```

**Kontrolli:** `/alerts` lehel peaks olema uus `LowDiskSpace` alert (roheline, kui piisavalt ruumi on).

**Testimiseks:** Täida ketas (ETTEVAATLIKULT!):
```bash
dd if=/dev/zero of=/tmp/bigfile bs=1M count=10000
```

---

### Kokkuvõte: Tunni 2

Õppisid:
- Alert rules'ide loomist
- AlertManager paigaldamist
- Slack integratsiooni
- Hoiatuste testimist

**Järgmine tund:** Grafana dashboardide loomine.

---

## Tunni 3: Grafana Visualiseerimine (45 min)

### Mida õpid selles tunnis?

- Paigaldad Grafana
- Lisad Prometheus data source'i
- Lood dashboardi põhipaneelidega
- Õpid erinevaid visualiseerimise tüüpe

---

### Grafana Paigaldamine

**Lisa Docker Compose'i:**

```bash
cd ~/prometheus-lab/docker-compose/prometheus
cat >> docker-compose.yml << 'EOF'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
    restart: unless-stopped
EOF
```

**Uuenda volumes sektsiooni faili alguses:**

```yaml
volumes:
  prometheus-data: {}
  grafana-data: {}
```

**Käivita:**
```bash
docker-compose up -d
```

**Kontrolli:**
```bash
docker ps | grep grafana
docker logs grafana
```

---

### Grafana UI

**Ava brauseris:**
```
http://192.168.1.150:3000
```

**Login:**
- Username: `admin`
- Password: `admin`

Esimesel sisselogimisel palub uut parooli seada. Vali uus parool või skip.

---

### Ülesanne 7: Lisa Prometheus Data Source

**Sammud:**

1. Grafana UI's kliki külgmenüüs ikooni (paremal pool) → **Configuration** (hammasratas) → **Data Sources**

2. **Add data source**

3. Vali **Prometheus**

4. Seaded:
   ```
   Name: Prometheus
   URL: http://prometheus:9090
   ```

   **Oluline:** Kasuta `prometheus:9090` mitte `localhost:9090`! Nad on samas Docker võrgus.

5. **Save & Test**

Peaksid nägema: **Data source is working**

---

### Ülesanne 8: Loo Dashboard

**Sammud:**

1. Kliki **+** (Create) → **Dashboard**

2. **Add new panel**

3. Panel editor avaneb

---

### Esimene Panel: CPU Kasutus

**Query tab (all):**

```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Panel settings (paremal):**
- **Title:** CPU Usage
- **Unit:** Percent (0-100)
- **Legend:** `{{instance}}`

**Visualisation:** Time series (default)

**Apply** (ülemine parempoolne nurk)

---

### Teine Panel: Mälu Kasutus

**Lisa uus panel:**
1. Dashboard'il kliki **Add panel** ikoonil (ülemine riba)

**Query:**
```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

**Panel settings:**
- **Title:** Memory Usage
- **Unit:** Percent (0-100)

**Apply**

---

### Kolmas Panel: Aktiivsed Konteinerid

**Lisa uus panel**

**Query:**
```promql
engine_daemon_container_states_containers{state="running"}
```

**Visualization:** Stat (muuda vasakult)

**Panel settings:**
- **Title:** Running Containers

**Apply**

---

### Salvesta Dashboard

1. Kliki ülemises ribal **Save dashboard** (ikoon)
2. Nimi: `System Monitoring`
3. **Save**

**Mida õppisid?**
- Dashboard'i loomist
- Paneeliide lisamist
- Erinevaid visualiseerimise tüüpe

---

### Visualiseerimise Tüübid

| Tüüp | Millal Kasutada | Näide |
|------|-----------------|-------|
| Time series | Ajalised graafikud | `rate(http_requests_total[5m])` |
| Stat | Üks number | `count(up == 1)` |
| Gauge | Protsent või suhe | `cpu_usage_percent` |
| Bar gauge | Võrdlus | `topk(5, memory_usage)` |
| Table | Detailne info | `node_uname_info` |

---

### Ülesanne 9: Impordi Valmis Dashboard

Grafana kogukond on loonud tuhandeid valmis dashboarde.

**Sammud:**

1. **+** (Create) → **Import**

2. **Import via grafana.com:**
   ```
   ID: 1860
   ```
   (Node Exporter Full dashboard)

3. **Load**

4. Vali **Prometheus** data source

5. **Import**

**Mida näed:**
- Professionaalset Node Exporter dashboardi
- CPU, mälu, disk, võrgu graafikud
- Korralikult organiseeritud

**Uurimine:** Vaata, kuidas paneelid on tehtud. Kliki paneel → **Edit** → vaata query'd.

---

### Ülesanne 10: Dashboard Täiustamine

**Lisa oma "System Monitoring" dashboard'ile:**

**1. Row'de kasutamine:**

Dashboard'il kliki **Add panel** kõrval **Add** → **Row**

Nimetage: "Application Metrics"

**2. Lisa paneel Application Metrics row'sse:**

**Query:**
```promql
rate(http_requests_total[5m])
```

**Title:** HTTP Requests Rate

**3. Lisa veel paneel:**

**Query:**
```promql
active_users
```

**Title:** Active Users

---

### Kokkuvõte: Tunni 3

Õppisid:
- Grafana paigaldamist
- Data source'i lisamist
- Dashboard'i ja paneeliide loomist
- Erinevaid visualiseerimise tüüpe
- Valmis dashboard'ide importimist

---

## Labor 2 Kokkuvõte

**Tunni 1:**
- Docker Engine metrics
- cAdvisor paigaldamine
- Infrastructure päringud

**Tunni 2:**
- Alert rules loomine
- AlertManager + Slack
- Hoiatuste testimine

**Tunni 3:**
- Grafana paigaldamine
- Dashboard loomine
- Erinevad visualiseerimised

---

## Kodutöö

1. Täienda oma dashboardi vähemalt 5 paneeliga
2. Loo uus alert rule (nt LowDiskSpace, HighNetworkTraffic)
3. Vaata **Lisaülesanded** faili, kui tahad õppida Pushgateway't ja Service Discovery't

**Valmis!** Sul on nüüd täielik Prometheus monitoring stack: metrics, alerts, dashboards!

---

## Probleemide Lahendamine

| Probleem | Kontrolli | Lahendus |
|----------|-----------|----------|
| AlertManager ei saada Slack'i | `docker logs alertmanager` | Kontrolli Webhook URL'i |
| Slack'i ei jõua teade | Webhook'i URL'i test | `curl -X POST -H 'Content-type: application/json' --data '{"text":"Test"}' WEBHOOK_URL` |
| Grafana ei näe Prometheus'e | Data source test | Kasuta `http://prometheus:9090` |
| Alert ei käivitu | `/alerts` leht | Kontrolli `for:` aega - oota piisavalt |
| cAdvisor DOWN | `docker logs cadvisor` | Kontrolli volume mount'e |

---

## Portide Koond

| Teenus | Port | URL |
|--------|------|-----|
| Prometheus | 9090 | `http://192.168.1.150:9090` |
| Node Exporter | 9100 | `http://192.168.1.150:9100/metrics` |
| Docker Engine | 9323 | `http://192.168.1.150:9323/metrics` |
| cAdvisor | 8080 | `http://192.168.1.150:8080` |
| AlertManager | 9093 | `http://192.168.1.150:9093` |
| Grafana | 3000 | `http://192.168.1.150:3000` |