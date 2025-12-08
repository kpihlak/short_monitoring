# Grafana Lab

## Eeldused

**Kohustuslik:**
- **Docker (Community Edition)** - konteinerite jooksutamiseks
- **docker compose CLI** - mitme konteineri haldamiseks käsurealt

### Kontrolli, kas on paigaldatud:

```bash
# Kontrolli Docker
docker --version
# Peaks väljastama: Docker version 20.x.x või uuem

# Kontrolli Docker Compose (uus süntaks)
docker compose version
# Peaks väljastama: Docker Compose version v2.x.x

# VÕI vana süntaks (vanemad versioonid)
docker-compose --version
```

### Paigaldamine (kui puudub)

**Linux (Ubuntu/Debian):**
```bash
# Docker paigaldamine
sudo apt-get update
sudo apt-get install docker.io

# Docker Compose paigaldamine (kui pole kaasas)
sudo apt-get install docker-compose-plugin

# VÕI vana versioon
sudo apt-get install docker-compose
```

**Muud platvormid:**
- Ametlik juhend: https://docs.docker.com/compose/install/

**🔑 Süntaksi erinevus:**
- **Uued versioonid (v2+):** `docker compose` (ilma sidekriipsuta)
- **Vanad versioonid (v1):** `docker-compose` (sidekriipsuga)

**Selles juhendis kasutame mõlemat** - vali oma versiooni järgi!

---

## 🎯 Õpiväljundid

Pärast selle labi läbimist õpilane:

**Teadmised (Knowledge):**
1. Selgitab Grafana arhitektuuri (data source → dashboard → panel)
2. Kirjeldab PromQL põhifunktsioone (`rate()`, `sum()`, `offset`)
3. Eristab visualization tüüpe (Time Series, Gauge, Pie Chart)

**Oskused (Skills):**
4. Seadistab Prometheus data source'i Grafanas
5. Loob dashboard'e koos paneelide ja päringutega
6. Kasutab variable'eid dünaamiliste dashboardide loomiseks
7. Rakendab threshold'e ja värve andmete visualiseerimiseks
8. Võrdleb andmeid ajas kasutades time shift'i (`offset`)

**Pädevused (Competencies):**
9. Analüüsib süsteemi metrics'eid läbi visualiseerimise
10. Tuvastab probleeme threshold'ide ja värvikoodide abil
11. Loob professionaalseid monitooringu dashboarde

**Hindamine:** Valmis dashboard kõigi nõutud paneelide ja funktsioonidega.

---

## 📦 Mis me täna ehitame?

**3 konteinerit Dockeris:**

1. **SHOEHUB** (port 8080) = valmis DEMO rakendus
   - Python app Docker image'is
   - Genereerib automaatselt fake jalanõude müügiandmeid
   - Ei pea koodi kirjutama - lihtsalt käivitame!

2. **PROMETHEUS** (port 9090) = metrics koguja
   - Küsib shoehub'ilt andmeid iga 15s
   - Salvestab ajaloolised andmed (time-series DB)

3. **GRAFANA** (port 3000) = visualiseerija
   - Loeb Prometheusest andmeid
   - Ehitame dashboarde ja graafikuid

**Tulemus:** Dashboard, kus näed "live" müügiandmeid graafikutena! 📊

---

## Ettevalmistus (5 min)

**💻 Töövahekonnad:** Kõik käsud käivitatakse terminalis/CLI-s, MITTE Docker Desktop GUI's!

### Loo töökaust
```bash
mkdir grafana-lab
cd grafana-lab
```

**💡 Selgitus:** Loome eraldi kausta, kus hoiame kõiki konfiguratsioone. Nii on lihtsam hallata ja hiljem kustutada.

### Loo failid

**Loo fail nimega `docker-compose.yml`** (tekstiredaktoriga: nano, vim, VSCode jne)

**docker-compose.yml**
```yaml
version: '3'

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'

  shoehub:
    image: aussiearef/shoehub:latest
    ports:
      - "8080:8080"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  grafana-data:
```

