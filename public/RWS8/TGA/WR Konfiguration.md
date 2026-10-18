 Der Wechselrichter (Growatt 4200TL XH) ist per RS485 via USB-Stick an einen Raspberry Pi angeschlossen, um die pv Leistung auszulesen und zu begrenzen und im Netzwerk verfügbar zu machen.

# Raspberry Pi Einstellung

## Verbindung:
 - OS: diet Pi OS, Debian 13 (Trixie)
 - IP-Adresse: 192.168.180.44
 - Name: PiAmWR
 - Benutzer: wr
 - Passwort: ${getSecret('rws8RpiWRPassword')}



Hier ist die kompakte Schritt-für-Schritt-Anleitung, um DietPi auf deinem Raspberry Pi Zero W für den USB-RS485-Betrieb und minimalen SD-Karten-Verschleiß einzurichten.
## 1. Image flashen & headless vorbereiten

   1. Lade das DietPi Raspberry Pi 1/Zero (32-bit) Image von der offiziellen Website herunter.
   2. Flashe es mit einem Tool wie Raspberry Pi Imager oder BalenaEtcher auf deine SD-Karte [DietPi].
   3. Wichtig für Headless-Betrieb: Öffne die SD-Karte nach dem Flashen am PC. Bearbeite die Datei dietpi.txt im Hauptverzeichnis, um dein WLAN einzurichten:
   
   AUTO_SETUP_NET_WIFI_ENABLED=1
   AUTO_SETUP_NET_WIFI_SSID=DeinWlanName
   AUTO_SETUP_NET_WIFI_KEY=DeinWlanPasswort
   
   
## 2. Erster Boot & System-Updates

   1. Stecke die Karte in den Pi Zero W und schließe den USB-RS485-Dongle an.
   2. Verbinde dich per SSH mit dem Pi (z. B. via Terminal oder Putty):
   * Host: dietpi.local (oder die IP-Adresse aus deinem Router)
      * Username: root
      * Passwort: dietpi
   3. Der Pi führt nun automatisch das Erst-Setup und alle Updates durch. Folge den Anweisungen auf dem Bildschirm. Ändere dabei unbedingt das Standardpasswort.

## 3. SD-Karten-Schutz maximieren (Swap deaktivieren)
DietPi lagert Logs bereits automatisch im RAM (dietpi-ramlog) [DietPi]. Deaktiviere nun noch die Auslagerungsdatei (Swap), um Schreibzugriffe komplett zu minimieren:

# Swap sofort abschalten
sudo dphys-swapfile swapoff# Autostart des Swap-Dienstes deaktivieren
sudo systemctl disable dphys-swapfile

## 4. USB-RS485 Dongle fest binden (UDEV-Rule)
Damit dein Dongle immer unter demselben Namen erreichbar bleibt:

   1. ID auslesen (z.B. 0403:6001):
   
   lsusb
   
   2. Regel erstellen:
   
   sudo nano /etc/udev/rules.d/99-rs485.rules
   
   3. Folgende Zeile einfügen (IDs anpassen) und speichern:
   
   SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", SYMLINK+="ttyUSB_RS485", MODE="0660", GROUP="dialout"
   
   4. Regeln neu laden:
   
   sudo udevadm control --reload-rules && sudo udevadm trigger
   
   
Dein Dongle ist ab jetzt dauerhaft unter /dev/ttyUSB_RS485 erreichbar.
## 5. mbusd & Python installieren

   1. Installiere das Gateway und Python:
   
   sudo apt update && sudo apt install -y mbusd python3 python3-pip
   
   2. Konfiguriere mbusd in der /etc/mbusd/mbusd.conf so, dass es auf /dev/ttyUSB_RS485 zugreift und stelle den Parameter loglevel = 0 ein, um jegliche Log-Schreibzugriffe auf die SD-Karte zu unterdrücken.

Möchtest du als Nächstes ein passendes Systemd-Service-Template haben, damit dein Python-Skript nach dem Booten automatisch und sicher im Hintergrund startet?





## mbusd

Auf dem Pi muss **mbusd** installiert werden, um die Zähler um Netzwerk verfügbar zu machen. Dazu folgendes Installationsskript wie folgt benützen:

