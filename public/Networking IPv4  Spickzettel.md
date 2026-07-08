Hier ist dein kompakter, übersichtlicher Spickzettel für IPv4 und Netzwerkgrundlagen.

## 1. IPv4-Adressen & Subnetzmasken

Eine IPv4-Adresse besteht aus 32 Bit, aufgeteilt in 4 Oktette. Die Subnetzmaske bestimmt, welcher Teil die Netzwerk-ID und welcher die Host-ID ist. \[1, 2, 3\]

| CIDR \[4, 5, 6, 7, 8\] | Subnetzmaske | Verfügbare Hosts | Beschreibung / Typische Nutzung |
| :--- | :--- | :--- | :--- |
| /32 | 255.255.255.255 | 1   | Einzelner Host (Loopback / Host-Route) |
| /30 | 255.255.255.252 | 2   | Punkt-zu-Punkt-Verbindungen (Router-Links) |
| /24 | 255.255.255.0 | 254 | Typisches kleines Büro- oder Heimnetzwerk |
| /23 | 255.255.254.0 | 510 | Mittlere Netzwerke |
| /22 | 255.255.252.0 | 1.022 | Größere Firmennetzwerke |
| /16 | 255.255.0.0 | 65.534 | Sehr große Netzwerke / Enterprise |
| /8  | 255.0.0.0 | 16.777.214 | Riesige Infrastrukturen / Provider |

*Hinweis zur Host-Berechnung:* `(2^Anzahl der Host-Bits) - 2`. Die erste Adresse ist die Netzwerkadresse, die letzte ist die Broadcastadresse. Beide dürfen nicht an Hosts vergeben werden. \[9, 10, 11, 12, 13\]

---

## 2. Öffentliche vs. Private IP-Bereiche

*   Öffentliche IPs: Weltweit einzigartig, im Internet routbar.
*   Private IPs: Nicht im Internet routbar, für interne Netzwerke reserviert (RFC 1918). \[14, 15, 16, 17, 18\]

| Klasse \[19, 20, 21, 22, 23\] | Privater IP-Bereich | CIDR-Block |
| :--- | :--- | :--- |
| A   | 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 |
| B   | 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 |
| C   | 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 |

## Besondere Adressbereiche

*   127.0.0.0/8: Loopback (Lokaler Host, `127.0.0.1`).
*   169.254.0.0/16: APIPA (Link-Local, wenn kein DHCP-Server antwortet). \[24, 25\]

---

## 3. VLAN & NAT

*   VLAN (Virtual LAN): Logische Trennung von Netzwerken auf Layer 2 (Switch). Isoliert Datenströme (z.B. Gastnetzwerk von Firmennetzwerk) ohne separate Hardware.
*   NAT (Network Address Translation): Übersetzt private IPs in eine (oder mehrere) öffentliche IPs auf dem Router. Spart IPv4-Adressen und schützt das interne Netz.
    
    *   SNAT (Source NAT / Masquerading): Viele interne Hosts teilen sich eine öffentliche IP beim Surfen.
    *   DNAT (Destination NAT / Portweiterleitung): Leitet Anfragen aus dem Internet an einen bestimmten internen Server weiter. \[26, 27, 28, 29, 30\]
    

---

## 4. Routing, Gateway & MAC

*   Standardgateway (Default Gateway): Die IP-Adresse des Routers im lokalen Netz. Dorthin werden alle Pakete geschickt, deren Ziel außerhalb des eigenen Subnetzes liegt. \[31, 32, 33, 34, 35\]
*   Routing: Der Prozess, bei dem Router den besten Weg für ein Datenpaket durch verschiedene Netzwerke anhand der IP-Zieladresse (Layer 3) bestimmen. \[36\]
*   MAC-Adresse: Physische, weltweit eindeutige Hardware-Adresse einer Netzwerkkarte (48 Bit, z. B. `00:1A:2B:3C:4D:5E`). Arbeitet auf Layer 2. \[37, 38, 39, 40\]
*   ARP (Address Resolution Protocol): Übersetzt eine bekannte IPv4-Adresse in die dazugehörige MAC-Adresse des lokalen Netzes. \[41, 42, 43\]

