# Linux Logimise Kodutöö: Logide Dashboard

**Eeldus:** Põhilab on tehtud (keskne logiserver töötab)  
**Tähtaeg:** Enne järgmist tundi  
**Kestus:** ~1h

---

## Eesmärk

Lood bash skriptid, mis annavad kiire ülevaate logiserveri seisundist. Päris IT-keskkonnas kasutatakse selleks Grafana/Kibana dashboarde, aga põhimõte on sama.

---

## Ülesanne 1: Põhi-dashboard (kohustuslik)

Loo serveris skript `/opt/log_dashboard.sh`:

```bash
sudo nano /opt/log_dashboard.sh
```

```bash
#!/bin/bash

# Logide Dashboard
# Käivita: sudo /opt/log_dashboard.sh

LOG_DIR="/var/log/remote"
LOG_FILE="$LOG_DIR/syslog.log"
REFRESH=5

# Kontrolli kas logifail eksisteerib
if [ ! -f "$LOG_FILE" ]; then
    echo "VIGA: $LOG_FILE ei eksisteeri!"
    exit 1
fi

while true; do
    clear
    
    echo "╔════════════════════════════════════════════════════════╗"
    echo "║         LOGISERVERI DASHBOARD - $(date '+%Y-%m-%d %H:%M:%S')       ║"
    echo "╚════════════════════════════════════════════════════════╝"
    echo ""
    
    # --- STATISTIKA ---
    TOTAL=$(wc -l < "$LOG_FILE" 2>/dev/null || echo "0")
    ERRORS=$(grep -ic "error" "$LOG_FILE" 2>/dev/null || echo "0")
    WARNINGS=$(grep -ic "warn" "$LOG_FILE" 2>/dev/null || echo "0")
    
    echo "📊 STATISTIKA:"
    echo "   Kokku logisid:  $TOTAL"
    echo "   Vigu:           $ERRORS"
    echo "   Hoiatusi:       $WARNINGS"
    echo ""
    
    # --- KETTAKASUTUS ---
    echo "💾 KETTAKASUTUS:"
    df -h "$LOG_DIR" 2>/dev/null | tail -1 | awk '{printf "   Kasutatud: %s / %s (%s)\n", $3, $2, $5}'
    du -sh "$LOG_DIR" 2>/dev/null | awk '{printf "   Logide suurus: %s\n", $1}'
    echo ""
    
    # --- TOP KLIENDID ---
    echo "🖥️  TOP 5 KLIENDID (viimased 500 rida):"
    tail -500 "$LOG_FILE" 2>/dev/null | \
        awk '{print $4}' | \
        sed 's/:$//' | \
        sort | uniq -c | sort -rn | head -5 | \
        while read count host; do
            printf "   %-25s %6d sõnumit\n" "$host" "$count"
        done
    echo ""
    
    # --- VIIMASED VEAD ---
    echo "❌ VIIMASED VEAD:"
    grep -i "error" "$LOG_FILE" 2>/dev/null | tail -3 | \
        cut -c 1-75 | sed 's/^/   /' || echo "   (vigu ei leitud)"
    echo ""
    
    echo "─────────────────────────────────────────────────────────"
    echo "  Uuendamine iga ${REFRESH}s | Ctrl+C = välju"
    
    sleep $REFRESH
done
```

```bash
sudo chmod +x /opt/log_dashboard.sh
```

### Testimine

```bash
# Käivita dashboard
sudo /opt/log_dashboard.sh
```

Teises terminalis (kliendis) genereeri logisid:
```bash
# Tavalised logid
for i in {1..10}; do logger "Test logi #$i"; done

# Veateated
for i in {1..5}; do logger -p user.err "ERROR: Test viga #$i"; done

# Hoiatused
for i in {1..5}; do logger -p user.warning "WARNING: Test hoiatus #$i"; done
```

---

## Ülesanne 2: Vigade analüüsija (kohustuslik)

Loo serveris skript `/opt/analyze_logs.sh`:

```bash
sudo nano /opt/analyze_logs.sh
```

