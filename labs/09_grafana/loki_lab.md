# Grafana Stack - Labor

**Aeg:** 4h  
**Eesmärk:** Lisa Grafanale kaks uut data source'i - Loki (logs) ja Tempo (traces)

---

## Sõnastik - olulised mõisted

| Mõiste | Selgitus | Näide |
|--------|----------|-------|
| **Logi** | Tekstiline kirje, mida rakendus kirjutab | "2025-11-03 14:30 ERROR payment failed" |
| **Trace** | Ühe päringu tee läbi mitme teenuse | Kasutaja makse → auth → payment → bank |
| **Span** | Üks operatsioon trace'is | "database query võttis 850ms" |
| **Label** | Metadata logi jaoks | service="payment", env="prod" |
| **Stream** | Logide grupp sama labelitega | Kõik "payment" teenuse logid |
| **Derived field** | Automaatne link logis | trace_id → kliki → Tempo avaneb |
| **LogQL** | Loki päringukeel | Sarnane PromQL'iga (Prometheus päringukeel, õpite järgmises teemas) |
| **TraceQL** | Tempo päringukeel | Otsi trace'e sisu järgi |

---

## Kuidas andmed liiguvad süsteemis

```
┌─────────────────────────────────────────────────┐
│ RAKENDUSED (HotROD, log-generator)              │
│ - Kirjutavad logisid                             │
│ - Genereerivad trace'e (OpenTelemetry)          │
└──────────┬────────────────────────┬──────────────┘
           │ logid                  │ traces
           ↓                        ↓
    ┌──────────────┐        ┌──────────────┐
    │  Promtail    │        │    Tempo     │
    │  (koguja)    │        │   (server)   │
    └──────┬───────┘        └──────┬───────┘
           │                       │
           ↓                       │
    ┌──────────────┐               │
    │     Loki     │               │
    │   (server)   │               │
    └──────┬───────┘               │
           │                       │
           └───────────┬───────────┘
                       ↓
              ┌─────────────────┐
              │    Grafana      │
              │ (visualiseering)│
              └─────────────────┘
                       ↓
              Te näete brauseris!
```

**Miks te seda teete:**
- **Promtail** kogub logisid automaatselt → ei pea SSH'ma igasse serverisse
- **Loki** salvestab logid odavalt → kõik logid ühes kohas
- **Tempo** salvestab trace'e → näete päringu teed läbi süsteemi
- **Grafana** näitab kõike → üks dashboard, kõik andmed

---

## Mida te juba teate

- Grafana dashboardid (teema 3)
- Prometheus metrics
- Panelid, visualiseerimised
- Time series graafikud

## Mida täna õpite

- Loki data source - logid Grafanas
- Tempo data source - trace'd Grafanas  
- Navigate logs ↔ traces
- Kõik kolm data source'i ühes dashboardis

---

## Ülesanne 1: Keskkonna käivitamine (45 min)

**Miks:** HotROD ja log-generator annavad meile automaatselt logisid ja trace'e harjutamiseks.

### Samm 1: Töökataloog

```bash
mkdir ~/grafana-stack-lab
cd ~/grafana-stack-lab
```

### Samm 2: Docker Compose fail

Loo `docker-compose.yml`:

