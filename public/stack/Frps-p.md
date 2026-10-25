> **Hinweis:** Der Abschnitt oben ist die einfache Single-Token-Variante (frps :7000, `[auth]`) und bleibt als Referenz stehen. Aktuell läuft die Multi-User-Variante mit `frps-panel` auf Port **7001** – siehe Abschnitt am Ende der Seite.


```yaml
services:
  frps:
    image: fatedier/frps:v0.70.1
    container_name: frps
    restart: unless-stopped
    command: ["-c", "/etc/frp/frps.toml"]
    volumes:
      - "/docker-vol/frps/frps.toml:/etc/frp/frps.toml:ro"
    ports:
      - "7000:7000"
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      
      - "traefik.http.routers.npm-tunnel.rule=HostRegexp(`^.+\\.rws8\\.mehrwert\\.icu$`)"
      - "traefik.http.routers.npm-tunnel.entrypoints=websecure"
      - "traefik.http.routers.npm-tunnel.tls.certresolver=letsencrypt"
      - "traefik.http.routers.npm-tunnel.tls.domains[0].main=rws8.mehrwert.icu"
      - "traefik.http.routers.npm-tunnel.tls.domains[0].sans=*.rws8.mehrwert.icu"
      - "traefik.http.routers.npm-tunnel.service=npm-tunnel-svc"

      - "traefik.http.services.npm-tunnel-svc.loadbalancer.server.port=8081"

networks:
  traefik-net:
    external: true
```

`/docker-vol/frps/frps.toml`:
```toml
bindPort = 7000

[auth]
method = "token"
token = "EIN_LANGES_ZUFAELLIGES_TOKEN"
```

# Client

```bash
wget https://github.com/fatedier/frp/releases/download/v0.70.1/frp_0.70.1_linux_amd64.tar.gz
tar -xzf frp_*.tar.gz
cd frp_*
```

`frpc.toml`:
```toml
serverAddr = "rws8.mehrwert.icu"
serverPort = 7000

[auth]
method = "token"
token = "EIN_LANGES_ZUFAELLIGES_TOKEN"

[[proxies]]
name = "npm-http"
type = "tcp"
localIP = "192.168.180.11"
localPort = 80
remotePort = 8081
```

Als systemd-Service einrichten (frp bringt ein Beispiel-Unit-File mit, `systemctl enable --now frpc`).

```bash
mkdir -p /etc/frp
cp frpc /usr/local/bin/frpc
cp frpc.toml /etc/frp/frpc.toml
cp systemd/frpc.service /etc/systemd/system/frpc.service
nano /etc/systemd/system/frpc.service
systemctl daemon-reload
systemctl enable --now frp
systemctl status frpc
journalctl -u frpc -f
```

# Multi-User mit frps-panel (aktueller Stand)

Statt des einfachen `[auth]`-Tokens läuft frps jetzt mit dem Server-Plugin
[`frps-panel`](https://github.com/yhl452493373/frps-panel) (v2.0.0, benötigt frp >= v0.52):
pro User ein eigener Token in `frps-tokens.toml`, Ports und Domains werden pro User limitiert,
Verwaltung über eine Web-UI auf Port 7200. frps bindet jetzt auf **7001**.

```yaml
services:
  frps:
    image: fatedier/frps:v0.70.1
    container_name: frps
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

  frps-panel:
    # kein offizielles Docker-Image im Projekt -> selbst bauen, siehe Abschnitt "Panel-Image"
    image: frps-panel:local
    container_name: frps-panel
    restart: unless-stopped
    command: ["-c", "/etc/frp/frps-panel.toml"]
    volumes:
      - "/docker-vol/frps-p/frps-panel.toml:/etc/frp/frps-panel.toml"
      # NICHT :ro - das Panel schreibt neue/geänderte User dort hinein
      - "/docker-vol/frps-p/frps-tokens.toml:/etc/frp/frps-tokens.toml"
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

- `ops` deckt nur `Login` und `NewProxy` ab. Damit greifen Auth und Port-/Domain-Limit, aber es kommen keine Verbindungs-Events an - für Traffic-Statistik im Panel zusätzlich `NewWorkConn`, `NewUserConn`, `Ping` in `ops` aufnehmen.
- `admin_user` / `admin_pwd` sind nicht gesetzt, die Web-UI startet also mit dem Default `admin/admin`. `plugin_addr = "0.0.0.0"` ohne veröffentlichten Port heißt: UI nur aus dem Docker-Netz erreichbar (Zugriff z. B. per Traefik-Router), das ist so gewollt.
- `dashboard_addr` / `dashboard_port` / `dashboard_user` / `dashboard_pwd` fehlen, deshalb bleiben die Server-Infos im Panel leer (optional nachtragen).
- `enable = false` oder User entfernen wirkt erst mit Verzögerung; `ports` / `domains` werden nur bei `NewProxy` geprüft.

## Client (User `hl`)

`user` + `metadatas.token` ersetzen den alten `[auth]`-Block - ohne `user` lehnt das Panel den Login ab.

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

## Traefik

Der Router im ersten Abschnitt zeigte auf Port **8081** (User `rws8`). Für `hl` entsprechend **8082**:

```yaml
- "traefik.enable=true"
- "traefik.http.routers.npm-tunnel-hl.rule=Host(`hl.mehrwert.icu`)"
- "traefik.http.routers.npm-tunnel-hl.entrypoints=websecure"
- "traefik.http.routers.npm-tunnel-hl.tls.certresolver=letsencrypt"
- "traefik.http.routers.npm-tunnel-hl.service=npm-tunnel-hl-svc"
- "traefik.http.services.npm-tunnel-hl-svc.loadbalancer.server.port=8082"
```

## Panel-Image

Das Projekt liefert nur Release-Binaries, kein Docker-Image. Minimaler Build aus dem Repo
(`make frps-panel` legt `./bin/frps-panel` plus `assets/` an, CGO ist deaktiviert):

```dockerfile
FROM golang:1.22-alpine AS build
RUN apk add --no-cache git make
WORKDIR /src
COPY . .
RUN make frps-panel

FROM alpine:3.20
WORKDIR /app
COPY --from=build /src/bin/frps-panel /usr/local/bin/frps-panel
COPY --from=build /src/assets ./assets
ENTRYPOINT ["frps-panel"]
```

```bash
docker build -t frps-panel:local .
```
