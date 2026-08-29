Die Stromzähler (SDM630) für die 3 WEs und Allgemeinstrom sind per RS485 via USB-Stick an einen Raspberry Pi angeschlossen, um die Zählerwerte auszulesen und im Netzwerk verfügbar zu machen.

# Raspberry Pi Einstellung

## Verbindung:
 - OS: Raspberry Pi OS, Debian 13 (Trixie)
 - IP-Adresse: 192.168.180.43
 - Name: PiImSchrank
 - Benutzer: schrank
 - Passwort: ${getSecret('rws8RpiZPassword')}

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

| Modbus-ID | Bezeichnung      |
|-----------|------------------|
| 021       | ALLG - Allgemein |
| 022       | WE01 - EG        |
| 023       | WE02 - OG li     |
| 024       | WE03 - OG re     |

### Begründung der Adressierung

Die zweite Stelle der Adresse wurde bewusst auf **2** gesetzt (021, 022, 023, 024 …). Dadurch bleiben die niedrigeren Adressen (z. B. 001–019) für eventuelle zukünftige Geräte mit Standard-ID oder für weitere Erweiterungen frei. So können später problemlos zusätzliche Zähler oder andere Modbus-Geräte am Bus hinzugefügt werden, ohne Adresskonflikte zu riskieren.

# Home Assistant

# SDM630 → Raspberry Pi → Home Assistant

Der Pi dient als Modbus-Bridge – HA spricht direkt per TCP.

## Home Assistant Konfiguration – `configuration.yaml`

 * https://www.home-assistant.io/common-tasks/os/#installing-and-using-the-visual-studio-code-vsc-app
 * insert `modbus: !include modbus_sdm630.yaml` into configuration.yaml
 * add following yaml as modbus\_sdm630.yaml


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


Hier die vollständige Registertabelle:

| Adresse (dez) | Hex | Bezeichnung | Einheit | Bedeutung |
| --- | --- | --- | --- | --- |
| 0   | 0x0000 | Spannung L1 | V   | Außenleiterspannung L1 gegen Neutralleiter |
| 2   | 0x0002 | Spannung L2 | V   | Außenleiterspannung L2 gegen Neutralleiter |
| 4   | 0x0004 | Spannung L3 | V   | Außenleiterspannung L3 gegen Neutralleiter |
| 6   | 0x0006 | Strom L1 | A   | Strom auf Phase L1 |
| 8   | 0x0008 | Strom L2 | A   | Strom auf Phase L2 |
| 10  | 0x000A | Strom L3 | A   | Strom auf Phase L3 |
| 12  | 0x000C | Wirkleistung L1 | W   | Aktuelle Wirkleistung Phase L1 |
| 14  | 0x000E | Wirkleistung L2 | W   | Aktuelle Wirkleistung Phase L2 |
| 16  | 0x0010 | Wirkleistung L3 | W   | Aktuelle Wirkleistung Phase L3 |
| 18  | 0x0012 | Scheinleistung L1 | VA  | Scheinleistung Phase L1 |
| 20  | 0x0014 | Scheinleistung L2 | VA  | Scheinleistung Phase L2 |
| 22  | 0x0016 | Scheinleistung L3 | VA  | Scheinleistung Phase L3 |
| 24  | 0x0018 | Blindleistung L1 | var | Blindleistung Phase L1 |
| 26  | 0x001A | Blindleistung L2 | var | Blindleistung Phase L2 |
| 28  | 0x001C | Blindleistung L3 | var | Blindleistung Phase L3 |
| 30  | 0x001E | Leistungsfaktor L1 | –   | cos φ Phase L1 (+kapazitiv/-induktiv) |
| 32  | 0x0020 | Leistungsfaktor L2 | –   | cos φ Phase L2 |
| 34  | 0x0022 | Leistungsfaktor L3 | –   | cos φ Phase L3 |
| 36  | 0x0024 | Phasenwinkel L1 | °   | Phasenwinkel Spannung/Strom L1 |
| 38  | 0x0026 | Phasenwinkel L2 | °   | Phasenwinkel Spannung/Strom L2 |
| 40  | 0x0028 | Phasenwinkel L3 | °   | Phasenwinkel Spannung/Strom L3 |
| 42  | 0x002A | Durchschnittsspannung L-N | V   | Mittelwert der 3 Phasenspannungen |
| 46  | 0x002E | Durchschnittsstrom | A   | Mittelwert der 3 Phasenströme |
| 48  | 0x0030 | Summe Ströme | A   | Summe aller Phasenströme |
| 52  | 0x0034 | Gesamtwirkleistung | W   | Momentane Gesamt-Wirkleistung (alle Phasen) |
| 56  | 0x0038 | Gesamtscheinleistung | VA  | Momentane Gesamt-Scheinleistung |
| 60  | 0x003C | Gesamtblindleistung | var | Momentane Gesamt-Blindleistung |
| 62  | 0x003E | Gesamtleistungsfaktor | –   | Gesamt cos φ |
| 66  | 0x0042 | Gesamtphasenwinkel | °   | Gesamt-Phasenwinkel |
| 70  | 0x0046 | Frequenz | Hz  | Netzfrequenz |
| 72  | 0x0048 | Bezug Wirkarbeit seit Reset | kWh | Importierte Wirkenergie seit letztem Reset |
| 74  | 0x004A | Einspeisung Wirkarbeit seit Reset | kWh | Exportierte Wirkenergie seit letztem Reset |
| 76  | 0x004C | Bezug Blindarbeit seit Reset | kvarh | Importierte Blindenergie seit Reset |
| 78  | 0x004E | Einspeisung Blindarbeit seit Reset | kvarh | Exportierte Blindenergie seit Reset |
| 80  | 0x0050 | Scheinarbeit seit Reset | kVAh | Scheinenergie seit letztem Reset |
| 82  | 0x0052 | Amperestunden seit Reset | Ah  | Ladungsdurchsatz seit Reset |
| 84  | 0x0054 | Gesamtwirkleistungsbedarf | W   | Aktueller Leistungsbedarf (Demand-Fenster) |
| 86  | 0x0056 | Max Gesamtwirkleistungsbedarf | W   | Maximaler Leistungsbedarf seit Reset |
| 100 | 0x0064 | Gesamtscheinleistungsbedarf | VA  | Aktueller Scheinleistungsbedarf |
| 102 | 0x0066 | Max Gesamtscheinleistungsbedarf | VA  | Maximaler Scheinleistungsbedarf seit Reset |
| 104 | 0x0068 | Neutralleiterstrombedarf | A   | Aktueller Bedarf N-Leiterstrom |
| 106 | 0x006A | Max Neutralleiterstrombedarf | A   | Maximaler N-Leiterstrombedarf seit Reset |
| 200 | 0x00C8 | Spannung L1-L2 | V   | Außenleiterspannung zwischen L1 und L2 |
| 202 | 0x00CA | Spannung L2-L3 | V   | Außenleiterspannung zwischen L2 und L3 |
| 204 | 0x00CC | Spannung L3-L1 | V   | Außenleiterspannung zwischen L3 und L1 |
| 206 | 0x00CE | Durchschnittsspannung L-L | V   | Mittelwert der 3 Außenleiterspannungen |
| 224 | 0x00E0 | Neutralleiterstrom | A   | Strom im Neutralleiter |
| 234 | 0x00EA | THD Spannung L1 | %   | Oberschwingungsgehalt Spannung L1 |
| 236 | 0x00EC | THD Spannung L2 | %   | Oberschwingungsgehalt Spannung L2 |
| 238 | 0x00EE | THD Spannung L3 | %   | Oberschwingungsgehalt Spannung L3 |
| 240 | 0x00F0 | THD Strom L1 | %   | Oberschwingungsgehalt Strom L1 |
| 242 | 0x00F2 | THD Strom L2 | %   | Oberschwingungsgehalt Strom L2 |
| 244 | 0x00F4 | THD Strom L3 | %   | Oberschwingungsgehalt Strom L3 |
| 248 | 0x00F8 | Durchschnitt THD Spannung L-N | %   | Mittlerer THD aller Phasenspannungen |
| 250 | 0x00FA | Durchschnitt THD Strom | %   | Mittlerer THD aller Phasenströme |
| 254 | 0x00FE | Gesamtleistungsfaktor Winkel | °   | Gesamt-Leistungsfaktor als Winkel |
| 258 | 0x0102 | Strombedarf L1 | A   | Aktueller Strombedarf Phase L1 |
| 260 | 0x0104 | Strombedarf L2 | A   | Aktueller Strombedarf Phase L2 |
| 262 | 0x0106 | Strombedarf L3 | A   | Aktueller Strombedarf Phase L3 |
| 264 | 0x0108 | Max Strombedarf L1 | A   | Maximaler Strombedarf L1 seit Reset |
| 266 | 0x010A | Max Strombedarf L2 | A   | Maximaler Strombedarf L2 seit Reset |
| 268 | 0x010C | Max Strombedarf L3 | A   | Maximaler Strombedarf L3 seit Reset |
| 334 | 0x014E | THD Spannung L1-L2 | %   | Oberschwingungsgehalt Spannung L1-L2 |
| 336 | 0x0150 | THD Spannung L2-L3 | %   | Oberschwingungsgehalt Spannung L2-L3 |
| 338 | 0x0152 | THD Spannung L3-L1 | %   | Oberschwingungsgehalt Spannung L3-L1 |
| 340 | 0x0154 | Durchschnitt THD Spannung L-L | %   | Mittlerer THD aller Außenleiterspannungen |
| 342 | 0x0156 | Gesamtenergie | kWh | Gesamte Wirkenergie (Bezug+Einspeisung, alle Phasen) |
| 344 | 0x0158 | Gesamt Blindenergie | kvarh | Gesamte Blindenergie (alle Phasen) |
| 346 | 0x015A | L1 Bezug Wirkenergie | kWh | Importierte Wirkenergie Phase L1 |
| 348 | 0x015C | L2 Bezug Wirkenergie | kWh | Importierte Wirkenergie Phase L2 |
| 350 | 0x015E | L3 Bezug Wirkenergie | kWh | Importierte Wirkenergie Phase L3 |
| 352 | 0x0160 | L1 Einspeisung Wirkenergie | kWh | Exportierte Wirkenergie Phase L1 |
| 354 | 0x0162 | L2 Einspeisung Wirkenergie | kWh | Exportierte Wirkenergie Phase L2 |
| 356 | 0x0164 | L3 Einspeisung Wirkenergie | kWh | Exportierte Wirkenergie Phase L3 |
| 358 | 0x0166 | L1 Gesamtenergie | kWh | Wirkenergie gesamt Phase L1 |
| 360 | 0x0168 | L2 Gesamtenergie | kWh | Wirkenergie gesamt Phase L2 |
| 362 | 0x016A | L3 Gesamtenergie | kWh | Wirkenergie gesamt Phase L3 |
| 364 | 0x016C | L1 Bezug Blindenergie | kvarh | Importierte Blindenergie Phase L1 |
| 366 | 0x016E | L2 Bezug Blindenergie | kvarh | Importierte Blindenergie Phase L2 |
| 368 | 0x0170 | L3 Bezug Blindenergie | kvarh | Importierte Blindenergie Phase L3 |
| 370 | 0x0172 | L1 Einspeisung Blindenergie | kvarh | Exportierte Blindenergie Phase L1 |
| 372 | 0x0174 | L2 Einspeisung Blindenergie | kvarh | Exportierte Blindenergie Phase L2 |
| 374 | 0x0176 | L3 Einspeisung Blindenergie | kvarh | Exportierte Blindenergie Phase L3 |
| 376 | 0x0178 | L1 Gesamt Blindenergie | kvarh | Blindenergie gesamt Phase L1 |
| 378 | 0x017A | L2 Gesamt Blindenergie | kvarh | Blindenergie gesamt Phase L2 |
| 380 | 0x017C | L3 Gesamt Blindenergie | kvarh | Blindenergie gesamt Phase L3 |

