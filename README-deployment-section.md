## Deployment

The box can be brought up two ways. Both start the same infrastructure; they
differ only in how the CTFd challenges are loaded. Use whichever you prefer.

### Prerequisites

- Docker Engine 24.x with the Compose v2 plugin
- `openssl`, `curl`, `git`
- A Linux host (tested on Ubuntu 22.04 LTS / Kali 2026.x)

### 1. Clone and configure

```bash
git clone <repo-url> raidtheroot
cd raidtheroot
cp .env.template .env
```

Edit `.env` and replace every `CHANGE_ME`. Generate strong values with
`openssl rand -hex 24`. The `STAGE6_USER` / `STAGE6_PASSWORD` pair is the
foothold credential embedded in the Stage 5 capture — leave it as provided
unless you also regenerate that capture.

Add the challenge hostnames to your hosts file:

```bash
echo "127.0.0.1   ctf.rtr.local vault-03.rtr.local" | sudo tee -a /etc/hosts
```

### 2. Deploy the infrastructure

```bash
./scripts/deploy.sh
```

This starts CTFd, MariaDB, Redis and Nginx, builds the Stage 3 and Stage 6
challenge containers, and generates a self-signed TLS certificate. When it
finishes, CTFd is running but **empty** — no challenges yet.

### 3. Load the challenges — choose ONE path

#### Path A — automatic (one command)

```bash
./scripts/setup-ctfd.sh
```

This completes CTFd's first-run setup using the `CTFD_ADMIN_*` values in
`.env`, then imports `ctfd-config/rtr-event-export.zip`. It finishes by
checking that six challenges are present. If anything fails it says so and
points you to Path B.

#### Path B — manual (browser)

1. Open <https://ctf.rtr.local> and accept the self-signed certificate warning.
2. Complete the CTFd setup wizard (set an admin account; choose **Users** mode).
3. Go to **Admin → Config → Backup → Import**.
4. Upload `ctfd-config/rtr-event-export.zip`.

Both paths produce the same result: six challenges with flags, hints, the
prerequisite chain, and all uploaded artefact files.

### Access points

| What | Where |
|---|---|
| CTFd control plane | <https://ctf.rtr.local> |
| Stage 3 portal (VAULT-03) | <https://vault-03.rtr.local> |
| Stage 6 foothold (BANK-CORE-01) | `ssh <STAGE6_USER>@127.0.0.1 -p 2222` |

### Verify and reset

```bash
./scripts/verify-isolation.sh     # confirms network segmentation (T-05/T-06)
./scripts/reset.sh stage6         # rebuild Stage 6 after a destructive solve
./scripts/reset.sh platform       # full teardown and redeploy
```

### Note on generated artefacts

The Stage 2 image, Stage 4 encrypted file and Stage 5 packet capture are
produced by their `generate.py` scripts and committed for convenience. If you
change `.env` (specifically `STAGE6_PASSWORD`), regenerate the Stage 5 capture
so it matches the Stage 6 container:

```bash
cd stage5-forensics && python3 generate.py && cd ..
# then re-upload capture_10.10.20.50.pcap to the Stage 5 challenge
```
