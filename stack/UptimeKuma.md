```yaml
services:
  tailscale:
    image: tailscale/tailscale:latest
    container_name: tailscale
    hostname: uptime-kuma
    environment:
      - TS_AUTHKEY=tskey-auth-...
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_USERSPACE=false
    volumes:
      - /docker-vol/tailscale:/var/lib/tailscale
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW
    ports:
      - "3001:3001"
    restart: always

  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    network_mode: service:tailscale
    depends_on:
      - tailscale
    volumes:
      - /docker-vol/uptime-kuma:/app/data
    restart: always
    security_opt:
      - no-new-privileges:true
```