```sh
sudo su
nano install_mbusd_sdm630.sh
chmod +x install_mbusd_sdm630.sh
paste -> CTRL+X -> Y -> Enter
./install_mbusd_sdm630.sh
reboot
```

**install_mbusd_sdm630.sh:**
```sh
#!/usr/bin/env bash
#
# install_mbusd_sdm630.sh
#
# Installs mbusd (Modbus TCP <-> Modbus RTU gateway) on Raspberry Pi OS Lite
# and configures it to bridge an Eastron SDM630 energy meter connected via
# a USB-RS485 adapter on /dev/ttyUSB0.
#
# Source: https://github.com/3cky/mbusd
#
# Run as: sudo ./install_mbusd_sdm630.sh
#
set -euo pipefail

# ---------------------------------------------------------------------------
# Configuration - adjust these if your setup differs from SDM630 defaults
# ---------------------------------------------------------------------------
SERIAL_DEVICE="/dev/ttyUSB0"   # USB-RS485 adapter device
BAUD_RATE="9600"               # SDM630 factory default is 9600 (check meter's own settings)
SERIAL_MODE="8n1"              # 8 data bits, no parity, 1 stop bit (SDM630 default)
TCP_PORT="502"                 # Standard Modbus TCP port (requires root to bind)
TRX_CONTROL="addc"             # "addc" = automatic direction control (most FTDI/CH340/CP210x
                                # USB-RS485 adapters handle this in hardware).
                                # Change to "rts" if your adapter needs manual RTS toggling.
BUILD_DIR="/usr/src/mbusd"
CONF_DIR="/etc/mbusd"
CONF_FILE="${CONF_DIR}/mbusd-$(basename "${SERIAL_DEVICE}").conf"

# ---------------------------------------------------------------------------
# Sanity checks
# ---------------------------------------------------------------------------
if [[ $EUID -ne 0 ]]; then
    echo "This script must be run as root (use sudo)." >&2
    exit 1
fi

echo "==> Installing build dependencies..."
apt-get update
apt-get install -y git cmake build-essential

# ---------------------------------------------------------------------------
# Fetch and build mbusd
# ---------------------------------------------------------------------------
echo "==> Fetching mbusd source..."
rm -rf "${BUILD_DIR}"
git clone https://github.com/3cky/mbusd.git "${BUILD_DIR}"

echo "==> Building mbusd..."
mkdir -p "${BUILD_DIR}/build"
cd "${BUILD_DIR}/build"
cmake -DCMAKE_INSTALL_PREFIX=/usr ..
make -j"$(nproc)"

echo "==> Installing mbusd..."
make install
# make install also installs the mbusd@.service systemd template if systemd
# is detected on the build host (true for Raspberry Pi OS).

# ---------------------------------------------------------------------------
# Serial port access
# ---------------------------------------------------------------------------
# Give the default 'pi'/first user dialout access to the USB serial adapter,
# in case you ever run mbusd outside systemd (systemd's mbusd@.service runs
# as root by default, so this isn't strictly required for the service itself).
if id -u pi &>/dev/null; then
    usermod -aG dialout pi
fi

# ---------------------------------------------------------------------------
# mbusd configuration for the SDM630 on ttyUSB0
# ---------------------------------------------------------------------------
echo "==> Writing configuration to ${CONF_FILE}..."
mkdir -p "${CONF_DIR}"
cat > "${CONF_FILE}" <<EOF
############# Serial port settings #############
# Serial port device name
device = ${SERIAL_DEVICE}

# Serial port speed
speed = ${BAUD_RATE}

# Serial port mode (data bits / parity / stop bits)
mode = ${SERIAL_MODE}

# RS-485 data direction control type (addc, rts, sysfs_0, sysfs_1)
trx_control = ${TRX_CONTROL}

############# TCP server settings #############
# TCP server port number
port = ${TCP_PORT}

############# Logging #############
# Log file (comment out to disable file logging; systemd journal captures
# stdout/stderr regardless)
# log = /var/log/mbusd.log
EOF

# ---------------------------------------------------------------------------
# Enable and start the systemd service
# ---------------------------------------------------------------------------
SERVICE_NAME="mbusd@$(basename "${SERIAL_DEVICE}").service"

echo "==> Enabling and starting ${SERVICE_NAME}..."
systemctl daemon-reload
systemctl enable --now "${SERVICE_NAME}"

sleep 1
systemctl --no-pager status "${SERVICE_NAME}" || true

cat <<EOF

==============================================================
mbusd is installed and running as: ${SERVICE_NAME}
Config file:  ${CONF_FILE}
TCP endpoint: <this-pi-ip>:${TCP_PORT}
Serial:       ${SERIAL_DEVICE} @ ${BAUD_RATE} ${SERIAL_MODE}

Your SDM630 should now be reachable as a Modbus TCP slave
(default Modbus address 1 unless you changed it on the meter)
through this gateway.

Useful commands:
  journalctl -u ${SERVICE_NAME} -f     # follow logs
  systemctl restart ${SERVICE_NAME}    # after editing config
  cat ${CONF_FILE}                     # view current config
==============================================================
EOF
```

