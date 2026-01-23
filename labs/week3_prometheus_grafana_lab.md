# Week 3 Labor: Prometheus & Grafana

Selles laboris ehitame täieliku monitooringulahenduse nullist. Kasutame demo rakendust nimega Shoehub, mis simuleerib jalanõude e-poodi ja genereerib automaatselt müügiandmeid. Prometheus kogub neid andmeid ja Grafana visualiseerib need dashboardina.

Labori lõpuks on sul töötav süsteem, kus näed reaalajas "müügiandmeid" ilusate graafikutena ning oskad luua oma dashboarde ja päringuid.

## Arhitektuur, mida ehitame

Meie süsteem koosneb kolmest konteinerist. Shoehub jookseb pordil 8080 ja genereerib pidevalt juhuslikke müügiandmeid - kui palju müüdi Loafers'eid Austraalias, kui palju Boots'e Ameerikas jne. Prometheus jookseb pordil 9090 ja küsib Shoehub'ilt iga 15 sekundi järel uusi andmeid. Grafana jookseb pordil 3000 ja võimaldab neid andmeid visualiseerida.

Kõik kolm konteinerit suhtlevad omavahel Docker'i sisevõrgu kaudu. See tähendab, et kui Grafana tahab Prometheusega rääkida, kasutab ta nime `prometheus`, mitte `localhost`.

## Osa 1: Keskkonna ettevalmistus

Alustame töökausta loomisest. Kõik konfiguratsioonifailid hoiame ühes kohas, et oleks hiljem lihtne puhastada.

```bash
mkdir -p ~/prometheus-grafana-lab
cd ~/prometheus-grafana-lab
```

Nüüd loome Docker Compose faili, mis kirjeldab meie kolme konteinerit. Docker Compose võimaldab mitut konteinerit korraga hallata ja tagab, et nad saavad omavahel suhelda.

```bash
cat > docker-compose.yml << 'EOF'
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
EOF
```

See fail ütleb Docker'ile, et tahame kolme teenust. Prometheus kasutab ametlikku `prom/prometheus` image'it ja me anname talle oma konfiguratsioonifaili. Shoehub on valmis demo rakendus, mida keegi on juba meie jaoks pakendanud. Grafana saab vaikimisi admin parooli "admin" ja me salvestame tema andmed volume'i, et need säiliksid ka konteineri taaskäivitamisel.

Järgmisena loome Prometheus'e konfiguratsioonifaili. See ütleb Prometheus'ele, mida jälgida.

```bash
cat > prometheus.yml << 'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'shoehub'
    static_configs:
      - targets: ['shoehub:8080']
EOF
```

Konfiguratsioon on minimaalne. Ütleme, et scrape'ime iga 15 sekundi järel ja meil on üks job nimega "shoehub", mis jälgib target'i aadressil `shoehub:8080`. Pane tähele, et kasutame konteineri nime, mitte IP-aadressi - Docker lahendab selle automaatselt.

Nüüd käivitame kõik konteinerid. Kasuta seda käsku, mis sinu Docker versioonis töötab:

```bash
docker compose up -d
```

Kui sul on vanem Docker, võib käsk olla sidekriipsuga:

```bash
docker-compose up -d
```

Flag `-d` tähendab detached mode - konteinerid jooksevad taustal. Kontrollime, kas kõik käivitus:

```bash
docker compose ps
```

Peaksid nägema kolme konteinerit, kõik staatuses "Up". Esimene kord võtab käivitamine natuke aega, sest Docker peab image'id alla laadima.

Nüüd kontrollime, kas kõik töötab. Ava brauseris http://localhost:8080/metrics - peaksid nägema Shoehub'i tooreid mõõdikuid tekstiformaadis. Read nagu `shoehub_sales{ShoeType="Loafers",CountryCode="AU"} 145.0` näitavad, et Shoehub genereerib andmeid.

Ava http://localhost:9090 - see on Prometheus'e UI. Ja http://localhost:3000 on Grafana, kuhu saad sisse logida kasutajaga admin ja parooliga admin.

## Osa 2: Tutvumine Prometheus'e päringutega

Enne Grafanasse minekut tutvume Prometheus'e UI-ga. See aitab mõista, kuidas päringud töötavad.

Ava http://localhost:9090 ja mine Graph tab'i. Sisesta päringukasti:

```promql
shoehub_sales
```

Vajuta Execute. Näed kõiki müügimõõdikuid - erinevad jalanõutüübid, erinevad riigid. Iga rida on eraldi aegrida oma label'ite kombinatsiooniga.

Nüüd filtreeri ainult Loafers'id:

```promql
shoehub_sales{ShoeType="Loafers"}
```

Tulemusi on vähem - ainult need, kus ShoeType on Loafers.

Counter näitab kogusummat, mis pidevalt kasvab. Et näha müügikiirust, kasuta rate() funktsiooni:

```promql
rate(shoehub_sales[1m])
```

Nüüd näed, kui kiiresti müük kasvab - mitu "müüki" sekundis. Nurksulgudes `[1m]` ütleb, et vaatame viimase minuti andmeid.

Kui tahad näha kogu müüki kokku, lisa sum():

```promql
sum(rate(shoehub_sales[1m]))
```

Ja kui tahad näha müüki jalanõutüübi kaupa:

```promql
sum by (ShoeType) (rate(shoehub_sales[1m]))
```

Mine ka Status → Targets menüüsse. Seal näed, et shoehub target on UP ja roheline. See tähendab, et Prometheus saab edukalt andmeid.

## Osa 3: Grafana andmeallika seadistamine

Nüüd ühendame Grafana Prometheus'ega. Ava http://localhost:3000 ja logi sisse kasutajaga admin, parooliga admin. Esimesel sisselogimisel küsib Grafana uut parooli - võid vajutada Skip, et jätkata vana parooliga.

Grafana UI võib versiooniti veidi erineda, aga põhiloogika on sama. Otsi menüüst Connections või Data Sources. Tavaliselt leiad selle hamburgermenüüst (☰) vasakul üleval.

Mine Connections → Data Sources ja kliki Add data source. Vali Prometheus.

URL väljale kirjuta `http://prometheus:9090`. Oluline on kasutada konteineri nime "prometheus", mitte "localhost" - Grafana jookseb Docker'i sees ja localhost viitaks Grafana enda konteinerile, mitte Prometheus'ele.

Kliki Save & Test. Kui näed rohelist "Data source is working", on ühendus olemas.

## Osa 4: Esimese dashboardi loomine

Nüüd hakkame ehitama dashboardi. Mine Dashboards menüüsse ja vali New Dashboard, seejärel Add visualization.

Esimene paneel näitab müügikiirust jalanõutüübi kaupa. Vali andmeallikaks Prometheus ja sisesta päring:

```promql
sum by (ShoeType) (rate(shoehub_sales[1m]))
```

Kohe peaksid nägema joongraafiku mitme joonega - üks iga jalanõutüübi kohta. Paremal pool on paneeli seaded. Anna talle pealkiri "Müügikiirus per toode". Legend sektsioonis võid lisada `{{ShoeType}}`, et joonte nimed oleksid loetavamad.

Kliki Apply, et paneel salvestada dashboardi.

Teise paneeli jaoks kliki ülaosas Add visualization. Seekord teeme Stat paneeli, mis näitab ühte suurt numbrit. Sisesta päring:

```promql
sum(shoehub_sales)
```

Paremal pool muuda visualisatsioonitüüp Stat'iks (vaikimisi on Time Series). Anna pealkiri "Kogumüük". See näitab nüüd üht suurt numbrit - kogu müükide summat.

Kolmas paneel on sektordiagramm, mis näitab makseid riikide kaupa. Lisa uus visualization ja muuda tüüp Pie Chart'iks.

Siin vajame mitut päringut. Esimese päringu (A) jaoks:

```promql
sum(shoehub_payments{CountryCode="AU"})
```

Lisa teine päring (B) klikkides + Query:

```promql
sum(shoehub_payments{CountryCode="US"})
```

Ja kolmas (C):

```promql
sum(shoehub_payments{CountryCode="IN"})
```

Iga päringu juures Options sektsioonis saad määrata Legend'i - pane vastavalt "Australia", "USA", "India". Anna paneelile pealkiri "Maksed per riik".

Nüüd salvesta dashboard. Kliki ülaosas disketiikoonil või vajuta Ctrl+S. Anna dashboardile nimi "Shoe Sales Dashboard".

## Osa 5: Dünaamilised valikud variable'itega

Praegu on riigid dashboardis "hardcoded". Aga mis siis, kui tahad, et kasutaja saaks ise valida, millist riiki vaadata? Selleks kasutame variable'eid.

Mine dashboardi seadetesse - kliki ülaosas hammasrattaikoonil (Dashboard settings). Vali vasakult Variables ja kliki New variable.

Täida väljad järgmiselt. Name väljale kirjuta `country` - see on nimi, mida kasutad päringutes. Label väljale kirjuta `Riik` - seda näeb kasutaja. Type vali Query. Data source vali Prometheus.

Query väljale kirjuta:

```
label_values(shoehub_payments, CountryCode)
```

See päring leiab kõik unikaalsed CountryCode väärtused shoehub_payments mõõdikust.

Lülita sisse Multi-value ja Include All option. See võimaldab kasutajal valida mitu riiki korraga või kõik korraga.

Kliki Apply ja seejärel Save dashboard.

Nüüd näed dashboardi ülaosas dropdown'i "Riik". Aga meie paneelid ei kasuta seda veel. Lisame uue paneeli, mis kasutab variable't.

Lisa uus visualization ja sisesta päring:

```promql
sum(rate(shoehub_payments{CountryCode="$country"}[1m]))
```

Pane tähele `$country` - see asendatakse kasutaja valikuga. Anna paneelile pealkiri "Maksed - $country", siis näitab pealkiri ka valitud riiki.

Nüüd kui valid dropdown'ist erinevaid riike, uueneb paneel automaatselt.

## Osa 6: Visuaalne tagasiside threshold'idega

Gauge paneel on hea viis näidata ühte väärtust nii, et värvid annavad kohe tagasisidet. Loome paneeli, mis näitab PayPal maksete osakaalu USAs.

Lisa uus visualization ja muuda tüüp Gauge'iks. Sisesta päring:

```promql
sum(shoehub_payments{CountryCode="US", PaymentMethod="Paypal"}) 
/ 
sum(shoehub_payments{CountryCode="US"}) 
* 100
```

See arvutab, mitu protsenti USA maksetest on PayPal.

Paremal pool seadetes mine Standard options sektsiooni ja vali Unit → Percent (0-100).

Seejärel mine Thresholds sektsiooni. Vaikimisi on üks threshold. Seadista nii:
- Base väärtus jääb 0, värv roheline
- Lisa uus threshold väärtusega 10, värv kollane  
- Lisa veel üks väärtusega 20, värv punane

Nüüd näitab gauge rohelist kui PayPal osakaal on alla 10%, kollast 10-20% vahel ja punast üle 20%.

Anna pealkiri "PayPal % USAs" ja salvesta.

## Osa 7: Ajaliste trendide võrdlus

Vahel tahad võrrelda praegust olukorda minevikuga. PromQL'i `offset` võimaldab seda teha.

Lisa uus visualization. Sisesta esimene päring:

```promql
sum(rate(shoehub_sales{ShoeType="Loafers"}[1m]))
```

Options sektsioonis pane Legend väärtuseks "Praegu".

Lisa teine päring:

```promql
sum(rate(shoehub_sales{ShoeType="Loafers"}[1m] offset 5m))
```

Legend väärtuseks pane "5 min tagasi".

Nüüd näed graafikul kahte joont - praegust müügikiirust ja seda, milline oli müügikiirus 5 minutit tagasi. Päris süsteemis kasutaksid pigem `offset 1d` (eile) või `offset 7d` (nädal tagasi), aga meie demo jaoks on 5 minutit piisav, et näha erinevust.

Anna paneelile pealkiri "Loafers: praegu vs 5 min tagasi" ja salvesta dashboard.

## Kokkuvõte

Sul on nüüd töötav Prometheus + Grafana süsteem, mis jälgib demo rakendust. Dashboard sisaldab erinevaid visualisatsioone - joongraafikud trendide jaoks, stat paneel kogunumbrite jaoks, sektordiagramm jaotuse näitamiseks, gauge threshold'idega ja ajaline võrdlus.

Samad põhimõtted kehtivad päris süsteemide monitoorimisel. Shoehub'i asemel oleks Node Exporter serverite jälgimiseks või sinu enda rakendus, mis eksponeerib mõõdikuid.

## Puhastamine

Kui labor on läbi, saad kõik kustutada ühe käsuga:

```bash
cd ~/prometheus-grafana-lab
docker compose down -v
```

Flag `-v` kustutab ka volume'd, sealhulgas Grafana salvestatud dashboardid.

## Kui midagi ei tööta

Kui konteiner ei käivitu, vaata logisid:

```bash
docker compose logs prometheus
docker compose logs grafana
docker compose logs shoehub
```

Kui Grafana ei saa Prometheus'ega ühendust, kontrolli et URL on `http://prometheus:9090`, mitte `http://localhost:9090`.

Kui paneel näitab "No data", testi päringut kõigepealt Prometheus'e UI-s aadressil http://localhost:9090. Kui seal töötab, on probleem Grafana seadistuses.

Kui variable ei tööta, kontrolli et Query on täpselt `label_values(shoehub_payments, CountryCode)` ja data source on Prometheus.
