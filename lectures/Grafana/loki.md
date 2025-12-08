# Grafana Stack: Loki ja Tempo

**Eeldused:** Grafana dashboardid (teema 3), Docker  
**Õpetaja aeg:** ~45 min

---

## Terminoloogia

Enne alustamist - põhimõisted, mida kasutame:

**Log** = rakenduse tekstirida  
Näide: `"2025-11-03 14:30 ERROR payment failed"`

**Label** = metadata logile, et grupeerida  
Näide: `{service="payment", env="prod"}`

**Stream** = logide grupp samade labelitega

**Trace** = ühe päringu tee läbi süsteemi  
Näide: Kasutaja klikk → 5 teenust → vastus

**Span** = üks operatsioon ühes teenuses  
Näide: Database query span, API call span

**Trace ID** = unikaalne identifikaator päringule  
Näide: `abc-123-def` - sama ID läbib kõik teenused

---

## Õpiväljundid

Pärast selle loengu läbimist oskad:

1. **Selgitada**, mida Grafana Stack lisab tavalisele Grafanale
2. **Kirjeldada**, kuidas Loki kogub ja salvestab logisid
3. **Mõista**, miks labelite disain on Lokis kriitiline
4. **Kasutada** LogQL päringuid logide leidmiseks
5. **Selgitada**, mis on distributed tracing ja milleks see vajalik
6. **Navigeerida** logidest trace'idele ja vastupidi Grafanas
7. **Hinnata**, kuidas Grafana Stack parandab debugging'u hajutatud süsteemides

---

## 1. Mis jäi Grafana teemast puudu?

Eelnevas teemas (teema 3) õppisite Grafana dashboarde. Olete loonud panele ja visualiseerinud süsteemi seisundit reaalajas. Kui olete juba seadistanud dashboard'e, siis teate, et saate näha CPU kasutust, mälu ja päringu kiirusi.

Kuid oletame, et teie dashboard näitab error rate'i tõusu kell 14:30:

```
Error Rate
100 ┤                    ╭───╮
 80 ┤                  ╭─╯   ╰─╮
 60 ┤                ╭─╯       ╰─╮
 40 ┤              ╭─╯           ╰─╮
  0 ┼──────────────╯
    13:00      14:00      15:00
                ↑
            Probleem!
```

**Mida te järgmisena teete?**

Tõenäoliselt SSH'te serverisse, käivitate `tail -f /var/log/app.log`, grepite erroreid ja proovite aru saada mis läks valesti.

### Kolm probleemi

**Probleem 1: Logid pole Grafanas**  
Te peate lahkuma Grafanast ja minema käsureale. Dashboard näitas probleemi, aga log on mujal.

**Probleem 2: Mitme serveri logid**  
Kui teil on 50 serverit? Peate SSH'ma igasse ja grepima igat logifaili. See võtab tunde.

**Probleem 3: Hajutatud süsteemid**  
Bolt'i maksesüsteemis liigub üks päring läbi:
```
Frontend → Auth → Payment → Database → Bank API → Notification
```

Kuidas leiate ÜHE kasutaja päringu logid kõigist neist teenustest?

### Lahendus

Logid peaksid olema Grafana data source. Klõpsad dashboard'il error spike'il → näed kohe logisid. Ei pea SSH'ma.

Hajutatud süsteemis ei piisa logidest. Vajad näha päringu teed: milline teenus oli aeglane? Kus error tekkis? Kui kaua iga samm võttis?

**Grafana Stack lahendab mõlemat.**

---

## 2. Grafana Stack = Grafana + Loki + Tempo

Grafana Stack lisab teie juba tuttavale Grafanale kaks uut data source'i:

```
         ┌──────────────┐
         │   Grafana    │ ← Teate juba (teema 3)
         └──────┬───────┘
                │
         ┌──────┴──────┐
         │             │
    ┌────▼────┐   ┌────▼────┐
    │  Loki   │   │  Tempo  │
    │ (Logs)  │   │(Traces) │
    └─────────┘   └─────────┘
         ↑             ↑
       UUED          UUED
```

