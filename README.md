# RaidTheRoot (RTR) — Ghost of the Vault
 
A six-stage Capture The Flag Play Box simulating the investigation of an
insider-originated breach at **Nexora Bank**, a fictional digital bank. The
participant plays a Junior Penetration Tester tracing an attacker known only
as **Ghost** from public reconnaissance through to full internal compromise.
 
**IE3132 Penetration Testing — Assignment 02**
Sri Lanka Institute of Information Technology
 
| Member | Student ID | Responsibility |
|---|---|---|
| Algiriya Vitharanage R.V. | IT24100446 | CTF Platform & Architecture |
| Jayashan D H J | IT24101430 | Challenge Design A (Stages 1–3) |
| Chandler S. R | IT24102116 | Challenge Design B (Stages 4–6) |
| Hansika H A S | IT24103666 | Integration, Testing & Documentation |
 
---
 
## The box at a glance
 
Six stages, each producing an artefact that unlocks the next — one continuous
investigation, not six separate puzzles.
 
| Stage | Challenge | Domain | Difficulty | Unlocks next via |
|---|---|---|---|---|
| 1 | The First Trace | OSINT / Recon | Easy | employee username |
| 2 | The Ghost File | Steganography | Easy | hostname VAULT-03 |
| 3 | The Back Door | Web Security | Moderate | encrypted file + cipher hint |
| 4 | The Locked Vault | Cryptography | Moderate | internal IP:port |
| 5 | Inside the Network | Network Forensics | Moderate–Hard | hostname + SSH credential |
| 6 | Find the Ghost | Linux / System Security | Hard | root — Ghost's identity |
 
Flag format: `RTR{stage-slug_descriptor}`, case-insensitive.
 
---
 
## Requirements
 
- Docker Engine 24.x with the Compose v2 plugin
- `openssl`, `curl`, `git`, `python3`
- A Linux host (tested on Ubuntu 22.04 LTS and Kali 2026.x)
- 4 vCPU / 8 GB RAM / 40 GB disk recommended
---
 
## Deployment
 
Three commands bring the whole box up. Run them in this order from the
repository root:
 
```bash
git clone https://github.com/avrviranga/raidtheroot.git
cd raidtheroot
chmod +x scripts/*.sh
 
./scripts/deploy.sh         # 1. build & start everything (auto-creates .env)
./scripts/setup-ctfd.sh     # 2. import the six challenges into CTFd
./scripts/doctor.sh         # 3. add the hosts entry and verify everything
```
 
Then open <https://ctf.rtr.local> (accept the self-signed certificate
warning) and the box is ready.
 
### What each step does
 
**1. `deploy.sh`** — creates `.env` automatically from `.env.template` if it
is missing, generates a self-signed TLS certificate, and starts CTFd,
MariaDB, Redis, Nginx, and the Stage 3 and Stage 6 challenge containers. When
it finishes, CTFd is running but **empty** — the challenges load in step 2.
 
For a shared deployment, review `.env` afterwards and set strong values
(`openssl rand -hex 24`). The `STAGE6_USER` / `STAGE6_PASSWORD` pair is the
Stage 6 foothold credential, embedded in the Stage 5 capture for participants
to recover; it is pinned in `.env.template` so the capture and the container
always match — leave it as provided unless you also regenerate the capture.
 
**2. `setup-ctfd.sh`** — imports `ctfd-config/rtr-event-export.zip` directly
inside the CTFd container using CTFd's own `import_ctf` command, then restarts
and verifies six challenges are present. This runs over `docker exec`, so it
does not need the hosts entry yet. The admin login afterwards is the account
stored in the export.
 
**3. `doctor.sh`** — adds the `ctf.rtr.local` / `vault-03.rtr.local` entries to
`/etc/hosts` (needs sudo), then checks Docker, `.env`, the certificate, the
containers, CTFd reachability, and the Stage 5 ↔ Stage 6 credential sync. It
reports everything green when the box is ready.
 
### Manual CTFd import (alternative to step 2)
 
If you prefer to import through the browser instead of `setup-ctfd.sh`:
 
1. Open <https://ctf.rtr.local> and complete the CTFd setup wizard (**Users** mode).
2. Go to **Admin → Config → Backup → Import**.
3. Upload `ctfd-config/rtr-event-export.zip`.
Either way the result is the same: six challenges with flags, hints, the
prerequisite chain, and all uploaded artefact files.
 
### Access points
 
| What | Where |
|---|---|
| CTFd control plane | <https://ctf.rtr.local> |
| Stage 3 portal (VAULT-03) | <https://vault-03.rtr.local> |
| Stage 6 foothold (BANK-CORE-01) | `ssh <STAGE6_USER>@127.0.0.1 -p 2222` |
 
