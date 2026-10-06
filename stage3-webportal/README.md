# Stage 3 — The Back Door (Web Security)

**Owner:** Member 2 (Challenge Design A)

Deliberately vulnerable Flask portal representing `VAULT-03`, Nexora Bank's
internal document portal.

## Expected contents

- `Dockerfile` — based on `python:3.11-slim`
- `requirements.txt` — pinned Flask / Jinja2 versions
- `app/` — the portal source
- `docker-compose.yml` — joins the external `stage3-net` network

## Compose requirements

The service **must** be named `rtr-stage3` (the Nginx config proxies to that
hostname) and must join the externally-created `stage3-net`:

```yaml
services:
  rtr-stage3:
    build: .
    container_name: rtr-stage3
    networks: [stage3-net]
    restart: unless-stopped
    privileged: false
    cap_drop: [ALL]
    mem_limit: 512m
    cpus: 0.5

networks:
  stage3-net:
    external: true
```

No published ports — reachable only through the reverse proxy at
`https://vault-03.rtr.local`.

Must return the `X-Cipher-Hint` response header on successful bypass; Stage 4
depends on it.
