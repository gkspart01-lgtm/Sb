# rathole (Tunnel für `hl.mehrwert.icu`)

[`rathole`](https://github.com/rapiz1/rathole) als Tunnel zwischen VPS und LAN - Alternative zu frp.
Der VPS hält den TLS-Endpunkt (Traefik), rathole transportiert nur TCP; der Client im LAN schiebt den Verkehr
weiter an NPM (`192.168.180.11:80`).

Aufteilung:

- **VPS**: Control-Port **8443/tcp** öffentlich, Service-Ports **8081/8082 nur intern** im `traefik-net`
- **LAN**: rathole-Client pro Service mit Token, Ziel NPM
- **hl** → Port 8082, Domain `hl.mehrwert.icu` (aktiv)
- **rws8** → Port 8081, Domain `rws8.mehrwert.icu` (vorbereitet, auskommentiert)

Docker-Image: `rapiz1/rathole:v0.5.0` (12,5 MB, Rust, `ENTRYPOINT ./rathole`, läuft als uid 1000).
Deshalb muss die Config für andere lesbar sein: `chmod 644 server.toml`.

`/docker-vol/rathole/compose.yaml`:
```yaml
services:
  rathole:
    image: rapiz1/rathole:v0.5.0
    container_name: rathole
    restart: unless-stopped
    command: ["--server", "/app/config/server.toml"]
    environment:
      - "RUST_LOG=info"
    volumes:
      - "/docker-vol/rathole/server.toml:/app/config/server.toml:ro"
    ports:
      - "8443:8443"          # Control-Channel (muss von außen erreichbar sein)
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"

      # --- hl (aktiv) ---
      - "traefik.http.routers.rathole-hl.rule=HostRegexp(`^(.+\\.)?hl\\.mehrwert\\.icu$`)"
      - "traefik.http.routers.rathole-hl.entrypoints=websecure"
      - "traefik.http.routers.rathole-hl.tls.certresolver=letsencrypt"
      - "traefik.http.routers.rathole-hl.tls.domains[0].main=hl.mehrwert.icu"
      - "traefik.http.routers.rathole-hl.tls.domains[0].sans=*.hl.mehrwert.icu"
      - "traefik.http.routers.rathole-hl.service=rathole-hl-svc"
      - "traefik.http.services.rathole-hl-svc.loadbalancer.server.port=8082"

      # --- rws8 (vorbereitet, noch deaktiviert) ---
      # - "traefik.http.routers.rathole-rws8.rule=HostRegexp(`^(.+\\.)?rws8\\.mehrwert\\.icu$`)"
      # - "traefik.http.routers.rathole-rws8.entrypoints=websecure"
      # - "traefik.http.routers.rathole-rws8.tls.certresolver=letsencrypt"
      # - "traefik.http.routers.rathole-rws8.tls.domains[0].main=rws8.mehrwert.icu"
      # - "traefik.http.routers.rathole-rws8.tls.domains[0].sans=*.rws8.mehrwert.icu"
      # - "traefik.http.routers.rathole-rws8.service=rathole-rws8-svc"
      # - "traefik.http.services.rathole-rws8-svc.loadbalancer.server.port=8081"

networks:
  traefik-net:
    external: true
```

Keypair für den verschlüsselten Control-Channel erzeugen (macht nur lokal ein X25519-Paar, kein Netzwerkzugriff):

```bash
docker run --rm rapiz1/rathole:v0.5.0 --genkey
# Private Key:
# <PRIVATER_SCHLUESSEL_SERVER>
# Public Key:
# <OEFFENTLICHER_SCHLUESSEL_SERVER>
```

Der **private** Key bleibt auf dem VPS, der **öffentliche** kommt in die Client-Config.

`/docker-vol/rathole/server.toml` (Token/Keys sind Platzhalter - echte Secrets gehören nicht ins Repo, `chmod 600`):
```toml
[server]
bind_addr = "0.0.0.0:8443"
heartbeat_interval = 30

[server.transport]
type = "noise"                       # Control-Channel verschlüsselt (Noise_NK_25519_ChaChaPoly_BLAKE2s)

[server.transport.noise]
local_private_key = "PRIVATER_SCHLUESSEL_SERVER"

[server.services.hl]
token = "TOKEN_FUER_HL"
bind_addr = "0.0.0.0:8082"       # nur intern, kein ports:-Eintrag in der Compose

# --- rws8 (vorbereitet, noch deaktiviert) ---
# [server.services.rws8]
# token = "TOKEN_FUER_RWS8"
# bind_addr = "0.0.0.0:8081"
```

# Client (LAN)

`client.toml`:
```toml
[client]
remote_addr = "hl.mehrwert.icu:8443"
retry_interval = 1

[client.transport]
type = "noise"                       # muss zum Server passen

[client.transport.noise]
remote_public_key = "OEFFENTLICHER_SCHLUESSEL_SERVER"

[client.services.hl]
token = "TOKEN_FUER_HL"
local_addr = "192.168.180.11:80"
```

```bash
wget https://github.com/rapiz1/rathole/releases/download/v0.5.0/rathole-x86_64-unknown-linux-musl.zip
unzip rathole-*.zip && chmod +x rathole && mv rathole /usr/local/bin/
mkdir -p /etc/rathole && cp client.toml /etc/rathole/
```

`/etc/systemd/system/rathole.service`:
```ini
[Unit]
Description=rathole client
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/rathole --client /etc/rathole/client.toml
Restart=always
RestartSec=2

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload && systemctl enable --now rathole
journalctl -u rathole -f
```

# Ports

- **8443/tcp** veröffentlichen (dorthin verbindet der Client). Bei aktiver UFW: `ufw allow 8443/tcp`
- **8081/8082** **nicht** in `ports:` eintragen - Traefik erreicht sie über `traefik-net`, von außen sind sie zu
- kein UDP nötig, Traefik bleibt auf 80/443

# Prüfen

```bash
# Control-Port offen?
nc -zv <vps-ip> 8443

# Service-Ports dürfen von außen NICHT offen sein
nc -zv <vps-ip> 8082      # erwartet: refused/filtered

# Client-Log (VPS-Seite)
docker logs -f rathole    # "service [hl] started" o.ä., keine Reconnect-Schleife

# Traefik: Zertifikat + Erreichbarkeit
echo | openssl s_client -connect whoami.hl.mehrwert.icu:443 -servername whoami.hl.mehrwert.icu 2>/dev/null \
  | openssl x509 -noout -subject -ext subjectAltName     # erwartet: SAN mit *.hl.mehrwert.icu
curl -k -o /dev/null -w "%{http_code}\n" https://hl.mehrwert.icu/
```

Fehlerbilder wie bei frp: `404` = kein Router matcht den Host, `502` = Router matcht, aber auf 8082 hängt kein Client
(Client down, falscher Token, Port belegt), `TRAEFIK DEFAULT CERT` = Zertifikat deckt den Namen nicht ab.

# Hinweise

- Der Control-Channel läuft mit **Noise** (`Noise_NK_25519_ChaChaPoly_BLAKE2s`): verschlüsselt und der Server wird
  per Public Key authentifiziert, ohne Zertifikate. `type = "noise"` muss auf **beiden** Seiten stehen, sonst
  schlägt der Handshake fehl (Client reconnectet endlos, Server-Log zeigt einen Transport-Fehler).
  Keypair nur einmal erzeugen: neuer `--genkey`-Lauf entwertet den alten Client.
  Bei anderer Noise-Pattern-Wahl müssen `pattern`-Angaben auf beiden Seiten identisch sein.
- Der Service-Token (`[server.services.*] token`) bleibt zusätzlich nötig - Noise sichert den Transport, der Token
  autorisiert den Service.
- Kein Healthcheck möglich (kein Shell im Image), `restart: unless-stopped` genügt.
- Router-Namen (`rathole-hl`) bewusst anders als beim frp-Stack (`hl-tunnel`), damit beide parallel laufen können.
  Zeigen zwei Router auf denselben Host, gewinnt der zuletzt registrierte - dann kommen zufällige 502er.
- rathole hat **keine** User-Verwaltung, keine Port-/Domain-Limits und keine Spend-Anzeige. Wenn du mehrere Nutzer
  mit eigenen Tokens und Kontingenten brauchst, ist der `frps-panel`-Stack die bessere Wahl; rathole ist der schlanke,
  schnelle Tunnel für ein bis wenige Ziele.