Stages 1, 2, 4 and 5 are static forensic artefacts downloaded from their CTFd
challenge pages once the preceding stage is solved.
 
---
 
## Verify, reset, clean up
 
```bash
./scripts/doctor.sh              # diagnose + auto-fix common setup issues
./scripts/verify-isolation.sh    # confirm network segmentation (T-05 / T-06)
python3 tests/chain_test.py      # end-to-end integration test (needs sshpass)
./scripts/reset.sh stage6        # rebuild Stage 6 after a destructive solve
./scripts/reset.sh platform      # full teardown and redeploy
./scripts/teardown.sh            # remove the environment (dry-run by default)
```
 
Stage 6 must be **rebuilt**, not restarted: the intended solution overwrites a
scheduled script inside the container, so only a force-recreate restores the
planted account, cron job, permissions and evidence file. Static stages are
reset by re-running their generator scripts, which use fixed seeds and produce
byte-identical output.
 
---
 
## Architecture
 
```
participant ──HTTPS/443──> Nginx ──┬──> CTFd ──> MariaDB
                                   │      └────> Redis        [rtr-control]
                                   └──> Stage 3 portal        [stage3-net]
 
participant ──SSH/2222───────────> Stage 6 BANK-CORE-01       [stage6-net]
```
 
One isolated bridge network per live challenge. Only Nginx spans more than one
network. Challenge containers have no route to each other or to the control
plane, so full compromise of Stage 6 — the intended outcome — yields no other
stage's flag and cannot reach the database where flags are stored.
 
Challenge containers run unprivileged, drop all Linux capabilities except the
minimum the intended path requires, mount no host volumes, and carry explicit
CPU and memory limits.
 
---
 
## Repository layout
 
```
raidtheroot/
├── README.md                   this file
├── docker-compose.yml          control plane: nginx, ctfd, mariadb, redis
├── .env.template               environment template (.env is gitignored)
├── nginx/nginx.conf            reverse proxy, TLS, rate limiting, host routing
├── scripts/
│   ├── deploy.sh               bring the box up from clean state
│   ├── setup-ctfd.sh           automatic CTFd event import
│   ├── verify-isolation.sh     network isolation checks (T-05 / T-06)
│   ├── reset.sh                per-stage and full-box recovery
│   ├── teardown.sh             remove the environment (dry-run default)
│   ├── doctor.sh               diagnose + auto-fix common setup issues
│   └── package-submission.sh   build the courseweb submission archive
├── ctfd-config/                CTFd event export (challenges, flags, prereqs)
├── stage1-osint/               OSINT artefacts + solution notes
├── stage2-stego/               carrier image + generator + solver
├── stage3-webportal/           vulnerable Flask portal (live) + exploit
├── stage4-crypto/              encrypted artefact + generator + solver
├── stage5-forensics/           synthetic PCAP + Scapy generator + solver
├── stage6-bankcore/            BANK-CORE-01 container (live) + solver
├── tests/                      chain test, methodology, recorded results
└── docs/                       architecture diagram, CTFd setup reference
```
 
Each stage folder has its own README with the build, the solution path, and
the reset procedure.
 
---
 
## Troubleshooting
 
Run the doctor; it diagnoses and auto-fixes the common issues (missing
`/etc/hosts` entry, missing `.env`, containers down, CTFd not answering, and
Stage 5 ↔ Stage 6 credential drift):
 
```bash
./scripts/doctor.sh          # fix what it safely can
./scripts/doctor.sh --check  # diagnose only, change nothing
```
 
| Symptom | Cause | Fix |
|---|---|---|
| `ERR_NAME_NOT_RESOLVED` on ctf.rtr.local | hosts entry missing | `./scripts/doctor.sh` (or add it manually, step 2) |
| 502 Bad Gateway | CTFd still starting | wait ~30s, or `docker compose restart rtr-ctfd` |
| Stage 6 SSH rejects the recovered password | PCAP and container out of sync | `cd stage5-forensics && python3 generate.py`, then re-upload the PCAP to CTFd |
| `permission denied` on docker | user not in docker group | `sudo usermod -aG docker $USER` then re-login |
 
---
 
## Ethical scope
 
All targets are synthetic and were created by the group for this CTF. Scenario
personal data is fabricated; Nexora Bank does not correspond to any real
institution. No third-party production system, real personal data, or
unauthorised target is involved. The box binds only to localhost and is not
reachable from any institutional network segment.
 
---
 
## Acknowledgements
 
CTFd · Docker · Nginx · MariaDB · Redis · Flask · Pillow · PyCryptodome ·
Scapy · Wireshark · CyberChef · OWASP Top 10 · MITRE ATT&CK
 