```yaml
version: "3.8"

services:
  grafana:
    image: grafana/grafana:10.2.0
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    volumes:
      - grafana-data:/var/lib/grafana
      - ./provisioning:/etc/grafana/provisioning
    networks:
      - monitoring

  loki:
    image: grafana/loki:2.9.3
    container_name: loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - loki-data:/loki
    networks:
      - monitoring

  promtail:
    image: grafana/promtail:2.9.3
    container_name: promtail
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - ./promtail-config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml
    networks:
      - monitoring

  tempo:
    image: grafana/tempo:2.3.1
    container_name: tempo
    command: ["-config.file=/etc/tempo.yaml"]
    volumes:
      - ./tempo-config.yaml:/etc/tempo.yaml
      - tempo-data:/tmp/tempo
    ports:
      - "3200:3200"
      - "4317:4317"
      - "4318:4318"
    networks:
      - monitoring

  hotrod:
    image: jaegertracing/example-hotrod:latest
    container_name: hotrod
    command: ["all", "--otel-exporter=otlp"]
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://tempo:4318
      - OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
    ports:
      - "8080:8080"
    labels:
      logging: "promtail"
    networks:
      - monitoring

  log-generator:
    image: mingrammer/flog:0.4.3
    container_name: log-generator
    command: -f json -t log -d 1s -l
    labels:
      logging: "promtail"
    networks:
      - monitoring

volumes:
  grafana-data:
  loki-data:
  tempo-data:

networks:
  monitoring:
```