**💡 Selgitus:** 
- **Shoehub** - see on VALMIS demo rakendus (Docker image), mis jookseb kui konteiner ja GENEREERIB automaatselt fake müügiandmeid. See on simuleeritud e-pood!
- **Prometheus** - kogub mõõdikuid (metrics) shoehub rakendusest
- **Grafana** - visualiseerib Prometheuse andmed dashboardideks
- **Volume** - säilitab Grafana seadistused ka peale restart'i

**docker-compose.yml** kirjeldab, kuidas konteinerid omavahel suhtlevad (network).

**🔄 KUST ANDMED TULEVAD?**

```
┌─────────────────────────────────────────┐
│  SHOEHUB KONTEINER                      │
│  (aussiearef/shoehub Docker image)      │
│                                         │
│  Python rakendus sees, mis:             │
│  • Genereerib FAKE müügiandmeid         │
│  • Loob juhuslikke numbreid             │
│  • Eksponeerib /metrics endpoint        │
│                                         │
│  Andmed:                                │
│  - shoehub_sales (müük)                 │
│  - shoehub_payments (maksed)            │
│  - Random riigid: AU, US, IN            │
└─────────────────────────────────────────┘
              ↓ (port 8080)
┌─────────────────────────────────────────┐
│  PROMETHEUS                             │
│  Küsib iga 15s: "shoehub, mis toimub?" │
│  Salvestab time-series DB'sse           │
└─────────────────────────────────────────┘
              ↓ (port 9090)
┌─────────────────────────────────────────┐
│  GRAFANA                                │
│  Küsib Prometheuselt ja visualiseerib  │
└─────────────────────────────────────────┘
```

**⚠️ OLULINE:** 
- Shoehub on **valmis Docker image** (sa ei pea koodi kirjutama!)
- See **jookseb automaatselt** kui käivitad `docker-compose up`
- Andmed on **random/fake** - iga kord erinevad numbrid
- Reaalsüsteemis shoehub asemel oleks PÄRIS rakendus (nt sinu veebipood, server, API)

**Loo teine fail nimega `prometheus.yml`** (samasse `grafana-lab` kausta)

**prometheus.yml**
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'shoehub'
    static_configs:
      - targets: ['shoehub:8080']
```

**💡 Selgitus:**
- **scrape_interval: 15s** - Prometheus küsib andmeid iga 15 sekundi tagant
- **job_name: 'shoehub'** - anname töö nimeks "shoehub"
- **targets: ['shoehub:8080']** - kust Prometheus andmeid küsib (shoehub konteiner, port 8080)

⚠️ **Tähtis:** Dockeri network'is kasutatakse konteineri nime, mitte `localhost`!

**📊 Praktiline näide:**
1. Shoehub genereerib andmeid → näiteks müüdi 5 Loafers'it
2. Prometheus küsib iga 15s: "Shoehub, mis on praegu?"
3. Shoehub vastab: "shoehub_sales{ShoeType='Loafers'} = 5"
4. Prometheus salvestab: `timestamp: 2025-10-30 18:00:00, value: 5`
5. Grafana küsib: "Prometheus, näita mulle Loafers müüki"
6. Prometheus saadab andmed → Grafana joonistab graafiku

**Kontrolli ise!** Käivita konteinerid ja mine: http://localhost:8080/metrics
Näed RAW andmeid, mida Prometheus kogub!

### Käivita

**⚠️ Kõik käsud töötavad CLI-s (terminal/shell), MITTE Docker Desktop GUI-s!**

```bash
# Uus süntaks (Docker Compose v2+)
docker compose up -d
docker compose ps

