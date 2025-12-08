# Labor Zabbix HTTP Monitooring ja Alarmid

**Kestus:** 60 minutit  
**Raskus:** Keskmine  
**Eeldused:** Lab 1 läbitud, Zabbix töötab

---

## Eesmärk

Selle labori lõpuks oskad:
- ✅ Luua HTTP Agent item'eid (agentless monitoring)
- ✅ Monitoorida veebirakendusi ilma agendita
- ✅ Töödelda JSON vastuseid (API monitoring)
- ✅ Luua triggereid (automaatsed alarmid)
- ✅ Ehitada Web scenarioid (multi-step testid)
- ✅ Koostada dashboardi (visualiseerimine)

---

## Mida me monitoorime?

**Zabbix Web ise!** (meta-monitoring)

```
┌─────────────────────────────────────┐
│     Zabbix Server                   │
│          ↓                          │
│     HTTP päringud                   │
│          ↓                          │
│  ┌───────────────────┐              │
│  │  Zabbix Web UI    │ ← monitoorime│
│  │  (port 8080)      │   seda!     │
│  │                   │              │
│  │  /               → login         │
│  │  /zabbix.php     → dashboard     │
│  │  /api_jsonrpc.php → API (JSON)   │
│  └───────────────────┘              │
└─────────────────────────────────────┘
```

**Miks Zabbix Web?**
- ✅ Juba töötab (Lab 1-st)
- ✅ Ei vaja lisa container'it
- ✅ Päris rakendus (mitte test mock)
- ✅ Pakub JSON API'd
- ✅ Meta-efekt: "Zabbix jälgib iseennast"

---

## Osa 1: HTTP Agent Basics (15 min)

### Mis on HTTP Agent?

**Zabbix Agent** (Lab 1):
```
Zabbix Server → küsib → Zabbix Agent (hostis)
                         ↓
                      CPU, RAM, disk
```

**HTTP Agent** (Lab 2):
```
Zabbix Server → HTTP GET/POST → URL
                                 ↓
                              vastus (HTML/JSON)
```

**Erinevus:**
- Agent: Vajab agenti installitud hostis
- HTTP Agent: Lihtsalt HTTP päring, nagu `curl`

**Millal kasutada:**
- Agent → Süsteemi mõõdikud (CPU, RAM, processes)
- HTTP Agent → Veebirakendused, API'd, health check'id

---

### Samm 1: Lisa Host

Loome uue hosti mis esindab Zabbix Web'i.

**Data collection → Hosts → Create host**

| Väli | Väärtus | Miks? |
|------|---------|-------|
| **Host name** | `Zabbix-Web-App` | Rakenduse nimi |
| **Groups** | `Applications` | Loo uus group (Add new) |
| **Monitored by** | `Server` | Zabbix server teeb päringud |

**NB!** EI lisa interface'i! HTTP Agent ei vaja seda.

Salvesta: **Add**

**💡 Märkused:**

**DNS resolution:**
- `http://zabbix-web:8080/` töötab Docker võrgus (container ↔ container)
- Brauserist kasuta `http://localhost:8080/` (host → container)

**Timeout soovitus:**
HTTP Agent item'itele võib lisada timeout (nt 5s), et vead tuvastataks kiiremini.

📝 **Q1:** Miks me ei lisanud interface'i nagu Lab 1'es?  
💡 **A1:** Interface on ainult Zabbix Agent'i jaoks. HTTP Agent kasutab URL'i otse.

---

### Samm 2: Loo esimene HTTP Check

**Data collection → Hosts → Zabbix-Web-App → Items → Create item**

| Väli | Väärtus | Selgitus |
|------|---------|----------|
| **Name** | `Web UI: Home page` | Kirjeldav nimi |
| **Type** | `HTTP agent` | ⚠️ Oluline! Ei ole "Zabbix agent" |
| **Key** | `web.ui.home` | Unikaalne identifikaator |
| **URL** | `http://zabbix-web:8080/` | DNS nimi (Lab 1 õpetuse järgi!) |
| **Request type** | `GET` | HTTP meetod |
| **Timeout** | `5s` | Vigade kiire tuvastamine |
| **Update interval** | `30s` | Iga 30 sekundi tagant |

Salvesta: **Add**

**Mis juhtub nüüd:**
```
Iga 30 sekundi tagant:
  Zabbix Server → GET http://zabbix-web:8080/
                → salvestab vastuse (HTML)
                → history MySQL'i
```

---

### Samm 3: Mõõda Response Time'i

**Probleem:** HTTP agent item ei salvesta response time'i automaatselt.

