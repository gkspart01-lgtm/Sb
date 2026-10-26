# frps auf rws8.mehrwert.icu (Multi-User mit frps-panel)

frps v0.70.1 läuft auf dem VPS, gebunden auf **7001**. Auth läuft nicht über einen globalen `[auth]`-Token,
sondern über das Server-Plugin [`frps-panel`](https://github.com/yhl452493373/frps-panel) (v2.0.0, benötigt frp >= v0.52):
pro User ein eigener Token in `frps-tokens.toml`, Ports und Domains werden pro User limitiert, Verwaltung per Web-UI auf Port 7200.

Aktive User:

- `hl` → Port 8082, Domain `hl.mehrwert.icu` (aktiv)
- `rws8` → Port 8081, Domain `rws8.mehrwert.icu` (vorbereitet, noch deaktiviert)

`/docker-vol/frps-p/compose.yaml`:
```yaml
services:
  frps-p:
    image: fatedier/frps:v0.70.1
    container_name: frps-p
    restart: unless-stopped
    command: ["-c", "/etc/frp/frps.toml"]
    volumes:
      - "/docker-vol/frps-p/frps.toml:/etc/frp/frps.toml:ro"
    ports:
      - "7001:7001"
    networks:
      - traefik-net
    depends_on:
      - frps-panel
    labels:
      - "traefik.enable=true"

      # --- rws8-tunnel (noch deaktiviert) ---
      # - "traefik.http.routers.rws8-tunnel.rule=HostRegexp(^.+\\.rws8\\.mehrwert\\.icu$)"
      # - "traefik.http.routers.rws8-tunnel.entrypoints=websecure"
      # - "traefik.http.routers.rws8-tunnel.tls.certresolver=letsencrypt"
      # - "traefik.http.routers.rws8-tunnel.tls.domains[0].main=rws8.mehrwert.icu"
      # - "traefik.http.routers.rws8-tunnel.tls.domains[0].sans=*.rws8.mehrwert.icu"
      # - "traefik.http.routers.rws8-tunnel.service=rws8-tunnel-svc"
      # - "traefik.http.services.rws8-tunnel-svc.loadbalancer.server.port=8081"

      - "traefik.http.routers.hl-tunnel.rule=HostRegexp(`^(.+\\.)?hl\\.mehrwert\\.icu$`)"
      - "traefik.http.routers.hl-tunnel.entrypoints=websecure"
      - "traefik.http.routers.hl-tunnel.tls.certresolver=letsencrypt"
      - "traefik.http.routers.hl-tunnel.tls.domains[0].main=hl.mehrwert.icu"
      - "traefik.http.routers.hl-tunnel.tls.domains[0].sans=*.hl.mehrwert.icu"
      - "traefik.http.routers.hl-tunnel.service=hl-tunnel-svc"
      - "traefik.http.services.hl-tunnel-svc.loadbalancer.server.port=8082"

  frps-panel:
    image: miku114514/frps-panel-docker
    container_name: frps-panel
    restart: unless-stopped
    volumes:
      - "/docker-vol/frps-p/frps-panel.toml:/app/frps-panel.toml:ro"
      - "/docker-vol/frps-p/frps-tokens.toml:/app/frps-tokens.toml"
    networks:
      - traefik-net

networks:
  traefik-net:
    external: true
```

`/docker-vol/frps-p/frps.toml`:
```toml
bindPort = 7001

[[httpPlugins]]
name = "frps-panel"
addr = "frps-panel:7200"
path = "/handler"
ops = ["Login", "NewProxy"]
```

`/docker-vol/frps-p/frps-panel.toml`:
```toml
[common]
plugin_addr = "0.0.0.0"
plugin_port = 7200
admin_keep_time = 0
tls_mode = false
```

`/docker-vol/frps-p/frps-tokens.toml` (Token sind Platzhalter - echte Secrets gehören nicht ins Repo):
```toml
[tokens]
# --- rws8 (noch deaktiviert, spaeter aktivieren) ---
# [tokens.rws8]
# user = "rws8"
# token = "TOKEN_FUER_RWS8"
# ports = [8081]
# domains = ["rws8.mehrwert.icu"]
# enable = true

[tokens.hl]
user = "hl"
token = "TOKEN_FUER_HL"
ports = [8082]
domains = ["hl.mehrwert.icu"]
enable = true
```

Hinweise zu dieser Konfiguration:

- `frps-tokens.toml` ist bewusst **rw** gemountet (kein `:ro`), weil das Panel neue/geänderte User dort hineinschreibt.
- `ops` deckt nur `Login` und `NewProxy` ab. Damit greifen Auth und Port-/Domain-Limit, aber es kommen keine Verbindungs-Events an - für Traffic-Statistik im Panel zusätzlich `NewWorkConn`, `NewUserConn`, `Ping` in `ops` aufnehmen.
- `admin_user` / `admin_pwd` sind nicht gesetzt, die Web-UI startet also mit dem Default `admin/admin`. `plugin_addr = "0.0.0.0"` ohne veröffentlichten Port heißt: UI nur aus dem Docker-Netz erreichbar (Zugriff z. B. per Traefik-Router), das ist so gewollt.
- `dashboard_addr` / `dashboard_port` / `dashboard_user` / `dashboard_pwd` fehlen, deshalb bleiben die Server-Infos im Panel leer (optional nachtragen).
- `enable = false` oder User entfernen wirkt erst mit Verzögerung; `ports` / `domains` werden nur bei `NewProxy` geprüft.
- Router `hl-tunnel` nimmt den Host `hl.mehrwert.icu` an und schiebt ihn auf Port 8082 - also auf denselben Port, den der Client unten als `remotePort` belegt. Für jeden weiteren User ein eigenes Router/Service-Paar mit eigenem Port anlegen.
- `miku114514/frps-panel-docker` ist ein Community-Image (das Projekt selbst liefert nur Release-Binaries); es erwartet `frps-panel.toml` und `frps-tokens.toml` unter `/app/`.

## Subdomains: Zertifikat + Router

`hl.mehrwert.icu` funktioniert, `whoami.hl.mehrwert.icu` aber nicht - dafür müssen zwei Dinge passen:

1. **Router muss Subdomains matchen.** `Host(hl.mehrwert.icu)` matcht nur den Apex. Subdomains bekommen dann
   gar keinen Router: Traefik antwortet `404` und liefert dabei sein Default-Zertifikat aus, weil für den Host
   kein Router mit TLS-Domain existiert. Deshalb `HostRegexp(`^(.+\.)?hl\.mehrwert\.icu$`)`, so wie bei rws8.
2. **Zertifikat muss die Subdomains abdecken.** Ein Client-Zertifikat mit nur `DNS:hl.mehrwert.icu` (Let's Encrypt,
   HTTP-01) hilft Subdomains nicht. Nötig ist `tls.domains[0].sans=*.hl.mehrwert.icu`; Traefik legt `main` auch ins
   Zertifikat, also `DNS:hl.mehrwert.icu` **+** `DNS:*.hl.mehrwert.icu`.

Wichtig: Wildcard geht bei Let's Encrypt **nur per DNS-01**, also muss der Certresolver einen DNS-Provider
(z. B. IONOS) mit API-Zugang konfiguriert haben. Dass es hier grundsätzlich geht, zeigt rws8: dort liegt ein
Zertifikat mit `DNS:*.rws8.mehrwert.icu, DNS:rws8.mehrwert.icu` -der Resolver kann DNS-01, für `hl` fehlt nur das `sans`.

```bash
# Zertifikat prüfen (erwartet: SAN mit *.hl.mehrwert.icu)
echo | openssl s_client -connect whoami.hl.mehrwert.icu:443 -servername whoami.hl.mehrwert.icu 2>/dev/null \
  | openssl x509 -noout -subject -ext subjectAltName

# Label-Änderungen greifen erst, wenn der Container neu erstellt wird
cd /docker-vol/frps-p && docker compose up -d
docker logs -f frps-p        # auf ACME-Fehler achten (DNS-01 / Provider-Credentials)
```

Fehlerbilder auseinanderhalten:

- `CN=TRAEFIK DEFAULT CERT` = kein passendes Zertifikat für diesen Namen (Router matcht nicht oder Cert deckt den Namen nicht ab)
- `404` bei `curl -k` = kein Router matcht den Host (Regel falsch)
- `502` bei `curl -k` = Router matcht, aber Backend ist tot: frps hat auf dem `remotePort` keinen verbundenen Client

```bash
curl -k -o /dev/null -w "%{http_code}\n" https://hl.mehrwert.icu/            # Apex
curl -k -o /dev/null -w "%{http_code}\n" https://whoami.hl.mehrwert.icu/    # Subdomain (Ziel)
```

### "send Login request to plugin error"

Kommt im `frps-p`-Log, wenn frps das Panel auf `frps-panel:7200` nicht erreicht. Prüfreihenfolge:

- läuft der Panel-Container? `docker compose ps`, `docker logs frps-panel`
- gleiche Netz? frps und Panel müssen beide auf `traefik-net` liegen (der `addr` ist ein Docker-DNS-Name)
- Panel-Name im `frps.toml` == Servicename in der Compose (`frps-panel`) und Port == `plugin_port` (`7200`)
- `frps-panel.toml` / `frps-tokens.toml` liegen im Container unter `/app/` (so mountet die Compose), die Tokens-Datei
  muss **rw** gemountet sein
- nach Änderung an `frps.toml` Container **neu erstellen**, ein `restart` reicht nicht

Hinweis: der alte Stack auf Port `7000` (frps + Router `rws8-tunnel` → 8081) läuft parallel weiter, solange `rws8`
nicht auf `frps-p` umgezogen ist.

# Client

```bash
wget https://github.com/fatedier/frp/releases/download/v0.70.1/frp_0.70.1_linux_amd64.tar.gz
tar -xzf frp_*.tar.gz
cd frp_*
```

`frpc.toml` für User `hl` - `user` + `metadatas.token` ersetzen den früheren `[auth]`-Block, ohne `user` lehnt das Panel den Login ab:

```toml
serverAddr = "rws8.mehrwert.icu"
serverPort = 7001

user = "hl"
[metadatas]
token = "TOKEN_FUER_HL"

[[proxies]]
name = "npm-http"
type = "tcp"
localIP = "192.168.180.11"
localPort = 80
remotePort = 8082
```

Als systemd-Service einrichten (frp bringt ein Beispiel-Unit-File mit):

```bash
mkdir -p /etc/frp
cp frpc /usr/local/bin/frpc
cp frpc.toml /etc/frp/frpc.toml
cp systemd/frpc.service /etc/systemd/system/frpc.service
nano /etc/systemd/system/frpc.service
systemctl daemon-reload
systemctl enable --now frpc
systemctl status frpc
journalctl -u frpc -f
```
