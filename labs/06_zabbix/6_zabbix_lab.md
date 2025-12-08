# Labor Zabbix Alused

**Kestus:** 60 minutit  
**Raskus:** Beginner  
**Eeldused:** Docker, Docker Compose v2

---

## Eesmärk

Selle labori lõpuks oskad:
- ✅ Ehitada Zabbix keskkonda komponent-komponendilt  
- ✅ Mõista iga komponendi rolli monitooringu arhitektuuris  
- ✅ Testida iga sammu eraldi  
- ✅ Seadistada esimest monitoritud hosti  
- ✅ Vaadata reaalseid mõõdikuid  

---

## Monitooringu Arhitektuur

```
┌──────────────────┐
│ MySQL (3306)     │ ← Andmebaas: config + ajalugu
└────────┬─────────┘
         ↕
┌────────┴─────────┐
│ Zabbix Server    │ ← Aju: kogub + töötleb
│ (10051)          │
└────────┬─────────┘
         ↕
┌────────┴─────────┐
│ Zabbix Web       │ ← Veebiliides (8080)
└──────────────────┘
         ↑
         │ küsib andmeid
         │
┌────────┴─────────┐
│ Zabbix Agent     │ ← Monitoorib süsteemi
│ (10050)          │    (CPU, RAM, disk)
└──────────────────┘
```

**Miks selline järjekord?**

Kui ehitad maja, alustad vundamendist:
1. **MySQL** – vundament (andmebaas)
2. **Zabbix Server** – tagumine osa (mootor)
3. **Zabbix Web** – esikülg (liides)
4. **Zabbix Agent** – mõõdik (mõõteseade)

---

## Osa 1: Projekti Alustamine (5 min)

```bash
mkdir zabbix-lab
cd zabbix-lab
nano docker-compose.yml
```

**Kopeeri see baas:**

```yaml
services:
  # Lisame komponendid järk-järgult

volumes:
  # Püsiv salvestus tuleb siia

networks:
  zabbix-net:
    driver: bridge
```

**💡 Miks bridge network?**  
DNS töötab automaatselt – `mysql-server` on hostname, mitte IP.

---

## Osa 2: MySQL – Andmebaas (10 min)

### Miks alustame andmebaasiga?

Zabbix Server vajab kohta kuhu config ja metrics salvestada. Ilma MySQL'ita server ei käivitu.

### Lisa docker-compose.yml faili:

```yaml
  mysql-server:
    image: mysql:8.0.39
    container_name: mysql-server
    environment:
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix_pwd
      MYSQL_ROOT_PASSWORD: root_pwd
      TZ: Europe/Tallinn
    command:
      - mysqld
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_bin
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - zabbix-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "-u", "root", "-p$$MYSQL_ROOT_PASSWORD"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s
```

**Lisa volumes sektsioon:**

```yaml
volumes:
  mysql-data:
```

### Selgitus rida-realt:

| Rida | Mis see teeb? | Miks oluline? |
|------|---------------|---------------|
| `MYSQL_DATABASE: zabbix` | Loob DB nimega "zabbix" | Server vajab seda nime |
| `TZ: Europe/Tallinn` | Ajatsoon | Logid ja timestamps õiged |
| `--character-set-server=utf8mb4` | UTF-8 support | Emoji ja mitme keele tugi |
| `volumes: mysql-data` | Andmed jäävad alles | Restart ei kustuta midagi |
| `healthcheck` | Kontrollib kas MySQL valmis | Server ootab selle signaali |
| `start_period: 60s` | Esimesed 60s eirab vigu | MySQL käivitub aeglaselt |

### Test:

```bash
docker compose up -d mysql-server
docker compose ps
```

**Ootus:** Status = `Up (healthy)`

```bash
docker exec mysql-server mysql -uzabbix -pzabbix_pwd -e "SHOW DATABASES;"
```

**Ootus:** Näed `zabbix` andmebaasi.

📝 **Q1:** Miks me ei lisa kohe kõiki komponente korraga?  
💡 **Vastus:** Kui miski failibki, oskad täpselt öelda kus. Debugging on 10× lihtsam.

---

## Osa 3: Zabbix Server – Aju (15 min)

### Mis see teeb?

Server on monitooringu mootor:
- Küsib agent'idelt andmeid (passive checks)
- Võtab vastu andmeid (active checks)
- Arvutab triggereid (alarmid)
- Salvestab kõik MySQL'i

### Lisa docker-compose.yml faili:

```yaml
  zabbix-server:
    image: zabbix/zabbix-server-mysql:ubuntu-7.0.6
    container_name: zabbix-server
    environment:
      DB_SERVER_HOST: mysql-server
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix_pwd
      TZ: Europe/Tallinn
    ports:
      - "10051:10051"
    networks:
      - zabbix-net
    restart: unless-stopped
    depends_on:
      mysql-server:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "nc -z 127.0.0.1 10051 || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 60s
```

