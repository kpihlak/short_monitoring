# Lecture 5: Elasticsearch Agregatsioonid

### 1. Sissejuhatus Agregatsionidesse

#### 1.1 Mis on Elasticsearch agregatsioonid?

Agregatsioonid on Elasticsearchi võimas funktsioon, mis võimaldab andmeid reaalajas analüüsida ja koondada. Need on võrreldavad SQL-i `GROUP BY` operatsioonidega, kuid pakuvad märksa rohkem võimalusi.

| Omadus           | Kirjeldus                              |
|------------------|----------------------------------------|
| Reaalajas töötlus | Tulemused arvutatakse päringute ajal  |
| Paindlikkus      | Võimalik kombineerida erinevaid agregatsioone |
| Skaleeruvus      | Töötab efektiivselt suurte andmemahtudega |
| Täpsus           | Võimaldab täpset analüütikat          |

#### 1.2 Põhilised kasutusjuhud

```json
// Näide lihtsa agregatsiooni päringust
{
  "aggs": {
    "keskmine_hind": {
      "avg": {
        "field": "hind"
      }
    }
  }
}
```

### 2. Agregatsiooni Tüübid

#### 2.1 Meetrika Agregatsioonid

Meetrika agregatsioonid arvutavad numbrilisi väärtusi dokumentide põhjal.

| Agregatsioon | Kirjeldus         | Näide                  |
|--------------|-------------------|------------------------|
| avg          | Keskmine väärtus   | Toodete keskmine hind  |
| sum          | Summa             | Päeva müügitulu        |
| min/max      | Miinimum/maksimum  | Madalaim/kõrgeim temperatuur |
| stats        | Põhilised statistikud | Täielik ülevaade hindadest |

#### 2.2 Bucketing Agregatsioonid

```json
// Näide terms agregatsiooni kohta
{
  "aggs": {
    "populaarsed_kategooriad": {
      "terms": {
        "field": "kategooria",
        "size": 5
      }
    }
  }
}
```

### 3. Praktiline Näide: Müügiandmete Analüüs

Vaatame reaalset näidet e-poe müügiandmete analüüsist:

```json
{
  "aggs": {
    "müük_kuude_lõikes": {
      "date_histogram": {
        "field": "müügi_kuupäev",
        "interval": "month"
      },
      "aggs": {
        "kogutulu": {
          "sum": {
            "field": "summa"
          }
        }
      }
    }
  }
}
```

### 4. Visualiseerimine Kibanas

Kibana võimaldab agregatsioone visualiseerida erinevate diagrammidena:

- **Tulpdiagrammid** (müük kategooriate kaupa)
- **Joondiagrammid** (ajalised trendid)
- **Sektordiagrammid** (turuosad)
- **Soojuskaardid** (geograafiline jaotus)

### 5. Harjutused

1. **Põhiline agregatsioon:**
   ```json
   // Leia keskmine tellimuse suurus
   {
     "aggs": {
       "keskmine_tellimus": {
         "avg": {
           "field": "tellimuse_summa"
         }
       }
     }
   }
   ```

2. **Kompleksne agregatsioon:**
   ```json
   // Analüüsi müüke piirkondade ja toodete lõikes
   {
     "aggs": {
       "piirkonnad": {
         "terms": {
           "field": "piirkond"
         },
         "aggs": {
           "tooted": {
             "terms": {
               "field": "toode"
             }
           }
         }
       }
     }
   }
   ```