`mbusd` hört jetzt auf **Port 502** (Modbus TCP).


Es wird noch ein Service installiert und gestartet, welcher die wr Leistung automatisch an den Hausverbrauch anpasst
Maximal 3 kw leistung des wr
Maximal 800w Einspeisung (also Hausverbrauch+800 = wr Leistung)
Das Script muss sich die sekündlichen Verbrauchswerte holen oder zugesendet bekommen
Und falls 3 Sekunden lang keine Werte kommen wird die Leistung auf 800w gesetzt
---

# Technische Beschreibung – PiAmWR (Raspberry Pi + Growatt 4200TL-XH)

**Stand:** Oktober 2026  
**Ziel:** Dynamische Leistungsbegrenzung des Growatt 4200TL-XH auf Basis des aktuellen Hausverbrauchs  
**Hardware:** Raspberry Pi Zero W + USB-RS485-Dongle + Growatt 4200TL-XH

---

## 1. Übersicht der Architektur

```
[Hauszähler / SDM630]
        │
        │ Modbus TCP (über HA oder direkt)
        ▼
[Home Assistant] ──MQTT──► [PiAmWR (Raspberry Pi)]
                                    │
                                    │ RS485 (USB-Dongle)
                                    ▼
                            [Growatt 4200TL-XH]
```

**Kernfunktionen des PiAmWR:**
- Empfang der aktuellen Hausverbrauchswerte (sekündlich)
- Berechnung der maximal erlaubten WR-Leistung:
  - Max. **3000 W** absolute Grenze
  - Max. **800 W** Einspeisung → `WR-Leistung = Hausverbrauch + 800 W`
- Schreiben der Begrenzung an den Wechselrichter
- Failsafe: Keine gültigen Werte für **3 Sekunden** → Leistung auf **800 W** setzen

---

## 2. Hardware & Betriebssystem

| Komponente              | Details                                      |
|-------------------------|----------------------------------------------|
| Raspberry Pi            | Pi Zero W                                    |
| OS                      | DietPi (Debian 13 Trixie)                    |
| Hostname                | `PiAmWR`                                     |
| IP-Adresse              | `192.168.180.44`                             |
| Benutzer                | `wr`                                         |
| USB-RS485-Dongle        | FTDI / CH340 / CP210x (wird als `/dev/ttyUSB_RS485` gebunden) |

### Wichtige Systemeinstellungen (SD-Kartenschutz)

```bash
# Swap deaktivieren
sudo dphys-swapfile swapoff
sudo systemctl disable dphys-swapfile

# Logs bleiben im RAM (dietpi-ramlog ist bereits aktiv)
```

### UDEV-Regel für festen Gerätenamen

Datei: `/etc/udev/rules.d/99-rs485.rules`

```
SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", SYMLINK+="ttyUSB_RS485", MODE="0660", GROUP="dialout"
```

Nach dem Anlegen:
```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
```

---

## 3. mbusd – Modbus TCP ↔ RTU Gateway

mbusd macht den Wechselrichter (oder den Zähler) im Netzwerk als Modbus-TCP-Gerät verfügbar.

- **TCP-Port:** 502
- **Serial:** `/dev/ttyUSB_RS485` @ 9600 8N1
- **trx_control:** `addc` (bei den meisten USB-RS485-Adaptern)

