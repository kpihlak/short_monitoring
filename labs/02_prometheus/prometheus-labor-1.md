# Prometheus Labor 1: Alused ja Instrumenteerimine

**Kestus:** 3 × 45 minutit  
**Eeldused:** Linux CLI põhitõed, Docker paigaldatud  
**Eesmärk:** Paigaldad Prometheus'e, kogud süsteemi mõõdikuid, instrumenteerid rakenduse

> **Loengust:** Vaata üle "Prometheus Alused" enne labi

---

## Tunni 1: Prometheus Setup ja Node Exporter (45 min)

### Mida õpid selles tunnis?

- Paigaldad Prometheus'e Docker Compose'iga
- Lisad Node Exporter'i süsteemi mõõdikute kogumiseks
- Mõistad Docker võrgustikku ja port mapping'u
- Teed esimesed PromQL päringud

---

### Oluline: Võrgustik Proxmox Keskkonnas

**Sinu setup:**
- **Proxmox host:** Füüsiline server
- **VM:** Ubuntu, bridged võrk, IP näiteks `192.168.1.150`
- **Docker konteinerid:** Töötavad VM sees

**Kuidas see töötab:**

```
Sinu arvuti (Windows/Mac)
    ↓ (brauser)
VM IP: 192.168.1.150:9090
    ↓ (Docker port mapping)
Prometheus kontainer: port 9090
```

**Kontrolli oma VM IP:**
```bash
ip addr show | grep inet
```

Näed midagi nagu: `inet 192.168.1.150/24`

**Kirjuta see üles! Vajad seda kogu labi vältel.**

Minu VM IP: `__________________`

---

### Ettevalmistus

**Kontrolli Dockerit:**
```bash
docker --version
docker-compose --version
```

Kui puudub, paigalda:
```bash
curl -fsSL https://get.docker.com -o install-docker.sh
sudo sh install-docker.sh
sudo usermod -aG docker ${USER}
newgrp docker
```

**Loo projekti struktuur:**
```bash
cd ~
mkdir -p prometheus-lab/docker-compose/prometheus/config
cd prometheus-lab/docker-compose/prometheus
```

---

### Docker Compose Konfiguratsioon

**Loo fail `docker-compose.yml`:**

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

volumes:
  prometheus-data: {}

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./config:/etc/prometheus
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--web.enable-lifecycle'
    restart: unless-stopped

  nodeexporter:
    image: prom/node-exporter:latest
    container_name: nodeexporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
    ports:
      - "9100:9100"
    restart: unless-stopped
    network_mode: host
EOF
```

**Selgitus:**

`ports: "9090:9090"` - VM port 9090 suunab Prometheus konteineri porti 9090

`network_mode: host` - Node Exporter kasutab VM võrku otse (loeb VM ressursse, mitte konteineri!)

`volumes: /proc:/host/proc:ro` - näitab VM `/proc` kausta konteinerile (seal on süsteemi info)

`--web.enable-lifecycle` - võimaldab Prometheus'e konfiguratsiooni uuesti laadida ilma taaskäivitamata

---

### Prometheus Konfiguratsioon

**Loo fail `config/prometheus.yml`:**

```bash
cat > config/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
EOF
```

**Selgitus:**

`scrape_interval: 15s` - Prometheus küsib mõõdikuid iga 15 sekundi järel

`job_name` - Grupeerib target'id loogiliselt

`targets: ['localhost:9100']` - Node Exporter kasutab `network_mode: host`, seega on kättesaadav localhost kaudu

> **Loengust:** Prometheus kasutab pull mudelit - küsib ise target'eid, ei oota et target'id saadaks.

---

### Käivita Teenused

```bash
docker-compose up -d
```

Flag `-d` tähendab "detached mode" - töötab taustal.

**Kontrolli:**
```bash
docker ps
```

Peaksid nägema kahte konteinerit:
```
CONTAINER ID   IMAGE                       STATUS
abc123         prom/prometheus:latest      Up 10 seconds
def456         prom/node-exporter:latest   Up 10 seconds
```

**Vaata logisid:**
```bash
docker logs prometheus
docker logs nodeexporter
```

Ei tohiks näha ERROR'eid.

---

### Ülesanne 1: Ava Prometheus UI

**Eesmärk:** Veendud, et Prometheus töötab.

**Sammud:**

1. Ava brauser oma arvutis (Windows/Mac)

2. Mine aadressile (asenda oma VM IP-ga!):
   ```
   http://192.168.1.150:9090
   ```

3. Peaksid nägema Prometheus UI

**Kui ei avane:**
- Kontrolli VM IP: `ip addr show`
- Kontrolli, kas Docker töötab: `docker ps`
- Kontrolli logisid: `docker logs prometheus`
- Kontrolli firewall'i: `sudo ufw status` (peaks olema inactive või 9090 avatud)

---

### Ülesanne 2: Kontrolli Target'eid

**Eesmärk:** Veendud, et Prometheus kogub mõõdikuid.

**Sammud:**

1. Prometheus UI's kliki ülaosas: **Status** → **Targets**

2. Peaksid nägema:

| Endpoint | State |
|----------|-------|
| prometheus (localhost:9090) | UP (roheline) |
| node (localhost:9100) | UP (roheline) |

**Kui näed DOWN (punane):**

```bash
# Kontrolli konteinereid
docker ps

