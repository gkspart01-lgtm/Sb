```yaml
Services:
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
    networks:
      - internal-net  # Internes Netzwerk für Kommunikation
      - traefik-net   # Traefik-Netzwerk für Routing
    restart: always

  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    network_mode: service:tailscale  # Nutzt Tailscale-Netzwerk
    depends_on:
      - tailscale
    volumes:
      - /docker-vol/uptime-kuma:/app/data
    restart: always
    security_opt:
      - no-new-privileges:true
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.uptime-kuma.rule=Host(`up.mehrwert.icu`)"
      - "traefik.http.routers.uptime-kuma.entrypoints=websecure"
      - "traefik.http.routers.uptime-kuma.tls.certresolver=letsencrypt"
      - "traefik.http.services.uptime-kuma.loadbalancer.server.port=3001"

networks:
  traefik-net:
    external: true
  internal-net:
    driver: bridge
```