**Lahendus:** Web scenario (isegi 1 sammuga) mõõdab aega automaatselt!

**Data collection → Hosts → Zabbix-Web-App → Web scenarios → Create web scenario**

| Väli | Väärtus |
|------|---------|
| **Name** | `Web UI: Response check` |
| **Update interval** | `1m` |

**Steps** tab → **Add:**

| Väli | Väärtus |
|------|---------|
| **Name** | `Home` |
| **URL** | `http://zabbix-web:8080/` |
| **Timeout** | `5s` |
| **Required status codes** | `200` |

Salvesta scenario: **Add**

**Mis juhtus:**  
Zabbix lõi automaatselt 4 item'i:
- Download speed
- **Response time** ← see on meil vaja!
- Response code
- Failed step

**Item key:** `web.test.time[Web UI: Response check,Home]`

---

### Samm 4: Vaata Andmeid

⏱️ **Oota 1-2 minutit**

**Monitoring → Hosts → Zabbix-Web-App → Web**

**Peaksid nägema:**
- Scenario: `Web UI: Response check`
- Status: **OK**
- Response time (nt 0.05s)

**Monitoring → Latest data**

Filter:
- Hosts: `Zabbix-Web-App`
- Name: `Response`
- Apply

**Peaksid nägema:**
- `Download speed for scenario "Web UI: Response check"`
- `Response time for step "Home" of scenario "Web UI: Response check"` ← see!
- `Response code for step "Home" of scenario "Web UI: Response check"`

**Kliki response time juures:** Graph ikoon

✅ **Checkpoint:** Näed response time graafikut!

📝 **Q2:** Miks me kasutame web scenario'sid response time'i jaoks?  
💡 **A2:** Web scenario mõõdab aega automaatselt. HTTP agent item salvestab ainult vastuse sisu, mitte aega.

---

## Osa 2: JSON API Monitoring (15 min)

### Mis on API monitoring?

**Veebileht** (HTML):
```html
<html><body>Zabbix</body></html>
```
→ Inimesele loetav, masinale raske

**API** (JSON):
```json
{"version": "7.0.6", "status": "ok"}
```
→ Masinale lihtne, struktureeritud

**Zabbix API:**
- URL: `http://zabbix-web:8080/api_jsonrpc.php`
- POST päring → JSON vastus
- Ei vaja autentimist (apiinfo.version)

---

### Samm 1: Loo API Master Item

**Create item:**

| Väli | Väärtus |
|------|---------|
| **Name** | `API: Raw response` |
| **Type** | `HTTP agent` |
| **Key** | `api.raw` |
| **URL** | `http://zabbix-web:8080/api_jsonrpc.php` |
| **Request type** | `POST` |
| **Request body** | (vt all) |
| **Request body type** | `JSON data` |
| **Update interval** | `1m` |
| **Type of information** | `Text` |

**Request body** (kopeeri täpselt):
```json
{
  "jsonrpc": "2.0",
  "method": "apiinfo.version",
  "id": 1
}
```

**Mis see teeb:**  
POST päring API'le → küsib Zabbix versiooni → saab JSON vastuse.

Salvesta.

---

### Samm 2: Vaata JSON Vastust

⏱️ **Oota 1 min**

**Monitoring → Latest data → Zabbix-Web-App**

Item: `API: Raw response`

**History** → **Values**

Peaksid nägema:
```json
{"jsonrpc":"2.0","result":"7.0.6","id":1}
```

✅ **Checkpoint:** JSON vastus nähtav!

---

### Samm 3: Parse JSON (Dependent Item)

**Probleem:**  
JSON on üks suur string. Me tahame versiooni numbrit (`7.0.6`) eraldi mõõdikuna!

**Lahendus:** Dependent Item + JSONPath

**Create item:**

| Väli | Väärtus |
|------|---------|
| **Name** | `API: Zabbix version` |
| **Type** | `Dependent item` |
| **Key** | `api.version` |
| **Master item** | `Zabbix-Web-App: API: Raw response` |
| **Type of information** | `Character` |

**Preprocessing** (scroll alla):
1. Kliki **Add**
2. Vali: `JSONPath`
3. Parameters: `$.result`

**Mis `$.result` tähendab:**
```json
{"jsonrpc":"2.0","result":"7.0.6","id":1}
                      ↑
                   võtame selle
```

`$` = JSON root  
`.result` = võta "result" väli

Salvesta.

---

### Samm 4: Vaata Parsed Value

⏱️ **Oota 1 min**

**Latest data → Zabbix-Web-App**

Item: `API: Zabbix version`

