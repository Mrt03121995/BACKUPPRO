
# Backup-pro
- **Backuppro:** …
- **SMART:** Spezifisch · Messbar · Attraktiv · Realistisch · Terminiert

## Ausgangslage

Schreibe ein Skript mit einer kleinen Dokumentation.


## Zielformulierung / To-dos

1. Es soll ein Skript für ein Backup task erstellt werden.
2. Im Backup nahme soll die Uhrzeit stehen.
3. Es soll dazu die Automatiserung und installation des Skript erläutert werden.
4. Das skript sollte erläutertert werden was wo gemacht wird 


## Vorausetzung 
# installierte poackete

1.SSH Sehrver 
2. sshpass

sudo apt-get update
sudo apt-get install -y sshpass
sudo apt update

sudo apt install openssh-server                 # Server installieren
sudo systemctl enable --now ssh                 # Dienst starten & beim Boot aktivieren
sudo systemctl status ssh                       # Status prüfen


## Bsckup code Beschreibung
- 2025-09-06:

#!/usr/bin/env bash
set -Eeuo pipefail

# === Quelle/Ziel ===
REMOTE_USER="marcel"
REMOTE_HOST="192.168.1.117"
REMOTE_FILE="/home/marcel/b1.txt"
LOCAL_DIR="/home/Desktop/Desktop/backuppool "

# === SSH-Setup ===
SSH_PORT=22
SSH_KEY=""              # z.B. /home/server1/.ssh/id_ed25519; leer = Standard-Key/Agent
# Falls du mit sudo startest und keinen SSH_KEY setzt, versuche Benutzer-Key zu verwenden:
if [[ -z "${SSH_KEY}" && "${SUDO_USER-}" && -r "/home/${SUDO_USER}/.ssh/id_ed25519" ]]; then
  SSH_KEY="/home/${SUDO_USER}/.ssh/id_ed25519"
fi

# === OPTIONAL: Passwort-Login via sshpass (unsicherer als Key-Login) ===
PASSWORD="marcel"             # direkt hier eintragen (sichtbar in Prozessliste) – besser leer lassen
PASSWORD_FILE=""        # oder: Pfad zu einer Datei mit NUR dem Passwort (sicherer als oben), z.B. /root/.ssh/remote_pw

# === Vorbereitung ===
mkdir -p "$LOCAL_DIR"

SSH_OPTS=(-p "$SSH_PORT" -o StrictHostKeyChecking=accept-new -o ConnectTimeout=10)
[[ -n "$SSH_KEY" ]] && SSH_OPTS+=(-i "$SSH_KEY")

# Prefix für ssh/rsync, wenn Passwort verwendet wird
SSHPASS_PREFIX=()
if [[ -n "$PASSWORD_FILE" ]]; then
  SSHPASS_PREFIX=(sshpass -f "$PASSWORD_FILE")
elif [[ -n "$PASSWORD" ]]; then
  SSHPASS_PREFIX=(sshpass -p "$PASSWORD")