**Mis toimub:**
- Grafana - te juba teate
- Loki - logide server
- Promtail - kogub logisid (agent igas serveris)
- Tempo - trace'ide server
- HotROD - test app (genereerib logisid ja trace'e)
- log-generator - genereerib JSON logisid

### Samm 3: Promtail konfiguratsioon

Loo `promtail-config.yml`:

```yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
        filters:
          - name: label
            values: ["logging=promtail"]
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: 'stream'
```

**Selgitus:**
- `server` - Promtail kuulab port 9080
- `clients` - saadab logid Lokisse
- `scrape_configs` - kogub Docker konteinerite logisid
- `filters` - ainult konteinerid, millel label `logging=promtail`

### Samm 4: Tempo konfiguratsioon

Loo `tempo-config.yaml`:

```yaml
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        http:
          endpoint: 0.0.0.0:4318
        grpc:
          endpoint: 0.0.0.0:4317

storage:
  trace:
    backend: local
    wal:
      path: /tmp/tempo/wal
    local:
      path: /tmp/tempo/blocks
```

**Selgitus:**
- `receivers` - võtab vastu OTLP protocol'iga trace'e
- `storage` - salvestab lokaalsesse failisüsteemi

### Samm 5: Grafana provisioning

```bash
mkdir -p provisioning/datasources
```

Loo `provisioning/datasources/datasources.yml`:

```yaml
apiVersion: 1

datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    jsonData:
      derivedFields:
        - name: TraceID
          matcherRegex: "trace_id[=:]([a-f0-9]+)"
          url: "$${__value.raw}"
          datasourceUid: tempo

  - name: Tempo
    type: tempo
    access: proxy
    uid: tempo
    url: http://tempo:3200
    jsonData:
      tracesToLogsV2:
        datasourceUid: 'loki'
        filterByTraceID: true
```

**Selgitus:**
- `derivedFields` - log'is olev trace_id muutub lingiks
- `tracesToLogsV2` - trace'ist saab vaadata logisid

### Samm 6: Käivita

```bash
docker-compose up -d
```

Kontrolli:

```bash
docker-compose ps
```

Peaks nägema:
```
grafana         running
loki            running
promtail        running
tempo           running
hotrod          running
log-generator   running
```

### Samm 7: Testi

**Grafana:** http://localhost:3000

Configuration → Data sources → peaks nägema:
- Loki (roheline - connected)
- Tempo (roheline - connected)

**HotROD:** http://localhost:8080

Kliki mõnel nupul (Rachel, Seth jne) - see genereerib trace'e.

### ⚠️ Kui midagi ei tööta

**Probleem: Konteiner ei käivitu**
```bash
# Vaata logisid
docker-compose logs <konteineri_nimi>

# Näiteks
docker-compose logs loki
docker-compose logs promtail
```

**Probleem: Loki ei ole "connected" Grafanas**
```bash
# Kontrolli Loki tervist
curl http://localhost:3100/ready

# Peaks tagastama "ready"
```

**Probleem: Ei näe logisid Grafanas**
```bash
# Kontrolli Promtail metrics
curl http://localhost:9080/metrics | grep promtail_sent_entries_total

# Peab olema > 0
```

**Probleem: Port 3000 on juba kasutusel**
```bash
# Muuda docker-compose.yml
ports:
  - "3001:3000"  # Kasuta 3001 asemel
```

**Abi vajadusel:**
- Docker dokumentatsioon: https://docs.docker.com
- Grafana kogukond: https://community.grafana.com
- Küsi õpetajalt!

### Checkpoint 1

✅ Kõik 6 konteinrit töötavad  
✅ Grafana avaneb  
✅ Loki ja Tempo data sources on seadistatud  
✅ HotROD app töötab  

---

## Ülesanne 2: Loki päringud (60 min)

**Miks me seda teeme:**
- LogQL on päringukeel logide jaoks (sarnane PromQL'iga, mida õpite järgmises teemas)
- Õpite, kuidas logidest informatsiooni leida
- Õpite, kuidas logidest metrics'eid genereerida

**Kuidas peaks välja nägema:**
```
Explore vaade Grafanas:
┌──────────────────────────────────────┐
│ Data source: [Loki ▼]               │
│ Query: {container="log-generator"}   │
│ [Run query]                          │
├──────────────────────────────────────┤
│ 2025-11-03 14:30:15                  │
│ {"host":"server01", "status":"200"}  │
│                                      │
│ 2025-11-03 14:30:16                  │
│ {"host":"server02", "status":"500"}  │
└──────────────────────────────────────┘
```

Grafanas: Explore → vali **Loki**

### Stream selector

Grafanas: Explore → vali **Loki**

Päring:

```logql
{container="log-generator"}
```

Vajuta Enter.

Näed JSON logisid - sarnane `docker logs` väljundile, aga Grafanas!

**Teine päring:**

```logql
{container="hotrod"}
```

Näed HotROD rakenduse logisid.

### Line filter (nagu grep)

**Leia read, mis sisaldavad "500":**

```logql
{container="log-generator"} |= "500"
```

`|=` = contains

**Leia POST või PUT:**

```logql
{container="log-generator"} |~ "POST|PUT"
```

`|~` = regex

**Välista GET:**

```logql
{container="log-generator"} != "GET"
```

### JSON parsing

```logql
{container="log-generator"} | json
```

Nüüd näed struktureeritud välju: host, method, status jne.

**Filtreeri parsitud välja järgi:**

```logql
{container="log-generator"} | json | status >= 500
```

Ainult error status koodid!

### Metrics logidest

**Loe logiridu minutis:**

```logql
count_over_time({container="log-generator"}[1m])
```

Vaheta Visualisation: **Time series**

Näed graafikut! Nagu metrics graafikud, mida olete Grafanas näinud!

**Error rate:**

```logql
rate({container="log-generator"} | json | status >= 500 [5m])
```

**Summeeri konteineri kaupa:**

```logql
sum(rate({container=~".*"}[5m])) by (container)
```

Funktsioonid nagu `sum()` ja `by()` - õpite neid põhjalikult järgmises teemas (Prometheus)!

### Quantile

```logql
quantile_over_time(0.95, 
  {container="log-generator"} 
  | json 
  | unwrap bytes_sent [5m])
```

`unwrap` = võta number logist, käsitle metric'una.

P95 bytes_sent!

### Top päringud

```logql
topk(5, 
  sum(count_over_time({container="log-generator"} | json [1h])) by (host)
)
```

Vaheta Visualisation: **Table**

### Ülesanded

**A) Successful requests**

Leia logid, kus status on 200-299.

<vihje>
Kasuta: `| json | status >= 200 | status < 300`
</vihje>

**B) Top 3 host'i**

Leia 3 host'i, kes genereerivad kõige rohkem logisid.

<vihje>
Kasuta: `topk(3, ...)` ja `count_over_time()`
</vihje>

**Kontrolli:** Tee ekraanipilt oma tulemustest!

