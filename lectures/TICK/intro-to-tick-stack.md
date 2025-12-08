# TICK Stack - Põhjalik ülevaade

## Mis on aegread?
![Aegread](https://www.influxdata.com/wp-content/uploads/time-series-graph-1-300x127.png)

Aegreadata mängivad olulist rolli selles, kuidas mõista asjade muutumist ajas. Olgu tegemist serveri metric'ute jälgimise või aktsiahindade jälgimisega, aeg on alati oluline tegur.

Aegread on andmepunktide jada, mis on kogutud samast allikast regulaarsete või ebaregulaarsete ajavahemike järel. Näiteks võib jälgida trenditeema kohta tehtud säutsude arvu ajas. Kui neid andmeid graafikule kanda, tähistab x-telg alati aega, muutes aja andmestiku oluliseks osaks.

### Aegreadata omadused:
- Miljardid individuaalsed andmepunktid
- Kõrge lugemis- ja kirjutamisvõimekus
- Suured kustutamised (andmete aegumine)
- Peamiselt lisamisega seotud töökoormus, väga vähe uuendusi

## Mis on TICK Stack?
![Aegridade kiirus](
https://images.ctfassets.net/o7xu9whrs0u9/gHXO09SuFNwCWrx6dREqV/9e815551a0ac27cda53f63a18f7d9fc1/influxdb-overview.png)

TICK Stack on avatud lähtekoodiga tarkvara kogum, mis on mõeldud ajaseeria andmete (time series data) töötlemiseks. "TICK" on lühend, mis tuleb neljast peamisest komponendist:

- **T**elegraf - andmete kogumise tööriist (Collection)
- **I**nfluxDB - andmebaas andmete salvestamiseks (Storage)
- **C**hronograf - visuaalne kasutajaliides (Visualization)
- **K**apacitor - andmete töötlemise ja häirete süsteem (Processing)

![TICK components](https://itcommunity.stanford.edu/sites/default/files/unconference/attachments/2019-11/tick_stack_components.png)

## TICK Stack komponendid tabelina

| Komponent | Funktsioon | Alternatiivid | Port |
|-----------|------------|---------------|------|
| **Telegraf** | Andmete kogumine | Prometheus exporters, Collectd, Beats | 8125 |
| **InfluxDB** | Andmete säilitamine | Prometheus, TimescaleDB, OpenTSDB | 8086 |
| **Chronograf** | Visualiseerimine | Grafana, Kibana, Datadog UI | 8888 |
| **Kapacitor** | Andmetöötlus ja häired | Prometheus Alertmanager, Nagios, ElastAlert | 9092 |

![TICK components 2](https://miro.medium.com/v2/resize:fit:720/format:webp/0*Ku-ohmJKRGswMqhH.png)

## Andmetüübid, mida saab jälgida

| Kategooria | Näited | Jälgimise kasu |
|------------|--------|----------------|
| **Süsteemi ressursid** | CPU, mälu, kettakasutus | Ressursside puuduse ennetamine |
| **Võrk** | Latentsus, läbilaskevõime, paketikadu | Võrguprobleemide tuvastamine |
| **Rakendused** | Vastusaeg, veateated, API päringud | Rakenduste jõudluse parandamine |
| **Andmebaasid** | Päringute arv, täitmisaeg, ühenduste arv | Andmebaasi optimeerimise võimaluste leidmine |
| **Kasutajate aktiivsus** | Sisselogimised, lehekülje vaatamised, ostud | Kasutajakogemuse parandamine |

## Lihtsustatud arhitektuur

```
┌────────────────────┐
│   ANDMEALLIKAD     │ ← Serverid, rakendused, seadmed, sensorid jne
└──────────┬─────────┘
           │
           ▼
┌────────────────────┐
│      TELEGRAF      │ ← Kogub andmeid erinevatest allikatest
└──────────┬─────────┘
           │
           ▼
┌────────────────────┐
│      INFLUXDB      │ ← Salvestab ajaseeria andmed
└────┬─────────┬─────┘
     │         │
     ▼         ▼
┌─────────┐ ┌────────────┐
│CHRONOGRAF│ │ KAPACITOR  │ ← Visualiseerimine ja häirete loomine
└─────────┘ └────────────┘
```

## Komponendid

### 📊 Telegraf - Andmete koguja

**Mis see on?** Telegraf on nagu "koristaja", kes käib ringi ja korjab andmeid kokku. See on lihtne, kuid võimas tööriist TICK Stack'is, mille peamine ülesanne on andmete kogumine.

**Mida see teeb?**
- Kogub erinevaid metric'uid sinu süsteemidest
- Suudab koguda andmeid paljudest erinevatest kohtadest (serverid, võrguseadmed, andmebaasid jne)
- Töötab väga efektiivselt ja vajab vähe ressursse

**Tehniline ülevaade:**
- Plugin'idel põhinev agent metric'ute kogumiseks ja edastamiseks
- Minimaalne mälujälg
- Üle 300 plugin'i paljude populaarsete teenuste jaoks
- Plugin'ide süsteem võimaldab hõlpsasti lisada uusi sisendeid ja väljundeid
- Võib töötada koos väliste skriptidega

**Kuidas see töötab:**
1. Paigaldad Telegraf oma serverisse või arvutisse
2. Määrad seadistustes, milliseid andmeid soovid koguda
3. Telegraf hakkab automaatselt andmeid koguma ja saatma InfluxDB-sse

**Näide elust:** Kujuta ette, et Telegraf on nagu elektriarvesti, mis pidevalt jälgib, kui palju elektrit sinu kodu erinevad seadmed tarbivad.

#### Telegraf'i plugin'id:
Telegraf'i tugevus tuleneb selle plugin'idest. Need on kahte tüüpi:

- **Sisendi plugin'id:** Need ütlevad Telegraf'ile, kust andmeid koguda (nt süsteemi metric'ud, andmebaasid, API-d)
- **Väljundi plugin'id:** Need ütlevad Telegraf'ile, kuhu andmeid pärast kogumist saata (nt InfluxDB-sse või teistesse teenustesse nagu Kafka)

#### Andmete voog Telegraf'is:
Protsess algab andmete kogumisega läbi sisend-plugin'ide (nt CPU, mälu, kettakasutus). Seejärel andmeid töödeldakse, kus saab rakendada transformatsioone või filtreerimist. Lõpuks saadab väljund-plugin kogutud ja töödeldud andmed määratud sihtkohta, näiteks InfluxDB instantsi.

### 💾 InfluxDB - Andmete säilitaja

**Mis see on?** InfluxDB on spetsiaalne andmebaas, mis on optimeeritud just ajaseeria andmete salvestamiseks. See on loodud käsitlema efektiivselt suuri ajatempliga andmete koguseid.

**Mida see teeb?**
- Salvestab suuri koguseid ajaseeria andmeid kiiresti ja efektiivselt
- Võimaldab neid andmeid hiljem pärida ja analüüsida
- Suudab töödelda miljoneid andmepunkte sekundis

**Tugevused:**
- Kiire sisestamine: võimaldab käsitleda miljoneid andmepunkte sekundis
- Kompaktne salvestus: kasutab tõhusaid salvestusmehhanisme, nagu tihendamine
- Säilitamispoliitikad: võimaldab konfigureerida poliitikaid, et automaatselt kustutada vanemaid andmeid

**Näide elust:** InfluxDB on nagu digitaalne päevik, mis salvestab automaatselt kõik sündmused koos täpse ajatempliga.

#### InfluxDB andmemudel:

InfluxDB andmete organiseerimise mõistmiseks on neli põhikomponenti:

- **Mõõtmised** (measurements) - andmete kategooriad (nt. CPU kasutus, temperatuur)
- **Väljad** (fields) - tegelikud väärtused (nt. CPU kasutus = 75%)
- **Sildid** (tags) - meta-andmed, mille järgi saab andmeid filtreerida (nt. server = "web-1")
- **Ajatempel** (timestamp) - millal andmed salvestati

#### Andmestruktuuri näide:
```
cpu,host=server01,region=us-west usage_user=20.5,usage_system=10.1 1632892800
```

Selles näites:
- Mõõtmine: cpu
- Sildid: host=server01, region=us-west
- Väljad: usage_user=20.5, usage_system=10.1
- Ajatempel: 1632892800 (Unix-i ajaformaat)

#### Päringud InfluxDB-st:
InfluxDB kasutab SQL-sarnast päringukeelt nimega InfluxQL. Näiteks:

```
SELECT * FROM cpu WHERE host = 'server01' AND time > now() - 1h;
```

See päring toob viimase tunni jooksul kõik serveri server01 CPU metric'ud.

#### Kus InfluxDB silma paistab:
- **IoT:** Suurte sensorandmete mahtude haldamisel Asjade Interneti rakendustes
- **Reaalajas jälgimine:** Süsteemi jälgimiseks, kus pidevad andmevood serveritest
- **DevOps ja rakenduste jõudluse jälgimine:** Rakenduste jõudluse metric'uid, serveri tervist jälgides
- **Finantsturud:** Reaalajas finantsanalüüsides, nagu aktsiahindade jälgimine

### 📈 Chronograf - Andmete visualiseerija

**Mis see on?** Chronograf on veebirakendus, mis võimaldab sul näha ja visualiseerida InfluxDB-s olevaid andmeid.

**Mida see teeb?**
- Loob graafikuid ja visuaalseid dashboarde
- Võimaldab andmeid uurida ilma keerulisi päringuid kirjutamata
- Aitab häireid ja teateid seadistada

**Võimalused:**
- Infrastruktuuri haldamine
- Hoiatuste haldamine: integreerib Kapacitoriga
- Andmete visualiseerimine: kasutage eelloodud dashboarde või looge oma dashboardi
- Andmebaasi haldamine: looge ja kustutage andmebaase ja kasutajaid
- Mitme organisatsiooni ja kasutaja tugi

**Miks see on kasulik:**
- Lihtne graafiline kasutajaliides
- Ei vaja programmeerimisoskusi
- Saad kiiresti ülevaate oma süsteemide seisundist

**Näide elust:** Chronograf on nagu ilmateade, mis näitab sulle graafiliselt, kuidas temperatuur, niiskus ja õhurõhk on päeva jooksul muutunud.

### ⚡ Kapacitor - Andmete töötleja ja häiresüsteem

**Mis see on?** Kapacitor on mootor, mis võimaldab andmeid töödelda ja nende põhjal häireid luua.

**Mida see teeb?**
- Analüüsib andmeid reaalajas
- Loob häireid, kui midagi olulist juhtub (nt. serveri mälu on otsas)
- Saab andmeid töödelda ja muuta

**Põhiomadused:**
- Töötleb nii voolavaid kui ka pakktöötluse andmeid
- Teeb päringuid InfluxDB-st vastavalt ajakavale
- Teostab transformatsioone, mis on võimalikud InfluxQL-is
- Salvestab transformeeritud andmeid tagasi InfluxDB-sse
- Võimaldab lisada kohandatud funktsioone anomaaliate tuvastamiseks
- Integreerib paljude teenustega (HipChat, OpsGenie, Slack jt)

**Näide elust:** Kapacitor on nagu valvesüsteem, mis jälgib pidevalt su kodu ja annab häiret, kui midagi kahtlast toimub.

**Milleks see kasulik on:**
- Automaatsed hoiatused, kui midagi läheb valesti
- Saad teada probleemidest enne, kui need suureks muutuvad
- Võimaldab automaatseid tegevusi (nt. e-maili saatmine või skripti käivitamine)

## Study Case

### Temperatuuri jälgimine
Sul on temperatuurisensorid, mis mõõdavad temperatuuri erinevates ruumides.

**Lahendus TICK Stackiga:**
1. Telegraf kogub sensorite andmed
2. InfluxDB salvestab temperatuurid koos ajatempliga
3. Chronograf näitab temperatuuri muutumist ajas igas ruumis
4. Kapacitor saadab häire, kui temperatuur langeb alla 18°C või tõuseb üle 28°C

## Miks peaks kasutama TICK Stacki?

### Plussid
- **Kerged binaarid (Go)** – lihtne paigaldada
- **Suurepärane meetrikate tugi** – veergude salvestus kõrge tihendusega
- **Salvestab loomulikult int, float, text ja bool andmetüüpe**
- **"SQL-inspired" päringukeel** on kasutajasõbralik
- **Andmetöötlus toimub reaalajas** "continuous queries" ja "subscriptions" kaudu

### Miinused
- **InfluxQL erinevused SQL-ist** ilmnevad kiiresti – ühe- ja kahekordsed jutumärgid on väga erinevad!
- **Kapacitor kasutab veel üht keelt** – "TICKScript". Plaanis on ühtlustada InfluxDB 2.0-s ("Flux")
- **Teavituste seadistamine on keeruline** ja mitte eriti nutikas
- **Veerupõhine andmebaas pole sobiv sündmuste logidele**. Ei saa omada kahte "rida" sama ajatempliga (kuigi toetab nanosekundilise täpsusega) ja tekstiotsing on kogu veeru jõuline läbivaatus
- **Ettearvamatu RAM-i kasutus** ja andmebaasi rikked (kuigi palju paranenud)
- **Horisontaalne skaleerimine** ainult kommertstoodetes

## Alustamine TICK Stack'iga

### Miinimumnõuded arvutile:
- **Protsessor:** Vähemalt 2 tuuma
- **Mälu:** Vähemalt 4GB RAM
- **Kõvaketas:** Vähemalt 10GB vaba ruumi
- **Operatsioonisüsteem:** Linux (Ubuntu 20.04 või uuem), macOS (10.15 või uuem)

### Soovituslikud nõuded suurema süsteemi jaoks:
- **Protsessor:** 4 või rohkem tuuma
- **Mälu:** 16GB või rohkem RAM
- **Kõvaketas:** SSD tüüpi, vähemalt 100GB
- **Operatsioonisüsteem:** Linux (Ubuntu 22.04 LTS)

## Turvalisus

TICK Stack on turvaline süsteem, kuid on oluline teada põhilisi turvafunktsioone:

- **SSL/TLS krüpteering** - See on nagu tähtsate dokumentide saatmine lukustatud kohvris
- **Kasutajaõiguste haldus (RBAC)** - See võimaldab määrata, kes mida teha saab
- **Token-põhine autentimine** - See on nagu spetsiaalne võti süsteemi sisenemiseks
- **Andmete krüpteerimine** - Tagab, et andmeid ei saa lugeda ilma õige võtmeta

## Probleemide lahendamine

Kui midagi ei tööta, siis esimene koht, kust otsida, on logifailid:

```bash
/var/log/telegraf/telegraf.log    - Telegraf'i logid
/var/log/influxdb/influxd.log     - InfluxDB logid
/var/log/chronograf/chronograf.log - Chronograf'i logid
/var/log/kapacitor/kapacitor.log  - Kapacitor'i logid
```

**Tüüpilised probleemid:**

1. **Telegraf ei kogu andmeid** - Kontrolli, kas õigused ja ühendused on korras
2. **InfluxDB ei käivitu** - Kontrolli, kas port 8086 on vaba ja kas sul on piisavalt kettaruumi
3. **Chronograf näitab tühja lehte** - Kontrolli, kas InfluxDB on käivitatud ja ühendus on seadistatud
4. **Häired ei tööta** - Kontrolli, kas Kapacitor on käivitatud ja õigesti seadistatud

## Komponentide asendamine

| Komponent | Populaarne alternatiiv | Eelised | Puudused |
|-----------|------------------------|---------|----------|
| Telegraf | Prometheus eksporterid | Parem integratsioon Kubernetes-ega | Rohkem seadistamist vaja |
| InfluxDB | Prometheus | Parem alerting, paremad Kubernetes integratsioonid | Mitte nii hea kirjutamise jõudlus |
| Chronograf | Grafana | Rohkem visualiseerimise võimalusi, rohkem andmeallikaid | Keerulisem seadistada |
| Kapacitor | Alertmanager | Lihtsam integratsioon Prometheus'ega | Vähem paindlik |

## Olulised mõisted monitoringus:

| Mõiste | Selgitus | Näide TICK Stack'is |
|--------|----------|---------------------|
| **Mõõtepunkt (Metric)** | Konkreetne mõõdetav väärtus | CPU kasutus = 75% |
| **Ajavahemik (Interval)** | Kui tihti andmeid kogutakse | Telegraf kogub andmeid iga 10 sekundi tagant |
| **Retensioon (Retention)** | Kui kaua andmeid säilitatakse | InfluxDB säilitab detailseid andmeid 30 päeva |
| **Häire (Alert)** | Teavitus, kui midagi olulist juhtub | Kapacitor saadab e-maili, kui server läheb maha |
| **Töölaud (Dashboard)** | Visuaalne andmete kogu | Chronograf näitab 4 graafikut ühel lehel |

## Lisaressursid õppimiseks:
- [TICK Stack õpetused](https://docs.influxdata.com/influxdb/v2.0/get-started/)
- [InfluxData YouTube kanal](https://www.youtube.com/influxdata)
- [Community foorumid](https://community.influxdata.com/)
- [GitHub projektid](https://github.com/influxdata)
- [Ametlik dokumentatsioon](https://docs.influxdata.com/)
- [Telegraf plugin'id](https://docs.influxdata.com/telegraf/latest/plugins/)
- [Kapacitor häired](https://docs.influxdata.com/kapacitor/latest/guides/alerts/)
- [Chronograf töölauad](https://docs.influxdata.com/chronograf/latest/guides/dashboard-template-variables/)