### Selgitus:

| Rida | Mis see teeb? |
|------|---------------|
| `DB_SERVER_HOST: mysql-server` | DNS nimi, mitte IP! |
| `ports: 10051` | Agent'id saadavad siia andmeid |
| `depends_on: service_healthy` | Ootab MySQL valmis olemist |
| `nc -z 127.0.0.1 10051` | Kontrollib kas port avatud (parem kui `pgrep`) |

### Test:

```bash
docker compose up -d zabbix-server
docker compose logs -f zabbix-server
```

**Oota seda rida:**
```
Zabbix Server started. Zabbix 7.0.6 (revision xxxxx).
```

**Kontrolli andmebaasi:**
```bash
docker exec mysql-server mysql -uzabbix -pzabbix_pwd zabbix -e "SHOW TABLES;" | wc -l
```

**Ootus:** ~140 tabelit (server lõi skeemi ise)

📝 **Q2:** Miks server vajab esimesel käivitumisel aega?  
💡 **Vastus:** Peab looma 140 tabelit + indeksid. See võtab ~30-60s.

---

## Osa 4: Zabbix Web – Liides (10 min)

### Mis see teeb?

Web on sinu juhtpaneel:
- UI kust vaatad graafikuid
- Seadistusliides (hosts, templates, alerts)
- API kust saad andmeid välja

### Lisa docker-compose.yml faili:

```yaml
  zabbix-web:
    image: zabbix/zabbix-web-nginx-mysql:ubuntu-7.0.6
    container_name: zabbix-web
    environment:
      ZBX_SERVER_HOST: zabbix-server
      DB_SERVER_HOST: mysql-server
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix_pwd
      PHP_TZ: Europe/Tallinn
      TZ: Europe/Tallinn
    ports:
      - "8080:8080"
    networks:
      - zabbix-net
    restart: unless-stopped
    depends_on:
      mysql-server:
        condition: service_healthy
```

### Selgitus:

- `ZBX_SERVER_HOST` – kust küsida live andmeid
- `DB_SERVER_HOST` – kust lugeda config'i
- `PHP_TZ` – PHP skriptide ajatsoon
- `8080:8080` – avame pordi välja

### Test:

```bash
docker compose up -d zabbix-web
```

**Ava brauseris:** http://localhost:8080

**Login:**
```
Username: Admin
Password: zabbix
```

✅ **Kohustuslik:** Vaheta parool kohe!  
1. Kliki profiilipilti (ülemine parem nurk)
2. *Profile → Change password*
3. Uus parool: `Zabbix123!`

📝 **Q3:** Miks web vajab nii DB'd kui ka server'it?  
💡 **Vastus:** DB = config (hosts, users), Server = live data (metrics).

---

## Osa 5: Zabbix Agent – Mõõteseade (10 min)

### Mis see teeb?

Agent jookseb süsteemis mida me monitoorime:
- Kogub metrics (CPU, RAM, disk)
- Vastab server'i küsimustele
- Saadab andmeid (kui active mode)

### Lisa docker-compose.yml faili:

```yaml
  zabbix-agent:
    image: zabbix/zabbix-agent:ubuntu-7.0.6
    container_name: zabbix-agent
    environment:
      ZBX_SERVER_HOST: zabbix-server
      ZBX_HOSTNAME: docker-agent-01
      TZ: Europe/Tallinn
    networks:
      - zabbix-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "zabbix_agentd -t agent.ping | grep -q '1' || exit 1"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 15s
```

### Oluline muudatus:

❌ **EI:** `ZBX_HOSTNAME: "Zabbix server"`  
✅ **JAH:** `ZBX_HOSTNAME: docker-agent-01`

**Miks?** "Zabbix server" on *server'i enda* nimi Zabbix UI's. Kui paneme sama nime agent'ile, tekib collision hiljem.

### Selgitus:

| Rida | Mis see teeb? |
|------|---------------|
| `ZBX_SERVER_HOST` | Kellel on luba küsida (passive) |
| `ZBX_HOSTNAME` | Agent'i identifikaator |
| `zabbix_agentd -t agent.ping` | Test kas agent vastab |

### Test:

```bash
docker compose up -d zabbix-agent
```

**Kontrolli ühendust:**
```bash
docker exec zabbix-server zabbix_get -s zabbix-agent -k agent.ping
```

**Ootus:** `1`

📝 **Q4:** Mis vahe on `zabbix_get` testil ja päris monitooringul?  
💡 **Vastus:** `zabbix_get` teeb ühe päringu, ei salvesta. Monitoring küsib iga 60s + salvestab.