See diagramm näitab, et Grafana on keskne UI, mis ühendub kahega uue data source'iga: Loki logide jaoks ja Tempo trace'ide jaoks.

**Täna õpite:**
- **Loki** - kuidas logid tulevad Grafanasse
- **Tempo** - kuidas trace'd tulevad Grafanasse  
- **Integratsioon** - kuidas kõik koos töötavad

### Observability kaks sammast (siin kursuses)

```
┌──────────┐  ┌──────────┐
│   LOGS   │  │  TRACES  │
├──────────┤  ├──────────┤
│ Tekstid  │  │ Teed     │
│          │  │          │
│ERROR:    │  │Frontend  │
│payment   │  │ └─Auth   │
│failed    │  │   └─DB   │
└──────────┘  └──────────┘
     ↓             ↓
   Mis juhtus?  Kuidas?
```

Grafana Stack ühendab mõlemad ühes dashboardis.

---

## 3. Loki - logid Grafana data source'ina

### Miks mitte SSH + grep?

Wise'i või Bolt'i süsteem = 50-100 serverit/konteinrit.

```
┌─────────┐ ┌─────────┐ ┌─────────┐     ┌─────────┐
│Server 01│ │Server 02│ │Server 03│ ... │Server 50│
│ app.log │ │ app.log │ │ app.log │     │ app.log │
└─────────┘ └─────────┘ └─────────┘     └─────────┘
     ↓           ↓           ↓               ↓
   SSH?        SSH?        SSH?            SSH?
```

Te ei saa käia 50 serveris SSH'ga. Vajate tsentraliseeritud logisüsteemi.

### Loki arhitektuur

```
Rakendused/Serverid
       ↓
   Promtail (agent)
       ↓
   Loki (server)
       ↓
   Grafana
```

See diagramm näitab logide liikumist: rakendused kirjutavad logisid, Promtail agent kogub need kokku, saadab Loki serverisse, ja Grafanas saate neid vaadata.

**Promtail** = agent (väike programm)
- Töötab igas serveris/konteineris
- Jälgib logifaile `/var/log/*.log`
- Või kogub Docker stdout/stderr
- Saadab logid Loki serverisse

**Loki** = server
- Võtab logisid vastu
- Lisab **labeleid** (service, environment, host)
- Salvestab kompresseeritult
- Võimaldab LogQL päringuid

**Grafana** = visualiseerimine
- Explore vaates näed logisid
- Dashboard'ides saad panele teha
- Navigate logidest trace'idele

### Näide: Kuidas logid liiguvad

```
1. Rakendus kirjutab logi:
   "2025-11-03 14:30 ERROR payment failed"
           ↓
2. Promtail kogub:
   Jälgib /var/log/app.log
           ↓
3. Promtail lisab labeleid:
   {service="payment", env="prod", host="srv01"}
           ↓
4. Saadab Lokisse:
   HTTP POST → Loki server
           ↓
5. Loki salvestab:
   Index: labelid
   Sisu: kompresseeritud
           ↓
6. Grafanas näed:
   {service="payment"} |= "error"
```

### Loki võimas trikk - ainult labelid indekseeritakse

```
Logi rida:
"2025-11-03 14:30:15 ERROR payment failed user_id=12345"

Loki salvestab:
┌─────────────────────────────────────────┐
│ Index (kiire otsing):                   │
│ {service="payment", env="prod"}         │
└─────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────┐
│ Chunks (kompresseeritud sisu):          │
│ "2025-11-03 14:30:15 ERROR payment..." │
└─────────────────────────────────────────┘
```

Index on väike (ainult labelid) → odav storage  
Sisu on kompresseeritud → 10x vähem kettaruumi

### Labelite disain - VÄGA OLULINE!

Iga unikaalne labelite kombinatsioon = uus **stream**.