---

## 5. DNS & DHCP

*   DHCP (Dynamic Host Configuration Protocol): Weist Clients im Netzwerk automatisch IP-Adressen, Subnetzmasken, Gateways und DNS-Server zu.
    
    *   *Der DHCP-Ablauf (DORA):* Discover (Client sucht) → Offer (Server bietet an) → Request (Client fordert an) → Acknowledge (Server bestätigt). \[44, 45, 46, 47, 48\]
    
*   DNS (Domain Name System): Übersetzt menschenlesbare Domainnamen (z.B. `google.com`) in maschinenlesbare IP-Adressen (z.B. `142.250.181.238`). \[49\]

---

Wenn du möchtest, kann ich dir zeigen, wie man Subnetting im Kopf berechnet, oder wir gehen tiefer in ein bestimmtes Thema wie Portweiterleitung (DNAT) oder VLAN-Tagging ein. Welcher Bereich interessiert dich am meisten?

  

\[1\] [https://www.juunit.com](https://www.juunit.com/blog/subnetting)

\[2\] [https://docs.oracle.com](https://docs.oracle.com/cd/E19957-01/820-2980/ipconfig-32/index.html)

\[3\] [https://www.studysmarter.de](https://www.studysmarter.de/ausbildung/ausbildung-in-it/fachinformatiker-systemintegration/subnetting-techniken/)

\[4\] [https://learn.microsoft.com](https://learn.microsoft.com/de-de/troubleshoot/windows-client/networking/tcpip-addressing-and-subnetting)

\[5\] [https://www.narrowin.ch](https://www.narrowin.ch/de/subnet_rechner.html)

\[6\] [https://www.expressvpn.com](https://www.expressvpn.com/de/blog/was-ist-eine-subnetzmaske/)

\[7\] [https://www.expressvpn.com](https://www.expressvpn.com/de/blog/ip-adress-klassen/)

\[8\] [https://faq.tickets.tu-dresden.de](https://faq.tickets.tu-dresden.de/otrs/public.pl?Action=PublicFAQZoom;ItemID=1315)

\[9\] [https://www.computerweekly.com](https://www.computerweekly.com/de/tipp/IP-Adressen-und-Subnetze-Wie-man-IPv4-Subnetzmasken-mit-der-Host-Formel-berechnet)

\[10\] [https://www.juunit.com](https://www.juunit.com/blog/subnetting)

\[11\] [https://docs.oracle.com](https://docs.oracle.com/cd/E19957-01/820-2980/6nehvsgau/index.html)

\[12\] [https://www.juunit.com](https://www.juunit.com/blog/subnetting)

\[13\] [https://www.expressvpn.com](https://www.expressvpn.com/de/blog/was-ist-eine-subnetzmaske/)

\[14\] [https://entwickler.de](https://entwickler.de/webentwicklung/ipv4-ipv6-dns-subnetz-co-was-sie-uber-ip-adressen-wissen-mussen)

\[15\] [https://de.linkedin.com](https://de.linkedin.com/pulse/understanding-ip-addresses-types-classes-aduragbemi-adebo-eljaf?tl=de)

\[16\] [https://de.linkedin.com](https://de.linkedin.com/pulse/complete-guide-ip-address-planning-subnet-masks-gateways-david-zhu-k4qfc?tl=de)

\[17\] [https://support.springernature.com](https://support.springernature.com/de/support/solutions/articles/6000206529-ip-bereiche-pflegen)

\[18\] [https://iktconcept.com](https://iktconcept.com/it-expertise/spezielle-ipv4-adressbereiche/)

\[19\] [https://support.whd.de](https://support.whd.de/hc/de/articles/26012141143313-Wissenswertes-zur-IP-Adresse)

\[20\] [https://www.stefanux.de](https://www.stefanux.de/wiki/doku.php/netzwerke/netzwerkklassen_subnetting_cidr)

\[21\] [https://geekflare.com](https://geekflare.com/de/cidr-ip-addressing/)

\[22\] [https://www.elektronik-kompendium.de](https://www.elektronik-kompendium.de/sites/net/2011211.htm)

\[23\] [https://www.biancahoegel.de](https://www.biancahoegel.de/computer/netz/ip-adresse.html)

\[24\] [https://openbook.rheinwerk-verlag.de](https://openbook.rheinwerk-verlag.de/windows_server_2012r2/04_002.html)

\[25\] [https://iktconcept.com](https://iktconcept.com/it-expertise/spezielle-ipv4-adressbereiche/)

\[26\] [https://www.studysmarter.de](https://www.studysmarter.de/studium/informatik-studium/netzwerke/nat-und-pat-techniken/)

\[27\] [https://www.placetel.de](https://www.placetel.de/ratgeber/nat-network-address-translation)

\[28\] [https://www.learnj.de](https://www.learnj.de/11/doku.php?id=kommunikation:adressierung:start)

\[29\] [https://www.elektronik-kompendium.de](https://www.elektronik-kompendium.de/sites/net/0812111.htm)

\[30\] [https://utm-shop.de](https://utm-shop.de/information/anleitungen/utm-online-hilfe-9.400/network-protection/nat/maskierung-masquerading/)

\[31\] [https://learn.microsoft.com](https://learn.microsoft.com/de-de/windows-server/networking/core-network-guide/core-network-guide)

\[32\] [https://wiki.2n.com](https://wiki.2n.com/hip/conf/latest/de/5-konfigurace-interkomu/5-6-system/5-6-1-sit)

\[33\] [https://www.elektronik-kompendium.de](https://www.elektronik-kompendium.de/sites/net/0811271.htm)

\[34\] [https://www.computerweekly.com](https://www.computerweekly.com/de/ratgeber/Wie-Sie-Subnetze-in-IPv4-und-IPv6-Netzwerken-erstellen)

\[35\] [https://dokumentation.agfeo.de](https://dokumentation.agfeo.de/agfeo_web/dokulib.nsf/ResolvRes.xsp?lu=5372)

\[36\] [https://dt.wara.de](https://dt.wara.de/fobiNetz/routing.pdf)

\[37\] [https://learning.lpi.org](https://learning.lpi.org/de/learning-materials/010-160/4/4.4/4.4_01/)

\[38\] [https://www.ionos.de](https://www.ionos.de/digitalguide/server/knowhow/classless-inter-domain-routing/)

\[39\] [https://de.linkedin.com](https://de.linkedin.com/pulse/understanding-tcpudpipmac-packet-structure-layered-approach-david-zhu-kuogc?tl=de)

\[40\] [https://de.linkedin.com](https://de.linkedin.com/pulse/ip-address-port-numbers-mac-basics-behind-every-network-shaik-reshma-ekisc?tl=de)

\[41\] [https://www.zenarmor.com](https://www.zenarmor.com/docs/de/netzwerkgrundlagen/was-ist-die-arp-tabelle)

\[42\] [https://docs.tia.siemens.cloud](https://docs.tia.siemens.cloud/r/de-de/v21/scalance-x/w/m-projektieren/scalance-m-projektieren/informationen-anzeigen/arp/nachbarn/arp-tabelle)

\[43\] [https://www.ip-insider.de](https://www.ip-insider.de/was-ist-arp-address-resolution-protocol-a-664001/)

\[44\] [https://infosys.beckhoff.com](https://infosys.beckhoff.com/content/1031/twincat_bsd/7872916619.html)

\[45\] [https://interlir.com](https://interlir.com/de/2024/11/05/dhcp-optionen/)

\[46\] [https://learn.microsoft.com](https://learn.microsoft.com/de-de/windows-hardware/customize/desktop/unattend/microsoft-windows-netbt-interfaces-interface-netbiosoptions)

\[47\] [https://docs.xenserver.com](https://docs.xenserver.com/de-de/xencenter/current-release/hosts-management-ip.html)

\[48\] [https://support.epson-europe.com](https://support.epson-europe.com/onlineguides/de/b42wd_nwg/html/setpn_5.htm)

\[49\] [https://www.thomas-krenn.com](https://www.thomas-krenn.com/de/wiki/IT-Glossar)