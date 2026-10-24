Der Wechselrichter **Growatt 4200TL-XH** (MIN-Serie) stellt über RS485 ein Modbus-RTU-Interface bereit. Diese Seite dokumentiert die Register, die über dieses Interface lesbar sind, inklusive der Skalierungen und der Fallstricke beim Auslesen.

**Stand:** 24.10.2026 · Alle Beispielwerte wurden live am Gerät gelesen.

# Quelle

- Herstellerdokument: **Growatt Inverter Modbus RTU Protocol II V1.39** (gültig für TL-X / TL-XH / MIN, MOD, MID, MAX, SPH, SPA, WIT)
- PDF: `AppNote_Growatt_WIT-Modbus-RTU-Protocol-II-V1.39-English-20240416.pdf` (129 Seiten)
- Aufbereitete Registerliste mit Einheiten und Skalierungen: <https://0xaha.github.io/Growatt_ModbusTCP/developer/protocol-v139/>

Laut Dokument gilt für TL-X/TL-XH (MIN-Typ) der Registerbereich **0–124** (Holding und Input) sowie **3000–3374** (erweiterte Daten). Alles, was in dieser Seite steht, stammt aus dem Bereich 0–124.

# Zugriffsweg

```
[Growatt 4200TL-XH] --RS485--> [USB-RS485 am PiAmWR] --mbusd--> 192.168.180.44:502
```

 - **Port/Pins am WR:** SYS COM, **Pin 3 = RS485A2**, **Pin 4 = RS485B2**
 - **Seriell:** 9600 8N1
 - **Modbus-TCP:** `192.168.180.44:502`, **Unit-ID 1** (Holding-Register 30 = COM-Adresse = 1, bestätigt)
 - **Achtung Polarität:** Bei vertauschtem A/B antwortet der Wechselrichter überhaupt nicht — keine Fehlermeldung, nur Timeouts. Das war die Ursache der ersten erfolglosen Versuche.
 - **Nicht geeignet:** der 8-polige COM-Port. Dessen Pins 3–6 sind in Europa potentialfreie Kontakte (PCI/DRM), keine RS485-Datenleitung.

## Lesen (funktionierendes Beispiel)

```python
import socket, struct

def read(fc, addr, count, unit=1, host="192.168.180.44", port=502):
    pdu = struct.pack(">BHH", fc, addr, count)
    s = socket.create_connection((host, port), timeout=6)
    s.sendall(struct.pack(">HHHB", 1, 0, len(pdu) + 1, unit) + pdu)
    n = struct.unpack(">HHHB", s.recv(7))[2]
    body = b""
    while len(body) < n - 1:
        body += s.recv(n - 1 - len(body))
    s.close()
    if body[0] & 0x80:
        raise IOError(f"Modbus-Exception {body[1]}")
    return [int.from_bytes(body[2 + i*2:4 + i*2], "big") for i in range(body[1] // 2)]

inp = read(4, 0, 125)                                    # alle Live-Werte am Stück
print("PV-Leistung :", (inp[1]  * 65536 + inp[2])  / 10, "W")
print("AC-Leistung :", (inp[35] * 65536 + inp[36]) / 10, "W")
print("Netzspannung:", inp[38] / 10, "V")
print("Frequenz    :", inp[37] / 100, "Hz")
```

Antwortzeit über mbusd: ca. 0,1–0,2 s. Blöcke bis 125 Register funktionieren; bei manchen Adressen scheitern größere Blöcke, dann in 10er-Schritten lesen.

# Wichtig: 32-Bit-Paare

Viele Leistungs- und Energie-Register sind **32 Bit breit und liegen auf zwei aufeinanderfolgenden Registern** (High-Wort zuerst, Low-Wort danach, Einheit 0,1).

Wird ein solches Register einzeln als 16 Bit gelesen, kommen scheinbar plausible, aber völlig falsche Werte heraus. Beispiel aus der Praxis: Register 2 einzeln gelesen ergab `4475` und wurde als „447,5 V PV-Spannung“ gedeutet — tatsächlich ist Register 2 das **Low-Wort der PV-Gesamtleistung** und der Wert bedeutet **447,5 W**.

Betroffene Paare (High/Low, jeweils 0,1-Skalierung):