### Checkpoint 2

✅ Oskad stream selector'e  
✅ Oskad line filter'eid  
✅ Oskad JSON parse'ida  
✅ Oskad logidest metrics'e teha  
✅ Mõistad LogQL põhitõdesid  

---

## Ülesanne 3: Tempo trace'd (45 min)

### Trace'ide vaatamine

HotROD: http://localhost:8080

Kliki nuppudel 10 korda (Rachel, Seth jne).

Grafanas: Explore → vali **Tempo**

Kliki **Search** → **Run query**

Näed listi trace'e:
- Trace ID
- Start time
- Duration
- Span count

### Trace detailvaade

Kliki ühel trace'il.

Näed **Gantt chart'i**:

```
[frontend    ]====================> 1000ms
  [customer  ]===> 100ms
  [route     ]======> 200ms
    [mysql   ]====> 150ms
```

- Iga tulp = üks span
- Tulba pikkus = kestus
- Vertikaalsed jooned = parent-child

**Kliki span'il** → näed detaile:
- Service name
- Operation name
- Duration
- Tags (http.method, http.url jne)

### TraceQL päringud

**Leia aeglased trace'd:**

```traceql
{duration > 500ms}
```

**Leia error'itega trace'd:**

```traceql
{status = error}
```

**Leia teenuse järgi:**

```traceql
{resource.service.name = "customer"}
```

**Kombineeri:**

```traceql
{resource.service.name = "frontend" && duration > 1s}
```

### Ülesanded

**D) Aeglane trace**

Leia trace, kus kogukestus üle 1 sekund. Dokumenteeri trace ID ja kõige aeglasem span.

**E) Error trace**

Leia trace, kus on error. Dokumenteeri, millises teenuses error tekkis.

**Kontrolli:** Tee ekraanipilt!

### Checkpoint 3

✅ Oskad trace'e otsida  
✅ Mõistad Gantt chart'i  
✅ Oskad span detaile vaadata  
✅ Oskad TraceQL päringuid  

---

## Ülesanne 4: Logs ↔ Traces integratsioon (60 min)

**See on kõige võimsam osa!**

### Logist trace'ile

Explore → **Loki** → päring:

```logql
{container="hotrod"} | json
```

Vaata logisid. Otsi `trace_id` välja.

**Märkad linki ikooni?** See on derived field!

**Kliki lingil** → Tempo avaneb automaatselt!

### Trace'ist logile

Explore → **Tempo** → vali trace → kliki span'il

Span detailides: **"Logs for this span"**

**Kliki sellel** → Loki avaneb!

### Töövoog: Debug

**Stsenaarium:** Klient kaebab aeglast rakendust.

**Samm 1 - Logi:**

```logql
{container="hotrod"} | json
```

Muuda Time range → vali aeg, kui genere erisid trace'e.

**Samm 2 - Navigate:**

Logis on trace_id → kliki → trace avaneb

**Samm 3 - Analüüs:**

Vaata trace't → näed, milline span on aeglane

### Multi-datasource vaade

Explore tab'is kliki **+** (ülevas menüüs) → lisab teise paneeli.

**Vasakul:** Loki  
**Paremal:** Tempo

Mõlemas vali sama Time range.

**Töövoog:**
1. Vasakul - näed logi
2. Kliki trace_id
3. Paremal - trace avaneb

### Ülesanded

**G) Log → Trace**

Leia HotROD logist trace_id. Kliki lingil. Dokumenteeri aeglaseim span.

**H) Trace → Log**

Tempo'st vali trace. Kliki "Logs for this span". Dokumenteeri, mitu logi kirjet leidsid.

**Kontrolli:** Tee ekraanipilt!

### Checkpoint 4

✅ Derived fields töötavad  
✅ TracesToLogs töötab  
✅ Oskad multi-datasource Explore't  
✅ Mõistad debug workflow'i  

---

## Ülesanne 5: Dashboard (45 min)

