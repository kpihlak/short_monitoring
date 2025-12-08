# Grafana - Monitooringu Visualiseerimise Platvorm

---

## Õpiväljundid

Pärast selle loengu läbimist oskad:

1. **Selgitada**, mis on Grafana ja milleks seda kasutatakse
2. **Kirjeldada** Grafana põhikomponente (dashboard, panel, data source, query)
3. **Mõista**, kuidas Grafana töötab ja kuidas andmed liiguvad süsteemis
4. **Tuvastada** Grafana põhifunktsioone (variables, thresholds, alerts)
5. **Eristada** LGTM stack komponente ja nende rolli
6. **Valida** sobivat andmeallikat vastavalt kasutusalale
7. **Hinnata** Grafana tugevusi ja piiranguid erinevates kontekstides

---

## 1. Mis on Grafana?

Grafana on visualiseerimise tööriist, mis muudab andmed graafikuteks ja dashboardideks. Kujuta ette, et sul on palju numbreid ja andmeid erinevatest süsteemidest – Grafana aitab sul need ühte kohta kokku tuua ja muuta visuaalselt arusaadavaks. See toimib nagu "juhtpaneel", kust näed kogu olulise info ühel pilgul.

### Põhiprintsiip

```
Andmeallikas (Prometheus, MySQL, InfluxDB)
          ↓
      Grafana (visualiseerimine)
          ↓
      Dashboard (graafikud brauseris)
```

**Oluline:** Grafana **EI KOGU** andmeid - see ainult visualiseerib neid.

### Milleks seda kasutatakse?

- **Serverite monitoorimine** - CPU, mälu, ketas
- **Rakenduste jälgimine** - päringute arv, vastuse aeg, vead
- **Äriandmed** - müük, kasutajad, konversioon
- **IoT seadmed** - temperatuur, niiskus, energiatarbimine

### Miks populaarne?

✓ Tasuta ja avatud lähtekoodiga  
✓ Toetab sadu erinevaid andmeallikaid  
✓ Visuaalselt atraktiivsed dashboardid  
✓ Lihtne kasutada  
✓ Suur kogukond

---

## 2. Põhikomponendid

Grafana koosneb mitmest põhikomponendist, mida on oluline mõista. Dashboard on peamine koht, kus sa näed oma andmeid, ja see koosneb väiksematest osadest nagu panelid ja päringud. Järgnevalt vaatame, millised need komponendid on ja kuidas nad omavahel seotud.

### Dashboard
- Lehekülg, kus on graafikud ja mõõdikud
- Koosneb panelidest
- Näide: "Serverite monitooring" dashboard

**Tüüpiline dashboard:**