# VÕI vana süntaks (Docker Compose v1)
docker-compose up -d
docker-compose ps
```

**💡 Selgitus:**
- `up -d` - käivitab kõik konteinerid taustal (detached mode)
- `ps` - näitab, kas konteinerid töötavad (peaks olema 3 konteinerit UP state'is)

**Oodatud väljund:**
```
NAME                STATUS
grafana-lab-grafana-1      Up
grafana-lab-prometheus-1   Up
grafana-lab-shoehub-1      Up
```

**Ava:**
- Grafana: http://localhost:3000 (admin/admin)
- Prometheus: http://localhost:9090

**Esimesel sisselogimisel** Grafana küsib uut parooli - võid jätta samaks (admin/admin).

**🔍 Kontrolli, kas shoehub toodab andmeid:**

Ava brauser: http://localhost:8080/metrics

Peaksid nägema midagi sellist:
```
# HELP shoehub_sales Total shoe sales
# TYPE shoehub_sales counter
shoehub_sales{ShoeType="Loafers",CountryCode="AU"} 145.0
shoehub_sales{ShoeType="Boots",CountryCode="US"} 89.0
shoehub_sales{ShoeType="HighHeels",CountryCode="IN"} 203.0

# HELP shoehub_payments Total payments
# TYPE shoehub_payments counter
shoehub_payments{PaymentMethod="Card",CountryCode="AU"} 67.0
shoehub_payments{PaymentMethod="Paypal",CountryCode="US"} 12.0
```

**Kui näed seda → shoehub TÖÖTAB ja genereerib andmeid!** 🎉

Refresh lehte mitu korda - numbrid muutuvad (kasvavad).

---

## Osa 1: Data Source (5 min)

1. **Grafana** → ☰ → **Connections** → **Data Sources**
2. **Add data source** → **Prometheus**
3. URL: `http://prometheus:9090`
4. **Save & Test** → ✅ "Data source is working"

**💡 Selgitus:**
**Data Source** on Grafana ühendus andmeallikaga. Grafana EI KOGU ise andmeid - ta ainult visualiseerib! Prometheus kogub, Grafana näitab.

**Miks `http://prometheus:9090`?** 
Docker network'is konteinerid räägivad omavahel konteineri nime järgi, mitte `localhost` järgi.

**✅ Õpiväljund #1, #4:** Mõistad Grafana arhitektuuri ja oskad seadistada data source'i.

---

## Osa 2: Esimene Dashboard (10 min)

### Loo Dashboard

1. ☰ → **Dashboards** → **New Dashboard**
2. **Add visualization**
3. Vali **Prometheus**

**💡 Selgitus:**
- **Dashboard** = konteiner, kus on mitu paneeli (graafikut)
- **Panel** = üks graafik/tabel/gauge
- **Visualization** = kuidas andmeid näidatakse (joongraafik, sektordiagramm jne)

### Panel 1: Loafers Müük

**Query:**
```promql
rate(shoehub_sales{ShoeType="Loafers"}[1m])
```

**💡 Selgitus:**
- `shoehub_sales` = metric (mõõdik), mida Prometheus kogub
- `{ShoeType="Loafers"}` = filter, näita ainult Loafers'i
- `[1m]` = vaata viimast 1 minutit
- `rate()` = arvuta muutuse kiirus sekundis (kui kiiresti kasvab)

**MIKS rate()?** Ilma selleta näeksime lihtsalt kasvavat numbrit (total). `rate()` näitab müügikiirust - mitu paari minutis müüakse.

**Seaded:**
- Panel title: `Loafers müük`
- Legend: `Loafers`

**Save** → nimetä Dashboard: `Shoe Sales`

### Lisa veel panelid (Add → Visualization)

**Panel 2: Boots**
```promql
rate(shoehub_sales{ShoeType="Boots"}[1m])
```

**Panel 3: High Heels**
```promql
rate(shoehub_sales{ShoeType="HighHeels"}[1m])
```

**Panel 4: Kokku**
```promql
sum(rate(shoehub_sales[1m]))
```

**✅ Kontroll:** 4 time series graafikut!

**✅ Õpiväljund #2, #3, #5:** Oskad kirjutada PromQL päringuid ja luua dashboarde koos visualiseerimistega.

---

## Osa 3: Variables (10 min)

