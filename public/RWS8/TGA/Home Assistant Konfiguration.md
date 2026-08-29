# Setup

## Installation

https://community-scripts.org/scripts/haos-vm

  * Using HAOS Version: 18.1
  - 🆔  Virtual Machine ID: 100
  - 📦  Machine Type: q35
  - 💾  Disk Size: 32G
  - 💾  Disk Cache: Write Through
  - 🏠  Hostname: haos18-1
  - 🖥️  CPU Model: Host
  - 🧠  CPU Cores: 2
  - 🛠️  RAM Size: 2048
  - 🌉  Bridge: vmbr0
  - 🔗  MAC Address: 02:5E:AF:7C:94:9C
  - 🏷️  VLAN: Default
  - ⚙️  Interface MTU Size: Default

Zugang:
 * IP 192.168.180.20
 * Passwort: ${getSecret('rws8HaosPassword')}


## Reverse proxy
configuration.yaml:
```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 192.168.180.11
```

## Single sign on (SSO)

https://community-scripts.org/scripts/authentik

https://integrations.goauthentik.io/miscellaneous/home-assistant/
We use https://github.com/christiaangoossens/hass-oidc-auth as oidc integration. 

Discovery URL:
https://auth.rws8.mehrwert.icu/application/o/home-assistant/.well-known/openid-configuration


## SDM630 Stromzähler

[[public/RWS8/TGA/Stromzähler SDM630 Konfiguration]]

configuration.yaml
```yaml
modbus: !include modbus_sdm630.yaml
```

modbus_sdm630.yaml
```yaml
- name: "sdm630_bus"
  type: tcp
  host: 192.168.178.43
  port: 502
  sensors:
    # ══════════ SDM630 Slave 21 — ALLG ══════════
    - name: "SDM630 ALLG Wirkleistung L1"
      slave: 21
      address: 12
      input_type: input
      data_type: float32
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      unique_id: sdm630_21_12
      scan_interval: 5
    - name: "SDM630 ALLG Wirkleistung L2"
      slave: 21
      address: 14
      input_type: input
      data_type: float32
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      unique_id: sdm630_21_14
      scan_interval: 5
    - name: "SDM630 ALLG Wirkleistung L3"
      slave: 21
      address: 16
      input_type: input
      data_type: float32
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      unique_id: sdm630_21_16
      scan_interval: 5
    - name: "SDM630 ALLG Gesamtwirkleistung"
      slave: 21
      address: 52
      input_type: input
      data_type: float32
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      unique_id: sdm630_21_52
      scan_interval: 1
    - name: "SDM630 ALLG Gesamtwirkenergie Bezug"
      slave: 21
      address: 72
      input_type: input
      data_type: float32
      unit_of_measurement: "kWh"
      device_class: energy
      state_class: total_increasing
      unique_id: sdm630_21_72
      scan_interval: 300
    - name: "SDM630 ALLG Gesamtwirkenergie Einspeisung"
      slave: 21
      address: 74
      input_type: input
      data_type: float32
      unit_of_measurement: "kWh"
      device_class: energy
      state_class: total_increasing
      unique_id: sdm630_21_74
      scan_interval: 300

    # ══════════ SDM630 Slave 22 — WE01 ══════════
    - name: "SDM630 WE01 Wirkleistung L1"
      slave: 22
      address: 12
      input_type: input
      data_type: float32
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      unique_id: sdm630_22_12
      scan_interval: 5
    - name: "SDM630 WE01 Wirkleistung L2"
      slave: 22
      address: 14
      input_type: input
      data_type: float32
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      unique_id: sdm630_22_14
      scan_interval: 5
    - name: "SDM630 WE01 Wirkleistung L3"
      slave: 22
      address: 16
      input_type: input
      data_type: float32
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      unique_id: sdm630_22_16
      scan_interval: 5
    - name: "SDM630 WE01 Gesamtwirkleistung"
      slave: 22
      address: 52
      input_type: input
      data_type: float32
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      unique_id: sdm630_22_52
      scan_interval: 1
    - name: "SDM630 WE01 Gesamtwirkenergie Bezug"
      slave: 22
      address: 72
      input_type: input
      data_type: float32
      unit_of_measurement: "kWh"
      device_class: energy
      state_class: total_increasing
      unique_id: sdm630_22_72
      scan_interval: 300
    - name: "SDM630 WE01 Gesamtwirkenergie Einspeisung"
      slave: 22
      address: 74
      input_type: input
      data_type: float32
      unit_of_measurement: "kWh"
      device_class: energy
      state_class: total_increasing
      unique_id: sdm630_22_74
      scan_interval: 300

    # ══════════ SDM630 Slave 23 — WE02 ══════════
    - name: "SDM630 WE02 Wirkleistung L1"
      slave: 23
      address: 12
      input_type: input
      data_type: float32
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      unique_id: sdm630_23_12
      scan_interval: 5
    - name: "SDM630 WE02 Wirkleistung L2"
      slave: 23
      address: 14
      input_type: input
      data_type: float32
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      unique_id: sdm630_23_14
      scan_interval: 5
    - name: "SDM630 WE02 Wirkleistung L3"
      slave: 23
      address: 16
      input_type: input
      data_type: float32
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      unique_id: sdm630_23_16
      scan_interval: 5
    - name: "SDM630 WE02 Gesamtwirkleistung"
      slave: 23
      address: 52
      input_type: input
      data_type: float32
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      unique_id: sdm630_23_52
      scan_interval: 1
    - name: "SDM630 WE02 Gesamtwirkenergie Bezug"
      slave: 23
      address: 72
      input_type: input
      data_type: float32
      unit_of_measurement: "kWh"
      device_class: energy
      state_class: total_increasing
      unique_id: sdm630_23_72
      scan_interval: 300
    - name: "SDM630 WE02 Gesamtwirkenergie Einspeisung"
      slave: 23
      address: 74
      input_type: input
      data_type: float32
      unit_of_measurement: "kWh"
      device_class: energy
      state_class: total_increasing
      unique_id: sdm630_23_74
      scan_interval: 300

    # ══════════ SDM630 Slave 24 — WE03 ══════════
    - name: "SDM630 WE03 Wirkleistung L1"
      slave: 24
      address: 12
      input_type: input
      data_type: float32
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      unique_id: sdm630_24_12
      scan_interval: 5
    - name: "SDM630 WE03 Wirkleistung L2"
      slave: 24
      address: 14
      input_type: input
      data_type: float32
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      unique_id: sdm630_24_14
      scan_interval: 5
    - name: "SDM630 WE03 Wirkleistung L3"
      slave: 24
      address: 16
      input_type: input
      data_type: float32
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      unique_id: sdm630_24_16
      scan_interval: 5
    - name: "SDM630 WE03 Gesamtwirkleistung"
      slave: 24
      address: 52
      input_type: input
      data_type: float32
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      unique_id: sdm630_24_52
      scan_interval: 1
    - name: "SDM630 WE03 Gesamtwirkenergie Bezug"
      slave: 24
      address: 72
      input_type: input
      data_type: float32
      unit_of_measurement: "kWh"
      device_class: energy
      state_class: total_increasing
      unique_id: sdm630_24_72
      scan_interval: 300
    - name: "SDM630 WE03 Gesamtwirkenergie Einspeisung"
      slave: 24
      address: 74
      input_type: input
      data_type: float32
      unit_of_measurement: "kWh"
      device_class: energy
      state_class: total_increasing
      unique_id: sdm630_24_74
      scan_interval: 300
```


