
# Backup-pro

## Ausgangslage

Schreiben eines Skripts mit einer kleinen Dokumentation.

#### Zielformulierung

1. Es soll ein Skript für ein Backup task erstellt werden.
2. Im Backup nahme soll das Erstellungsdatum stehen.
3. Der Code sollte erläutert werden, was dieser macht.
4. Die Automatisierung wird auch Dokumentiert

### Infrastuktur 
- localhost Server (192.168.1.120)
- (Remote) Pc (192.168.1.117)

#### Umgebungsvariablen
- Datei "b1.txt" ist von dem Remote-pc auf den localserver zu backupen.

#### Pakete, die installiert werden sollen
- Remote-Gerät (192.168.1.117): openssh-server
- sshpass localhost

#### --- Remote-Gerät: SSH-Server installieren & aktivieren ---
sudo apt install -y openssh-server
sudo systemctl enable --now ssh         # Dienst starten & beim Boot aktivieren
systemctl status ssh --no-pager         # Status prüfen

#### --- Skript-Rechner: sshpass installieren (optional) ---
sudo apt install -y sshpass
sshpass -V                              # Version prüfen (optional)

## Backupcode (backup.sh) und Beschreibung
### Source-Code
#!/usr/bin/env bash
set -Eeuo pipefail

### ================================
###   Einstellungen Quelle / Ziel
### ================================
REMOTE_USER="marcel"                       # Remote-User*
REMOTE_HOST="192.168.1.117"                # Remote-Host / IP*
REMOTE_FILE="/home/marcel/b1.txt"          # Remote-Datei Die zu Backupen ist*
LOCAL_DIR="/home/marcel/Desktop/backuppool"  # Lokaler Zielordner der gebacupten datei (ohne Leerzeichen!)*
LOG_FILE="/var/log/remote_b1_backup.log"   # Logdatei (Root-Recht nötig)

### ================================
###   SSH-Setup
### ================================
SSH_PORT=22
SSH_KEY=""                                  # Angeben wenn du mitels SSH-Schlüssel die anmelden wilst. z.b /home/server1/.ssh/id_ed25519; leer -> Standard-Key/Agent

#### Optional: Wenn das Skript per sudo läuft und kein SSH_KEY gesetzt ist, versuche Benutzer-Key zu nutzen.
if [[ -z "${SSH_KEY}" && "${SUDO_USER-}" && -r "/home/${SUDO_USER}/.ssh/id_ed25519" ]]; then
  SSH_KEY="/home/${SUDO_USER}/.ssh/id_ed25519"
fi

### ================================
###   OPTIONAL: Passwort-Login (sshpass)  Sicherer ist Key-Login -> beide Felder leer lassen!
### ================================
PASSWORD=""                                 # Möglichkeit: Passwort anzugeben.
PASSWORD_FILE=""                            # Besser: Datei mit NUR dem Passwort, z.B. /root/.ssh/remote_pw (600)

### ================================
###   Vorbereitung
### ================================
mkdir -p "$LOCAL_DIR"                       # Setzt angegebener Speicherort.

###### SSH-Optionen (Fingerprints beim ersten Mal automatisch annehmen)
SSH_OPTS=(-p "$SSH_PORT" -o StrictHostKeyChecking=accept-new -o ConnectTimeout=10)
[[ -n "$SSH_KEY" ]] && SSH_OPTS+=(-i "$SSH_KEY")

###### Prefix für ssh/rsync, falls Passwort-Login verwendet wird
SSHPASS_PREFIX=()
if [[ -n "$PASSWORD_FILE" ]]; then
  SSHPASS_PREFIX=(sshpass -f "$PASSWORD_FILE")
elif [[ -n "$PASSWORD" ]]; then
  SSHPASS_PREFIX=(sshpass -p "$PASSWORD")