**💡 Mis on Variable?**
Variable teeb dashboardi dünaamiliseks! Selle asemel, et teha 5 paneeli iga riigi kohta, teed ühe paneeli ja variable valib riigi.

**Näide:** `$country` võib olla AU, US, IN - kasutaja valib dropdown'ist.

### Loo Variable

1. **Dashboard Settings** ⚙️ → **Variables** → **New variable**
2. **Seaded:**
   - Name: `country`
   - Label: `Riik`
   - Type: **Query**
   - Data source: **Prometheus**
   - Query: `label_values(shoehub_payments, CountryCode)`
   - Multi-value: ✅
   - Include All option: ✅

**💡 Selgitus:**
- **Name:** kuidas viitad variable'ile (`$country`)
- **Label:** mis kuvatakse kasutajale (dropdown)
- **Query:** `label_values(shoehub_payments, CountryCode)` - leia kõik erinevad `CountryCode` väärtused
- **Multi-value:** kasutaja saab valida mitu riiki korraga
- **Include All:** lisab "All" valiku

3. **Apply** → **Save**

### Kasuta Variable't

**Add → Visualization** (uus panel)

**Query:**
```promql
rate(shoehub_payments{CountryCode="$country", PaymentMethod="Card"}[1m])
```

**💡 Selgitus:**
`$country` asendatakse dropdown'ist valitud väärtusega! Kui valid "AU", siis päring muutub:
```promql
rate(shoehub_payments{CountryCode="AU", PaymentMethod="Card"}[1m])
```

**Panel title:**
```
Kaardimaksed - $country
```

**Panel Options → Repeat options:**
- Repeat by variable: `country`

**💡 Mis on Repeat?**
Repeat loob AUTOMAATSELT eraldi paneeli iga variable väärtuse jaoks!
- Kui valid 3 riiki → 3 paneeli
- Kui valid "All" → panel iga riigi kohta

See on võimas feature - säästad aega, ei pea käsitsi kopeerima!

**✅ Kontroll:** Iga riigi kohta eraldi panel!

**✅ Õpiväljund #6:** Oskad kasutada variable'eid dünaamiliste dashboardide loomiseks.

---

## Osa 4: Gauges ja Thresholds (10 min)

**💡 Mis on Gauge?**
Gauge näitab ÜHT numbrit visuaalselt. Nagu autol kiirusemõõdik - näitab praegust olekut, mitte ajalugu.

**Mis on Threshold?**
Threshold määrab VÄRVI reeglid:
- 0-50 = roheline (OK)
- 50-80 = kollane (tähelepanu!)
- 80+ = punane (probleem!)

### Panel 1: PayPal % USAs

1. **Add → Visualization**
2. **Visualization type:** Gauge

**Query:**
```promql
sum(shoehub_payments{CountryCode="US", PaymentMethod="Paypal"}) / sum(shoehub_payments{CountryCode="US"}) * 100
```

**💡 Selgitus:**
- `sum(...)` - liida kokku KÕIK PayPal maksed USAs
- Jaga kõigi maksete summaga USAs
- Korruta 100ga, et saada protsent

**Tulemus:** Mitu protsenti USA maksudest on PayPal?

**Seaded:**
- Title: `PayPal % USAs`
- Unit: **Percent (0-100)**
- Thresholds:
  - Base: Green
  - 5: Red

**💡 Selgitus:**
- **Percent (0-100):** näita kui "87%" mitte "0.87"
- **Base: Green** = vaikimisi roheline (0-5%)
- **5: Red** = kui üle 5%, muutu punaseks

**MIKS RED kui PayPal > 5%?** See on näide - võib-olla tahad hoiatada kui liiga palju kasutavad PayPal'i (kõrged tasud).

### Panel 2: Memory Gauge (Australias)

**Query:**
```promql
sum(shoehub_payments{CountryCode="AU", PaymentMethod="Paypal"}) / sum(shoehub_payments{CountryCode="AU"}) * 100
```

**Seaded:**
- Title: `PayPal % Australias`
- Thresholds: 0 (green), 10 (yellow), 20 (red)

