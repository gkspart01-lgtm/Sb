
```yaml
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-117-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Tue Jun  2 05:48:00 UTC 2026

  System load:  0.16              Processes:             126
  Usage of /:   67.6% of 8.65GB   Users logged in:       0
  Memory usage: 65%               IPv4 address for ens6: 212.227.139.254
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s   just raised the bar for easy, resilient and secure K8s cluster deployment.
   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

26 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


Last login: Tue Jun  2 01:04:33 2026 from 80.133.215.230
root@ubuntu:~# cat /docker-vol/portainer-traefik-stack.yml
services:
  traefik:
    image: traefik:v3.7
    container_name: traefik
    command:
      # Dashboard
      - "--api.dashboard=true"

      # Docker Provider
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"

      # Dynamische Config (Basic Auth etc.)
      - "--providers.file.directory=/config"

      # EntryPoints
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"

      # Let's Encrypt
      - "--certificatesresolvers.letsencrypt.acme.tlschallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.email=f.thomale@gmail.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "/docker-vol/traefik/acme.json:/acme.json"
      - "/docker-vol/traefik/config:/config"
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"

      # === Global HTTP → HTTPS Redirect ===
      - "traefik.http.routers.http-catchall.rule=hostregexp(`{host:.+}`)"
      - "traefik.http.routers.http-catchall.entrypoints=web"
      - "traefik.http.routers.http-catchall.middlewares=redirect-to-https"
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.permanent=true"

      # Dashboard
      - "traefik.http.routers.dashboard.rule=Host(`traefik.mehrwert.icu`)"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls.certresolver=letsencrypt"
      - "traefik.http.routers.dashboard.middlewares=auth-traefik@file"

  portainer:
    image: portainer/portainer-ce:sts
    container_name: portainer
    command: -H unix:///var/run/docker.sock
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "/docker-vol/portainer:/data"
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"

      # Router (nur HTTPS)
      - "traefik.http.routers.portainer.rule=Host(`portainer.mehrwert.icu`)"
      - "traefik.http.routers.portainer.entrypoints=websecure"
      - "traefik.http.routers.portainer.tls.certresolver=letsencrypt"
      - "traefik.http.routers.portainer.middlewares=auth-portainer@file"

      # Wichtig: Portainer läuft intern auf 9000
      - "traefik.http.services.portainer.loadbalancer.server.port=9000"

networks:
  traefik-net:
    name: traefik-net
    driver: bridge
```