```yaml
# ✅ HEA - 3 streami
{service="payment", env="prod"}
{service="payment", env="staging"}
{service="auth", env="prod"}

# ❌ HALB - 10 000 streami!
{service="payment", user_id="1"}
{service="payment", user_id="2"}
{service="payment", user_id="3"}
...
{service="payment", user_id="10000"}
```

**Reegel:** Labelid = mida TEATE enne otsingut.

Te teate: "Otsin payment teenuse logisid"  
Te EI tea: "Otsin kasutaja 12345 logisid" ← user_id peaks olema logi SISUS!

**Õige:**
```yaml
Labelid: {service="payment", env="prod"}
Log sisu: "user_id=12345 request_id=abc payment failed"
```

### Miks see oluline?

Valesti disainitud labelid kukutavad Loki serveri maha. Kui teete iga user_id jaoks eraldi streami, siis 10 000 kasutajaga on teil 10 000 streami. Loki ei suuda seda hallata ja jõudlus kukub kokku. Reaaltöös olete sina, kes selle üles seab - pead teadma õiget disaini.

### Levinud viga

❌ **VIGA:**
```yaml
{service="payment", user_id="12345"}
```

✅ **ÕIGE:**
```yaml
Labelid: {service="payment", env="prod"}
Log sisu: "user_id=12345 amount=100 status=failed"
```

User_id on dünaamiline väärtus - pane see logi sisusse, mitte labeliks!

### LogQL - Loki päringukeel

LogQL on päringukeel logidele.

**LogQL sümboolid:**
- `|=` = sisaldab (contains)
- `!=` = ei sisalda (not contains)
- `|~` = regex match
- `|` = pipe (edasta järgmisele operaatorile)

#### Stream selector - vali õiged logid

```logql
{service="payment"}
{service="payment", env="prod"}
{service="payment", env="prod", host="srv01"}
```

**Mis siin toimub?** See valib välja õige streami - grupi logisid samade labelitega. Esimene päring võtab kõik payment teenuse logid, teine lisab filtri et ainult prod keskkonnas, kolmas veel täpsem - ainult srv01 serverist.

#### Line filter - filtreeri sisu

```logql
{service="payment"} |= "error"
{service="payment"} |= "error" != "timeout"  
{service="payment"} |~ "error|fail"
```

**Mis siin toimub?** Esimene leiab kõik logid, kus on sõna "error". Teine leiab logid, kus on "error" aga MITTE "timeout". Kolmas kasutab regex'i - leiab "error" VÕI "fail".

**Näide:**
```logql
{service="payment", env="prod"} |= "ERROR"
```

See leiab kõik production payment teenuse logid, kus on sõna ERROR.

#### JSON parsing

Kui logid on JSON formaadis:

```logql
{service="payment"} | json
{service="payment"} | json | status_code >= 500
{service="payment"} | json | user_id="12345"
```

**Mis siin toimub?** Esimene käsklus `| json` parsib JSON väljad. Seejärel saate filtreerida nende väljade järgi - näiteks leida kõik HTTP 500 errorid või konkreetse kasutaja logid.

**Näide logi JSON formaadis:**
```json
{"level":"error","msg":"payment failed","user_id":"12345","status_code":500}
```

**Päring:**
```logql
{service="payment"} | json | status_code >= 500
```

#### Statistika logidest

```logql
count_over_time({service="payment"} |= "error" [1h])
rate({service="payment"} |= "error" [5m])
sum(count_over_time({service="payment"} [1h])) by (host)
```

**Mis siin toimub?** Need päringud loevad logisid ja annavad ajaseeriad. Esimene loeb, mitu errori oli viimase tunni jooksul. Teine arvutab errorite rate'i (erroreid sekundis) viimase 5 minuti jooksul. Kolmas summeerib errorid serveri kaupa.

Need annavad ajaseeriad → saad time series graafikud dashboardis!

---

## 4. Tempo - distributed tracing

### Probleem hajutatud süsteemides

Wise'i rahakanne:

```
Kasutaja
  ↓
Frontend (50ms)
  ↓
API Gateway (20ms)
  ↓
Auth (30ms)
  ↓
User Service (100ms)
  ↓
Account Service (200ms)
  ↓
Payment Processor (5000ms) ← PROBLEEM!
  ↓
Bank API (4800ms)
  ↓
Notification (50ms)
```

Kui kogu protsess võtab 6 sekundit, **KUS probleem on?**

Logid näitavad: Iga teenus logib eraldi  
Aga **millises** teenuses kulub aeg?

### Miks see oluline?

Reaaltöös Bolt'il või Wise'il on kasutajad vihased, kui makse võtab 6 sekundit. Sul on 5 minutit leida, kus probleem on. Ilma trace'ideta peaksid lugema logisid kõigist 9 teenusest eraldi ja proovima ise kokku panna, mis juhtus. Võtaks tunde. Trace'iga näed kohe.

### Distributed tracing lahendus

**Trace ID** = unikaalne identifikaator päringule

```
Kasutaja päring
  ↓
[Trace ID: abc-123-def]
  ↓
Frontend: trace_id=abc-123-def GET /transfer
  ↓
Auth: trace_id=abc-123-def user=valid
  ↓
Payment: trace_id=abc-123-def processing
  ↓
Bank: trace_id=abc-123-def timeout!
  ↓
Notification: trace_id=abc-123-def not_sent
```

**Mis siin toimub?** Iga päring saab unikaalse trace ID. See ID liigub läbi kõigi teenuste. Kui otsite Lokis `trace_id=abc-123-def`, leiate KÕIK selle päringu logid kõigist teenustest!

### Trace struktuur - span'id

```
Trace ID: abc-123

[frontend      ]====================> 1000ms
  [auth        ]===> 50ms
  [payment     ]===============> 900ms
    [database  ]============> 850ms ← Probleem siin!
    [notify    ]==> 30ms
```

See diagramm näitab ühe päringu trace'i. Iga riba on span - üks operatsioon ühes teenuses. Riba pikkus näitab, kui kaua operatsioon võttis. Parent-child struktuur näitab, kes keda kutsus. Siin näete kohe: database span võttis 850ms - seal on probleem!

**Span** = üks operatsioon ühes teenuses  
**Parent-child** = kes keda kutsus  
**Kestus** = tulba pikkus

Näete kohe: database span = 850ms → seal probleem!

### OpenTelemetry (OTel)

Rakendused kasutavad **OTel SDK'd**:
- Instrumenteerib automaatselt HTTP, database, jne
- Genereerib trace ID ja span'e
- Edastab trace ID järgmisele teenusele
- Saadab span'id Tempo'sse

```
Rakendus + OTel SDK
       ↓
OTLP protocol
       ↓
Tempo server
       ↓
Grafana
```

**Mis siin toimub?** OTel SDK on library, mille lisate oma rakendusse. See genereerib automaatselt trace ID iga päringu jaoks, lisab selle HTTP headeritesse (nii liigub see järgmisesse teenusesse), ja saadab span'id Tempo serverisse.

### Tempo on lihtne

- EI indekseeri span'e täielikult
- Otsing ainult trace ID järgi
- Salvestab kompresseeritult
- Väga odav

**Küsimus:** Kui otsin ainult trace ID järgi, kust saan trace ID?

**Vastus:** LOGIDEST! OTel lisab trace_id igasse logi kirjesse.

---

## 5. Integratsioon - Grafana Stack'i võimsus

See on KÕIGE OLULISEM osa!

### Töövoog ILMA Grafana Stack'ita

```
1. Dashboard: Error rate kõrge! 📈
2. SSH serverisse 💻
3. grep "error" /var/log/app.log 🔍
4. Leiad error, aga ei tea konteksti ❓
5. Proovid leida, mis juhtus ⏰
6. Võtab tunde ⏳
```

### Töövoog Grafana Stack'iga

