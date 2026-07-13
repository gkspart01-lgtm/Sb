
Zugang:
 * IP: 192.168.180.10
 * Benutzer: root
 * Passwort: ${getSecret('rws8ProxmoxPassword')}

## Reverse Proxy
 * IP: 192.168.180.11
 * Tailscale + Nginx Proxy Manager (NPM)
 * Installation: https://community-scripts.org/scripts/nginxproxymanager

### Tailscale Installation
Die due Erreichbarkeit von außen richten wir einen tailscale funnel ein. 

#### 1. In den NPM-LXC-Container gehen
```bash
# Falls du noch nicht drin bist:
pct enter ID-des-NPM-Containers
```

#### 2. Tailscale installieren (Debian/Ubuntu)
```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

#### 3. Tailscale starten & authentifizieren
```bash
tailscale up --advertise-routes=10.0.0.0/8   # oder dein internes Subnetz
```
- Folge dem Link im Terminal und melde dich mit deinem Tailscale-Account an.
- Aktiviere **Funnel** für den Node (im Tailscale Admin-Panel → Machine → Edit → Enable Funnel).

#### 4. Tailscale Funnel konfigurieren
Damit der Traffic auf NPM landet:

```bash
# Beispiel: Funnel für HTTP/HTTPS auf Port 80/443
tailscale funnel --bg --tcp=80 --tcp=443
```

### Wichtige Konfigurationstipps

- **NPM-Ports**: Stelle sicher, dass NPM auf **alle Interfaces** (0.0.0.0) hört, nicht nur localhost.
- **Tailscale Subnet Routes**: Wenn du andere Container/VMs erreichbar machen willst, aktiviere `--advertise-routes` und genehmige sie im Tailscale-Admin-Panel.
- **SSL**: Du kannst Let’s Encrypt über NPM nutzen **oder** Tailscale’s MagicDNS + HTTPS (Tailscale stellt automatisch Zertifikate bereit – oft einfacher).
- **Reihenfolge**: Starte erst Tailscale, dann NPM/OpenResty.