fi
###### Prüfen, ob sshpass installiert ist, wenn gebraucht
if [[ ${#SSHPASS_PREFIX[@]} -gt 0 ]]; then
  command -v sshpass >/dev/null || { echo "FEHLER: sshpass ist nicht installiert."; exit 3; }
fi

### ================================
###   Checks: Existenz + Zeitstempel
### ================================
#### 1) Sicherstellen, dass die Datei remote existiert und lesbar ist
"${SSHPASS_PREFIX[@]}" ssh "${SSH_OPTS[@]}" "$REMOTE_USER@$REMOTE_HOST" "test -r '$REMOTE_FILE'"

##### 2) Zeitstempel der Remote-Datei ermitteln:
######    - %W = Birth/Erschaffung in EPOCH (sek)  (-1 falls nicht verfügbar)
######    - %Y = Mtime/Änderungszeit in EPOCH (sek)
######    Wir nutzen Birth, falls vorhanden; sonst Mtime.
ts_line=$("${SSHPASS_PREFIX[@]}" ssh "${SSH_OPTS[@]}" "$REMOTE_USER@$REMOTE_HOST" "stat -c '%W %Y' '$REMOTE_FILE'")
birth_epoch=$(awk '{print $1}' <<<"$ts_line")
mtime_epoch=$(awk '{print $2}' <<<"$ts_line")
if [[ "${birth_epoch}" -gt 0 ]]; then
  used_epoch="${birth_epoch}"
  used_label="birth"
else
  used_epoch="${mtime_epoch}"
  used_label="mtime"
fi

##### 3) Zeitstempel hübsch formatieren (UTC, ISO-ähnlich, ohne Doppelpunkte für Dateinamen) Beispiel: 2025-09-04T082206Z
TS_HUMAN=$(date -u -d "@${used_epoch}" "+%Y-%m-%dT%H%M%SZ")

### ================================
###   Zieldateinamen mit Datum bauen
### ================================
###### Original-Basename zerlegen: Name + Erweiterung
BASENAME=$(basename "$REMOTE_FILE")
if [[ "$BASENAME" == .* || "$BASENAME" != *.* ]]; then
  ####### Keine/verborgene Erweiterung -> einfach Suffix anhängen
  NAME="$BASENAME"
  EXT=""
else
  NAME="${BASENAME%.*}"
  EXT=".${BASENAME##*.}"fi
###### Ziel: name_TIMESTAMP.ext  (z.B. b1_2025-09-04T082206Z.txt)
TARGET="$LOCAL_DIR/${NAME}_${TS_HUMAN}${EXT}"

### ================================
###   Kopieren (rsync via ssh)
### ================================
RSYNC_SSH="ssh ${SSH_OPTS[*]}"
"${SSHPASS_PREFIX[@]}" rsync -a --partial --inplace --progress \
  -e "$RSYNC_SSH" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_FILE" "$TARGET"

### ================================
###   Integritätscheck (SHA256)
### ================================
remote_sum=$("${SSHPASS_PREFIX[@]}" ssh "${SSH_OPTS[@]}" "$REMOTE_USER@$REMOTE_HOST" "sha256sum '$REMOTE_FILE' | awk '{print\$1}'") local_sum=$(sha256sum "$TARGET" | awk '{print \$1}')
if [[ "$remote_sum" != "$local_sum" ]]; then
  echo "FEHLER: Checksumme stimmt nicht überein!" >&2
  exit 2
fi

### ================================
###   Log / Ausgabe
### ================================
###### Log-Zeile enthält: Laufzeit, Ziel, Checksumme, benutzter Zeitstempel (birth/mtime) + EPOCH
msg="$(date -Iseconds) – Backup ok: $TARGET (sha256 $local_sum) – source_time=${used_label}:${TS_HUMAN} (epoch ${used_epoch})"
echo "$msg"
###### Ins Log schreiben (falls Pfad beschreibbar ist)
{ echo "$msg"; } | tee -a "$LOG_FILE" >/dev/null || true


## Automatisierung mittels crontab
crontab -e

### Code 
SHELL=/bin/bash
HOME=/home/marcel/Desktop/Automatisation  # Homepfad
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=""

### Täglich um 13:10 Uhr ausführen
10 13 * * * /home/marcel/Desktop/Automatisation/Backup2.bash >> /home/marcel/Desktop/Automatisation/Backup2.log 2>&1

## Fazit
Die Erstellung des Skripts war für mich zunächst Neuland unter Bash. 
Durch gezielte Recherche konnte ich jedoch wertvolle Kenntnisse gewinnen und mein Verständnis vertiefen. 
Sehr hilfreich waren dabei die Grundkenntnisse, die wir in der Schule besprochen haben. 
Oft reicht es schon, einfach zu wissen, dass es beispielsweise crontab gibt, mit dem man solche Skripte automatisieren kann. 
Das fertige Skript lässt sich flexibel an die jeweiligen Anforderungen anpassen – etwa indem es ein TAR-Archiv erstellt und dieses automatisch an einen anderen Speicherort kopiert.