Nüüd paned kõik kolm data source'i ühte dashboardi!

### Panel 1: Log Volume

Dashboards → New → Add visualization

- Data source: **Loki**
- Query:
```logql
sum(count_over_time({container=~".*"}[1m])) by (container)
```
- Visualisation: **Time series**
- Title: "Log Volume by Container"

Save panel.

### Panel 2: Error Rate

Add visualization:

- Data source: **Loki**
- Query:
```logql
sum(rate({container="log-generator"} | json | status >= 500 [5m]))
```
- Visualisation: **Stat**
- Title: "Error Rate"
- Unit: ops/sec (Standard options → Unit)

### Panel 3: Status Distribution

Add visualization:

- Data source: **Loki**
- Query:
```logql
sum by (status) (count_over_time({container="log-generator"} | json [5m]))
```
- Visualisation: **Pie chart**
- Title: "HTTP Status Distribution"

### Panel 4: Recent Errors

Add visualization:

- Data source: **Loki**
- Query:
```logql
{container="log-generator"} | json | status >= 500
```
- Visualisation: **Logs**
- Title: "Recent Errors"

### Dashboard salvestamine

Save dashboard: "Observability Dashboard"

### Ülesanded

**J) P95 panel**

Lisa panel, mis näitab P95 latency. Kasuta `quantile_over_time(0.95, ...)`.

**Kontrolli:** Tee ekraanipilt dashboardist!

### Checkpoint 5

✅ Dashboard loodud vähemalt 4 paneeliga  
✅ Panelid näitavad live andmeid  
✅ Dashboard salvestatud  

---

## Ülesanne 6: Troubleshooting (45 min)

### Stsenaarium

HotROD app: mõned tellimused on aeglased.

**Teie ülesanne:**
1. Reprodutseeri - kliki ühel nupul 20 korda
2. Leia aeglased logid - `{container="hotrod"}`
3. Navigate trace'ile - kliki trace_id
4. Dokumenteeri - milline span oli aeglane ja miks

### Incident report (3-4 lauset)

**Probleem:**  
_____________________

**Mida leidsin:**  
_____________________

**Root cause:**  
_____________________

**Lahendus:**  
_____________________

### Checkpoint 6

✅ Aeglased logid leitud  
✅ Trace vaadatud  
✅ Report kirjutatud  

---

## Kokkuvõte

Olete läbinud:
- Grafana Stack setup
- Loki päringud (LogQL)
- Tempo traces (TraceQL)
- Logs ↔ Traces navigation
- Dashboard loomine
- Troubleshooting

### 💭 Reflection (5 min)

**1. Mis oli kõige raskem ja miks?**

<vastus>
_______________________________
_______________________________
</vastus>

**2. Kuidas Grafana Stack aitab debugging'ul?**

<vastus>
_______________________________
_______________________________
</vastus>

**3. Üks asi, mida veel ei saa aru:**

<vastus>
_______________________________
</vastus>

---

### 💡 Soovitus: Co-coding

Seda labori saab teha **koos kellegagi**:
- Üks kirjutab päringuid, teine kontrollib tulemusi
- Vahetate rolle iga ülesande järel
- Arutate koos, miks midagi töötab või ei tööta
- Õppimine on kiirem ja lõbusam!

**Pair programming workflow:**
1. Driver (klaviatuuri taga) + Navigator (juhendab)
2. Iga 15 min vahetus
3. Mõlemad peavad aru saama, mis toimub

---

## Puhastamine

```bash
cd ~/grafana-stack-lab
docker-compose down -v
```

---

## Hindamine

- [ ] Kõik 6 ülesannet läbitud
- [ ] Checkpoints täidetud
- [ ] Ülesanded A-L dokumenteeritud
- [ ] Incident report kirjutatud

**Boonus:**
- [ ] Dashboard eksporditud JSON'i
- [ ] Reflection küsimustele vastatud