fi
# Prüfen, ob sshpass vorhanden ist, wenn benötigt
if [[ ${#SSHPASS_PREFIX[@]} -gt 0 ]]; then
  command -v sshpass >/dev/null || { echo "FEHLER: sshpass ist nicht installiert."; exit 3; }
fi

# === Checks & Kopie ===
# Existenz/Lesbarkeit remote prüfen
"${SSHPASS_PREFIX[@]}" ssh "${SSH_OPTS[@]}" "$REMOTE_USER@$REMOTE_HOST" "test -r '$REMOTE_FILE'"

# Kopieren (rsync ist robust; alternativ ginge scp)
BASENAME=$(basename "$REMOTE_FILE")
TARGET="$LOCAL_DIR/$BASENAME"
RSYNC_SSH="ssh ${SSH_OPTS[*]}"
"${SSHPASS_PREFIX[@]}" rsync -a --partial --inplace --progress \
  -e "$RSYNC_SSH" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_FILE" "$TARGET"

# Integritätscheck (SHA256)
remote_sum=$("${SSHPASS_PREFIX[@]}" ssh "${SSH_OPTS[@]}" "$REMOTE_USER@$REMOTE_HOST" "sha256sum '$REMOTE_FILE' | awk '{print \$1}'")
local_sum=$(sha256sum "$TARGET" | awk '{print $1}')
if [[ "$remote_sum" != "$local_sum" ]]; then
  echo "FEHLER: Checksumme stimmt nicht überein!" >&2
  exit 2
fi

echo "$(date -Iseconds) – Backup ok: $TARGET (sha256 $local_sum)" | tee -a /var/log/remote_b1_backup.log

  NR2___________________________________________________________________________

  #!/usr/bin/env bash
set -Eeuo pipefail

# ================================
#   Einstellungen Quelle / Ziel
# ================================
REMOTE_USER="marcel"                        # Remote-User
REMOTE_HOST="192.168.1.117"                # Remote-Host / IP
REMOTE_FILE="/home/marcel/b1.txt"          # Remote-Datei
LOCAL_DIR="/home/marcel/Desktop/backuppool"  # Lokaler Zielordner (ohne Leerzeichen!)
LOG_FILE="/var/log/remote_b1_backup.log"   # Logdatei (Root-Rechte nötig)

# ================================
#   SSH-Setup
# ================================
SSH_PORT=22
SSH_KEY=""                                  # z.B. /home/server1/.ssh/id_ed25519; leer -> Standard-Key/Agent

# Wenn das Skript per sudo läuft und kein SSH_KEY gesetzt ist, versuche Benutzer-Key zu nutzen.
if [[ -z "${SSH_KEY}" && "${SUDO_USER-}" && -r "/home/${SUDO_USER}/.ssh/id_ed25519" ]]; then
  SSH_KEY="/home/${SUDO_USER}/.ssh/id_ed25519"
fi

# ================================
#   OPTIONAL: Passwort-Login (sshpass)
#   Sicherer ist Key-Login -> beide Felder leer lassen!
# ================================
PASSWORD=""               # Achtung: im Prozess sichtbar – möglichst leer lassen
PASSWORD_FILE=""          # Besser: Datei mit NUR dem Passwort, z.B. /root/.ssh/remote_pw (600)

# ================================
#   Vorbereitung
# ================================
mkdir -p "$LOCAL_DIR"

# SSH-Optionen (Fingerprints beim ersten Mal automatisch annehmen)
SSH_OPTS=(-p "$SSH_PORT" -o StrictHostKeyChecking=accept-new -o ConnectTimeout=10)
[[ -n "$SSH_KEY" ]] && SSH_OPTS+=(-i "$SSH_KEY")

# Prefix für ssh/rsync, falls Passwort-Login verwendet wird
SSHPASS_PREFIX=()
if [[ -n "$PASSWORD_FILE" ]]; then
  SSHPASS_PREFIX=(sshpass -f "$PASSWORD_FILE")
elif [[ -n "$PASSWORD" ]]; then
  SSHPASS_PREFIX=(sshpass -p "$PASSWORD")
fi
# Prüfen, ob sshpass installiert ist, wenn gebraucht
if [[ ${#SSHPASS_PREFIX[@]} -gt 0 ]]; then
  command -v sshpass >/dev/null || { echo "FEHLER: sshpass ist nicht installiert."; exit 3; }
fi

# ================================
#   Checks: Existenz + Zeitstempel
# ================================
# 1) Sicherstellen, dass die Datei remote existiert und lesbar ist
"${SSHPASS_PREFIX[@]}" ssh "${SSH_OPTS[@]}" "$REMOTE_USER@$REMOTE_HOST" "test -r '$REMOTE_FILE'"

# 2) Zeitstempel der Remote-Datei ermitteln:
#    - %W = Birth/Erschaffung in EPOCH (sek)  (-1 falls nicht verfügbar)
#    - %Y = Mtime/Änderungszeit in EPOCH (sek)
#    Wir nutzen Birth, falls vorhanden; sonst Mtime.
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

# 3) Zeitstempel hübsch formatieren (UTC, ISO-ähnlich, ohne Doppelpunkte für Dateinamen)
#    Beispiel: 2025-09-04T082206Z
TS_HUMAN=$(date -u -d "@${used_epoch}" "+%Y-%m-%dT%H%M%SZ")

# ================================
#   Zieldateinamen mit Datum bauen
# ================================
# Original-Basename zerlegen: Name + Erweiterung
BASENAME=$(basename "$REMOTE_FILE")
if [[ "$BASENAME" == .* || "$BASENAME" != *.* ]]; then
  # Keine/verborgene Erweiterung -> einfach Suffix anhängen
  NAME="$BASENAME"
  EXT=""
else
  NAME="${BASENAME%.*}"
  EXT=".${BASENAME##*.}"fi

# Ziel: name_TIMESTAMP.ext  (z.B. b1_2025-09-04T082206Z.txt)
TARGET="$LOCAL_DIR/${NAME}_${TS_HUMAN}${EXT}"

# ================================
#   Kopieren (rsync via ssh)
# ================================
RSYNC_SSH="ssh ${SSH_OPTS[*]}"
"${SSHPASS_PREFIX[@]}" rsync -a --partial --inplace --progress \
  -e "$RSYNC_SSH" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_FILE" "$TARGET"

# ================================
#   Integritätscheck (SHA256)
# ================================
remote_sum=$("${SSHPASS_PREFIX[@]}" ssh "${SSH_OPTS[@]}" "$REMOTE_USER@$REMOTE_HOST" "sha256sum '$REMOTE_FILE' | awk '{print \$1}'")
local_sum=$(sha256sum "$TARGET" | awk '{print $1}')
if [[ "$remote_sum" != "$local_sum" ]]; then
  echo "FEHLER: Checksumme stimmt nicht überein!" >&2
  exit 2
fi

# ================================
#   Log / Ausgabe
# ================================
# Log-Zeile enthält: Laufzeit, Ziel, Checksumme, benutzter Zeitstempel (birth/mtime) + EPOCH
msg="$(date -Iseconds) – Backup ok: $TARGET (sha256 $local_sum) – source_time=${used_label}:${TS_HUMAN} (epoch ${used_epoch})"
echo "$msg"
# Ins Log schreiben (falls Pfad beschreibbar ist)
{ echo "$msg"; } | tee -a "$LOG_FILE" >/dev/null || true


## Automatisierung

- KPI: … — Zielwert: …

## Risiken & Annahmen
- Risiko: … — Gegenmaßnahme: …

  ## Meilensteine & Termine
- 2025-01-15: …

## Kennzahlen (KPIs)
- KPI: … — Zielwert: …

## Risiken & Annahmen
- Risiko: … — Gegenmaßnahme: …
