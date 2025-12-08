# Lecture 4: Elasticsearchi päringud

## 1. Query DSL põhikomponendid

Elasticsearchi Query DSL (Domain Specific Language) on võimas päringukeel, mis võimaldab luua keerukaid otsinguid ja filtreid. See koosneb neljast põhikomponendist:

| Komponent | Kirjeldus | Näide |
|-----------|-----------|--------|
| Query | Põhiline otsingu element, mis arvutab iga dokumendi jaoks vastavusskoori | `{"match": {"field": "value"}}` |
| Filter | Filtreerib dokumente ilma skoori arvutamata (kiirem kui query) | `{"term": {"status": "active"}}` |
| Aggregations | Statistilised operatsioonid andmete kohta | `{"aggs": {"avg_price": {"avg": {"field": "price"}}}}` |
| Sort | Tulemuste järjestamine | `{"sort": [{"price": {"order": "desc"}}]}` |

### Lihtsa päringu näide

```json
{
  "query": {
    "match": {
      "log_message": "error"
    }
  }
}
```

See päring otsib kõiki dokumente, mille `log_message` väljal esineb sõna "error".

### Terminipäring vs match päring

Elasticsearchis on oluline mõista erinevust `term` ja `match` päringute vahel:

- **Term päring**: Otsib täpset vastavust, ei analüüsi sisendteksti
  ```json
  {"term": {"field": "value"}}
  ```

- **Match päring**: Analüüsib sisendteksti enne otsingut, sobilik täistekstotsinguteks
  ```json
  {"match": {"field": "value"}}
  ```

## 2. Liitpäringud

Liitpäringud võimaldavad kombineerida mitu erinevat otsingutingimust. Kõige levinumad on bool päringud:

| Bool operaator | Kirjeldus | Loogika |
|----------------|-----------|---------|
| must | Dokument peab vastama kõigile tingimustele | AND |
| must_not | Dokument ei tohi vastata ühelegi tingimustele | NOT |
| should | Dokument võib vastata tingimustele (suurendab skoori) | OR |
| filter | Sarnane must'ile, aga ei mõjuta skoori | AND (ilma skoorita) |

### Bool päringu näide

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "service": "authentication" }},
        { "match": { "level": "error" }}
      ],
      "must_not": [
        { "match": { "environment": "test" }}
      ],
      "should": [
        { "match": { "priority": "high" }}
      ],
      "filter": [
        { "range": { "timestamp": { "gte": "now-1d" }}}
      ]
    }
  }
}
```

See päring:
1. **Peab leidma** logid, kus teenus on "authentication" JA tase on "error"
2. **Ei tohi leida** logisid, kus keskkond on "test"
3. **Võiks leida** (eelistatud) logid, kus prioriteet on "high"
4. **Filtreerib** ainult viimase päeva logid

## 3. Täistekstotsing ja hägus otsing

Elasticsearchis on mitmeid võimalusi teksti otsinguteks, mis on olulised logide analüüsimisel.

### Match päring

Põhiline täistekstotsingu päring:

```json
{
  "query": {
    "match": {
      "message": "connection refused"
    }
  }
}
```

See otsib dokumente, kus väljal "message" on kas "connection" VÕI "refused".

### Match phrase päring

Kui vajad täpset fraasi samas järjekorras:

```json
{
  "query": {
    "match_phrase": {
      "message": "connection refused"
    }
  }
}
```

See otsib dokumente, kus "connection" ja "refused" esinevad kõrvuti, samas järjekorras.

### Hägus otsing

Hägus otsing on kasulik, kui otsinguterminites võib esineda trükivigu:

```json
{
  "query": {
    "fuzzy": {
      "message": {
        "value": "conection",
        "fuzziness": "AUTO"
      }
    }
  }
}
```

See võib leida "connection", isegi kui otsingusõnas on trükiviga.

| Fuzziness väärtus | Kirjeldus |
|-------------------|-----------|
| 0 | Täpne vaste |
| 1 | Lubatud maksimaalselt 1 muudatus (tähe lisamine, eemaldamine või asendamine) |
| 2 | Lubatud maksimaalselt 2 muudatust |
| AUTO | Automaatselt määratud fuzziness sõna pikkuse põhjal |

## 4. Lehekülgede kaupa kuvamine ja sorteerimine

Logide analüüsimisel on sageli vaja tulemusi lehekülgede kaupa kuvada ja sorteerida.

### Lehekülgede kaupa kuvamine

```json
{
  "from": 0,  // Alguspositsioon
  "size": 20, // Tulemuste arv
  "query": {
    "match": {
      "level": "error"
    }
  }
}
```

See päring tagastab esimesed 20 dokumenti, mis vastavad päringule.

### Sorteerimine

```json
{
  "sort": [
    { "timestamp": { "order": "desc" }},
    { "_score": { "order": "desc" }}
  ],
  "query": {
    "match": {
      "message": "error"
    }
  }
}
```

See päring sorteerib tulemused esmalt ajatempli järgi kahanevas järjekorras, seejärel skoori järgi kahanevas järjekorras.

### Väljad ja nende filtreerimine

Kui tahad, et päring tagastaks ainult teatud väljad, saad kasutada `_source` parameetrit:

```json
{
  "_source": ["timestamp", "level", "message"],
  "query": {
    "match_all": {}
  }
}
```

See päring tagastab kõigi dokumentide puhul ainult timestamp, level ja message väljad.

## 5. Praktiline näide: logide otsing

Kujutame ette, et meil on logid, kus on järgmised väljad:
- timestamp: aja märge
- service: teenuse nimi
- level: logi tase (error, warn, info, debug)
- message: logi sõnum
- host: serveri nimi

Otsime kõiki kriitilisi vigu mitmest teenusest viimase tunni jooksul:

```json
{
  "size": 100,
  "sort": [
    { "timestamp": { "order": "desc" }}
  ],
  "_source": ["timestamp", "service", "level", "message", "host"],
  "query": {
    "bool": {
      "must": [
        { "match": { "level": "error" }}
      ],
      "should": [
        { "match": { "message": "critical" }},
        { "match": { "message": "failure" }},
        { "match": { "message": "crashed" }}
      ],
      "minimum_should_match": 1,
      "filter": [
        { "terms": { "service": ["authentication", "payment", "database"] }},
        { "range": { "timestamp": { "gte": "now-1h" }}}
      ]
    }
  }
}
```

See päring:
1. Otsib error-tasemel logisid
2. Annab kõrgema skoori logidele, mis sisaldavad "critical", "failure" või "crashed"
3. Filtreerib ainult authentication, payment ja database teenuste logid
4. Filtreerib ainult viimase tunni logid
5. Tagastab kuni 100 tulemust
6. Sorteerib tulemused ajatempli järgi kahanevas järjekorras
7. Tagastab ainult olulisemad väljad
