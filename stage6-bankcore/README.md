# Stage 6 — Find the Ghost (Linux / System Security)

**Owner:** Member 3 (Challenge Design B)

`BANK-CORE-01` — Debian container with a planted service account, scheduled
persistence mechanism, privilege-escalation path and root-owned evidence file.

## Expected contents

- `Dockerfile` — based on `debian:bookworm-slim`
- `provision/` — scripts creating `svc_ghost`, the cron job, permissions
- `docker-compose.yml` — own isolated `stage6-net`

## Compose requirements

Service **must** be named `rtr-stage6` (verify-isolation.sh checks it):

```yaml
services:
  rtr-stage6:
    build: .
    container_name: rtr-stage6
    networks: [stage6-net]
    ports: ["2222:22"]
    restart: unless-stopped
    privileged: false
    cap_drop: [ALL]
    cap_add: [CHOWN, SETUID, SETGID]
    mem_limit: 1g
    cpus: 0.5

networks:
  stage6-net:
    driver: bridge
```

No host volume mounts — the reset depends on the filesystem being ephemeral.

The foothold credential must match `STAGE6_USER` / `STAGE6_PASSWORD` in `.env`
**and** the credential Member 3 embeds in the Stage 5 packet capture.