```
  1/2   PV-Leistung gesamt        35/36  AC-Leistung (Pac)
  5/6   PV1-Leistung              40/41  AC-Scheinleistung
  9/10  PV2-Leistung              53/54  Ertrag heute
 13/14  PV3-Leistung              55/56  Ertrag gesamt
 17/18  PV4-Leistung              57/58  Betriebszeit (Einheit 0,5 s)
 59/60  PV1-Ertrag heute          61/62  PV1-Ertrag gesamt
 63/64  PV2-Ertrag heute          65/66  PV2-Ertrag gesamt
 91/92  PV-Ertrag gesamt         102/103 Max-Leistungsgrenze
```

# Live-Werte (FC04, nur lesbar)

| Adresse | Name | Einheit | Skalierung |
|---|---|---|---|
| 0 | Inverter-Status | — | 0 = Warten, 1 = Normal, 3 = Fehler |
| 1–2 | PV-Leistung gesamt | W | ×0,1 (32 Bit) |
| 3 | PV1-Spannung | V | ×0,1 |
| 4 | PV1-Strom | A | ×0,1 |
| 5–6 | PV1-Leistung | W | ×0,1 (32 Bit) |
| 7 | PV2-Spannung | V | ×0,1 |
| 8 | PV2-Strom | A | ×0,1 |
| 9–10 | PV2-Leistung | W | ×0,1 (32 Bit) |
| 11–18 | PV3 / PV4 (Spannung, Strom, Leistung) | | wie PV1/PV2, beim 4200TL-XH nicht belegt |
| 35–36 | **AC-Ausgangsleistung (Pac)** | W | ×0,1 (32 Bit) |
| 37 | Netzfrequenz | Hz | ×0,01 |
| 38 | Netzspannung | V | ×0,1 |
| 39 | Netzstrom | A | ×0,1 |
| 40–41 | Scheinleistung | VA | ×0,1 (32 Bit) |
| 53–54 | Ertrag heute | kWh | ×0,1 (32 Bit) |
| 55–56 | Ertrag gesamt | kWh | ×0,1 (32 Bit) |
| 57–58 | Betriebszeit gesamt | 0,5 s | ×0,5 (32 Bit) |
| 59–62 | PV1-Ertrag heute / gesamt | kWh | ×0,1 (32 Bit) |
| 63–66 | PV2-Ertrag heute / gesamt | kWh | ×0,1 (32 Bit) |
| 91–92 | PV-Ertrag gesamt | kWh | ×0,1 (32 Bit) |
| 93 | Temperatur Inverter | °C | ×0,1 |
| 94 | Temperatur IPM | °C | ×0,1 |
| 95 | Temperatur Boost | °C | ×0,1 |
| 97 | Batteriespannung (DSP) | V | ×0,1 (ohne Speicher Phantomwert) |
| 98 | Zwischenkreisspannung P-Bus | V | ×0,1 |
| 99 | Zwischenkreisspannung N-Bus | V | ×0,1 |
| 100 | Leistungsfaktor (IPF) | — | 0–20000 |
| 102–103 | Max. Ausgangsleistung (OPFullwatt) | W | ×0,1 (32 Bit) |
| 104 | Derating-Modus | — | 0 = keiner, 7 = **Regelung/Begrenzung aktiv** |
| 105 | Fehlercode (Haupt) | — | 0 = kein Fehler |
| 107 | Fehlercode (Sub) | — | 0 = kein Fehler |
| 110 | Warnbits | — | 0 = keine Warnung |
| 113 | Aktueller Leistungs-Prozentsatz | % | 1 |
| 119 | Batterietyp | — | 0 = Blei, 1 = Lithium |

# Einstellungen (FC03, teilweise schreibbar)

| Adresse | Name | Einheit | Schreibbar | Bemerkung |
|---|---|---|---|---|
| 0 | Ein/Aus (Remote) | — | ja | 1 = an |
| 1 | Safety-Bitmaske | — | ja | Netzanforderung (Ländersatz) |
| 3 | **Active Power Rate** | % | **ja** | Leistungsbegrenzung, 0–100, 255 = aus |
| 4 | Reactive Power Rate | % | ja | Blindleistung, hier 0 |
| 5 | Leistungsfaktor | ×10000 | ja | 10000 = 1,000 |
| 6–7 | Nennleistung (Pmax) | W | nein | ×0,1 (32 Bit), 42000 = 4200 W |
| 8 | Normale PV-Spannung (Vnormal) | V | ja | ×0,1 |
| 9–11 | Firmware-Version (H/M/L) | ASCII | nein | Modell-/Versionskennung |
| 12–14 | Firmware-Version 2 (H/M/L) | ASCII | nein | Steuerungs-Firmware |
| 17 | Einschaltspannung PV (Vpv start) | V | ja | ×0,1 |
| 19 | Neustart-Verzögerung | — | ja | nach Fehler |
| 20 | Start-Rampe | — | ja | |
| 23–27 | Seriennummer | ASCII | nein | 2 Zeichen pro Register |
| 30 | COM-Adresse | — | ja | 1 |
| 43 | Gerätetypcode (DTC) | — | nein | **5100 = MIN TL-XH** |
| 44 | Tracker-/Phasen-Konfiguration (TP) | — | nein | 513 |
| 52–83 | Netzschutz-Parameter | — | ja | Spannungs-/Frequenzgrenzen, Zeiten |
| 122 | ExportLimit aktiv | — | ja | 0 = aus |
| 123 | ExportLimit Power Rate | 0,1 % | ja | wirkt nur bei 122 = 1 |

