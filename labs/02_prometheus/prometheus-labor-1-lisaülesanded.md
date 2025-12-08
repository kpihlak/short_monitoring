# Prometheus Labor 1: Lisaülesanded

**Eesmärk:** Süvendatud harjutused neile, kes tahavad rohkem õppida

> Need ülesanded on vabatahtlikud! Tee neid kodus, kui tahad praktiseerida.

---

## Lisaülesanne 1: Flask Rakendus Dockeris

### Eesmärk

Õpid paigaldama instrumenteeritud Flask rakenduse Dockeris ja lisama selle Prometheus'e.

**Erinevus lihtsast Python serverist:**
- Flask on päris web framework
- Docker teeb paigalduse lihtsamaks
- Production-ready setup

---

### Projekti Struktuur

```bash
cd ~/prometheus-lab
mkdir -p instrumentation/flask_app/app
cd instrumentation/flask_app
```

---

### Flask Rakendus

**Loo fail `app/wsgi_prom.py`:**

```bash
cat > app/wsgi_prom.py << 'EOF'
from flask import Flask, Response
from prometheus_client import Counter, generate_latest

app = Flask(__name__)
REQUESTS = Counter('flask_requests_total', 'Flask päringute arv')

@app.route('/')
def hello():
    REQUESTS.inc()
    return 'Tere Flask rakendusest!'

@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype='text/plain')

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

**Selgitus:**

`generate_latest()` genereerib kõik mõõdikud Prometheus formaadis

`mimetype='text/plain'` määrab õige content-type

---

### uWSGI Konfiguratsioon

**Loo fail `app/uwsgi.ini`:**

```bash
cat > app/uwsgi.ini << 'EOF'
[uwsgi]
module = wsgi_prom
callable = app
master = true
processes = 2
socket = /tmp/uwsgi.sock
chmod-socket = 666
vacuum = true
die-on-term = true
EOF
```

**Selgitus:**

uWSGI on production-ready Python application server.

`processes = 2` käivitab 2 worker protsessi.

---

### Dockerfile

**Loo fail `Dockerfile`:**

```bash
cat > Dockerfile << 'EOF'
FROM tiangolo/uwsgi-nginx-flask:python3.8
ENV LISTEN_PORT=5000
EXPOSE 5000
RUN pip install prometheus-client
COPY ./app /app
EOF
```

**Selgitus:**

Kasutab valmis base image'i, mis sisaldab nginx + uWSGI + Flask.

`COPY ./app /app` kopeerib meie koodi konteinerisse.

---

### Ehita ja Käivita

```bash
# Ehita image
docker build -t flask-prom-app .

# Käivita kontainer
docker run -d --name flask_app -p 5002:5000 flask-prom-app
```

**Kontrolli:**
```bash
docker ps
docker logs flask_app
```

**Testi:**
```bash
curl http://localhost:5002
curl http://localhost:5002/metrics
```

---

### Lisa Prometheus'e

**Uuenda `config/prometheus.yml`:**

```yaml
  - job_name: 'flask-app'
    static_configs:
      - targets: ['192.168.1.150:5002']  # Asenda oma VM IP-ga
```

**Lae uuesti:**
```bash
curl -X POST http://localhost:9090/-/reload
```

**Kontrolli:** Status → Targets → flask-app peaks olema UP

---

### Ülesanne

1. Genereeri liiklust:
```bash
while true; do curl -s http://localhost:5002 > /dev/null; sleep 1; done &
```

2. Vaata mõõdikuid Prometheus'es:
```promql
flask_requests_total
rate(flask_requests_total[5m])
```

3. Võrdle:
   - Python HTTP server vs Flask rakendus
   - Mis on erinevused mõõdikutes?
   - Kumb on lihtsam instrumenteerida?

---

## Lisaülesanne 2: Textfile Collector

### Eesmärk

Õpid eksportima mõõdikuid bash scriptidest Node Exporter'i kaudu.

**Millal kasulik:**
- Cron job'id
- Legacy skriptid
- Kui ei saa HTTP endpoint'i pakkuda

---

### Node Exporter Seadistamine

**Uuenda `docker-compose.yml`:**

```yaml
  nodeexporter:
    image: prom/node-exporter:latest
    container_name: nodeexporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
      - /tmp:/tmp  # Lisa see rida!
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.textfile.directory=/tmp'  # Lisa see rida!
    ports:
      - "9100:9100"
    restart: unless-stopped
    network_mode: host
```

**Taaskäivita:**
```bash
cd ~/prometheus-lab/docker-compose/prometheus
docker-compose up -d nodeexporter
```

**Kontrolli:**
```bash
curl http://localhost:9100/metrics | grep textfile
```

Peaksid nägema: `node_textfile_scrape_error 0`

---

### Mõõdikute Eksportimise Skript

**Loo skript `instrumentation/file_counter.sh`:**

```bash
cd ~/prometheus-lab/instrumentation
cat > file_counter.sh << 'EOF'
#!/bin/bash