# Vaata logisid
docker logs prometheus
docker logs nodeexporter

# Testi käsitsi
curl http://localhost:9100/metrics
```

**Mida õppisid?**
- Docker Compose kasutamine
- Port mapping mõistmine
- Target'ite staatuse kontroll

---

### Ülesanne 3: Esimene PromQL Päring

**Eesmärk:** Õpid Prometheus UI kasutama ja teed esimese päringu.

**Sammud:**

1. Prometheus UI's mine **Graph** tab'i

2. Kirjuta päringukasti:
   ```promql
   up
   ```

3. Vajuta **Execute**

4. Vali **Table** vaade

**Mida näed?**
```
up{instance="localhost:9090", job="prometheus"} 1
up{instance="localhost:9100", job="node"} 1
```

**Selgitus:**
- `up` on sisseehitatud mõõdik
- `1` = target on kättesaadav (UP)
- `0` = target on maas (DOWN)
- `instance` ja `job` on label'id

> **Loengust:** Label'id võimaldavad filtreerida ja grupeerida mõõdikuid.

---

### Ülesanne 4: Node Exporter Mõõdikud

**Eesmärk:** Uurid süsteemi mõõdikuid.

**Sammud:**

1. Ava uus brauser tab:
   ```
   http://192.168.1.150:9100/metrics
   ```

2. Näed tuhandeid ridu teksti - see on Prometheus exposition formaat:
   ```
   # HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
   # TYPE node_cpu_seconds_total counter
   node_cpu_seconds_total{cpu="0",mode="idle"} 123456.78
   node_cpu_seconds_total{cpu="0",mode="user"} 5678.90
   ...
   ```

3. Prometheus UI's tee päring:
   ```promql
   node_cpu_seconds_total
   ```

4. Liiga palju tulemusi? Filtreeri:
   ```promql
   node_cpu_seconds_total{mode="idle"}
   ```

**Mida õppisid?**
- `/metrics` endpoint formaat
- HELP ja TYPE read
- Label'ite kasutamine filtreerimiseks

---

### Ülesanne 5: Põhilised Süsteemi Päringud

**Proovi neid päringuid Prometheus UI's:**

**1. Mitu CPU core'i on?**
```promql
count(node_cpu_seconds_total{mode="idle"})
```

**2. RAM kogus gigabaitides:**
```promql
node_memory_MemTotal_bytes / 1024 / 1024 / 1024
```

**3. Vaba kettaruum:**
```promql
node_filesystem_free_bytes{mountpoint="/"}
```

**4. Millal server käivitati?**
```promql
node_boot_time_seconds
```

Tulemus on Unix timestamp. Kasuta [epochconverter.com](https://epochconverter.com) kuupäeva jaoks.

**5. Serveri info:**
```promql
node_uname_info
```

Näed OS nime, kerneli versiooni jms.

---

### Terminali Nõuanded

Sageli vajad mitut terminali korraga.

**Terminal 1: Logi jälgimine**
```bash
docker logs -f prometheus
```

Flag `-f` = follow (jälgib pidevalt). Näed reaalajas, kui Prometheus target'eid scrape'ib.

Jäta see terminal lahti!

**Terminal 2: Käsud**

Ava uus SSH seanss või kasuta `tmux`/`screen`.

```bash
curl http://localhost:9090/api/v1/query?query=up
docker ps
```

**Peata logi:** Vajuta `Ctrl+C` esimeses terminalis.

---

### Kontrolli, Kas Kõik Töötab

| Kontroll | Käsk | Oodatav tulemus |
|----------|------|-----------------|
| Konteinerid töötavad? | `docker ps` | 2 konteinerit RUNNING |
| Prometheus logis vigu? | `docker logs prometheus` | Ei näe ERROR |
| Node Exporter töötab? | `curl localhost:9100/metrics` | Näed mõõdikuid |
| UI avaneb? | Brauser: `http://<VM-IP>:9090` | Prometheus UI |
| Targets UP? | UI: Status → Targets | Mõlemad rohelised |

