# Loeng 2: Elasticsearchi andmemodelleerimise ja indekseerimise alused

## Elasticsearchi andmemodelleerimise alused

![Elasticsearch Clusters](https://s3.amazonaws.com/media-p.slid.es/uploads/239568/images/5067910/es_clusters.jpg)

*Source: [Elasticsearch Clusters Image](https://s3.amazonaws.com/media-p.slid.es/uploads/239568/images/5067910/es_clusters.jpg)*

Elasticsearchi andmemodelleerimine on protsess, mille käigus struktuuritakse andmeid selliselt, et need oleksid tõhusalt salvestatavad ja hõlpsasti otsitavad. Võrreldes traditsiooniliste relatsioonandmebaasidega pakub Elasticsearch rohkem paindlikkust ja võimalust hallata suuri, mitmekesiseid andmemahtusid.

## Skeemitus struktuur (Schema-less Design)

![Legacy and Up-to-Date Entities in NoSQL Databases](https://figures.semanticscholar.org/fbf995276f7454aaed71c2172040932cb636f901/1-Figure1-1.png)

*Source: [Semantic Scholar](https://figures.semanticscholar.org/fbf995276f7454aaed71c2172040932cb636f901/1-Figure1-1.png)*

Elasticsearch järgib nn skeemitut lähenemist, mis tähendab, et enne andmete indekseerimist ei ole vaja kindlaks määrata jäika skeemi. See võimaldab samas indeksis salvestada erineva struktuuriga dokumente. Kuid kuigi see lähenemine on paindlik, võib dünaamiline skeem mõnikord viia ootamatute tulemusteni, seega on oluline mõista, kuidas Elasticsearch automaatselt tüüpe tuvastab.

## JSON-põhised dokumendid

![Elasticsearch Data Mapping](https://aravind.dev/static/3768f3f352da47aad193daae9e2df260/c0566/es-data-mapping.png)

*Source: [Aravind's Blog](https://aravind.dev/static/3768f3f352da47aad193daae9e2df260/c0566/es-data-mapping.png)*

Elasticsearchis esitatakse andmed JSON-dokumentidena. Iga dokument esindab üksikut objekti või kirjet, kus andmed on määratletud võtme-väärtuse paaridena. Näiteks toodete kataloogi loomisel võib iga dokument sisaldada toote nime, hinda, kategooriat ja saadavust.

## Dünaamiline ja eksplitsiitne skeem (Dynamic and Explicit Mapping)

Kui dokumente indekseeritakse ilma spetsiaalse skeemita, loob Elasticsearch automaatselt dünaamilise skeemi, määrates andmetüüpidele sobivad väärtused. See on mugav, kuid keerukamate otsingute või analüüside puhul on kasulikum määratleda eksplitsiitne skeem. Eksplitsiitne skeem võimaldab täpselt kontrollida, kuidas andmeväljad on indekseeritud ja analüüsitud, parandades otsingutulemuste asjakohasust ja jõudlust.

![Elasticsearch Arbitrary Data](https://smnh.me/resized-images/elasticsearch/elasticsearch-arbitrary-data-sharp-768x702.webp)

*Source: [Elasticsearch Arbitrary Data Visualization](https://smnh.me/resized-images/elasticsearch/)*

## Elasticsearchi andmetüüpide ülevaade

Elasticsearch toetab mitmesuguseid andmetüüpe, mis võimaldavad erinevaid otsingu- ja analüüsivõimalusi. Peamised tüübid on:

1. **Text (tekst)** – kasutatakse täieliku tekstotsingu jaoks. Tekst analüüsitakse (nt tokeniseeritakse ja tüvestatakse).
2. **Keyword (märksõna)** – sobib täpseks vastendamiseks ja agregatsioonideks. Seda ei analüüsita.
3. **Numeric (numbrilised tüübid)** – hõlmab täisarve, murdarve ja muid arvulisi väärtusi.
4. **Date (kuupäevad ja ajad)** – kasutatakse ajaandmete salvestamiseks.
5. **Object ja Nested (objektid ja pesastatud andmed)** – võimaldavad salvestada keerulisemaid hierarhilisi andmestruktuure.

![Elasticsearch Field Datatypes](https://miro.medium.com/v2/resize:fit:1306/format:webp/1*Dhm4W0G1hqH-LVygZAYaYw.png)

*Source: [Understanding Elasticsearch Field Datatypes](https://medium.com/your-article-link-here)*

## Indekseerimine ja CRUD-tegevused

### Dokumentide indekseerimine (Indexing Documents)

Dokumentide indekseerimine on protsess, kus JSON-põhised dokumendid lisatakse Elasticsearchi indeksisse. Näiteks järgmine päring lisab toote nimega "Juhtmevaba hiir":

```bash
POST /tooted/_doc/1
{
  "nimi": "Juhtmevaba hiir",
  "hind": 25.99,
  "kategooria": "Elektroonika",
  "saadavus": true,
  "lisatud": "2024-02-10 15:30:00"
}
```

![Elasticsearch Index Operations](https://duydo.me/images/posts/es_index_operations.png)

*Source: [Duy Do's Blog](https://duydo.me/images/posts/es_index_operations.png)*

[Elasticsearch Index Management Guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-mgmt.html)

### Dokumentide päring

Andmete otsimiseks kasutatakse GET-meetodit. Näiteks järgmine päring otsib ülaltoodud dokumendi:
```bash
GET /tooted/_doc/1
```

### Dokumentide uuendamine

Dokumentide uuendamine võimaldab muuta olemasolevaid välju. Näiteks toote hinna muutmine:
```bash
POST /tooted/_doc/1/_update
{
  "doc": {
    "hind": 20.99
  }
}
```

### Dokumentide kustutamine

Dokumentide eemaldamiseks kasutatakse DELETE-meetodit. Näiteks:
```bash
DELETE /tooted/_doc/1
```

## Pesastatud dokumendid ja dünaamilised mallid

Pesastatud dokumente kasutatakse hierarhiliste andmestruktuuride, nagu toodete arvustuste, haldamiseks. Näiteks:

```bash
PUT /tooted/_mapping
{
  "properties": {
    "arvustused": {
      "type": "nested",
      "properties": {
        "kasutajanimi": { "type": "text" },
        "hinnang": { "type": "integer" },
        "kommentaar": { "type": "text" }
      }
    }
  }
}
```

## Jõudluse optimeerimine

Elasticsearchi jõudluse optimeerimiseks tuleks tähelepanu pöörata:

1. Shardide ja replikaatide konfiguratsioonile.
2. Kohandatud analüsaatorite määratlusele, mis võimaldavad tõhusat tekstianalüüsi.
3. Jälgimisvahenditele, nagu `_cat/nodes` ja `_cluster/health`.