**Peaks näitama:** `7.0.6`

✅ **Checkpoint:** JSON parsing töötab!

📝 **Q3:** Miks Dependent Item on parem kui uus HTTP Agent item?  
💡 **A3:** 1 HTTP päring → mitu mõõdikut. Säästab võrguliiklust ja server'i koormust.

---

### Samm 5: Lisa HTTP Status Code Check

**Miks see vajalik?**  
Peame teadma kas server tagastas HTTP 200 (OK) või error (404, 500...).

**Create item:**

| Väli | Väärtus |
|------|---------|
| **Name** | `Web UI: HTTP status code` |
| **Type** | `Dependent item` |
| **Key** | `web.ui.status` |
| **Master item** | `Zabbix-Web-App: Web UI: Home page` |
| **Type of information** | `Numeric (unsigned)` |

**Preprocessing** (scroll alla):
1. Kliki **Add**
2. Vali: `HTTP status code`
3. (parameters jääb tühjaks)

**Mis see teeb:**  
Võtab HTTP agent vastusest status code (200, 404, 500...) → salvestab numbrina.

Salvesta.

⏱️ **Oota 1 min**

**Latest data** → peaks näitama: `200`

✅ **Checkpoint:** HTTP status code nähtav!

---

## Osa 3: Triggerid (Alarmid) (15 min)

### Mis on Trigger?

**Trigger** = tingimus mis loob alarmi.

**Komponendid:**
- **Expression** → Tingimus (nt `response time > 1s`)
- **Severity** → Kui tõsine (Warning, High, Disaster...)
- **Problem event** → Tingimus TRUE → Alarm!
- **OK event** → Tingimus FALSE → Alarm kaob

**Severity tasemed** (vähem → rohkem tõsine):
```
Not classified < Information < Warning < Average < High < Disaster
```

---

### Samm 1: Loo Response Time Trigger

**Data collection → Hosts → Zabbix-Web-App → Triggers → Create trigger**

| Väli | Väärtus |
|------|---------|
| **Name** | `Web UI: Slow response` |
| **Severity** | `Warning` |

**Expression:**
1. Kliki **Add**
2. Item: vali `Response time for step "Home" of scenario "Web UI: Response check"`
3. Function: `last()`
4. Operator: `>`
5. Value: `1`

**Peaks olema:**
```
last(/Zabbix-Web-App/web.test.time[Web UI: Response check,Home])>1
```

**Mis see tähendab:**  
"Viimane response time on suurem kui 1 sekund → Alarm!"

**NB!** Web scenario item key on pikk: `web.test.time[scenario,step]`

Salvesta.

---

### Samm 2: Loo Availability Trigger

**Create trigger:**

| Väli | Väärtus |
|------|---------|
| **Name** | `Web UI: Down or Error` |
| **Severity** | `High` |

**Expression:**
```
last(/Zabbix-Web-App/web.ui.status)<>200
```

**Kuidas see töötab:**  
HTTP status code ei ole 200 (OK) → Alarm!

**Võimalikud väärtused:**
- `200` = OK
- `404` = Not Found
- `500` = Server Error
- `0` = Connection failed

**Alternative expression** (kui tahad nodata detection):
```
last(/Zabbix-Web-App/web.ui.status)<>200 or nodata(/Zabbix-Web-App/web.ui.status,2m)
```

See alarmib ka siis kui andmeid pole 2 minutit tulnud (server down täielikult).

Salvesta.

---

### Samm 3: Testi Triggereid

**Test 1: UI Down**

Peata Zabbix Web 5 sekundiks:
```bash
docker stop zabbix-web && sleep 5 && docker start zabbix-web
```

**⚠️ NB!** Tee see **eraldi terminali aknas**, sest Web UI ise läheb kinni! Kui Web käivitub tagasi, saad jälle UI'sse.

⏱️ **Oota 1-2 min**

**Monitoring → Problems**

**Peaksid nägema:**
- 🔴 `Web UI: Down or Error` (High)

Kui Web käivitub:
- 🟢 Problem resolved

**Test 2: Slow response**

Aeglast vastust on raske tekitada ilma koodi muutmata, aga näeme et trigger konfiguratsioon on õige!

✅ **Checkpoint:** Trigger aktiveerus ja lahenes.

📝 **Q4:** Miks "Down" on High severity, aga "Slow" on Warning?  
💡 **A4:** Down = keegi ei saa kasutada (kriitilisem). Slow = aeglane, aga töötab veel.

---

## Osa 4: Web Scenario (Multi-Step) (10 min)