---

### Kokkuvõte: Tunni 1

Õppisid:
- Docker Compose kasutamist
- Prometheus ja Node Exporter paigaldamist
- Docker võrgustiku põhimõtteid (port mapping, network modes)
- Prometheus UI kasutamist
- Esimesi PromQL päringuid

**Järgmine tunni:** Instrumenteerid Python rakenduse kõigi 4 mõõdikutüübiga.

---

## Tunni 2: Rakenduse Instrumenteerimine (45 min)

### Mida õpid selles tunnis?

- Lood Python HTTP serveri
- Lisad kõik 4 mõõdikutüüpi (Counter, Gauge, Histogram, Summary)
- Genereerid liiklust
- Lisad rakenduse Prometheus'e

> **Loengust:** 4 mõõdikutüüpi - Counter (ainult kasvab), Gauge (tõuseb-langeb), Histogram (jaotus), Summary (protsentiilid).

---

### Python Keskkonna Seadistamine

**Loo virtuaalne keskkond:**
```bash
cd ~/prometheus-lab
mkdir -p instrumentation
cd instrumentation
python3 -m venv venv
source venv/bin/activate
pip install prometheus_client
```

**Selgitus:**

`venv` on eraldatud Python keskkond. Teegid paigaldatakse siia, mitte süsteemi.

