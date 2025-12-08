# Linux Logimise Labor: Keskne Logiserver

**Kestus:** 3x45min klassis + kodus lõpetamine  
**Eesmärk:** Ehitad keskse logiserveri, kuhu teine masin saadab oma logid

---

## Sisukord

1. [Mida sa õpid?](#mida-sa-õpid)
2. [Ettevalmistus](#ettevalmistus)
3. [Samm 1: Paigalda rsyslog](#samm-1-paigalda-rsyslog)
4. [Samm 2: Seadista LogServer](#samm-2-seadista-logserver)
5. [Samm 3: Seadista LogClient](#samm-3-seadista-logclient)
6. [Samm 4: Testi!](#samm-4-testi)
7. [Tõrkeotsing](#tõrkeotsing)
8. [Lisaülesanne 1: TCP edastamine](#lisaülesanne-1-tcp-edastamine-10-min)
9. [Lisaülesanne 2: Auth logid eraldi](#lisaülesanne-2-auth-logid-eraldi-faili-10-min)
10. [Lisaülesanne 3: Logrotate](#lisaülesanne-3-logrotate-15-min)
11. [Boonusülesanne: auditd](#boonusülesanne-auditd-20-min)
12. [Esitamine](#esitamine)

---

## Mida sa õpid?

| Oskus | Miks oluline? |
|-------|---------------|
| rsyslog serveri seadistamine | Keskne koht kõikide logide jaoks |
| Kliendi → serveri edastamine | Logid turvaliselt teises kohas |
| Logide testimine ja debug | Kiire probleemide leidmine |
| Logrotate seadistamine | Ketas ei täitu |

---

## Arhitektuur

```
┌─────────────────┐                    ┌─────────────────┐
│   LogClient     │                    │   LogServer     │
│                 │      UDP 514       │                 │
│  rsyslog        │ ──────────────────▶│  rsyslog        │
│  (saadab)       │                    │  (kuulab)       │
│                 │                    │                 │
│  192.168.x.20   │                    │  192.168.x.10   │
└─────────────────┘                    └─────────────────┘
                                              │
                                              ▼
                                    /var/log/remote/syslog.log
```

---

## Ettevalmistus

Võta Proxmoxist **2 Ubuntu/Debian VM-i**.

| Roll | Hostname |
|------|----------|
| Server | logserver |
| Klient | logclient |

> ⚠️ **Ära unusta:** Seadista mõlemale VM-ile staatilised IP-d ja kontrolli, et nad pingivad teineteist!

---

## Samm 1: Paigalda rsyslog

> **Mida teeme:** Paigaldame rsyslog mõlemasse VM-i. See on logimise "mootor", mis kogub ja edastab logisid.

**Mõlemas VM-is:**

```bash
sudo apt update
sudo apt install -y rsyslog
```

### ✅ Valideerimine

```bash
systemctl status rsyslog
```

**Oodatud:** `Active: active (running)`

---

## Samm 2: Seadista LogServer

> **Mida teeme:** Seadistame serveri kuulama porti 514 ja salvestama saabuvad logid eraldi kausta. Nii ei sega kauglogid kohalikke logisid.

### 2.1 Luba võrgust logide vastuvõtt

```bash
sudo nano /etc/rsyslog.conf
```

Otsi ja eemalda kommentaar (`#`) nende ridade eest:

```bash
module(load="imudp")
input(type="imudp" port="514")
```

### 2.2 Loo kaust ja salvestamise reegel

```bash
sudo mkdir -p /var/log/remote
sudo chown syslog:adm /var/log/remote
```

```bash
sudo nano /etc/rsyslog.d/remote.conf
```

```bash
*.* /var/log/remote/syslog.log
```

### 2.3 Taaskäivita

```bash
sudo systemctl restart rsyslog
```

### ✅ Valideerimine

```bash
sudo ss -uln | grep 514
```

**Oodatud:** `UNCONN 0 0 0.0.0.0:514 0.0.0.0:*`

❌ **Ei tööta?** Kontrolli: `sudo rsyslogd -N1` ja `sudo journalctl -u rsyslog -n 20`

---

## Samm 3: Seadista LogClient

> **Mida teeme:** Ütleme kliendile, et ta saadaks kõik logid serveri IP-le. Üks `@` = UDP protokoll.

### 3.1 Lisa edastamise reegel

```bash
sudo nano /etc/rsyslog.d/forward.conf
```

```bash
*.* @<SERVERI-IP>:514
```

> ⚠️ Asenda `<SERVERI-IP>` oma serveri IP-ga!

### 3.2 Taaskäivita

```bash
sudo systemctl restart rsyslog
```

### ✅ Valideerimine

```bash
sudo rsyslogd -N1
```

**Oodatud:** `rsyslogd: End of config validation run. Bye.`

---

## Samm 4: Testi!

> **Mida teeme:** Saadame kliendist testsõnumi ja kontrollime, kas see jõuab serverisse. `logger` käsk saadab sõnumi otse syslog'i.

### Kliendis:

```bash
logger "TEST: Logi kliendilt - $(hostname)"
```

### Serveris:

```bash
# Variant 1: tail (lihtne)
sudo tail /var/log/remote/syslog.log

# Variant 2: tail -f (reaalajas jälgimine, Ctrl+C väljumiseks)
sudo tail -f /var/log/remote/syslog.log

# Variant 3: journalctl (rsyslog teenuse logid)
sudo journalctl -u rsyslog -f
```

### Proovi ka neid käske serveris:

```bash
# Otsi ainult ERROR sõnumeid
grep -i "error" /var/log/remote/syslog.log

# Loe viimased 20 rida
tail -n 20 /var/log/remote/syslog.log

# Sirvi faili (q = välju)
less /var/log/remote/syslog.log

# Loe mitu rida failis on
wc -l /var/log/remote/syslog.log
```

### ✅ Valideerimine

**Oodatud serveris:**
```
Dec 11 10:15:32 logclient sysadmin: TEST: Logi kliendilt - logclient
```

🎉 **Näed sõnumit?** Põhiosa valmis!

❌ **Ei näe?** Vaata [Tõrkeotsing](#tõrkeotsing)

---

## Tõrkeotsing

| Probleem | Kontrolli | Käsk |
|----------|-----------|------|
| Võrk ei tööta | Kas pingib? | `ping <serveri-ip>` |
| Server ei kuula | Port 514 lahti? | `sudo ss -uln \| grep 514` |
| Tulemüür blokeerib | UFW reegel | `sudo ufw allow 514/udp` |
| rsyslog ei tööta | Teenuse staatus | `systemctl status rsyslog` |
| Konfig viga | Süntaksi kontroll | `sudo rsyslogd -N1` |
| Logid rsyslog'is | Vaata vigu | `sudo journalctl -u rsyslog -n 20` |

**Võrguliikluse jälgimine (viimane abinõu):**
```bash
# Serveris
sudo tcpdump -i any port 514 -n
# Kliendis samal ajal
logger "TCPDUMP TEST"
```

---

## Lisaülesanne 1: TCP edastamine (10 min)

> **Miks:** UDP ei garanteeri kohaletoimetamist. TCP on aeglasem, aga usaldusväärsem - sobib kui logid on kriitilised.

### Serveris

Lisa `/etc/rsyslog.conf` faili:
```bash
module(load="imtcp")
input(type="imtcp" port="514")
```

```bash
sudo systemctl restart rsyslog
```

### Kliendis

Muuda `/etc/rsyslog.d/forward.conf` - lisa teine `@`:
```bash
*.* @@<SERVERI-IP>:514
```

```bash
sudo systemctl restart rsyslog
```

### ✅ Valideerimine

```bash
# Serveris - TCP port kuulab?
sudo ss -tln | grep 514

# Kliendis - testi
logger "TEST: TCP töötab!"
```

---

## Lisaülesanne 2: Auth logid eraldi faili (10 min)

> **Miks:** SSH ja sudo logid eraldi = lihtsam turvaintsidente uurida, ei pea tuhandete ridade seast otsima.

### Serveris

Muuda `/etc/rsyslog.d/remote.conf`:
```bash
auth,authpriv.* /var/log/remote/auth.log
*.* /var/log/remote/syslog.log
```

```bash
sudo systemctl restart rsyslog
```

### ✅ Valideerimine

```bash
# Kliendis - genereeri auth log
sudo su -c "echo test"

# Serveris
sudo tail /var/log/remote/auth.log
```

---

## Lisaülesanne 3: Logrotate (15 min)

> **Miks:** Ilma selleta kasvavad logid lõpmatult → ketas täis → süsteem crashib. Logrotate pöörab ja pakib vanad logid automaatselt.

### Serveris

Loo `/etc/logrotate.d/remote-logs`:
```bash
/var/log/remote/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 syslog adm
    postrotate
        /usr/bin/systemctl kill -s HUP rsyslog.service
    endscript
}
```

| Parameeter | Tähendus |
|------------|----------|
| `daily` | Pööra iga päev |
| `rotate 7` | Hoia 7 vana faili |
| `compress` | Paki .gz formaati |
| `postrotate` | Ütle rsyslog'ile avada uus fail |

### ✅ Valideerimine

```bash
sudo logrotate -f /etc/logrotate.d/remote-logs
ls -la /var/log/remote/
```

**Oodatud:** `syslog.log` (uus) + `syslog.log.1` (vana)

---

## Boonusülesanne: auditd (20 min)

> **Miks:** auditd jälgib KES ja MILLAL faile muutis. Vajalik compliance jaoks (GDPR, PCI-DSS) ja turvaintsidentide uurimiseks.

### Paigalda ja käivita

```bash
sudo apt install -y auditd
sudo systemctl enable --now auditd
```

### Seadista jälgimine

```bash
sudo auditctl -w /var/log/remote -p wa -k remote_logs
sudo auditctl -w /etc/rsyslog.conf -p wa -k rsyslog_config
```

### Testi

```bash
sudo touch /var/log/remote/test.txt
sudo ausearch -k remote_logs -i
```

### Proovi ka neid käske:

```bash
# Otsi tänaseid sündmusi
sudo ausearch -k remote_logs -ts today -i

# Üldine kokkuvõte
sudo aureport --summary

# Failide muudatuste aruanne
sudo aureport -f

# Kasutajate tegevuste aruanne
sudo aureport -u
```

### Tee püsivaks

Loo `/etc/audit/rules.d/logging.rules`:
```bash
-w /var/log/remote -p wa -k remote_logs
-w /etc/rsyslog.conf -p wa -k rsyslog_config
```

```bash
sudo augenrules --load
```

---

## Kontrollnimekiri

| Osa | Tehtud? |
|-----|---------|
| **Põhiosa** | |
| VM-id pingivad teineteist | ☐ |
| Server kuulab porti 514 | ☐ |
| `logger` sõnum jõuab serverisse | ☐ |
| **Lisaülesanded** | |
| TCP edastamine (`@@`) | ☐ |
| Auth logid eraldi failis | ☐ |
| Logrotate seadistatud | ☐ |
| **Boonus** | |
| auditd jälgib logide kausta | ☐ |

---

## Esitamine

**Google Classroom'i esita:**

1. **Screenshot'id:**
   - Server kuulab: `sudo ss -uln | grep 514`
   - Logid jõuavad: `logger` kliendis + `tail` serveris
   - Failid olemas: `ls -la /var/log/remote/`

2. **Konfiguratsioonifailid** (või repo link):
   - `/etc/rsyslog.d/remote.conf`
   - `/etc/rsyslog.d/forward.conf`
   - `/etc/logrotate.d/remote-logs` (kui tegid)

### Hindamine

| Osa | % |
|-----|---|
| Põhiosa töötab | 60% |
| TCP edastamine | 10% |
| Auth logid eraldi | 10% |
| Logrotate | 10% |
| Screenshot'id + konfid | 10% |
| **Boonus:** auditd | +10% |

---

## Abi vajad?

| Probleem | Lahendus |
|----------|----------|
| Konfig viga | `sudo rsyslogd -N1` |
| rsyslog vead | `sudo journalctl -u rsyslog -f` |
| Muu | Küsi õpetajalt |

---

## Kasulikud viited

| Teema | Link |
|-------|------|
| rsyslog dokumentatsioon | https://www.rsyslog.com/doc/v8-stable/ |
| rsyslog konfiguratsioon | https://www.rsyslog.com/doc/v8-stable/configuration/index.html |
| logrotate manual | https://man7.org/linux/man-pages/man8/logrotate.8.html |
| journalctl manual | https://www.freedesktop.org/software/systemd/man/journalctl.html |
| auditd dokumentatsioon | https://man7.org/linux/man-pages/man8/auditd.8.html |
| ausearch manual | https://man7.org/linux/man-pages/man8/ausearch.8.html |
| Ubuntu Server logimine | https://ubuntu.com/server/docs/about-logging |