# Week 3: Prometheus & Grafana

Selles loengus tutvume kahe võimsa tööriistaga, mis koos moodustavad tänapäeva ühe populaarseima monitooringulahenduse. Prometheus kogub ja salvestab mõõdikuid, Grafana visualiseerib neid ilusate dashboardidena.

## Miks Prometheus loodi?

Aastal 2012 seisis SoundCloud silmitsi probleemiga. Nende süsteem oli kasvanud kümnest serverist sadadeks mikroteenusteks, mis jooksid konteinerites. Traditsioonilised monitooringutööriistad nagu Nagios ja Zabbix ei sobinud enam - need eeldasid, et servereid on vähe ja need on stabiilsed. Kubernetes'e maailmas aga konteinerid tekivad ja kaovad pidevalt, IP-aadressid muutuvad, teenuste arv kõigub vastavalt koormusele.

Julius Volz ja tema meeskond otsustasid luua midagi uut. Nii sündis Prometheus - monitooringusüsteem, mis on loodud dünaamiliste keskkondade jaoks.

## Pull vs Push - põhimõtteline erinevus

Traditsioonilised süsteemid nagu Zabbix kasutavad push-mudelit: agent serveris saadab andmeid keskserverisse. Prometheus töötab vastupidi - ta kasutab pull-mudelit, kus Prometheus ise küsib target'itelt mõõdikuid.

See vahe on oluline. Kui Zabbix'i agent lakkab saatmast, ei pruugi sa kohe aru saada, kas agent on maas või lihtsalt pole midagi saata. Prometheus aga märkab kohe, kui target ei vasta - scrape ebaõnnestub ja sa näed "target down" hoiatust.

Pull-mudel annab ka parema kontrolli. Prometheus otsustab, millal ja kui tihti andmeid koguda. Vaikimisi toimub see iga 15 sekundi järel, aga saad seda seadistada vastavalt vajadusele.

