
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
      # - "traefik.http.routers.rws8-tunnel.rule=HostRegexp(`^.+\\.rws8\\.mehrwert\\.icu$`)"
      # - "traefik.http.routers.rws8-tunnel.entrypoints=websecure"
      # - "traefik.http.routers.rws8-tunnel.tls.certresolver=letsencrypt"
      # - "traefik.http.routers.rws8-tunnel.tls.domains[0].main=rws8.mehrwert.icu"
      # - "traefik.http.routers.rws8-tunnel.tls.domains[0].sans=*.rws8.mehrwert.icu"
      # - "traefik.http.routers.rws8-tunnel.service=rws8-tunnel-svc"
      # - "traefik.http.services.rws8-tunnel-svc.loadbalancer.server.port=8081"

      - "traefik.http.routers.hl-tunnel.rule=Host(`hl.mehrwert.icu`)"
      - "traefik.http.routers.hl-tunnel.entrypoints=websecure"
      - "traefik.http.routers.hl-tunnel.tls.certresolver=letsencrypt"
      - "traefik.http.routers.hl-tunnel.tls.domains[0].main=hl.mehrwert.icu"
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

`/docker-vol/frps-p/frps-tokens.toml`:
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

# Client

```bash
wget https://github.com/fatedier/frp/releases/download/v0.70.1/frp_0.70.1_linux_amd64.tar.gz
tar -xzf frp_*.tar.gz
cd frp_*
```

`frpc.toml`:
```toml
serverAddr = "rws8.mehrwert.icu"
serverPort = 7001

[auth]
method = "token"
token = "TOKEN_FUER_RWS8"

[[proxies]]
name = "npm-http"
type = "tcp"
localIP = "192.168.180.11"
localPort = 80
remotePort = 8081
```

`frpc.toml`:
```toml
serverAddr = "hl.mehrwert.icu"
serverPort = 7001
user = "hl"

[auth]
method = "token"
token = "TOKEN_FUER_HL"

[[proxies]]
name = "hl-http"
type = "tcp"
localIP = "<ZIEL-IP>"
localPort = 80
remotePort = 8082
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