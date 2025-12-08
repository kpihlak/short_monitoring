# Linux Logimine ja Haldamine: Põhitõed

**Eeldused:** Linux põhikäsud, terminali kasutamine • **Kestus:** 45 min loeng

---

## Sisukord

1. [Sissejuhatus Linux logimisse](#1-sissejuhatus-linux-logimisse)
2. [Kust logid tulevad?](#2-kust-logid-tulevad)
3. [Linux logimise põhikomponendid](#3-linux-logimise-põhikomponendid)
4. [Syslog protokoll: Facility ja Severity](#4-syslog-protokoll-facility-ja-severity)
5. [Keskne logiserver](#5-keskne-logiserver)
6. [Parimad praktikad](#6-parimad-praktikad)
7. [Kokkuvõte](#7-kokkuvõte)

---

# 1. Sissejuhatus Linux logimisse

## Mis on logimine?

Logimine on protsess, mille käigus salvestatakse süsteemi või rakenduse sündmused, tegevused ja teated. See on oluline osa igast operatsioonisüsteemist, sealhulgas Linuxist. Logid salvestatakse kronoloogiliselt ja jagatakse erinevatesse kategooriatesse.

Tüüpiline logisõnum sisaldab nelja põhikomponenti: ajatempel (millal sündmus toimus), allikas (milline teenus või rakendus logi genereeris), sõnumi tase (näiteks INFO, WARNING, ERROR) ja sõnumi sisu ise.

Näide logisõnumist:
```
May 7 10:23:45 myserver sshd[12345]: Failed password for invalid user test from 192.168.1.100 port 54321 ssh2
```

Siin näeme, et 7. mail kell 10:23:45 prooviti serverisse `myserver` sisse logida kasutajaga `test` IP-aadressilt 192.168.1.100, kuid parool oli vale.

## Miks on logimine oluline?

Kujuta ette: sinu firma veebileht on maas. Juht küsib: "Mis juhtus? Millal see algas? Kes tegi viimati muudatusi?" 

Ilma logideta vastad: "Ei tea." 

Logidega vastad: "Kell 14:32 hakkas andmebaas tagastama vigu, sest ketas sai täis. Viimane deployment oli kell 14:15 kasutaja admin poolt."

Logimine on kriitilise tähtsusega mitmel põhjusel:

**Vigade tuvastamine ja lahendamine.** Kui rakendus crashib, logid näitavad täpset viga ja asukohta. Saad jälgida, millal probleem tekkis ja mis selle põhjustas. Näiteks kui näed logis "Database connection timeout", saad kohe uurida andmebaasi poole.

**Turvalisuse tagamine.** Logidest näed kõiki sisselogimise katseid - nii õnnestunuid kui ebaõnnestunuid. Kui näed 1000 ebaõnnestunud SSH sisselogimist 5 minutiga samalt IP-lt, on see selge märk brute-force rünnakust.

**Süsteemi jõudluse jälgimine.** Logid näitavad, millal süsteem muutus aeglaseks, kus on kitsaskohad ja kui palju ressursse on vaja.

**Vastavus regulatsioonidele.** GDPR nõuab logimist - kes milliseid isikuandmeid vaatas. Panganduses peab iga tehingu logi säilima 7 aastat. Auditite ajal küsivad inspektorid: "Näita, mis 15. märtsil kell 10:00 juhtus."

## Peamised logifailid Linuxis

Linuxis asuvad logifailid kataloogis `/var/log/`. Iga logifail on mõeldud teatud tüüpi sündmuste jaoks:

| Logifail | Mida sealt leida |
|----------|------------------|
| `/var/log/syslog` | Kõik süsteemi sündmused (üldine logi) |
| `/var/log/auth.log` | SSH sisselogimised, sudo kasutamine |
| `/var/log/kern.log` | Kerneli sõnumid, riistvara vead |
| `/var/log/dmesg` | Boot protsess, draiverid |
| `/var/log/cron` | Ajastatud ülesannete täitmine |
| `/var/log/nginx/` | Veebserveri päringud ja vead |

---

# 2. Kust logid tulevad?

Enne kui hakkad logisid lugema ja analüüsima, on oluline mõista, kuidas logimine Linuxis tegelikult töötab. Kust need logisõnumid tulevad ja kuidas nad jõuavad failidesse?

## Kaks erinevat maailma: Kernel Space ja User Space

Linux jagab mälu kaheks osaks: **kernel space** (tuuma ruum) ja **user space** (kasutaja ruum). See jaotus kehtib ka logimise kohta - meil on kaks erinevat logimise mehhanismi.

![Linux User Space vs Kernel Space](https://devconnected.com/wp-content/uploads/2019/11/linux-spaces-768x397.png)
*Allikas: devconnected.com*

**Kernel logging** tegeleb kerneli ehk tuumaga seotud sõnumitega - riistvara vead, draiverite probleemid, boot protsessi info. Need logid tekivad süsteemi kõige madalamal tasemel.

**User logging** tegeleb kasutajaruumi protsesside ja teenustega - veebiserverid, andmebaasid, SSH, sinu enda rakendused. See põhineb Syslog protokollil.

## Kernel Logging: Ring Buffer

Kerneli logid salvestatakse struktuuri nimega **kernel ring buffer**. See on ringpuhver, mis hakkab täituma kohe, kui süsteem käivitub - veel enne, kui kõvaketas on üldse kättesaadav.

![Kernel Logging internals](https://devconnected.com/wp-content/uploads/2019/11/kernel-logging-internals-768x557.png)
*Allikas: devconnected.com*

Kui sa käivitad arvuti ja näed ekraanil jooksvaid sõnumeid riistvara tuvastamise kohta - need kõik salvestatakse ring buffer'isse. Hiljem saad neid vaadata `dmesg` käsuga.

Tehniline pool on selline: `/dev/kmsg` on virtuaalne seade, mis toimib sissepääsuna ring buffer'isse. Logging utiliidid (syslogd, rsyslog) loevad sealt andmeid ja salvestavad need failidesse.

![Kernel Logging Complete](https://devconnected.com/wp-content/uploads/2019/11/internals-3-768x560.png)
*Allikas: devconnected.com - Täielik kerneli logimise arhitektuur*

## User Space Logging: Syslog protokoll

Kasutajaruumi logimine põhineb **Syslog protokollil**, mis loodi juba 1980. aastatel Eric Allman'i poolt. See on standard, mis määrab, kuidas logisõnumeid vormindada, saata ja vastu võtta.

![Syslog protocol](https://devconnected.com/wp-content/uploads/2019/11/syslog-card.png)
*Allikas: devconnected.com*

Syslog defineerib kolm põhilist rolli:

**Originator** (algataja) - see on rakendus, mis loob logisõnumeid. Näiteks Nginx, SSH deemon või sinu enda skript.

**Relay** (vahendaja) - valikuline komponent, mis võtab logisid vastu ja saadab need edasi. Võib logisid ka rikastada või filtreerida. Näited: Logstash, Fluentd.

**Collector** (koguja) - server, mis võtab logid vastu ja salvestab need. See võib kirjutada failidesse, andmebaasi või saata edasi analüüsitööriistadele.

![Syslog Architecture](https://devconnected.com/wp-content/uploads/2019/11/syslog-1-1920x383.png)
*Allikas: devconnected.com - Originator → Relay → Collector*

## Kuidas see kõik kokku töötab?

Kaasaegses Linux süsteemis (Ubuntu, Debian jt) töötavad **kaks logimissüsteemi kõrvuti**: rsyslog ja systemd-journal.

![User Space Logging Architecture](https://devconnected.com/wp-content/uploads/2019/11/linux-logging-2.png)
*Allikas: devconnected.com - rsyslog ja systemd-journal koostöö*

**rsyslog** on traditsiooniline Syslog'i teostus. Ta kuulab spetsiaalsel socket'il, võtab logisõnumeid vastu ja salvestab need tekstifailidesse kataloogis `/var/log/`. Rsyslog oskab ka logisid võrgu kaudu teistesse serveritesse saata.

**systemd-journal** (journald) tuli koos systemd'ga. Ta salvestab logid binaarformaadis andmebaasi, mis võimaldab kiiret otsingut ja struktureeritud päringuid. Vaikimisi ei säilita journald logisid püsivalt - need kaovad taaskäivitusel.

Need kaks süsteemi on mõeldud koos töötama. Journald kogub logisid systemd teenustelt ja rsyslog saab neid sealt lugeda, et salvestada traditsioonilistesse logifailidesse.

---

# 3. Linux logimise põhikomponendid

Nüüd, kui mõistad üldist arhitektuuri, vaatame lähemalt nelja põhilist tööriista, millega Linux administraator peab tuttav olema.

## 3.1 rsyslog

**rsyslog** (Rocket-fast System for Log processing) on peamine logikoguja enamikus Linux distributsioonides. Ta on kiire, paindlik ja toetab palju erinevaid sisend- ja väljundformaate.

![rsyslog info](https://devconnected.com/wp-content/uploads/2019/11/rsyslog-card.png)
*Allikas: devconnected.com*

Rsyslog töötab moodulite põhimõttel:

| Mooduli tüüp | Prefiks | Näited | Otstarve |
|--------------|---------|--------|----------|
| Sisendmoodulid | `im` | `imudp`, `imtcp`, `imjournal` | Koguvad logisid erinevatest allikatest |
| Väljundmoodulid | `om` | `omfile`, `ommysql`, `omfwd` | Saadavad logisid sihtkohtadesse |

Rsyslog'i konfiguratsioon asub failis `/etc/rsyslog.conf` ja lisakonfiguratsioonid kataloogis `/etc/rsyslog.d/`. Seal määrad reeglid, mis otsustavad, millised logid kuhu salvestatakse.

## 3.2 journald

**journald** on systemd osa ja pakub struktureeritud logimist. Erinevalt rsyslog'ist, mis salvestab tekstifaile, hoiab journald logisid binaarformaadis andmebaasis.

See tähendab, et sa ei saa journald logisid lugeda tavalise `cat` või `grep` käsuga - pead kasutama `journalctl` tööriista. Selle eest on otsing palju kiirem, sest andmed on indekseeritud.

| Omadus | rsyslog | journald |
|--------|---------|----------|
| Formaat | Tekstifailid | Binaarfailid |
| Asukoht | `/var/log/*.log` | `/var/log/journal/` |
| Vaatamine | `cat`, `grep`, `tail` | `journalctl` |
| Võrgu tugi | Native | Vajab rsyslog'i |

Peamine erinevus: rsyslog sobib paremini pikaajalise säilitamise ja võrgu kaudu edastamise jaoks, journald sobib paremini kiireks lokaalseks otsinguks ja systemd teenuste logimiseks.

## 3.3 logrotate

**logrotate** lahendab lihtsa, aga kriitilise probleemi: logifailid kasvavad lõputult.

Kujuta ette, et sinu Nginx server teenindab tuhandeid päringuid päevas. Iga päring = rida logifailis. Kuu ajaga võib `access.log` kasvada gigabaitide suuruseks. Lõpuks saab ketas täis ja süsteem crashib.

Logrotate pöörab faile automaatselt: praegune `access.log` nimetatakse ümber `access.log.1`, vana `.1` saab `.2` jne. Vanad failid kompresseeritakse ja lõpuks kustutatakse. Nii püsib kettakasutus kontrolli all.

| Parameeter | Tähendus |
|------------|----------|
| `daily/weekly` | Pööramise sagedus |
| `rotate 7` | Säilita 7 vana faili |
| `compress` | Paki vanad `.gz` formaati |
| `missingok` | Ära viska viga kui puudub |

## 3.4 auditd

**auditd** on Linuxi tuuma auditisubsüsteem ja vastab küsimusele, mida teised logimissüsteemid ei vasta: **KES täpselt MIDA tegi?**

Rsyslog ütleb sulle, et fail muutus. Auditd ütleb, kes seda muutis, mis protsess, mis kellaajal, milliste õigustega.

See on hädavajalik compliance jaoks. Kui auditor küsib "Näita mulle, kes vaatas klientide andmeid eelmisel nädalal" - auditd on see, mis annab vastuse. GDPR, PCI-DSS ja teised regulatsioonid nõuavad sellist detailset jälgimist.

---

# 4. Syslog protokoll: Facility ja Severity

Enne kui saad rsyslog reegleid kirjutada, pead mõistma kahte põhikontseptsiooni: **facility** ja **severity**.

## Miks see on oluline?

Kujuta ette, et sul on üks suur `/var/log/syslog` fail, kuhu lähevad KÕIK logid - mail server, SSH, cron, kernel, Nginx, MySQL... See fail kasvab 10GB päevas ja sealt millegi leidmine on võimatu.

Facility ja severity võimaldavad logisid kategoriseerida ja suunata erinevatesse failidesse. SSH logid lähevad `auth.log` faili, kerneli logid `kern.log` faili, ja kui tahad näha ainult vigu, saad filtreerida severity järgi.

## Facility - kust sõnum tuli?

Facility näitab, milline süsteemi osa või programm logi genereeris. Igal facility'l on oma number.

| Facility | Kasutus |
|----------|---------|
| `kern` | Kerneli sõnumid |
| `auth` / `authpriv` | Autentimine - SSH, sudo, login |
| `mail` | Meilisüsteem |
| `daemon` | Süsteemi deemonid (Apache, Nginx jt) |
| `cron` | Ajastatud ülesanded |
| `local0` - `local7` | Kohandatud kasutuseks |

Facility'd `local0` kuni `local7` on reserveeritud sinu enda rakenduste jaoks. Kui kirjutad skripti, mis peaks logisid saatma, kasutad üht neist.

## Severity - kui tõsine see on?

Severity näitab sõnumi tõsidust skaalal 0-7, kus 0 on kõige kriitilisem.

| Nr | Severity | Millal kasutada |
|----|----------|-----------------|
| 0 | `emerg` | Süsteem on täiesti maas |
| 1 | `alert` | Vaja tegutseda kohe |
| 2 | `crit` | Kriitiline viga |
| 3 | `err` | Viga, aga süsteem töötab |
| 4 | `warning` | Hoiatus - võib probleemiks muutuda |
| 5 | `notice` | Normaalne, aga oluline |
| 6 | `info` | Informatiivne |
| 7 | `debug` | Silumisinfo |

Praktiline reegel: kui ketas on 90% täis, on see `warning`. Kui teenus ei käivitu, on see `crit` või `err`. `emerg` kasutad ainult siis, kui kogu süsteem on ohus.

## Kuidas reegleid kirjutada?

Rsyslog reegel koosneb kahest osast: **filter** (mis logisid valida) ja **action** (kuhu need saata).

![Rsyslog Rules](https://devconnected.com/wp-content/uploads/2019/11/rsyslog-rules-585x331.png)
*Allikas: devconnected.com - facility.severity → destination*

Põhisüntaks:
```
facility.severity    action
```

Näited:
- `auth.*` → kõik autentimise logid, iga severity
- `*.err` → kõik vead, igast facility'st  
- `mail.info` → ainult mail server info tase
- `*.*;mail.none` → kõik logid, välja arvatud mail

---

# 5. Keskne logiserver

Siiani oleme rääkinud logimisest ühes masinas. Aga mis siis, kui sul on 10, 50 või 100 serverit?

## Miks on keskne logiserver vajalik?

**Turvalisus.** Kui ründaja pääseb serverisse, on esimene asi mida ta teeb - logide kustutamine, et jälgi peita. Kui logid on juba teises serveris, ei saa ta neid kustutada.

**Analüüs ja korrelatsioon.** Kui andmebaas crashib samal ajal kui veebserver hakkab vigu andma, tahad näha mõlema serveri logisid kõrvuti. Keskse logiserveri puhul on see lihtne.

**Varundamine.** Lihtsam on varundada ühte logiserveri kui 50 erinevat masinat.

**Pikaajaline säilitamine.** Tootmisserverites hoiad logisid ehk 1 kuu (kettaruum on kallis). Keskses logiserveris saad hoida 1 aasta.

## Arhitektuur

Tüüpiline keskse logiserveri ülesehitus on lihtne:

```
┌──────────────┐
│ Web Server 1 │───┐
└──────────────┘   │
┌──────────────┐   │     ┌──────────────┐
│ Web Server 2 │───┼────▶│   Keskne     │
└──────────────┘   │     │  Logiserver  │
┌──────────────┐   │     │  (rsyslog)   │
│  Database    │───┘     └──────────────┘
└──────────────┘
```

Kõik serverid (kliendid) saadavad oma logid ühte kohta (logiserver). Logiserver salvestab need ja võib edasi saata analüüsitööriistadele nagu Elasticsearch/Kibana või Grafana Loki.

## UDP vs TCP

Logisid saab saata kahel viisil:

**UDP (port 514)** on kiirem ja lihtsam. "Fire and forget" - klient saadab sõnumi ja ei oota vastust. Miinus: kui võrk on ülekoormatud, võivad sõnumid kaduda.

**TCP (port 514)** on usaldusväärsem. Server kinnitab iga sõnumi kättesaamist. Miinus: aeglasem ja keerulisem.

Praktikas: kui logid on kriitilised (turvalisus, audit), kasuta TCP-d. Kui kiirus on olulisem (debug logid), sobib UDP.

> **Laboris** seadistad keskse logiserveri kahe VM-iga ja näed, kuidas see praktikas töötab!

---

# 6. Parimad praktikad

## Turvalisus

Logifailid võivad sisaldada tundlikku infot - IP-aadresse, kasutajanimesid, mõnikord isegi paroole (kui rakendus on halvasti kirjutatud). 

Failide õigused peavad olema piiratud. Tüüpiline seadistus on `640` (omanik loeb/kirjutab, grupp loeb) ja owner `syslog:adm`. Auth logid võivad olla veelgi piiratumad - `600`.

Kui saadad logisid üle võrgu, kasuta TLS krüpteerimist. Tavaline UDP port 514 on krüptimata - keegi võib logisid pealt kuulata või võltsida.

## Säilitamine

Kui kaua logisid säilitada? See sõltub logi tüübist ja regulatsioonidest:

- **Auth logid** - 1-2 aastat (turvalisus, audit)
- **Süsteemilogid** - 30-90 päeva (troubleshooting)
- **Debug logid** - 7 päeva (need võtavad palju ruumi)
- **Audit logid** - 7+ aastat (GDPR, PCI-DSS nõuded)

## Jõudlus

Liiga palju logimist võib süsteemi aeglustada. Kui näed, et ketas on 100% koormatud ja rsyslog kasutab palju CPU-d, kontrolli:

- Kas debug logid on production'is sees? Lülita välja.
- Kas logrotate töötab? Võib-olla on failid liiga suured.
- Kas ketas on täis? Vabasta ruumi või suurenda rotate count'i.

---

# 7. Kokkuvõte

## Mida me õppisime?

Selles loengus käsitlesime Linux logimise põhitõdesid:

1. **Kust logid tulevad** - kernel space (ring buffer, dmesg) ja user space (syslog protokoll)
2. **rsyslog** - peamine logikoguja, mis salvestab tekstifailidesse
3. **journald** - systemd logid binaarformaadis, kiire otsing
4. **logrotate** - automaatne failide pööramine, et ketas ei saaks täis
5. **auditd** - turva-audit, "kes mida tegi"
6. **Facility ja Severity** - kuidas logisid kategoriseerida
7. **Keskne logiserver** - kõik logid ühes kohas

## Järgmine samm

**Laboris** rakendad kõike praktiliselt:
- Seadistad keskse logiserveri kahe VM-iga
- Konfigureerid rsyslog'i klient → server edastust
- Testid logger käsuga, kas logid jõuavad kohale
- Seadistad logrotate reeglid

**Kodutöös** lood bash skriptid logide analüüsiks ja monitoorimiseks.

---

## Kasulikud viited

- [rsyslog dokumentatsioon](https://www.rsyslog.com/doc/index.html)
- [journalctl manual](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [logrotate manual](https://man7.org/linux/man-pages/man8/logrotate.8.html)
- [Linux Logging Complete Guide](https://devconnected.com/linux-logging-complete-guide/) - artikkel koos diagrammidega

---

**Valmis laboriks!**