![Prometheus töövoog](https://training.promlabs.com/static/prometheus-abstract-pipeline-fcb092ef3974c2ada13032abdec22c29.svg)

## Kuidas andmed liiguvad

Prometheus'e arhitektuur on lihtne, aga võimas. Keskmes on Prometheus server, mis teeb kolme asja: kogub mõõdikuid target'itelt, salvestab need spetsiaalsesse aegrea andmebaasi (TSDB) ja hindab hoiatuste reegleid.

Target'id on rakendused või süsteemid, mida jälgid. Iga target pakub `/metrics` endpoint'i, kust Prometheus andmeid loeb. Kui sul on Linux server, paigaldad sinna Node Exporter'i - väikese programmi, mis loeb süsteemi statistikat ja pakub seda Prometheus'e formaadis.

![Prometheus arhitektuur](https://prometheus.io/assets/architecture.png)

AlertManager on eraldi komponent, mis tegeleb hoiatustega. Prometheus tuvastab probleemi, aga AlertManager otsustab, kuidas sellest teavitada - kas saata email, Slack'i sõnum või PagerDuty alert. Ta oskab ka sarnaseid hoiatusi grupeerida, et sa ei saaks kümmet emaili kümne serveri kohta, vaid ühe kokkuvõtva teate.

Grafana on visualiseerimiskiht. Prometheus'el on küll oma lihtne UI päringute tegemiseks, aga tõeliselt ilusad dashboardid teed Grafanas.

## Mõõdikute formaat

Kui külastad mõne target'i `/metrics` endpoint'i, näed lihtsat tekstiformaati. See on inimloetav ja kergesti parsitav. Iga mõõdik koosneb nimest, valikulistest label'itest ja väärtusest.

Näiteks HTTP päringute loendur võiks välja näha nii:

```
http_requests_total{method="GET",status="200"} 1234
http_requests_total{method="POST",status="201"} 567
```

Label'id on võimsad - need võimaldavad ühe mõõdiku jagada mitmeks dimensiooniks. Selle asemel, et luua eraldi mõõdikud `http_get_requests` ja `http_post_requests`, on sul üks `http_requests_total` mõõdik, mida saad filtreerida method'i järgi.

![Prometheus andmemudel](https://training.promlabs.com/static/prometheus-data-model-7756d9169168839cd7145f4aaa7e39df.svg)

## Neli mõõdikutüüpi

Prometheus tunneb nelja tüüpi mõõdikuid ja on oluline mõista nende erinevusi.

**Counter** on lihtsaim - see ainult kasvab. Päringute arv, vigade arv, töödeldud baitide hulk. Counter ei lähe kunagi alla, välja arvatud kui teenus taaskäivitub ja nullist alustab. Kui tahad teada päringute arvu sekundis, pead kasutama `rate()` funktsiooni.

**Gauge** tõuseb ja langeb vabalt. CPU kasutus, mälu hulk, aktiivsete ühenduste arv - need kõik on gauge'id. Gauge'i väärtust saad kasutada otse, ilma täiendavate funktsioonideta.

**Histogram** ja **Summary** on keerulisemad - need mõõdavad jaotust. Kui tahad teada, et 95% päringutest vastatakse alla 200 millisekundi, vajad histogrammi või summary't. Histogram jagab väärtused eelmääratud bucket'itesse, Summary arvutab protsentiilid kliendi poolel.

## PromQL - päringukeel

PromQL on Prometheus'e päringukeel. See on spetsiaalselt loodud aegrea andmete jaoks ja erineb oluliselt SQL-ist.

Lihtsaim päring on lihtsalt mõõdiku nimi:

```promql
http_requests_total
```

See tagastab kõik aegread selle mõõdiku jaoks. Kui sul on mitu serverit ja mitu endpoint'i, võid saada kümneid tulemusi.

Filtreerimiseks kasuta label'eid loogelistes sulgudes:

```promql
http_requests_total{method="GET"}
```

Kõige olulisem funktsioon on `rate()`. Counter näitab kogusummat - näiteks 12345 päringut alates teenuse käivitamisest. See number ise ei ütle sulle midagi. `rate()` arvutab, kui kiiresti counter kasvab:

```promql
rate(http_requests_total[5m])
```

Nurksulgudes olev `[5m]` on ajavahemik - Prometheus vaatab viimase 5 minuti andmeid ja arvutab keskmise kasvu sekundis. Tulemus on päringute arv sekundis.

Agregeerimisfunktsioonid nagu `sum()`, `avg()`, `max()` võimaldavad andmeid kokku võtta:

```promql
sum(rate(http_requests_total[5m]))
```

See annab kõigi serverite ja endpoint'ide päringud kokku. Kui tahad näha tulemust gruppide kaupa, lisa `by`:

```promql
sum by (method) (rate(http_requests_total[5m]))
```

Nüüd näed eraldi GET, POST, PUT päringute arvu.

## Grafana - visualiseerimine

Grafana on visualiseerimisplatvorm, mis ühendab erinevaid andmeallikaid. Oluline on mõista, et Grafana ise ei kogu andmeid - ta ainult küsib neid teistest süsteemidest ja kuvab graafikutena.

![Grafana dashboard näide](https://grafana.com/static/assets/img/blog/kubernetes_nginx_dash.png)

Dashboard on Grafanas lehekülg, mis sisaldab paneele. Iga paneel on üks visualiseering - joongraafik, gauge, tabel või muu. Paneel teeb päringu andmeallikasse ja kuvab tulemuse.

Time Series on klassikaline joongraafik, mis näitab väärtuse muutumist ajas. See sobib CPU kasutuse, traffic'u ja muude ajaliste trendide jaoks.

Gauge näitab ühte väärtust poolringina - sobib hästi protsentide jaoks nagu ketta täituvus.

Stat on lihtsalt suur number - aktiivsete kasutajate arv, vigade hulk viimase tunni jooksul.

Pie Chart näitab jaotust - kui palju müügist tuli igast kategooriast.

Variables teevad dashboardi dünaamiliseks. Defineerid muutuja `$server` ja kasutad seda päringutes. Kasutaja saab dropdown'ist valida serveri ja kõik panelid uuenevad automaatselt.

Thresholds määravad värvid väärtuste põhjal. Seadistad, et alla 70% on roheline, 70-90% kollane ja üle 90% punane. Nii näed kohe, kui miski vajab tähelepanu.

## Millal Prometheus sobib

Prometheus on ideaalne mikroteenuste ja Kubernetes'e keskkondade jaoks. Ta on loodud dünaamilisuse jaoks - teenused tulevad ja lähevad, Prometheus kohaneb.

Samas pole Prometheus universaalne lahendus. Ta on mõeldud mõõdikute jaoks, mitte logide jaoks. Kui vajad logide analüüsi, kasuta Loki't või ELK stack'i. Prometheus hoiab andmeid tavaliselt 15-30 päeva - kui vajad aastate pikkust ajalugu, pead lisama Thanos'e või Mimir'i. Ja kui vajad 100% täpsust arvelduse jaoks, kasuta parem SQL andmebaasi - Prometheus on optimeeritud kiiruse, mitte täpsuse jaoks.

## Kokkuvõte

Prometheus ja Grafana koos moodustavad võimsa monitooringulahenduse. Prometheus kogub mõõdikuid pull-mudeliga, salvestab need aegrea andmebaasi ja võimaldab PromQL päringuid. Grafana visualiseerib need andmed ilusate dashboardidena.

Järgmisena läheme laborisse, kus paigaldame mõlemad tööriistad ja ehitame oma esimese dashboardi.
