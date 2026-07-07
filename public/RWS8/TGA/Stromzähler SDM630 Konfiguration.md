Die Stromzähler (SDM630) für die 3 WEs und Allgemeinstrom sind per RS485 via USB-Stick an einen Raspberry Pi angeschlossen, um die Zählerwerte auszulesen und im Netzwerk verfügbar zu machen.

# Raspberry Pi Einstellung

## Verbindung:
 - OS: Raspberry Pi OS, Debian 13 (Trixie)
 - IP-Adresse: 192.168.178.43
 - Name: PiImSchrank
 - Benutzer: schrank
 - Passwort: ${getSecret('rpizPassword')}

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

# SDM630 Device Adressen

Beim Betrieb mehrerer Zähler an einem RS-485-Bus, benötigt jedes Gerät eine eindeutige Adresse. Die Konfiguration erfolgt über die Tasten an der Gerätefront:

## Modbus-Adresse manuell am Gerät ändern

1. **Setup-Modus aufrufen**  
   Halten Sie die Taste **E** (bzw. die Fronttaste) ca. 3 Sekunden gedrückt, bis die Passwortabfrage erscheint.

2. **Passwort eingeben**  
   Geben Sie das Standardpasswort `1000` ein.

3. **Menü navigieren**  
   Drücken Sie die Taste wiederholt kurz, um durch die Parameter zu blättern, bis das Display `id` (Modbus Slave ID) anzeigt.

4. **Adresse ändern**  
   Halten Sie die Taste gedrückt, um den Wert zu editieren. Durch kurzes Klicken erhöhen Sie die Adresse (Bereich 1–247).

5. **Speichern**  
   Drücken Sie die Taste nach der Einstellung erneut lang, um den neuen Wert zu bestätigen und das Menü zu verlassen.

## Zugewiesene Stromzähler Modbus-Adressen

| Modbus-ID | Bezeichnung     |
|-----------|-----------------|
| 021       | Allgemein       |
| 022       | EG              |
| 023       | OG li           |
| 024       | OG re           |

### Begründung der Adressierung

Die zweite Stelle der Adresse wurde bewusst auf **2** gesetzt (021, 022, 023, 024 …). Dadurch bleiben die niedrigeren Adressen (z. B. 001–019) für eventuelle zukünftige Geräte mit Standard-ID oder für weitere Erweiterungen frei. So können später problemlos zusätzliche Zähler oder andere Modbus-Geräte am Bus hinzugefügt werden, ohne Adresskonflikte zu riskieren.

# Home Assistant

# SDM630 → Raspberry Pi → Home Assistant

Der Pi dient als Modbus-Bridge – HA spricht direkt per TCP.

## Home Assistant Konfiguration – `configuration.yaml`

 * https://www.home-assistant.io/common-tasks/os/#installing-and-using-the-visual-studio-code-vsc-app
 * 

```yaml
modbus:
  - name: "sdm630_bus"
    type: tcp
    host: 192.168.178.43
    port: 502

sensor:
  # ── SDM630 #1 (Slave-Adresse 1) ──
  - platform: modbus
    name: "SDM630_1 Spannung L1"
    hub: sdm630_bus
    slave: 1
    address: 0       # Register 0x0000
    input_type: input
    data_type: float32
    unit_of_measurement: "V"
    device_class: voltage
    scan_interval: 30

  - platform: modbus
    name: "SDM630_1 Strom L1"
    hub: sdm630_bus
    slave: 1
    address: 6       # 0x0006
    input_type: input
    data_type: float32
    unit_of_measurement: "A"
    device_class: current
    scan_interval: 30

  - platform: modbus
    name: "SDM630_1 Wirkleistung gesamt"
    hub: sdm630_bus
    slave: 1
    address: 52      # 0x0034
    input_type: input
    data_type: float32
    unit_of_measurement: "W"
    device_class: power
    scan_interval: 30

  - platform: modbus
    name: "SDM630_1 Gesamtenergie"
    hub: sdm630_bus
    slave: 1
    address: 342     # 0x0156
    input_type: input
    data_type: float32
    unit_of_measurement: "kWh"
    device_class: energy
    state_class: total_increasing
    scan_interval: 60

  # ── SDM630 #2 (Slave-Adresse 2) ──
  - platform: modbus
    name: "SDM630_2 Wirkleistung gesamt"
    hub: sdm630_bus
    slave: 2
    address: 52
    input_type: input
    data_type: float32
    unit_of_measurement: "W"
    device_class: power
    scan_interval: 30

  # ... (gleiche Register, slave: 3 für #3)
```

## Wichtige SDM630 Register-Übersicht

| Messgröße | Register (dez) | Register (hex) |
|---|---|---|
| Spannung L1/L2/L3 | 0 / 2 / 4 | 0x0000–0x0004 |
| Strom L1/L2/L3 | 6 / 8 / 10 | 0x0006–0x000A |
| Leistung L1/L2/L3 | 12 / 14 / 16 | 0x000C–0x0010 |
| **Gesamtleistung** | 52 | 0x0034 |
| Frequenz | 70 | 0x0046 |
| **Gesamtenergie (Import)** | 342 | 0x0156 |
| Gesamtenergie (Export) | 344 | 0x0158 |

Alle Register sind **32-bit Float (2 Register)**, Input-Type.