```bash
#!/bin/bash

# Logide analüüsija
# Käivita: sudo /opt/analyze_logs.sh

LOG_FILE="/var/log/remote/syslog.log"

if [ ! -f "$LOG_FILE" ]; then
    echo "VIGA: $LOG_FILE ei eksisteeri!"
    exit 1
fi

echo "═══════════════════════════════════════════════════"
echo "        LOGIDE ANALÜÜS - $(date '+%Y-%m-%d %H:%M')"
echo "═══════════════════════════════════════════════════"
echo ""

# Üldstatistika
echo "📈 ÜLDSTATISTIKA:"
echo "   Ridu kokku:    $(wc -l < "$LOG_FILE")"
echo "   Faili suurus:  $(du -h "$LOG_FILE" | cut -f1)"
echo ""

# Logitasemed
echo "📊 LOGITASEMED:"
echo "   ERROR:   $(grep -ic 'error' "$LOG_FILE")"
echo "   WARNING: $(grep -ic 'warn' "$LOG_FILE")"
echo "   INFO:    $(grep -ic 'info' "$LOG_FILE")"
echo ""

# Top 10 kliendid
echo "🖥️  TOP 10 KLIENDID:"
awk '{print $4}' "$LOG_FILE" | sed 's/:$//' | \
    sort | uniq -c | sort -rn | head -10 | \
    while read count host; do
        printf "   %-30s %8d\n" "$host" "$count"
    done
echo ""

# Logid tunni kaupa (täna)
echo "⏰ LOGID TUNNI KAUPA (täna):"
TODAY=$(date '+%b %d')
for hour in $(seq -w 0 23); do
    count=$(grep "^$TODAY $hour:" "$LOG_FILE" 2>/dev/null | wc -l)
    if [ "$count" -gt 0 ]; then
        bar=$(printf '%*s' $((count / 10)) '' | tr ' ' '█')
        printf "   %s:00  %6d  %s\n" "$hour" "$count" "$bar"
    fi
done
echo ""

# Viimased 5 viga
echo "❌ VIIMASED 5 VIGA:"
grep -i "error" "$LOG_FILE" | tail -5 | \
    while read line; do
        echo "   ${line:0:70}"
    done
echo ""

echo "═══════════════════════════════════════════════════"
```

```bash
sudo chmod +x /opt/analyze_logs.sh
```

### Testimine

```bash
sudo /opt/analyze_logs.sh
```

---

## Ülesanne 3: Klientide raport (boonus)

Loo serveris skript `/opt/client_report.sh`:

```bash
sudo nano /opt/client_report.sh
```

```bash
#!/bin/bash

# Klientide raport
# Käivita: sudo /opt/client_report.sh [kliendi_nimi]

LOG_FILE="/var/log/remote/syslog.log"

if [ ! -f "$LOG_FILE" ]; then
    echo "VIGA: $LOG_FILE ei eksisteeri!"
    exit 1
fi

# Kui anti kliendi nimi, näita ainult selle kliendi logisid
if [ -n "$1" ]; then
    CLIENT="$1"
    echo "═══════════════════════════════════════════════════"
    echo "        KLIENDI RAPORT: $CLIENT"
    echo "═══════════════════════════════════════════════════"
    echo ""
    
    COUNT=$(grep " $CLIENT " "$LOG_FILE" | wc -l)
    ERRORS=$(grep " $CLIENT " "$LOG_FILE" | grep -ic "error")
    
    echo "📊 STATISTIKA:"
    echo "   Logisid kokku: $COUNT"
    echo "   Vigu:          $ERRORS"
    echo ""
    
    echo "📝 VIIMASED 10 LOGI:"
    grep " $CLIENT " "$LOG_FILE" | tail -10 | \
        while read line; do
            echo "   ${line:0:70}"
        done
    
    exit 0
fi

# Muidu näita kõiki kliente
echo "═══════════════════════════════════════════════════"
echo "        KÕIK KLIENDID - $(date '+%Y-%m-%d %H:%M')"
echo "═══════════════════════════════════════════════════"
echo ""

echo "🖥️  KLIENDID:"
printf "   %-30s %10s %10s\n" "KLIENT" "LOGID" "VEAD"
echo "   ─────────────────────────────────────────────────"

awk '{print $4}' "$LOG_FILE" | sed 's/:$//' | sort -u | \
    while read client; do
        total=$(grep " $client " "$LOG_FILE" | wc -l)
        errors=$(grep " $client " "$LOG_FILE" | grep -ic "error" || echo "0")
        printf "   %-30s %10d %10d\n" "$client" "$total" "$errors"
    done

echo ""
echo "   Kokku kliente: $(awk '{print $4}' "$LOG_FILE" | sed 's/:$//' | sort -u | wc -l)"
echo ""
```

```bash
sudo chmod +x /opt/client_report.sh
```

### Testimine

```bash
# Kõik kliendid
sudo /opt/client_report.sh

# Konkreetne klient
sudo /opt/client_report.sh logclient
```

---

## Esitamine

**Google Classroom'i esita:**

1. **Screenshot dashboard'ist** töötamas (koos andmetega)
2. **Screenshot analyze_logs.sh** väljundist
3. **Skriptide failid** (või repo link)

### Hindamine

| Ülesanne | Punktid |
|----------|---------|
| Dashboard töötab | 40% |
| Analyze_logs töötab | 40% |
| Client_report (boonus) | +20% |

---

## Tõrkeotsing

| Probleem | Lahendus |
|----------|----------|
| "Permission denied" | Käivita `sudo`-ga |
| "File not found" | Kontrolli kas `/var/log/remote/syslog.log` eksisteerib |
| Tühi väljund | Genereeri kõigepealt logisid kliendist |
| Skript ei käivitu | Kontrolli: `chmod +x /opt/skript.sh` |

---

## Kasulikud viited

| Teema | Link |
|-------|------|
| Bash skriptimine | https://www.gnu.org/software/bash/manual/bash.html |
| awk tutorial | https://www.grymoire.com/Unix/Awk.html |
| grep manual | https://man7.org/linux/man-pages/man1/grep.1.html |