DIR=/tmp
RESULT=$(ls ${DIR}/test-* 2>/dev/null | wc -l)
TIMESTAMP=$(date +%s)

# Kirjuta ajutisse faili
cat > "$DIR/filecount.prom.$$" << END
# HELP test_files_total Testfailide arv /tmp kataloogis
# TYPE test_files_total gauge
test_files_total $RESULT

# HELP test_files_last_check Viimase kontrolli timestamp
# TYPE test_files_last_check gauge
test_files_last_check $TIMESTAMP
END

# Liiguta atomaarselt
mv "$DIR/filecount.prom.$$" "$DIR/filecount.prom"
EOF

chmod +x file_counter.sh
```

**Selgitus:**

`.$$` on protsessi ID - tagab, et fail kirjutatakse atomaarselt (ei poolikut faili)

`mv` on atomic operatsioon Linux'is

---

### Testi Skripti

**Loo testfaile:**
```bash
touch /tmp/test-{1..5}.txt
```

**Käivita skript:**
```bash
./file_counter.sh
```

**Kontrolli:**
```bash
cat /tmp/filecount.prom
```

Peaksid nägema:
```
test_files_total 5
test_files_last_check 1731312345
```

---

### Vaata Prometheus'es

**Oota 15-30 sekundit** (Node Exporter scrape'ib faili).

**Tee päring:**
```promql
test_files_total
test_files_last_check
```

**Kui ei näe:**
- Kontrolli Node Exporter logi: `docker logs nodeexporter`
- Kontrolli faili: `cat /tmp/filecount.prom`
- Kontrolli, kas Node Exporter näeb: `curl localhost:9100/metrics | grep test_files`

---

### Cron Job

**Lisa cron job, mis käivitab skripti iga minut:**

```bash
crontab -e
```

Lisa rida:
```
* * * * * /home/ubuntu/prometheus-lab/instrumentation/file_counter.sh
```

**Kontrolli:**
```bash
crontab -l
tail -f /var/log/syslog | grep CRON
```

---

### Ülesanne

**1. Loo uusi testfaile:**
```bash
touch /tmp/test-{6..10}.txt
```

**2. Vaata, kuidas `test_files_total` muutub Prometheus'es.**

**3. Kirjuta oma skript, mis ekspordib:**
- Kausta suurus (MB)
- Protsesside arv
- Viimase backup'i timestamp

**Vihje:**
```bash
# Kausta suurus
du -sm /var/log | awk '{print "log_size_mb " $1}'

# Protsesside arv
ps aux | wc -l | awk '{print "process_count " $1}'
```

---

## Lisaülesanne 3: PromQL Harjutused

### Eesmärk

Harjutad keerukamaid PromQL päringuid.

---

### Võrgu Statistika

**Sissetulev traffic MB/s:**
```promql
rate(node_network_receive_bytes_total[5m]) / 1024 / 1024
```

**Väljaminev traffic MB/s:**
```promql
rate(node_network_transmit_bytes_total[5m]) / 1024 / 1024
```

**Kokku traffic (sisse + välja):**
```promql
sum(rate(node_network_receive_bytes_total[5m])) + 
sum(rate(node_network_transmit_bytes_total[5m]))
```

**Ülesanne:** Millise network interface kaudu liigub kõige rohkem andmeid?

```promql
topk(1, rate(node_network_receive_bytes_total[5m]))
```

---

### Ketta Statistika

**Disk read rate (MB/s):**
```promql
rate(node_disk_read_bytes_total[5m]) / 1024 / 1024
```

**Disk write rate (MB/s):**
```promql
rate(node_disk_written_bytes_total[5m]) / 1024 / 1024
```

**IOPS (Input/Output Operations Per Second):**
```promql
rate(node_disk_io_time_seconds_total[5m])
```

**Ketta täituvus protsentides:**
```promql
100 - ((node_filesystem_free_bytes{mountpoint="/"} / 
         node_filesystem_size_bytes{mountpoint="/"}) * 100)
```

**Ülesanne:** Kui kiiresti ketas täitub (kui praegune kirjutamise kiirus jätkub)?

```promql
# Vaba ruum GB
node_filesystem_free_bytes{mountpoint="/"} / 1024 / 1024 / 1024

# Jagatud kirjutamise kiirusega (GB/s)
node_filesystem_free_bytes{mountpoint="/"} / 
rate(node_disk_written_bytes_total[5m])