**💡 3-värvi süsteem:**
- 0-10% = roheline (OK)
- 10-20% = kollane (jälgi!)
- 20%+ = punane (probleem!)

### Panel 3: Pie Chart - Jalanõude tüübid

**💡 Millal kasuta Pie Chart?**
Kui tahad näidata PROTSENTUAALSET JAOTUST - mitu % on Loafers, mitu Boots jne.

1. **Visualization type:** Pie Chart

**Queries (Add 3 query't):**
```promql
# A
sum(shoehub_sales{ShoeType="Loafers"})

# B
sum(shoehub_sales{ShoeType="Boots"})

# C
sum(shoehub_sales{ShoeType="HighHeels"})
```

**Legend:**
- A: `Loafers`
- B: `Boots`
- C: `HighHeels`

**✅ Kontroll:** Sektordiagramm 3 osaga!

**✅ Õpiväljund #3, #7, #10:** Oskad valida õiget visualization tüüpi ja rakendada threshold'e probleemide tuvastamiseks.

---

## Osa 5: Time Shift (5 min)

**💡 Mis on Time Shift?**
`offset` võimaldab vaadata MINEVIKKU! Võid võrrelda:
- Täna vs eile
- See nädal vs eelmine nädal
- See kuu vs eelmine kuu

See on VÕIMAS võrdlemiseks - näed trende ja muutusi!

### Võrdle Täna vs Nädal Tagasi

**Add → Visualization**

**Query A:** Täna
```promql
rate(shoehub_sales{ShoeType="Loafers"}[1m])
```
Legend: `Täna`

**Query B:** Nädal tagasi
```promql
rate(shoehub_sales{ShoeType="Loafers"}[1m] offset 7d)
```
Legend: `7 päeva tagasi`

**💡 Selgitus:**
- `offset 7d` = liigu 7 päeva tagasi ajas
- **Tulemus:** sama päring aga NÄDAL TAGASI andmetega

**Kasutusjuht:** "Kas see nädal on müük parem kui eelmisel nädalal?"

**Title:** `Loafers: Täna vs Nädal Tagasi`

**✅ Kontroll:** 2 joont graafikul!

**⚠️ Meie demo:** Kuna shoehub genereerib juhuslikke andmeid, ei näe suurt erinevust. Reaalsüsteemis see on VÄGA kasulik!

**✅ Õpiväljund #2, #8, #9:** Oskad võrrelda andmeid ajas ja analüüsida trende.

---

## Lõpptulemus

Dashboard peaks sisaldama:
- ✅ 4 Time Series paneeli (Loafers, Boots, HighHeels, Kokku)
- ✅ Variable "country" mis töötab
- ✅ Repeat panels riikide kaupa
- ✅ 3 Gauge paneeli threshold'idega
- ✅ 1 Pie Chart
- ✅ Time Shift võrdlus

---

## 🎓 Mida õppisid?

**Tehnilised oskused:**
1. **Docker Compose** - mitme konteineri haldamine
2. **Prometheus** - metrics'te kogumine
3. **Grafana** - visualiseerimine ja dashboardid
4. **PromQL** - päringute kirjutamine

**Grafana kontseptsioonid:**
1. **Data Source** - andmeallika ühendamine
2. **Dashboard** - paneelide konteiner
3. **Panel** - üksik visualiseering
4. **Variable** - dünaamiline filtreerimine
5. **Repeat** - automaatne paneelide loomine
6. **Threshold** - värvi reeglid hoiatusteks
7. **Time Shift** - mineviku võrdlemine

**Praktiline kasutus:**
- Saad jälgida süsteeme reaalajas
- Märkad probleeme ENNE kui kasutajad kurdavad
- Saad võrrelda trende
- Teed andmeid visuaalselt mõistetavaks

**Järgmine samm:** Lisa **Loki** ja jälgi logisid koos metrics'tega! 🚀

---

## ✅ Kontrolli oma oskusi (Self-Assessment)

**Teadmised:** Suudan selgitada...
- [ ] Kuidas Grafana, Prometheus ja shoehub omavahel suhtlevad
- [ ] Mida teeb `rate()` funktsioon ja miks seda kasutame
- [ ] Erinevust Time Series, Gauge ja Pie Chart vahel

**Oskused:** Suudan iseseisvalt...
- [ ] Ühendada uue Prometheus data source'i
- [ ] Luua uue dashboard'i koos paneelide ja päringutega
- [ ] Kirjutada PromQL päringuid koos filtritega
- [ ] Seadistada variable'eid ja repeat funktsiooni
- [ ] Määrata threshold'e ja värve Gauge'dele
- [ ] Võrrelda andmeid kasutades `offset`

**Pädevused:** Suudan rakendada...
- [ ] Loon professionaalse dashboard'i reaalseks monitooringuks
- [ ] Tuvastan probleeme värvikoodi põhjal (punane = probleem)
- [ ] Annan kolleegile ülevaate, kuidas dashboard töötab

**Kui kõik linnukesed peal → LÄBITUD! 🎓**

---

## 📝 Kiire Käskluste Referents

**Uus süntaks (Docker Compose v2+):**
```bash
docker compose up -d          # Käivita
docker compose ps             # Kontrolli olekut
docker compose logs <service> # Vaata loge
docker compose restart        # Taaskäivita
docker compose down           # Peata ja kustuta
docker compose down -v        # Kustuta koos volume'idega
```

**Vana süntaks (Docker Compose v1):**
```bash
docker-compose up -d          # Käivita
docker-compose ps             # Kontrolli olekut
docker-compose logs <service> # Vaata loge
docker-compose restart        # Taaskäivita
docker-compose down           # Peata ja kustuta
docker-compose down -v        # Kustuta koos volume'idega
```

**Mõlemad töötavad - vali oma versiooni järgi!**

---

## Puhastamine

**Uus süntaks (v2+):**
```bash
# Peata konteinerid
docker compose down

# Kustuta kõik (kaasa andmed ja volume'd)
docker compose down -v

# Restart
docker compose restart
```

**Vana süntaks (v1):**
```bash
# Peata konteinerid
docker-compose down

# Kustuta kõik (kaasa andmed ja volume'd)
docker-compose down -v

# Restart
docker-compose restart
```

---

## Abi

**Docker/Compose ei tööta?**
```bash
# Kontrolli, kas Docker töötab
docker ps
# Kui error "Cannot connect to Docker daemon" → Docker ei ole käivitatud

# Linux: käivita Docker
sudo systemctl start docker
sudo systemctl enable docker

# Kontrolli õigusi (Linux)
sudo usermod -aG docker $USER
# Logi välja ja uuesti sisse, et muudatused jõustuksid!

# Kui "docker compose" ei tööta, proovi vana sünaksit
docker-compose version
```

**Konteinerid ei käivitu?**
```bash
# Vaata logisid (uus süntaks)
docker compose logs grafana
docker compose logs prometheus
docker compose logs shoehub

# VÕI vana süntaks
docker-compose logs grafana
```

**Kust andmed tulevad?**
```bash
# Kontrolli, kas shoehub genereerib andmeid
curl http://localhost:8080/metrics

# Peaksid nägema:
# shoehub_sales{ShoeType="Loafers",CountryCode="AU"} 123.0
# shoehub_payments{PaymentMethod="Card",CountryCode="US"} 456.0
```

**Prometheus ei näita andmeid?**
- http://localhost:9090/targets
- Kontrolli kas `shoehub` on UP
- Kui DOWN → kontrolli kas shoehub konteiner töötab: `docker-compose ps`

**Grafana ei ühenda?**
- URL peab olema `http://prometheus:9090`
- MITTE `http://localhost:9090` (Docker networking!)

**Variable ei tööta?**
- Kontrolli Query: `label_values(shoehub_payments, CountryCode)`
- Apply + Save dashboard

---

## Järgmine kord: Loki

Lisame logide visualiseerimise! 🚀