### Mis vahe on 1-sammulise ja mitme-sammulise scenario'l?

**1-sammuline** (juba tegime Osas 1):
```
GET /  → Response time
```

**Mitme-sammuline** (teeme nüüd):
```
Step 1: GET /login        → Login screen
Step 2: GET /dashboard    → Dashboard loads
Step 3: GET /api          → API responds
```

**Miks see oluline:**  
Veebileht võib vastata, aga login ei tööta! Multi-step testib kogu kasutaja teekonda.

---

### Samm 1: Loo Multi-Step Scenario

**Data collection → Hosts → Zabbix-Web-App → Web scenarios → Create web scenario**

| Väli | Väärtus |
|------|---------|
| **Name** | `Web UI: User Journey` |
| **Update interval** | `2m` |

**Agent** (scroll alla):
- `Zabbix` (jätame default'i)

**NB!** See on **teine** scenario. Esimene (`Web UI: Response check`) jääb alles!

---

### Samm 2: Lisa Steps

**Steps** tab → **Add**

**Step 1:**
- **Name:** `Login page`
- **URL:** `http://zabbix-web:8080/`
- **Required status codes:** `200`

**Add**

**Step 2:**
- **Name:** `Dashboard`
- **URL:** `http://zabbix-web:8080/zabbix.php?action=dashboard.view`
- **Required status codes:** `200`
- **Required string:** `Dashboard`

**Add**

**Step 3:**
- **Name:** `API health`
- **URL:** `http://zabbix-web:8080/api_jsonrpc.php`
- **Required status codes:** `200`

**Add**

Salvesta scenario: **Add**

---

### Samm 3: Vaata Scenario Tulemusi

⏱️ **Oota 2 min**

**Monitoring → Hosts → Zabbix-Web-App → Web**

**Peaksid nägema:**
- Scenario nimi: `Web UI: User Journey`
- Status: **OK** (0 failed steps)

**Kliki scenario nimel** → näed iga sammu:
- Download speed
- Response time
- Failed step (0 = OK)

✅ **Checkpoint:** Web scenario töötab!

📝 **Q5:** Millal kasutaksid Web Scenario vs lihtsat HTTP Agent item'i?  
💡 **A5:** Scenario = mitme-sammuline flow (login → dashboard). HTTP item = üks URL check.

---

## Osa 5: Dashboard (10 min)

### Mis on Dashboard?

**Dashboard** = kohandatud vaade mõõdikutele.

**Widgets:**
- **Graph** → Ajaline graafik
- **Item value** → Üks number
- **Gauge** → Visuaalne meter
- **Problems** → Alarmid

**Lab 1:** Kasutasime valmis dashboardi (vaatasime)  
**Lab 2:** Loome oma dashboardi (koostame)

---

### Samm 1: Loo Dashboard

**Dashboards → Create dashboard**

| Väli | Väärtus |
|------|---------|
| **Name** | `Web Monitoring` |
| **Default page name** | `Overview` |

**Add** (salvesta)

---

### Samm 2: Lisa Widgets

**Edit dashboard** (paremal üleval)

**Widget 1: Response Time Graph**

**Add widget:**
- **Type:** `Graph (classic)`
- **Name:** `Response Time`
- **Data set:**
  - Host: `Zabbix-Web-App`
  - Item: `Response time for step "Home" of scenario "Web UI: Response check"`
- **Time period:** `Last 1 hour`
- **Width:** `6` (pool ekraanist)

**Add**

**Widget 2: Zabbix Version**

**Add widget:**
- **Type:** `Item value`
- **Name:** `Zabbix Version`
- **Item:** `Zabbix-Web-App: API: Zabbix version`
- **Show:** `As is`
- **Width:** `3`

**Add**

**Widget 3: Current Response**

**Add widget:**
- **Type:** `Item value`
- **Name:** `Current Response Time`
- **Item:** `Response time for step "Home" of scenario "Web UI: Response check"`
- **Show:** `As is` + `Change indicator`
- **Decimal places:** `3`
- **Width:** `3`

**💡 Märkus:** Item value widget näitab hetkeväärtust ja muutust. Trendi/ajaloo jaoks kasuta Graph (classic).

**Add**

**Widget 3.5: HTTP Status**

**Add widget:**
- **Type:** `Item value`
- **Name:** `HTTP Status Code`
- **Item:** `Zabbix-Web-App: Web UI: HTTP status code`
- **Show:** `As is`
- **Thresholds:**
  - `200` → Green (OK)
  - `All other values` → Red
- **Width:** `3`

**Add**

**Widget 4: Problems**

**Add widget:**
- **Type:** `Problems`
- **Name:** `Web Issues`
- **Show:** `Recent problems`
- **Host groups:** `Applications`
- **Width:** `6`

**Add**

**Save changes** (dashboard'i salvesta)

---

### Samm 3: Vaata Dashboard'i

**Dashboards → Web Monitoring**

**Peaksid nägema:**
- ✅ Response time graafik (1h ajalugu)
- ✅ Zabbix version number (7.0.6)
- ✅ Praegune response time (nt 0.045s)
- ✅ Problems widget (tühi kui pole probleeme)

✅ **Checkpoint:** Dashboard näitab kõiki widget'eid!

📝 **Q6:** Miks dashboard on parem kui Latest data leht?  
💡 **A6:** Dashboard = kohandatud vaade. Näed ainult olulisi asju, ühe pilguga.

---

## Lõpetuseks

### ✅ Sul on nüüd oskused:

**HTTP Monitoring:**
- ✅ Lood HTTP Agent item'eid
- ✅ Monitoorid veebirakendusi ilma agendita
- ✅ Mõõdad response time'i

**JSON & API:**
- ✅ Teed POST päringuid
- ✅ Kasutad dependent item'eid
- ✅ Parse'id JSON'i (JSONPath: `$.field`)

**Alarmid:**
- ✅ Lood triggereid
- ✅ Seadistad severity taset
- ✅ Testad probleeme

**User Journey:**
- ✅ Lood Web scenarioid
- ✅ Testid mitme-sammulist flow'd

**Visualiseerimine:**
- ✅ Ehitad dashboardi
- ✅ Lisad erinevaid widget'eid
- ✅ Kohandad layout'i

---

### 🎓 Peamised kontseptsioonid:

**Item Types:**
| Type | Millal kasutada | Näide |
|------|----------------|-------|
| **Zabbix agent** | Süsteemi mõõdikud | CPU, RAM, disk |
| **HTTP agent** | Veebirakendused | Response time, API |
| **Dependent item** | Parse andmeid | JSON → eraldi väljad |

**Monitoring Levels:**
```
Lab 1: Infrastructure (host CPU, RAM, disk)
         ↓
Lab 2: Application (web response, API health)
         ↓
Lab 3: ??? (logs, alerts, automation)
```

**Trigger Severity:**
```
Information → kasutaja info
Warning     → tähelepanu väärt
Average     → tegelema peab
High        → kiire tegevus vajalik
Disaster    → kõik käed tööle!
```

---

## Küsimuste Vastused

**Q1:** HTTP Agent ei vaja interface'i – kasutab URL'i otse item'i konfist.

**Q2:** Zabbix Web UI kättesaadavust ja response time'i (meta-monitoring).

**Q3:** Dependent item kasutab master item'i andmeid → 1 HTTP päring, mitu mõõdikut.

**Q4:** Down = kriitilisem (keegi ei saa kasutada). Slow = aeglane, aga toimib.

**Q5:** Scenario = multi-step flow. HTTP item = üks URL check.

**Q6:** Dashboard = kohandatud, ülevaatlik. Latest data = kõik mõõdikud (info overload).

---

## Troubleshooting

### DNS ei lahene?

**Test:** Kas Zabbix Server näeb Zabbix Web DNS nime:
```bash
docker exec zabbix-server curl -I http://zabbix-web:8080/
```

**Ootus:** `HTTP/1.1 200 OK`

**Kui töötab:**
- Docker võrgus DNS OK ✅
- Zabbix item'id töötavad ✅
- Brauserist kasuta `http://localhost:8080/`

**Kui ei tööta:**
```bash
# Kontrolli võrku
docker network inspect zabbix-net

# Kontrolli kas zabbix-web on võrgus
docker network inspect zabbix-net | grep zabbix-web
```

### HTTP Agent item ei kogu andmeid?

1. **Kontrolli URL'i:** `http://zabbix-web:8080/` (DNS, mitte IP)
2. **Kontrolli timeout'i:** Vähemalt 5s
3. **Vaata item history:** Latest data → History tab
4. **Vaata logisid:** `docker logs zabbix-server | grep HTTP`

### Web scenario fail'ib?

1. **Kontrolli required string:** Kas leheküljel on see tekst?
2. **Kontrolli status code:** Kas server tagastab 200?
3. **Test käsitsi:** `curl -I http://zabbix-web:8080/`

---

## Koristus

**Stop ilma andmeid kustutamata:**
```bash
docker compose stop
```

**Eemalda kõik (ka DB):**
```bash
docker compose down -v
```