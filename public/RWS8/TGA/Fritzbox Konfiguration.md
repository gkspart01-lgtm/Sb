# Login
   - http://192.168.178.1/
   * Benutzer: rws8
   * Passwort: ${getSecret('rws8FritzboxPassword')}
   * 2FA OTP Code: ${getSecret('rws8FritzboxOtpCode')}

# Einstellungen

## Netzwerk
  * http://fritz.box/#/network/settings/critical/ipv4
  - IPv4-Adresse: 192.168.178.1
  - Subnetzmaske: 255.255.248.0

### DHCP
DHCP-Server vergibt IPv4-Adressen:
  - von: 192.168.178.20
  - bis: 192.168.178.199

### Gastnetz
  - IPv4-Adresse: 192.168.184.1
  - Subnetzmaske: 255.255.255.0

## WLAN-Zugang
  - SSID: RWS-8
  - Schlüssel: ${getSecret('rws8Wlan')}

## WLAN-Zugang für Gastzugang/Hotspot
  - SSID: Pappendorfer Pappnasen
  - Schlüssel: ${getSecret('rws8GastWlan')}

# Mesh

## FritzBox DG
  - http://192.168.178.31/
  - Passwort: ${getSecret('rws8FritzboxDGPassword')}

## FritzBox OG
  * http://192.168.178.32/
  * Passwort: ${getSecret('rws8FritzboxOGPassword')}


