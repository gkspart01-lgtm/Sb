
```yaml
services:
  silverbullet:
    image: ghcr.io/silverbulletmd/silverbullet:v2
    container_name: silverbullet
    restart: unless-stopped
    volumes:
      - "/docker-vol/silverbullet:/space"
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      
      # =============================================
      # 1. Öffentliche Routen OHNE Authentifizierung
      # =============================================
      - "traefik.http.routers.silverbullet-public.rule=Host(`sb.mehrwert.icu`) && (Path(`/service_worker.js`) || PathPrefix(`/.client/`))"
      - "traefik.http.routers.silverbullet-public.entrypoints=websecure"
      - "traefik.http.routers.silverbullet-public.tls.certresolver=letsencrypt"
      - "traefik.http.routers.silverbullet-public.priority=100"
      
      # Router
      - "traefik.http.routers.silverbullet.rule=Host(`sb.mehrwert.icu`)"
      - "traefik.http.routers.silverbullet.entrypoints=websecure"
      - "traefik.http.routers.silverbullet.tls.certresolver=letsencrypt"
      - "traefik.http.routers.silverbullet.middlewares=auth-user@file"
      - "traefik.http.routers.silverbullet.priority=50"
      
      # Service Port (SilverBullet läuft intern auf 3000)
      - "traefik.http.services.silverbullet.loadbalancer.server.port=3000"

networks:
  traefik-net:
    external: true
```