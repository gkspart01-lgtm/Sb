
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