# Setup

## Installation

https://community-scripts.org/scripts/haos-vm
Zugang:
 * IP 192.168.180.20
 * Passwort: ${getSecret('rws8HaosPassword')}


## Reverse proxy

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 192.168.180.11
```

## Single sign on (SSO)

https://community-scripts.org/scripts/authentik

https://integrations.goauthentik.io/miscellaneous/home-assistant/
We use https://github.com/christiaangoossens/hass-oidc-auth as oidc integration. 