# Tulemus on sekundites, jaga 86400 = päevades
```

---

### Matemaatika ja Võrdlused

**Mitu sekundit on server töötanud?**
```promql
time() - node_boot_time_seconds
```

**Mitu päeva?**
```promql
(time() - node_boot_time_seconds) / 86400
```

**CPU core'ide kasutus individuaalselt:**
```promql
100 - (rate(node_cpu_seconds_total{mode="idle"}[5m]) * 100)
```

**Top 3 ketta IO'd:**
```promql
topk(3, rate(node_disk_io_time_seconds_total[5m]))
```

---

### Keerukamad Ülesanded

**Ülesanne 1:** Kirjuta päring, mis näitab ainult CPU core'e, kus kasutus on üle 50%

**Vihje:**
```promql
(100 - (rate(node_cpu_seconds_total{mode="idle"}[5m]) * 100)) > 50
```

**Ülesanne 2:** Leia network interface, kus on traffic aga mitte kõige rohkem

**Vihje:** Kasuta `topk()` ja `bottomk()`

**Ülesanne 3:** Arvuta keskmine vastuse aeg viimase tunni kohta

**Vihje:** Kasuta subquery `[1h:5m]`

---

## Lisaülesanne 4: Mitmik-Target Setup

### Eesmärk

Simuleerid multi-server setup'i (mitme serveri jälgimine).

---

### Käivita Mitu Node Exporter'it

**Käivita 2 lisaks Node Exporter'it:**

```bash
docker run -d --name nodeexporter2 \
  -p 9101:9100 \
  prom/node-exporter

docker run -d --name nodeexporter3 \
  -p 9102:9100 \
  prom/node-exporter
```

**Kontrolli:**
```bash
docker ps | grep nodeexporter
curl http://localhost:9101/metrics | head
curl http://localhost:9102/metrics | head
```

---

### Lisa Prometheus'e

**Uuenda `config/prometheus.yml`:**

```yaml
  - job_name: 'node-cluster'
    static_configs:
      - targets:
        - '192.168.1.150:9100'
        - '192.168.1.150:9101'
        - '192.168.1.150:9102'
        labels:
          cluster: 'local'
          environment: 'test'
```

**Selgitus:**

`labels` lisab ühise label'i kõigile target'idele selles grupis.

**Lae uuesti:**
```bash
curl -X POST http://localhost:9090/-/reload
```

---

### Ülesanded

**1. Näita kõigi node'ide CPU kasutust:**
```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**2. Leia node kõige kõrgema CPU kasutusega:**
```promql
topk(1, 
  100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
)
```

**3. Arvuta keskmine mälu kasutus kõigi node'ide peale:**
```promql
avg(
  (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
)
```

**4. Mitu node'i on cluster'is:**
```promql
count(up{job="node-cluster"})
```

**5. Mitu node'i on UP:**
```promql
count(up{job="node-cluster"} == 1)
```

---

## Challenge: Mini-Projekt

### Eesmärk

Loo täielik monitoring setup, mis demonstreerib kõiki õpitud oskusi.

---

### Projekt: 3 Teenust + Monitoring

**1. Teenused:**
- Python HTTP server (port 5000-5001)
- Flask app Dockeris (port 5002)
- 3x Node Exporter (port 9100-9102)

**2. Cron Job:**
- Ekspordib failistatistikat iga minut
- Textfile Collector Node Exporter'i kaudu

**3. Prometheus:**
- Jälgib kõiki teenuseid
- Konfigureeritud job'idega

**4. PromQL Päringud:**

Kirjuta vähemalt 5 päringut, mis näitavad:
- Teenuste health (UP/DOWN)
- CPU kasutus per node
- Päringute määr per rakendus
- Mälu kasutus trend
- Failide arvu muutus

**5. Dokumentatsioon:**

Kirjuta README.md:
```markdown
# Minu Prometheus Setup

## Arhitektuur
[kirjelda komponendid]

## Paigaldamine
[sammud]

## PromQL Päringud
[5 päringut koos selgitustega]

## Screenshots
[lisa pildid Prometheus UI-st]
```

---

## Lisalugemist

**Prometheus:**
- [Prometheus Best Practices](https://prometheus.io/docs/practices/)
- [Instrumentation Guidelines](https://prometheus.io/docs/practices/instrumentation/)

**PromQL:**
- [PromQL Functions](https://prometheus.io/docs/prometheus/latest/querying/functions/)
- [PromQL Examples](https://logz.io/blog/promql-examples-introduction/)

**Node Exporter:**
- [Node Exporter Guide](https://prometheus.io/docs/guides/node-exporter/)
- [Enabled Collectors](https://github.com/prometheus/node_exporter#enabled-by-default)

**Python Client:**
- [Prometheus Python Client](https://github.com/prometheus/client_python)
- [Flask Integration](https://github.com/rycus86/prometheus_flask_exporter)