```
1. Dashboard: Error rate kõrge! 📈
   ↓ kliki graafikul
2. "View logs" → Loki avaneb 📋
   ↓ näed erroreid
3. Logis: trace_id=xyz789 🔗
   ↓ kliki lingil
4. Tempo: näitab kogu trace'i 🎯
   ↓ näed span'e
5. Probleem database span'is! ✅
   ↓
6. Leidsid 5 minutiga! ⚡
```

**KÕIK ÜHES BRAUSERI AKNAS!**

### Miks see oluline?

Bolt'il on 50+ teenust. Wise'il 100+. Ilma integreeritud stack'ita oled pime. Sul kulub 2 tundi probleemi leidmiseks - peate SSH'ma igasse serverisse, grepima logisid, proovima manuaalselt kokku panna, mis juhtus. Grafana Stack'iga leiate 5 minutiga. See on industry standard - pead seda oskama.

### Derived fields - automaatne linking

Grafana Loki data source'is seadistatakse:

```
Regex: trace_id=([a-f0-9]+)
Link: Tempo data source
```

Nüüd kui logi sisaldab trace_id, näed linki:

```
┌────────────────────────────────────────┐
│ 2025-11-03 14:30 ERROR payment failed │
│ trace_id=abc123 [🔗 View trace]        │ ← Link!
└────────────────────────────────────────┘
```

**Mis siin toimub?** Grafana otsib logi seast trace_id'sid (regex abil) ja teeb automaatselt lingi Tempo'sse. Klõpsad lingil → Tempo avaneb automaatselt õige trace'iga!

### Reaalne näide - Pipedrive CRM

**Incident:** Kasutaja kaebab, et CRM laadimine on aeglane kell 14:30.

**Samm 1:** Grafana dashboard näitab latency spike'i kell 14:30 - API vastused võtavad 2 sekundit.

**Samm 2:** Kliki "View logs" → Loki avaneb:
```logql
{service="crm-api"}
```
Time range: 14:25-14:35

Näed:
```
14:30:15 ERROR slow query detected
14:30:15 trace_id=def456 duration=1800ms
14:30:16 database connection pool exhausted
```

**Samm 3:** Kliki `trace_id=def456` lingil → Tempo avaneb:

```
[frontend    ]==> 100ms
  [crm-api   ]=================> 1900ms
    [database]===============> 1800ms ← Probleem!
      [query ]===============> 1780ms
    [cache   ]==> 20ms
```

**Samm 4:** Kliki database span'il, näed:
```
Span: database.query
Duration: 1780ms
Tags:
  sql.query: "SELECT * FROM contacts WHERE ..."
  sql.rows: 15000
```

**Samm 5:** Root cause leitud - SQL query võttis 1.8s ja tagastas 15 000 rida. Query ei ole optimeeritud, puudub index.

**Aeg:** 5 minutit probleemist root cause'ini!

---

## 6. Kokkuvõte

**Grafana Stack = Grafana + Loki + Tempo**

Teate juba:
- Grafana dashboardid (teema 3)
- Visualiseerimised

Täna õppisite:
- **Loki** - logid Grafana data source'ina
- **Tempo** - distributed tracing
- **Integratsioon** - logs → traces

### Võtmeoskused

**LogQL päringud:**
- Stream selector: `{service="payment"}`
- Line filter: `|= "error"`
- JSON parsing: `| json | status_code >= 500`

**Labelite disain:**
- Madal kardinaliteet!
- Labelid = mida TEATE enne otsingut
- Dünaamilised väärtused logi sisusse

**Trace'ide lugemine:**
- Span'id, parent-child
- Gantt chart
- Trace ID linkimine logidega

### Praktikas

```
Dashboard näitab probleemi
  ↓ kliki
Loki näitab logisid
  ↓ kliki trace_id
Tempo näitab trace'i
  ↓
Root cause leitud!
```

### Järgmised sammud

- **Laboris:** Proovid seda praktikas käed-küljes
- **Soovitatud lugemine:** Grafana Loki dokumentatsioon (https://grafana.com/docs/loki/)
- **Järgmine teema:** ELK Stack - alternatiivne lähenemine logimisele