---

## Osa 6: Seadista Host Web'is (15 min)

### Nüüd ühendame agent'i Zabbix'iga!

**Sammud:**

1. **Ava:** *Data collection → Hosts*
2. **Vajuta:** *Create host* (paremal üleval)
3. **Täida:**

| Väli | Väärtus | Miks? |
|------|---------|-------|
| Host name | `docker-agent-01` | **Peab ühtima** agent'i `ZBX_HOSTNAME` väärtusega |
| Groups | `Linux servers` | Organiseerimiseks |
| Interfaces → Add | Type: `Agent` | Passive check'ide jaoks |
| - IP address | `zabbix-agent` | DNS nimi (mitte IP!) |
| - Port | `10050` | Agent'i default port |

4. **Templates tab:**
   - Vajuta *Select*
   - Otsi: `Linux by Zabbix agent`
   - Lisa see

5. **Salvesta:** *Add* (all pool)

### Kontrolli:

**Vaata:** *Data collection → Hosts*

**Availability veerg:**

| Ikoon | Tähendus |
|-------|----------|
| 🟢 ZBX | Ühendus OK! |
| 🔴 ZBX | Ei saa ühendust |
| ⚪ ZBX | Veel pole kontrollitud |

⏱️ **Oota ~60s** esimeseks kontrolliks.

### Debugging (kui punane):

```bash
# 1. Kas agent töötab?
docker ps | grep zabbix-agent

# 2. Kas port avatud?
docker exec zabbix-server nc -zv zabbix-agent 10050

# 3. Kas hostname õige?
docker exec zabbix-agent cat /etc/zabbix/zabbix_agentd.conf | grep Hostname
```

📝 **Q5:** Miks me ei kasuta IP aadressi?  
💡 **Vastus:** DNS on paindlikum. Container restart = uus IP, aga DNS jääb samaks.

---

## Osa 7: Vaata Mõõdikuid (5 min)

### Nüüd tuleb huvitav osa!

**Ava:** *Monitoring → Latest data*

**Filtreeri:**
- Host: `docker-agent-01`
- Apply

⏱️ **Oota 2-3 minutit** – esimesed andmed võtavad aega.

### Mida näed:

| Metric | Mis see on? |
|--------|-------------|
| `CPU utilization` | CPU kasutus % |
| `Available memory` | Vaba RAM |
| `Free disk space on /` | Vaba kettaruum |
| `System uptime` | Kui kaua süsteem töötanud |

### Testi CPU koormust:

```bash
docker exec zabbix-agent sh -c 'dd if=/dev/zero of=/dev/null & pid=$!; sleep 30; kill $pid 2>/dev/null'
```

**Kliki:** *Graph* ikoonile CPU juures  
✅ **Näed:** CPU spike graafikus!

---

## Lõpetuseks

### ✅ Sul on nüüd:

- MySQL andmebaas (püsiv salvestus)
- Zabbix Server (mootor)
- Zabbix Web (UI)
- Zabbix Agent (monitooringualune)

### 🎓 Õppisid:

1. **Arhitektuuri** – iga komponendi roll
2. **Healthcheck'id** – miks ootame valmis olemist
3. **DNS vs IP** – miks DNS parem
4. **Template'id** – valmis monitooring konfid
5. **Passive check** – server küsib, agent vastab
6. **Debugging** – kuidas leida vigu

### 🔐 Turvalisus:

❌ **Ära jäta default paroole!**  
✅ **Muuda kohe:**
- Zabbix Admin: `Admin/zabbix` → `Admin/Zabbix123!`
- MySQL root: `root_pwd` → midagi paremat

---

## Koristus

**Stop ilma andmeid kustutamata:**
```bash
docker compose stop
```

**Stop + kustuta kõik (ka andmebaas):**
```bash
docker compose down -v
```

**Kontrolli:**
```bash
docker volume ls | grep zabbix
```

---

## Küsimuste Vastused

**Q1:** Kui kõik lisada korraga ja miski crashib → kus viga? Järk-järgult = täpne debug.

**Q2:** Server loob esimesel käivitumisel ~140 DB tabelit. Võtab 30-60s.

**Q3:** Web vajab DB'd config'i jaoks (hosts, users) ja server'it live andmete jaoks (metrics).

**Q4:** `zabbix_get` on üks päring, ei salvesta. Monitoring = perioodiline + salvestamine.

**Q5:** DNS jääb samaks, ka kui container saab uue IP. Paindlikum ja loetavam.

---

## Järgmine Samm

**Lab 2:** Active vs Passive checks, templates, triggers

**Valmis?** Restart keskkond:
```bash
docker compose restart
```