Der Dienst läuft als systemd-Service (`mbusd@ttyUSB_RS485.service` bzw. analog).

---

## 4. Leistungsbegrenzung am Growatt 4200TL-XH

### Verwendetes Register

| Parameter              | Wert                                      |
|------------------------|-------------------------------------------|
| **Register**           | **3** (Holding Register)                  |
| **Name**               | Active Power Rate                         |
| **Funktion**           | Maximale Ausgangsleistung in % der Nennleistung |
| **Datentyp**           | uint16                                    |
| **Wertebereich**       | 0–100 (255 = keine Begrenzung)           |
| **Funktionscode**      | FC06 (Write Single Register)              |
| **Slave-Adresse**      | 1 (Standard)                              |

### Umrechnung

- Nennleistung des WR: **4200 W**
- Gewünschte Leistung in Watt → Prozent:

```
percent = round(desired_power_w / 42)          # 4200 W / 100
percent = max(0, min(100, percent))
```

**Beispiele:**
- 3000 W → **71 %**
- 800 W  → **19 %**
- 0 W    → **0 %**

### Failsafe-Logik

```
IF (Zeit seit letztem gültigen Verbrauchswert > 3 Sekunden)
    THEN setze Register 3 auf 19   # ≈ 800 W
```

---

## 5. Datenfluss der Verbrauchswerte

**Empfohlene Variante: MQTT**

1. Home Assistant publiziert den aktuellen Hausverbrauch (in Watt) auf einem Topic, z. B.:
   ```
   home/power/consumption
   ```
   Payload: reine Zahl (z. B. `1247`)

2. Der Python-Dienst auf dem Pi abonniert das Topic und speichert:
   - letzten Wert
   - Timestamp des letzten Updates

**Alternative Möglichkeiten:**
- Direkter HTTP-Aufruf der Home-Assistant REST-API
- Direkter Modbus-TCP-Zugriff auf den Zähler (falls dieser bereits im Netz erreichbar ist)

---

## 6. Empfohlene Software-Struktur auf dem Pi

```
/opt/piamwr/
├── power_limiter.py          # Haupt-Skript
├── config.yaml               # Konfiguration (Topics, Limits, Serial-Port …)
└── requirements.txt
```

**Wichtige Python-Bibliotheken:**
- `paho-mqtt` – MQTT-Client
- `pymodbus` – Modbus-Kommunikation
- `pyyaml` – Konfiguration

Der Dienst sollte als **systemd-Service** laufen und nach dem Boot automatisch starten.

---

## 7. Sicherheits- und Betriebsregeln

| Regel                              | Umsetzung                                      |
|------------------------------------|------------------------------------------------|
| Max. WR-Leistung                   | 3000 W (hart im Code)                          |
| Max. Einspeisung                   | 800 W (Hausverbrauch + 800)                    |
| Kommunikationsausfall              | Nach 3 s → 800 W                               |
| SD-Karten-Verschleiß               | Swap aus, Logs im RAM, mbusd-Loglevel = 0      |
| Serielle Schnittstelle             | Fester Symlink `/dev/ttyUSB_RS485`             |

---

## 8. Nächste Schritte (Checkliste)

- [ ] MQTT-Topic in Home Assistant einrichten und testen
- [ ] Python-Skript mit Register-3-Schreiben implementieren
- [ ] Systemd-Service anlegen und aktivieren
- [ ] Failsafe (3-Sekunden-Timeout) testen
- [ ] Grenzwerte 3000 W / 800 W verifizieren
- [ ] Logging (nur journald, keine Dateien auf der SD-Karte)

---

## 9. Wichtige Befehle (Schnellreferenz)

```bash
# mbusd Status
systemctl status mbusd@ttyUSB_RS485

# Live-Logs des Limiters
journalctl -u piamwr-limiter -f

# Manuell Register 3 lesen
# (mit pymodbus oder mbpoll)
```

---

**Dokument erstellt für:** PiAmWR – Growatt 4200TL-XH Leistungsbegrenzung  
**Register-Quelle:** Offizielles Growatt Modbus RTU Protocol (MIN TL-XH Serie) – Holding Register 3