![Dashboard Example](https://grafana.com/static/assets/img/blog/kubernetes_nginx_dash.png?w=1504)

### Panel
- Üks graafik või mõõdik dashboardil
- Näide: CPU kasutuse joongraafik

### Data Source (andmeallikas)
- Koht, kust andmed tulevad
- Näited: Prometheus, MySQL, InfluxDB, Elasticsearch

### Query (päring)
- Küsimus andmeallikale
- Näide: "Anna mulle serveri CPU andmed"

### Visualiseerimise tüübid

### Visualiseerimise tüübid

**Time Series (joongraafik)**
```
100% ┤     ╭╮
 80% ┤   ╭╯╰╮
 60% ┤ ╭╯   ╰╮
 40% ┤╭╯     ╰╮
  0% ┼─────────
     0  5  10 15 min
```
Kasutus: CPU kasutus aja jooksul

**Gauge (poolring)**
```
    ╱───────╲
   ╱    75%  ╲
  ╱           ╲
  ────────────
  0    50   100
```
Kasutus: Ketta täituvus protsendina

**Stat (suur number)**
```
┌────────────┐
│            │
│    1,234   │ ← suur number
│            │
└────────────┘
```
Kasutus: Aktiivsete kasutajate arv

**Table (tabel)**
```
┌────────┬────────┬────────┐
│ Server │ CPU    │ Status │
├────────┼────────┼────────┤
│ srv-01 │ 45%    │ OK     │
│ srv-02 │ 78%    │ Warn   │
│ srv-03 │ 12%    │ OK     │
└────────┴────────┴────────┘
```
Kasutus: Serverite nimekiri

![Grafana visualizations](https://grafana.com/media/blog/explore-profiles/explore-profiles-flame-graph.png?w=1920)

---

## 3. Kuidas Grafana töötab?

Grafana töö põhineb lihtsale printsiibile: andmeallikas annab andmeid, Grafana visualiseerib need ja kasutaja näeb tulemust brauseris. See protsess toimub automaatselt ja reaalajas. Vaatame, kuidas see samm-sammult käib.

### Lihtne skeem

```
┌─────────────────┐
│   ANDMEALLIKAS  │ (Prometheus, MySQL, InfluxDB)
│   CPU, mälu,    │
│   logid, jne    │
└────────┬────────┘
         │
         ↓ päring
┌─────────────────┐
│     GRAFANA     │
│  Visualiseerimine│
└────────┬────────┘
         │
         ↓ graafik
┌─────────────────┐
│   BRAUSER       │
│   Dashboard     │
└─────────────────┘
```

### Sammud

**1. Ühenda andmeallikas**
```
Configuration → Data Sources → Add data source
Vali: Prometheus (või muu)
Sisesta: URL ja autentimine
Test: "Data source is working" ✓
```

**2. Loo dashboard**
```
+ → New Dashboard
```

**3. Lisa panel**
```
Add panel → Vali visualiseerimise tüüp
```

**4. Tee päring**
```
Vali data source
Kirjuta päring (nt SQL või PromQL)
Run query → näed graafikut
```

**5. Salvesta**
```
Save dashboard
```

### Andmevoog

```
1. Kasutaja teeb päringu Grafanas
   "Näita serveri CPU kasutust"
           ↓
2. Grafana saadab päringu andmeallikale
   SELECT cpu_usage FROM prometheus
           ↓
3. Andmeallikas tagastab andmed
   [45%, 50%, 48%, 52%, ...]
           ↓
4. Grafana visualiseerib need
   Joongraafik
           ↓
5. Kasutaja näeb graafikut brauseris
   📈
```

**Näide dashboard:**

![Grafana Dashboard](https://grafana.com/static/img/docs/grafana-cloud/arch_diagrams/hostedonly.jpg)

---

## 4. Olulised funktsioonid

Grafana pakub mitmeid võimsaid funktsioone, mis muudavad monitooringu lihtsamaks ja efektiivsemaks. Variables võimaldavad dashboarde dünaamiliseks muuta, thresholds aitavad probleeme kiiresti märgata, ja alerts teavitavad sind automaatselt, kui midagi läheb valesti. Need funktsioonid on põhitööriistad, mida igapäevaselt kasutatakse.

### Variables (muutujad)
- Muudab dashboardi dünaamiliseks
- Näide: vali server dropdown menüüst → kõik graafikud uuenevad

```
┌──────────────────────────────────┐
│ Server: [server1 ▼] [7 päeva ▼] │ ← Variables
├──────────────────────────────────┤
│  ┌────────┐  ┌────────┐          │
│  │CPU 45% │  │RAM 60% │          │ ← Panelid
│  └────────┘  └────────┘          │
└──────────────────────────────────┘
```

### Thresholds (piirid)
Määrab, millal värv muutub

```
CPU Kasutus:
  0-70%  ████████████ (roheline = OK)
 70-90%  ████████████ (kollane = hoiatus)
90-100%  ████████████ (punane = kriitiline!)
```

**Näide Gauge paneelis:**

```
    ╱───────╲
   ╱    78%  ╲  ← Kollane (hoiatus)
  ╱───────────╲
  0    50   100
```

### Alerts (hoiatused)
Automaatsed teavitused

```
Tingimus: CPU > 80%
         ↓
    Kontrolli iga 1 min
         ↓
    Kui > 80% 5 min järjest
         ↓
    Saada EMAIL / SLACK
```

**Alert näide:**
```
┌──────────────────────────────┐
│ 🔴 ALERT: High CPU Usage     │
│                               │
│ Server: srv-02                │
│ CPU: 92%                      │
│ Threshold: 80%                │
│ Duration: 7 minutes           │
└──────────────────────────────┘
```

### Time Range
Vali, millist ajavahemikku vaadata

```
[Last 15 min ▼] [Last 1 hour] [Last 24 hours] [Last 7 days]
```

### Auto-refresh
Dashboard uueneb automaatselt

```
⟳ Refresh every: [5s ▼] [10s] [30s] [1m]
```

---

## 5. Grafana LGTM Stack

LGTM Stack on Grafana Labs'i poolt välja töötatud täielik monitooringu lahendus. See katab kõik kolm observability põhisammast: logid, mõõdikud ja traces. Kui sul on vaja terviklikku lahendust, mis jälgib kõike ühest kohast, siis LGTM stack on selleks ideaalne valik.

**LGTM** = täielik monitooringu lahendus

```
┌──────────────────────────────────────┐
│         RAKENDUS / SÜSTEEM           │
└────────┬──────────┬─────────┬────────┘
         │          │         │
         ↓          ↓         ↓
    ┌────────┐ ┌────────┐ ┌────────┐
    │  LOKI  │ │ MIMIR  │ │ TEMPO  │
    │ (Logid)│ │(Metrics)│ │(Traces)│
    └────┬───┘ └────┬───┘ └────┬───┘
         │          │          │
         └──────────┼──────────┘
                    ↓
            ┌──────────────┐
            │   GRAFANA    │
            │(Visualiseer.)│
            └──────────────┘
```

| Komponent | Eesmärk |
|-----------|---------|
| **L**oki | Logide kogumine ja salvestamine |
| **G**rafana | Visualiseerimine (dashboardid) |
| **T**empo | Distributed tracing (päringute jälgimine) |
| **M**imir | Mõõdikute pikaajaline salvestus |

![LGTM Stack](https://grafana.com/media/about/stack/grafana-stack-diagram@3x.png)

**3 observability sammast:**
1. **Logs** (logid) - mis juhtus?
2. **Metrics** (mõõdikud) - kui palju?
3. **Traces** (jäljed) - kuidas voog käis?

Grafana ühendab kõik kolm üheks süsteemiks.

---

## 6. Populaarsed andmeallikad

Grafana tugevus seisneb selles, et ta suudab ühenduda väga paljude erinevate andmeallikatega. Olenevalt sellest, milliseid andmeid sa kogud ja kust, on sul erinevaid valikuid. Siin on kõige populaarsemad andmeallikad, mida Grafanaga kasutatakse.

### Prometheus
- Mõõdikute kogumine serveritelt
- Kasutatakse kõige sagedamini
- PromQL päringukeel

### MySQL/PostgreSQL
- SQL andmebaasid
- Äriandmed, kasutajad, tehingud

### InfluxDB
- Time-series andmebaas
- IoT seadmed, sensorid

### Elasticsearch
- Logide salvestamine ja otsing
- ELK stack (Elasticsearch + Logstash + Kibana)

### Cloud teenused
- AWS CloudWatch
- Azure Monitor
- Google Cloud Monitoring

---

## 7. Versioonid

Grafana on saadaval mitmes erinevas versioonis, sõltuvalt sellest, kas sa tahad seda ise hallata või kasutada pilvelahendust. Kõige lihtsam viis alustamiseks on Grafana Cloud, mis ei vaja mingit paigaldamist. Kui sa vajad rohkem kontrolli või erikonfigureerimist, võid valida OSS või Enterprise versiooni.

| Versioon | Kirjeldus | Hind |
|----------|-----------|------|
| **Grafana OSS** | Avatud lähtekoodiga, paigalda ise | Tasuta |
| **Grafana Cloud** | Hallatud pilvelahendus | Tasuta kuni 10k metrics<br/>Tasuline alates $8/kuu |
| **Grafana Enterprise** | Lisafunktsioonidega ettevõtetele | Lepinguline |

**Paigaldamine:**
- Cloud: registreeri ja kasuta kohe (ei vaja credit kaarti)
- Docker: `docker run -d -p 3000:3000 grafana/grafana`
- Linux: apt/yum paketid

---

## 8. Kasutusalad

Grafanat kasutatakse väga erinevates valdkondades ja stsenaariumites. Kõige levinum on infrastruktuuri ja rakenduste monitoorimine, kuid sama hästi sobib see ka äriandmete visualiseerimiseks või IoT seadmete jälgimiseks. Vaatame, kuidas Grafanat erinevates valdkondades kasutatakse.

### Infrastruktuuri monitoorimine
**Mida jälgitakse:**
- Serverid: CPU, mälu, ketas, võrk
- Konteinerid: Docker, Kubernetes
- Võrguseadmed: routerid, switchid

```
┌────────────┐  ┌────────────┐  ┌────────────┐
│  Server 1  │  │  Server 2  │  │  Server 3  │
│ CPU: 45%   │  │ CPU: 78%   │  │ CPU: 23%   │
│ RAM: 60%   │  │ RAM: 85%   │  │ RAM: 45%   │
└──────┬─────┘  └──────┬─────┘  └──────┬─────┘
       │                │                │
       └────────────────┼────────────────┘
                        ↓
                ┌──────────────┐
                │ Node Exporter│
                └──────┬───────┘
                       ↓
                ┌──────────────┐
                │  Prometheus  │
                └──────┬───────┘
                       ↓
                ┌──────────────┐
                │   GRAFANA    │
                └──────────────┘
```

![Grafana Dashboard Example](https://grafana.com/static/assets/img/blog/appdynamics_enterprise_plugin_dash.png?w=1800)

### Rakenduse monitoorimine (APM)
**Mida jälgitakse:**
- Päringute arv
- Vastuse aeg
- Vigade arv
- Database päringud

```
Kasutaja → Veebileht → API → Andmebaas
   ↓          ↓        ↓         ↓
   └──────────┴────────┴─────────┘
              ↓
      ┌──────────────┐
      │  Prometheus  │
      │  + Tempo     │
      │  + Loki      │
      └──────┬───────┘
             ↓
      ┌──────────────┐
      │   GRAFANA    │
      └──────────────┘
```

### Ärianalüütika
**Mida jälgitakse:**
- Müüginumbrid
- Kasutajate arv
- Konversioon
- Revenue

```
SQL Andmebaas → Grafana → Dashboard
(müük, kasutajad)           ↓
                      CEO/Manager
```

### IoT ja tööstus
**Mida jälgitakse:**
- Temperatuur
- Niiskus
- Energiatarbimine
- Seadme olek

```
Sensorid → MQTT → InfluxDB → Grafana
(temp, niiskus)                ↓
                          Dashboard
```

---

## 9. Tugevused ja piirangud

Nagu iga tööriista puhul, on ka Grafanal oma tugevused ja nõrkused. Enne kui otsustad Grafanat kasutada, on oluline mõista, mida ta hästi teeb ja mis on tema piirangud. See aitab sul teha õigeid otsuseid oma monitooringu arhitektuuri planeerimisel.

### ✓ Tugevused
- Visuaalselt atraktiivsed dashboardid
- Toetab paljusid andmeallikaid
- Tasuta ja avatud lähtekoodiga
- Lihtne alustada
- Suur kogukond (valmis dashboardid, abi)

### ✗ Piirangud
- Ei kogu andmeid ise (vajab Prometheus, InfluxDB vms)
- Rohkem komponente, mida hallata
- Ei ole "all-in-one" lahendus

---

## 10. Kokkuvõte

Nüüd, kui oled läbi lugenud kogu materjali, on aeg kokku võtta kõige olulisem. Grafana on võimas visualiseerimise tööriist, mis aitab sul jälgida süsteeme ja andmeid reaalajas. Järgmised punktid annavad sulle kiire ülevaate sellest, mida sa peaksid meelde jätma ja kuidas edasi minna.

**Grafana on:**
- Visualiseerimise tööriist (mitte andmete koguja)
- Dashboard platvorm
- Ühendab erinevaid andmeallikaid

**Mida pead teadma:**
- Dashboard = lehekülg graafikutega
- Panel = üks graafik
- Data source = andmete allikas
- Query = päring andmetele

**Praktikas:**
- Ühenda andmeallikas
- Loo dashboard
- Lisa panelid
- Tee päringuid
- Visualiseeri

**Järgmine samm:**
→ Mine laboritesse ja tee hands-on harjutused! 🚀