`source venv/bin/activate` aktiveerib keskkonna (näed `(venv)` prompt'is).

**Hiljem uues terminalis:**
```bash
cd ~/prometheus-lab/instrumentation
source venv/bin/activate
```

---

### HTTP Server Kõigi Mõõdikutega

**Loo fail `httpserver_test.py`:**

```bash
cat > httpserver_test.py << 'EOF'
import http.server
import random
import time
from prometheus_client import start_http_server, Counter, Gauge, Histogram, Summary

print("HTTP Server käivitub...")
print("Rakendus: http://localhost:5000")
print("Mõõdikud: http://localhost:5001/metrics")

# Mõõdikud
REQUESTS = Counter('http_requests_total', 'HTTP päringute koguarv')
ACTIVE_USERS = Gauge('active_users', 'Aktiivsete kasutajate arv')
REQUEST_DURATION = Histogram('request_duration_seconds', 'Päringu kestus sekundites')
RESPONSE_TIME = Summary('response_time_seconds', 'Vastuse aja kokkuvõte')

class MyHandler(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        # Counter - ainult kasvab
        REQUESTS.inc()
        
        # Gauge - võib tõusta ja langeda
        ACTIVE_USERS.set(random.randint(1, 20))
        
        # Histogram ja Summary - mõõdavad kestust
        with REQUEST_DURATION.time():
            with RESPONSE_TIME.time():
                time.sleep(random.uniform(0.1, 0.5))
                self.send_response(200)
                self.end_headers()
                self.wfile.write(b'Tere!')
    
    def log_message(self, format, *args):
        pass  # Vähem logi

if __name__ == '__main__':
    start_http_server(5001)
    server = http.server.HTTPServer(('0.0.0.0', 5000), MyHandler)
    server.serve_forever()
EOF
```

**Selgitus:**

`start_http_server(5001)` käivitab `/metrics` endpoint'i port 5001-l

`Counter.inc()` suurendab counter'it +1

`Gauge.set(value)` määrab gauge'i väärtuse

`.time()` mõõdab, kui kaua `with` blokk kestab

---

### Ülesanne 6: Käivita Rakendus

**Sammud:**

1. Veendu, et venv on aktiivne:
   ```bash
   source venv/bin/activate
   ```

2. Käivita server:
   ```bash
   python3 httpserver_test.py
   ```

3. Näed:
   ```
   HTTP Server käivitub...
   Rakendus: http://localhost:5000
   Mõõdikud: http://localhost:5001/metrics
   ```

**Jäta see terminal lahti!** Rakendus peab töötama kogu labi vältel.

---

### Ülesanne 7: Testi Rakendust

**Ava uus terminal:**

```bash
# Testi rakendust
curl http://localhost:5000
```

Vastus: `Tere!`

**Vaata mõõdikuid:**
```bash
curl http://localhost:5001/metrics | grep -E "http_requests|active_users"
```

Näed:
```
# TYPE http_requests_total counter
http_requests_total 1.0

# TYPE active_users gauge
active_users 15.0
```

**Mida õppisid?**
- Prometheus Client Library kasutamine
- `/metrics` endpoint'i loomine
- Erinevate mõõdikutüüpide vahe

---

### Liikluse Genereerimine

**Loo skript `curler.sh`:**

```bash
cat > curler.sh << 'EOF'
#!/bin/bash
while true; do
  curl -s http://localhost:5000 > /dev/null
  sleep $((RANDOM % 3 + 1))
done
EOF

chmod +x curler.sh
```

**Käivita taustal:**
```bash
./curler.sh &
```

**Selgitus:**

`&` käivitab protsessi taustal

`RANDOM % 3 + 1` genereerib juhuslik paus 1-3 sekundit

**Peata hiljem:**
```bash
pkill -f curler.sh
```

---

### Rakenduse Lisamine Prometheus'e

**Probleem:** Python rakendus töötab VM-s, aga MITTE Dockeris.

**Lahendus:** Kasuta VM IP-d Prometheus config'is.

**Uuenda `config/prometheus.yml`:**

```bash
cd ~/prometheus-lab/docker-compose/prometheus
cat > config/prometheus.yml << 'EOF'
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
      - targets: ['192.168.1.150:5001']  # Asenda oma VM IP-ga!
EOF
```

**Selgitus:**

Prometheus kontainer saab jõuda VM IP-sse, sest Docker bridge võrk võimaldab seda.

`localhost` EI tööta, sest see tähendaks Prometheus konteineri localhost'i!

**Lae konfiguratsioon uuesti:**
```bash
curl -X POST http://localhost:9090/-/reload
```

---

### Ülesanne 8: Kontrolli Python App Target'i

**Sammud:**

1. Ava Prometheus UI: Status → Targets

2. Peaksid nägema:
   - prometheus: UP
   - node: UP
   - python-app: UP (uus!)

**Kui python-app on DOWN:**
```bash
# Kontrolli, kas app töötab
ps aux | grep httpserver_test.py

# Kontrolli, kas port on avatud
curl http://localhost:5001/metrics

# Kontrolli Prometheus logi
docker logs prometheus | grep python-app
```

---

### Ülesanne 9: Vaata Rakenduse Mõõdikuid

**Tee päringuid Prometheus UI's:**

**1. Päringute koguarv:**
```promql
http_requests_total
```

**2. Aktiivseid kasutajaid:**
```promql
active_users
```

**3. Request duration bucket'id:**
```promql
request_duration_seconds_bucket
```

Näed erinevaid bucket'e: `le="0.005"`, `le="0.01"`, `le="0.025"` jne.

> **Loengust:** Histogram jagab väärtused bucket'idesse (näiteks < 0.1s, < 0.5s, < 1s).

**4. Summary kvantiile:**
```promql
response_time_seconds{quantile="0.5"}
response_time_seconds{quantile="0.9"}
```

`0.5` = mediaan (50% päringutest on kiiremad)
`0.9` = 90% päringutest on kiiremad

---

### 4 Mõõdikutüüpi Kokkuvõte

| Tüüp | Käitumine | Näide | Millal kasutada |
|------|-----------|-------|-----------------|
| Counter | Ainult kasvab | `http_requests_total` | API päringud, vead, sündmused |
| Gauge | Tõuseb ja langeb | `active_users` | CPU, mälu, temperatuur, järjekorra pikkus |
| Histogram | Jaotus bucket'ites | `request_duration_seconds` | Vastuse ajad, faili suurused |
| Summary | Protsentiilid | `response_time_seconds` | SLA tracking (p95, p99) |

---

### Kokkuvõte: Tunni 2

Õppisid:
- Python rakenduse instrumenteerimist
- Kõiki 4 mõõdikutüüpi
- Liikluse genereerimist
- Rakenduse lisamist Prometheus'e
- VM IP kasutamist Docker ja host vahel

**Järgmine tund:** PromQL põhitõed - päringute kirjutamine ja andmete analüüs.

---

## Tunni 3: PromQL Põhitõed (45 min)

### Mida õpid selles tunnis?

- Kasutad `rate()` funktsiooni (kõige olulisem!)
- Filtreerid label'ite järgi
- Kasutad agregeerimist (`sum`, `avg`, `max`, `min`)
- Kirjutad praktilisi päringuid

> **Loengust:** PromQL on Prometheus'e päringukeel, nagu SQL andmebaasidele.

---

### rate() Funktsioon

**Probleem:** Counter'id ainult kasvavad.

```promql
http_requests_total
```

Näitab: `1234`

See number ise ei ütle midagi! Kas see on palju või vähe? Kui kiiresti kasvab?

**Lahendus:** `rate()` arvutab kasvu sekundis.

```promql
rate(http_requests_total[5m])
```

Näitab: `2.3` - päringuid sekundis!

**Selgitus:**

`[5m]` on range vector - viimased 5 minutit

`rate()` võtab esimese ja viimase väärtuse, arvutab vahe ja jagab ajaga

Tulemus: kasv sekundis

> **Loengust:** Counter'id vajavad alati `rate()` või `increase()` funktsiooni.

---

### Ülesanne 10: Rate Harjutamine

**Proovi neid päringuid:**

**1. Päringute määr:**
```promql
rate(http_requests_total[5m])
```

**2. Päringute määr 1-minutilise akna kohta:**
```promql
rate(http_requests_total[1m])
```

Võrdle tulemusi. Milline on stabiilsem?

**3. CPU kasutus (mitte idle):**
```promql
rate(node_cpu_seconds_total{mode!="idle"}[5m])
```

`mode!="idle"` = kõik peale idle

**4. Võrgu sissetulev traffic:**
```promql
rate(node_network_receive_bytes_total[5m])
```

**Mida õppisid?**
- `rate()` kasutamine
- Range vector'ite `[5m]` tähendus
- Filtreerimist `!=` operaatoriga

---

### Label'ite Filtreerimine

**Põhilised operaatorid:**

```promql
# Täpne match
node_cpu_seconds_total{mode="idle"}

# Negatiivne match
node_cpu_seconds_total{mode!="idle"}

# Regex match
node_filesystem_size_bytes{device=~"/dev/sd.*"}

# Negatiivne regex
node_filesystem_size_bytes{device!~"/dev/sd.*"}

# OR
http_requests_total{job=~"prometheus|python-app"}
```

---

### Ülesanne 11: Filtreerimine

**Proovi:**

**1. Ainult CPU 0 idle aeg:**
```promql
node_cpu_seconds_total{cpu="0", mode="idle"}
```

**2. Kõik CPU'd peale idle:**
```promql
node_cpu_seconds_total{mode!="idle"}
```

**3. Ainult SD kettad:**
```promql
node_disk_io_time_seconds_total{device=~"sd.*"}
```

**4. Root filesystem:**
```promql
node_filesystem_free_bytes{mountpoint="/"}
```

---

### Agregeerimine

**Põhilised funktsioonid:**

```promql
sum()    # Liidab kokku
avg()    # Keskmine
max()    # Suurim
min()    # Vähim
count()  # Loendab
```

**Näited:**

```promql
# Kõik päringud kokku
sum(rate(http_requests_total[5m]))

# Keskmine CPU kasutus
avg(rate(node_cpu_seconds_total{mode="user"}[5m]))

# Suurim mälu kasutus
max(node_memory_MemAvailable_bytes)

# Kui palju target'e on UP?
count(up == 1)
```

---

### Grupeerimine

**Grupeeri label'i järgi:**

```promql
sum by (job) (rate(http_requests_total[5m]))
```

Tulemus:
```
{job="prometheus"} 0.5
{job="python-app"} 2.3
```

**Grupeeri, jättes label'id välja:**

```promql
sum without (instance) (up)
```

Grupeerib kõigi label'ite järgi peale `instance`.

---

### Ülesanne 12: Agregeerimine ja Grupeerimine

**1. Päringute summa per job:**
```promql
sum by (job) (rate(http_requests_total[5m]))
```

**2. Keskmine CPU kasutus per core:**
```promql
avg by (cpu) (rate(node_cpu_seconds_total{mode="user"}[5m]))
```

**3. Kokku UP target'e:**
```promql
sum(up)
```

**4. Kui palju CPU core'e:**
```promql
count(node_cpu_seconds_total{mode="idle"})
```

---

### Praktilised Päringud

**CPU kasutus protsentides:**
```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Selgitus:**
1. `rate(...)` - idle kasvu sekundis
2. `avg(...)` - keskmine üle kõigi CPU'de
3. `* 100` - konverdi protsentideks
4. `100 -` - idle'ist saab kasutatud

**Mälu kasutus protsentides:**
```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

**Vaba kettaruum gigabaitides:**
```promql
node_filesystem_free_bytes{mountpoint="/"} / 1024 / 1024 / 1024
```

**95. protsentiil (Histogram):**
```promql
histogram_quantile(0.95, rate(request_duration_seconds_bucket[5m]))
```

95% päringutest on kiiremad kui see väärtus.

---

### Ülesanne 13: Keerukamad Päringud

**Proovi neid:**

**1. Mitu päeva on server töötanud?**
```promql
(time() - node_boot_time_seconds) / 86400
```

`time()` on praegune aeg, `86400` = sekundeid päevas

**2. Võrgu traffic megabaitides sekundis:**
```promql
rate(node_network_receive_bytes_total[5m]) / 1024 / 1024
```

**3. Top 3 CPU core'i kasutuse järgi:**
```promql
topk(3, rate(node_cpu_seconds_total{mode="user"}[5m]))
```

**4. Ketta täituvus protsentides:**
```promql
100 - ((node_filesystem_free_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100)
```

---

### Graph vs Table Vaated

**Table:** Näitab hetke väärtusi  
**Graph:** Näitab trende ajas

**Ülesanne 14:**

Proovi samu päringuid Graph vaates:

1. `rate(http_requests_total[5m])`
2. `active_users`
3. `100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`

**Mida näed?**
- Kas päringud suurenevad?
- Kas CPU kasutus kõigub?
- Kas kasutajate arv muutub?

---

### Kokkuvõte: Tunni 3

Õppisid:
- `rate()` funktsiooni (kõige olulisem!)
- Label'ite filtreerimist
- Agregeerimist ja grupeerimist
- Praktilisi päringuid (CPU, mälu, disk, võrk)
- Graph ja Table vaadete vahet

## Probleemide Lahendamine

| Probleem | Kontrolli | Lahendus |
|----------|-----------|----------|
| UI ei avane | `docker ps` | Kas kontainer töötab? |
| Target DOWN | `docker logs prometheus` | Vaata scrape vigu |
| Port kinni | `sudo netstat -tulpn | grep 9090` | `sudo ufw allow 9090` |
| Python app ei tööta | `ps aux | grep httpserver` | Käivita uuesti, kontrolli venv |
| localhost ei tööta brauserist | Kasutad VM IP-d? | `http://192.168.1.150:9090` |

---

## Kasulikud Käsud

**Docker:**
```bash
docker ps                    # Töötavad konteinerid
docker logs <container>      # Logi vaatamine
docker logs -f <container>   # Logi jälgimine (follow)
docker-compose up -d         # Käivita kõik
docker-compose down          # Peata kõik
docker-compose restart       # Taaskäivita
```

**Prometheus:**
```bash
curl -X POST http://localhost:9090/-/reload  # Lae konfig uuesti
```

**Python venv:**
```bash
source venv/bin/activate     # Aktiveeri
deactivate                   # Deaktiveeri
```

**Protsesside haldamine:**
```bash
ps aux | grep <nimi>         # Leia protsess
kill <PID>                   # Peata protsess
pkill -f <nimi>              # Peata kõik sobivad
```