# Leistungsbegrenzung in der Praxis

## Weg 1: Register 3 (Active Power Rate)

- Ganzzahlige Prozentwerte, **1 % = 42 W** bei 4200 W Nennleistung
- `22 % = 924 W`, `24 % = 1008 W`, `29 % = 1218 W`, `71 % = 2982 W`
- **100 = volle Leistung, 255 = keine Begrenzung**

## Weg 2: ExportLimit (Register 122/123)

- Auflösung **0,1 %** statt 1 %, also 4,2 W-Schritte
- Setzt voraus, dass Register 122 auf 1 steht; 123 enthält den Prozentwert ×10
- Derzeit **nicht** verwendet (122 = 0). Falls feinere Stufen gebraucht werden, ist das der Weg — vorher prüfen, wie sich ExportLimit und Register 3 zueinander verhalten (beide begrenzen die Wirkleistung, die kleinere Grenze sollte gewinnen).

## Schreiben über mbusd: Exception 11 ist kein Fehlschlag

Ein `FC06`-Schreibzugriff auf Register 3 über mbusd antwortet mit **Exception 11 (Gateway Target Failed To Respond)**, obwohl der Wert **tatsächlich geschrieben wird**. Mehrfach reproduziert und jeweils per Rücklesung bestätigt.

Deshalb gilt: Nach jedem Schreibvorgang das Register **zurücklesen** und erst dann von Erfolg sprechen. Wer die Exception als Fehler wertet, baut eine Fehlerbehandlung, die bei jedem funktionierenden Schreibvorgang Alarm auslöst.

# Momentaufnahme 24.10.2026, 16:30

```
Status                    1 (Normal), keine Fehler, keine Warnungen
PV-Leistung gesamt        306,5 W
  String 1                120,0 V   1,1 A   138,7 W
  String 2                187,4 V   0,9 A   167,8 W
AC-Leistung               301,0 W
Netzspannung              234,6 V   1,3 A   (301,3 VA)
Netzfrequenz              49,98 Hz
Leistungsfaktor           1,000
Ertrag heute              2,3 kWh
Ertrag gesamt             375,5 kWh
  PV1 gesamt              21,9 kWh
  PV2 gesamt              368,4 kWh
  PV gesamt               390,3 kWh
Betriebszeit              ca. 49 Tage
Temperatur                22,4 °C (IPM 22,4 °C)
Zwischenkreis P-Bus       360,4 V
Derating-Modus            7 (Regelung/Begrenzung aktiv)
Leistungs-Prozentsatz     7 %
Active Power Rate         24 % (= 1008 W)
Gerätetyp (DTC)           5100 (MIN TL-XH)
Seriennummer              ZKL1CKL050
```

# Offene Punkte / Beobachtungen

- **Register oberhalb 124** sind für die MIN-Serie im Protokolldokument nicht beschrieben. Werte, die dort trotzdem antworten (z. B. 186–199), lassen sich keinem dokumentierten Feld zuordnen — nicht als Messwerte verwenden.
- **String-Asymmetrie:** String 1 liefert bei 120 V deutlich weniger als String 2 bei 187 V, kumuliert 21,9 kWh gegen 368,4 kWh. Das deutet auf weniger Module an String 1 oder einen später angeschlossenen String hin. Zu prüfen, wenn beide Strings gleich aufgebaut sein sollten.
- **Verhalten bei Neustart:** Ob Register 3 einen Neustart des Wechselrichters (Nacht/Wiedereinschalten) überlebt, ist noch nicht verifiziert. Für eine dauerhafte Begrenzung muss der Wert nach dem Aufwachen neu gesetzt werden.

# Änderungsverlauf

- **24.10.2026** — Erstfassung: Registerübersicht, Zugriffsweg, 32-Bit-Fallstrick, Momentaufnahme
