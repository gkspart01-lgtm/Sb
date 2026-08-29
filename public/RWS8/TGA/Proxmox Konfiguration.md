
Zugang:
  * IP: 192.168.180.10
  * Benutzer: root
  * Passwort: ${getSecret('rws8ProxmoxPassword')}

# Installation

https://community-scripts.org/scripts/post-pve-install

## Reverse Proxy
  * IP: 192.168.180.11
  * Tailscale + Nginx Proxy Manager (NPM)
  * Installation: https://community-scripts.org/scripts/nginxproxymanager
    * 💡  PVE Version 9.2.4 (Kernel: 7.0.14-5-pve)
    - 🖥️  Operating System: debian
    - 🌟  Version: 13
    - 📦  Container Type: Unprivileged
    - 🆔  Container ID: 101
    - 🏠  Hostname: nginxproxymanager
    - 💾  Disk Size: 8 GB
    - 🧠  CPU Cores: 2
    - 🛠️  RAM Size: 2048 MiB
    - 🌉  Bridge: vmbr0
    - 📡  IPv4: dhcp
    - 📡  IPv6: auto
    - 🗂️  FUSE Support: no
    - 📡  TUN/TAP Support: yes
    - 📦  Nesting: Enabled
    - 📦  Keyctl: Enabled
    - 🎮  GPU Passthrough: no
    - 💡  Timezone: Europe/Berlin
    - 🔍  Verbose Mode: no

### Tailscale Installation
Für die Erreichbarkeit von außen richten wir einen tailscale funnel ein. 

#### 1. In den NPM-LXC-Container gehen, falls du noch nicht drin bist:
```bash
pct enter 101
```

#### 2. Tailscale installieren (Debian/Ubuntu)
```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

#### 3. Tailscale starten & authentifizieren
```bash
tailscale up
```
- Folge dem Link im Terminal und melde dich mit deinem Tailscale-Account an.

#### 4. Tailscale Funnel konfigurieren
Damit der Traffic auf NPM landet:

```bash
tailscale funnel --bg 80
```

- Aktiviere **Funnel** für den Node (im Tailscale Admin-Panel → Machine → Edit → Enable Funnel).
- SSL/HTTPS läuft nun direkt über tailscale.
- die funnel URL dann als cname Eintrag in der Domain Verwaltung eintragen.

## SSO / Authentifizierung

https://community-scripts.org/scripts/authentik
🧩  Using Advanced Install on node pve

  💡  PVE Version 9.2.4 (Kernel: 7.0.14-5-pve)
  🖥️    erating System: debian
  🌟  Version: 13
  📦  Container Type: Unprivileged
  🆔  Container ID: 103
  🏠  Hostname: rws8.einighof.de
  💾  Disk Size: 16 GB
  🧠  CPU Cores: 4
  🛠️    M Size: 8192 MiB
  🌉  Bridge: vmbr0
  📡  IPv4: dhcp
  📡  IPv6: auto
  🗂️    SE Support: no
  📦  Nesting: Enabled
  📦  Keyctl: Enabled
  🎮  GPU Passthrough: no
  💡  Timezone: Europe/Berlin
  🔍  Verbose Mode: no

