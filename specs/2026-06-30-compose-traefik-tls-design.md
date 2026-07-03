# COMPOSE-1 — Traefik v3 + Cloudflare DNS-01 + Wildcard HTTPS — Design Spec

!!! note "Historical design spec"
    This records the original design. The internal application domain was finalized
    as **`*.int.v2e.sh`** (the bare `v2e.sh` is the public zone, used only for the
    ACME DNS-01 challenge and email). App hostnames here have been corrected to
    `int.v2e.sh`; the current, authoritative reference is the
    [Application estate](../system/applications.md) page.

**Date:** 2026-06-30
**Phase:** COMPOSE-1 — see `MASTER-PLAN.md` §5
**Branch:** `feat/compose-traefik-tls`
**Repo:** `v2e-compose` · **Consumed by:** ANS-3 (`docker_compose_v2` deploy) · **Followed by:** COMPOSE-2 (TinyAuth)
**Status:** design approved — ready for implementation planning (writing-plans).

---

## 1. Goal

Stand up the TLS-terminating front door for the whole `v2e-compose` application
layer: **Traefik v3** issuing a **wildcard `*.v2e.sh`** certificate from Let's Encrypt
over the **Cloudflare DNS-01** challenge (no inbound ports), redirecting HTTP→HTTPS, and
a **`whoami`** service proving the path end-to-end. **Let's Encrypt staging CA first**,
with a one-variable flip to production.

This is the COMPOSE-side realisation of two locked master-plan decisions: *"HTTPS
everywhere"* (Traefik + LE + Cloudflare DNS-01 → wildcard certs, zero inbound ports) and
*D-1* (one SOPS-encrypted file read by Terraform, Ansible, and Compose).

---

## 2. Scope

**In scope (this phase):**

- Traefik v3 with `web(:80)`→`websecure(:443)` permanent redirect.
- Two ACME resolvers — `staging` and `production` — both Cloudflare DNS-01, each with its
  own storage file, selected per-router by one variable (`CERT_RESOLVER`).
- Wildcard `v2e.sh` + `*.v2e.sh` requested via the whoami router's `tls.domains`.
- Scoped `CF_DNS_API_TOKEN` (Zone:Read + DNS:Edit), sourced from SOPS.
- `no-new-privileges` on every service; read-only `docker.sock`; exact-tag image pins.
- Explicit shared **external** `frontend` network.
- A `whoami` test service as its own stack.
- Per-stack folder layout; a single shared root `.env` (config) + `secrets.sops.yaml`
  (secrets); a `Makefile` for local bootstrap/operations.

**Deferred to a named "COMPOSE-1.x hardening" pass (documented, not done here):**

- `cap_drop: ALL` (+ `NET_BIND_SERVICE` for Traefik), read-only root filesystem + `tmpfs`.
- `tecnativa/docker-socket-proxy` (removes the `:ro`-socket caveat — see §11).
- Image **digest** pinning (on top of the exact tags pinned now).
- HSTS (intentionally off until a production cert is active — see §10, §11).
- File-based TLS options (min version / cipher suites) — needs a dynamic config file.

**Owned by other phases (explicitly *not* COMPOSE-1):**

- The Traefik **dashboard** and forward-auth — **COMPOSE-2** (TinyAuth). COMPOSE-2 also
  re-anchors the wildcard onto the authenticated dashboard router so cert lifecycle no
  longer depends on the `whoami` test service.
- The `backend` (internal, no-egress) data-tier network — realised in **COMPOSE-3** where
  Semaphore's Postgres needs it. Declaring it here, unused, would be dead config.
- Automated host provisioning of secrets/config — **TF-2** (ships the SOPS file + age key)
  and **ANS-3** (decrypts, deploys). See §8 and §14.

---

## 3. Locked decisions (this phase)

1. **Replace the experimental stack.** The current root `compose.yml` (an experimental
   Dockge + Homepage stack, raw exposed ports, rw `docker.sock`) is replaced by the
   per-stack layout below. It remains recoverable in git history; Dockge returns properly
   behind Traefik + TinyAuth in COMPOSE-3, Homepage is not in the master plan.

2. **Per-stack folders.** Each service is its own Compose project in its own folder
   (`traefik/`, `whoami/`, future `semaphore/`, `dockge/`…). This is the Dockge-managed
   model (COMPOSE-3) and keeps stacks independently deployable.

3. **Shared `frontend` is an external network.** Separate Compose projects do not share a
   network automatically, so `frontend` is created once (`docker network create frontend`,
   automated by Ansible in ANS-3) and every stack joins it as `external: true`. This is
   what lets Traefik (in `traefik/`) discover and route to services in other folders.

4. **Static config as CLI command flags** in each `compose.yml` (the current Christian
   Lempa "Boilerplates v2" style), not a `traefik.yml` static file. Rationale: command
   flags interpolate `${VAR}` natively, so config is driven from the one environment
   channel without a file-plus-flag-override split. One file per stack to read.

5. **Two resolvers, not one.** `staging` and `production` each have their own ACME storage
   file; the active one is chosen by `${CERT_RESOLVER}` on a router label. A single
   resolver with a flipped `caServer` would force deleting `acme.json` on every
   staging→prod flip (Traefik will not re-request when it sees a cached cert) and lose
   instant rollback. The extra five flags buy a no-wipe, instant, reversible flip.

6. **`frontend` / `backend` network naming** (clearer two-tier names than the plan's
   "proxy/internal"). `frontend` = edge tier (Traefik ↔ routed services); `backend` =
   data tier (`internal: true`, no egress), realised in COMPOSE-3.

7. **Config vs secret split.** Non-secret config (`DOMAIN`, `ACME_EMAIL`, `CERT_RESOLVER`)
   lives in a plaintext, diff-readable `.env`; the one secret (`CF_DNS_API_TOKEN`) lives in
   `secrets.sops.yaml`. They have opposite change-frequency and readability needs — the
   token is long-lived and must be encrypted; `CERT_RESOLVER` is flipped routinely and
   wants to be legible in a git diff. Compose interpolates `${VAR}` from the process
   environment regardless of source, so the channel is pluggable (see §8).

8. **No dashboard in COMPOSE-1** (user decision). `--api.dashboard` stays off (default);
   nothing is published on `:8080`. The wildcard is anchored on the whoami router until
   COMPOSE-2 introduces the authenticated dashboard.

9. **Exact-tag image pins:** `traefik:v3.7.6`, `traefik/whoami:v1.11.0` (current stable as
   of 2026-06-30; the 3.7.x line carries recent CVE fixes). Renovate tracks bumps (Phase F).

10. **Cert storage at `traefik/data/certs/`** — `acme-staging.json` and `acme.json` (and
    any future cert material) live here, created by Traefik at runtime, mode `0600`,
    gitignored. Bind-mounted as a directory (not a file) to avoid Docker's
    create-a-directory-when-the-file-is-missing footgun.

---

## 4. Repository layout

```
v2e-compose/
├── Makefile                          # local bootstrap + up/down/logs + staging↔prod flip
├── .gitignore                        # repo-wide
├── README.md                         # layout · secrets/config · network bootstrap · deploy · gotchas
├── .env.example                      # shared NON-secret config: DOMAIN, ACME_EMAIL, CERT_RESOLVER
├── secrets.sops.yaml.example         # shared secrets template: CF_DNS_API_TOKEN
├── .sops.yaml                        # SOPS creation rule: path_regex + age recipient (public key)
├── traefik/
│   ├── compose.yml                   # Traefik (command flags); joins external `frontend`
│   └── data/
│       └── certs/
│           └── .gitkeep              # acme-staging.json / acme.json at runtime (0600, gitignored)
└── whoami/
    └── compose.yml                   # whoami test as its own stack; joins `frontend`; anchors wildcard
```

Generated/runtime-only (never committed): `.env`, `secrets.sops.yaml`,
`traefik/data/certs/*`.

`.gitignore`:

```gitignore
.env
secrets.sops.yaml
traefik/data/certs/*
!traefik/data/certs/.gitkeep
.DS_Store
```

(The `certs/` rule ignores *all* cert material — future `.pem`/`.crt`/`.key` too — keeping
only the directory via `.gitkeep`. Same burn-everything-but-the-dir trick `v2e-tf` uses
for tfstate.)

---

## 5. Architecture

**Networks**

- `frontend` — external bridge, shared, created once. Traefik + whoami (and every future
  routed service) join it. Traefik's Docker provider is pinned to it
  (`--providers.docker.network=frontend`).
- `backend` — **not defined in COMPOSE-1.** The two-tier model is documented in the README;
  `backend` (`internal: true`, no egress) is realised in COMPOSE-3 for Semaphore↔Postgres.

**Request flow**

```
client ──:443──► Traefik (websecure) ──Host(whoami.int.v2e.sh)──► whoami:80   (frontend)
client ──:80───► Traefik (web) ──301 permanent──► websecure
```

**Certificate flow (DNS-01, wildcard)**

```
Traefik `cloudflare` resolver ─► lego writes TXT _acme-challenge.v2e.sh
   (scoped CF_DNS_API_TOKEN: Zone:Read + DNS:Edit)         │
   LE validates the TXT ◄───────────────────────────────────┘
   wildcard  v2e.sh + *.v2e.sh  ─► stored in /certs/acme(-staging).json ─► served for every matching SNI
```

The wildcard is requested via the whoami router's `tls.domains` (`main=v2e.sh`,
`sans=*.v2e.sh`). Once in the store it is served for any `*.v2e.sh` SNI regardless of which
router matched.

---

## 6. The stacks

### 6.1 `traefik/compose.yml` (representative — final values at implementation)

```yaml
services:
  traefik:
    image: traefik:v3.7.6
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - frontend
    ports:
      - "80:80"
      - "443:443"
    environment:
      # The one secret. Sourced from the process env: `sops exec-env` (automation) or a
      # local export. NOT placed in .env. lego reads it for the DNS-01 challenge.
      - CF_DNS_API_TOKEN=${CF_DNS_API_TOKEN:?set via `make up` (sops exec-env) or a local export}
    command:
      - --global.checkNewVersion=false
      - --global.sendAnonymousUsage=false
      - --log.level=INFO
      - --ping=true
      - --providers.docker=true
      - --providers.docker.exposedByDefault=false
      - --providers.docker.network=frontend
      - --entryPoints.web.address=:80
      - --entryPoints.web.http.redirections.entryPoint.to=websecure
      - --entryPoints.web.http.redirections.entryPoint.scheme=https
      - --entryPoints.websecure.address=:443
      # --- staging resolver ---
      - --certificatesResolvers.staging.acme.email=${ACME_EMAIL}
      - --certificatesResolvers.staging.acme.storage=/certs/acme-staging.json
      - --certificatesResolvers.staging.acme.caServer=https://acme-staging-v02.api.letsencrypt.org/directory
      - --certificatesResolvers.staging.acme.dnsChallenge.provider=cloudflare
      - --certificatesResolvers.staging.acme.dnsChallenge.resolvers=1.1.1.1:53,1.0.0.1:53
      # --- production resolver ---
      - --certificatesResolvers.production.acme.email=${ACME_EMAIL}
      - --certificatesResolvers.production.acme.storage=/certs/acme.json
      - --certificatesResolvers.production.acme.caServer=https://acme-v02.api.letsencrypt.org/directory
      - --certificatesResolvers.production.acme.dnsChallenge.provider=cloudflare
      - --certificatesResolvers.production.acme.dnsChallenge.resolvers=1.1.1.1:53,1.0.0.1:53
    volumes:
      - ./data/certs:/certs
      - /var/run/docker.sock:/var/run/docker.sock:ro
    labels:
      - traefik.enable=true
      # Reusable security-headers middleware. HSTS deliberately omitted until prod cert.
      - traefik.http.middlewares.secure-headers.headers.frameDeny=true
      - traefik.http.middlewares.secure-headers.headers.contentTypeNosniff=true
      - traefik.http.middlewares.secure-headers.headers.browserXssFilter=true
      - traefik.http.middlewares.secure-headers.headers.referrerPolicy=strict-origin-when-cross-origin
    healthcheck:
      test: ["CMD", "traefik", "healthcheck", "--ping"]
      interval: 10s
      timeout: 5s
      retries: 3

networks:
  frontend:
    external: true
```

### 6.2 `whoami/compose.yml`

```yaml
services:
  whoami:
    image: traefik/whoami:v1.11.0
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - frontend
    labels:
      - traefik.enable=true
      - traefik.http.routers.whoami.rule=Host(`whoami.${DOMAIN}`)
      - traefik.http.routers.whoami.entryPoints=websecure
      - traefik.http.routers.whoami.tls.certResolver=${CERT_RESOLVER:-staging}
      - traefik.http.routers.whoami.tls.domains[0].main=${DOMAIN}
      - traefik.http.routers.whoami.tls.domains[0].sans=*.${DOMAIN}
      - traefik.http.routers.whoami.middlewares=secure-headers@docker
      - traefik.http.services.whoami.loadbalancer.server.port=80

networks:
  frontend:
    external: true
```

The wildcard is anchored here (`tls.domains`). Traefik's Docker provider reads these labels
across stacks, so the cross-stack arrangement (Traefik in one folder, whoami in another)
proves end-to-end discovery + routing + TLS — a stronger acceptance test than
whoami-inside-the-traefik-stack.

---

## 7. Configuration & secret contract

| Variable | Kind | Home | Used by |
|---|---|---|---|
| `DOMAIN` | config | `.env` / group_vars | whoami router rule + wildcard `tls.domains` |
| `ACME_EMAIL` | config | `.env` / group_vars | both resolvers' ACME registration |
| `CERT_RESOLVER` | config | `.env` / group_vars | whoami router — selects `staging`/`production` |
| `CF_DNS_API_TOKEN` | **secret** | `secrets.sops.yaml` / group_vars | Traefik container env → lego DNS-01 |

The compose files declare these as `${VAR}`. That is the contract; the *source* is
pluggable (§8). Only `CF_DNS_API_TOKEN` must reach the **container** environment (via
`environment:`); the rest are interpolated into flags/labels at launch.

---

## 8. How values reach the environment (the two channels)

Compose interpolates `${VAR}` from the process environment. It does not matter how that
environment was populated, which gives two channels onto one contract:

```
   compose.yml declares:  ${DOMAIN} ${ACME_EMAIL} ${CERT_RESOLVER} ${CF_DNS_API_TOKEN}
                                          ▲ injected at launch
        ┌─────────────────────────────────┴──────────────────────────────┐
   LOCAL / standalone (now)                              AUTOMATED lab (ANS-3)
   .env (config) + sops exec-env (token)        values set locally up-front in SOPS/group_vars,
   wrapped by `make up`                          BEFORE `terraform apply`; TF ships the file;
                                                 Ansible decrypts; `sops exec-env` injects them;
                                                 no `.env` is ever copied to the host
```

- **Local / standalone** (this repo, for COMPOSE-1's own acceptance test): a plaintext
  `.env` provides config; `sops exec-env secrets.sops.yaml '…'` adds the token; `make`
  wraps the incantation.
- **Automated lab**: the operator fills the values *once, locally, before automation
  starts*, in the SOPS-managed Ansible vars (the same D-1 file that already carries the
  token). Terraform ships it (existing channel, TF-2); Ansible decrypts (`community.sops`);
  ANS-3 runs the deploy under `sops exec-env`, so the values land directly in the
  container environment. **No `.env` is transported to the host** — it is purely a local
  convenience. (If a literal on-host `.env` is ever wanted for inspectability, ANS-3 can
  template one from the same vars; optional.)

This is the resolution of "send the variables before the automation starts": they live in
the one local input you author up-front; the pipeline carries them to the container.

`.env.example`:

```bash
# Non-secret config for the v2e-compose stacks. Copy to .env and edit (or let Ansible
# render it from group_vars in the automated flow). The secret (CF_DNS_API_TOKEN) lives
# in secrets.sops.yaml, NOT here.
DOMAIN=v2e.sh
ACME_EMAIL=you@example.com
CERT_RESOLVER=staging          # flip to `production` once staging issues cleanly
```

`secrets.sops.yaml.example` (encrypt to produce `secrets.sops.yaml`):

```yaml
# Scoped Cloudflare API token — Zone:Read + DNS:Edit (NOT the global key).
# Encrypt: sops --encrypt secrets.sops.yaml.example > secrets.sops.yaml
CF_DNS_API_TOKEN: "<your-scoped-cloudflare-token>"
```

---

## 9. Bootstrap & operations (`Makefile`)

The Makefile is the **local** front door only (it never invokes Terraform/Ansible; the
automated path is `terraform apply` → Ansible → ANS-3). Targets:

- `make bootstrap` — idempotent first-run setup:
  1. `docker network create frontend` (ignore "already exists")
  2. `cp -n .env.example .env` (no-clobber) and prompt to edit
  3. `mkdir -p traefik/data/certs`
  4. seed `secrets.sops.yaml`: `sops --encrypt secrets.sops.yaml.example > secrets.sops.yaml`
     then open `sops secrets.sops.yaml` to insert the real token. Pre-checks that an age
     key is available (`SOPS_AGE_KEY_FILE` or `~/.config/sops/age/keys.txt`); if not, tells
     the operator to run `age-keygen` first. **Bootstrap uses an existing age key from the
     global SOPS setup (D-1); it does not mint one.**
- `make up` — deploy both stacks:
  ```bash
  sops exec-env secrets.sops.yaml 'docker compose --env-file .env -f traefik/compose.yml up -d'
  docker compose --env-file .env -f whoami/compose.yml up -d   # no secret needed
  ```
- `make prod` — same as `up` with `CERT_RESOLVER=production` exported (shell env overrides
  `.env` in Compose), flipping the active resolver without editing files.
- `make down` — stop both stacks.
- `make logs` — `docker compose -f traefik/compose.yml logs -f traefik` (watch ACME).

`--env-file .env` is passed explicitly because the shared `.env` lives at the repo root
while each `compose.yml` is in a sub-folder (Compose auto-discovers `.env` only relative to
the stack dir). Explicit beats guessing.

---

## 10. Staging → production flip

1. Bring up with `CERT_RESOLVER=staging` (`.env` default). Confirm a **staging** cert
   issues and the redirect works (§13).
2. Flip: `make prod` (or set `CERT_RESOLVER=production` in `.env` and `make up`).
3. The `production` resolver uses its own `acme.json`, so the staging cert is untouched and
   rollback is instant (set it back to `staging`). No file deletion, ever.

Staging is mandatory first: production LE limits duplicate certs to 5/week, and the
two-resolver split means staging mistakes never burn production quota.

---

## 11. Hardening — now vs the COMPOSE-1.x follow-up

**Applied now** (required + near-zero risk to the cert bring-up — the one thing this phase
must prove): `no-new-privileges` on both services; read-only `docker.sock`; explicit
external `frontend`; `--providers.docker.exposedByDefault=false` (only labelled containers
route); exact image tags; non-HSTS security headers.

**Deferred to a named "COMPOSE-1.x hardening" pass** (all additive; none restructure the
stack):

- `cap_drop: ALL` (+ `NET_BIND_SERVICE` for Traefik), `read_only: true` root FS + `tmpfs`.
- `tecnativa/docker-socket-proxy` on a dedicated network exposing only the read endpoints
  Traefik needs; Traefik's Docker endpoint is repointed to `tcp://socket-proxy:2375`. This
  is the *real* socket hardening — note the caveat below.
- Image **digest** pins.
- **HSTS** on the `secure-headers` middleware — turned on only after the production cert is
  active.

**`:ro` docker.sock caveat (documented honestly):** `:ro` on the socket bind is largely
cosmetic — it makes the socket *file* read-only but does not stop a client issuing
write/`POST`/`DELETE` calls over the Docker API through it. Real protection is the
socket-proxy above. The master plan explicitly schedules socket-proxy as a follow-up, so
COMPOSE-1 ships `:ro` now and tees it up.

**Why HSTS is off now:** HSTS + a *staging* (untrusted) cert pins the domain in the browser
and turns the staging cert's warning into an **un-bypassable** error — which would block the
very staging test this phase exists for. Non-HSTS headers are staging-safe; HSTS flips on
with production.

---

## 12. Operational gotchas (→ README)

- **Gray-cloud internal records.** Any public A/AAAA record for an internal service must be
  **DNS-only (gray cloud)** — orange-cloud serves Cloudflare's edge cert, not the LE cert,
  and cannot reach a private IP. The `lab.v2e.sh` tunnel record stays proxied (a different
  record). DNS-01 itself needs no public record for the cert.
- **Local testing without a public record.** Point `whoami.int.v2e.sh` at the Docker host via
  `/etc/hosts` to exercise the cert path; the wildcard still validates over DNS-01.
- **DNS-01 resolvers pinned** to `1.1.1.1:53,1.0.0.1:53` so the ACME propagation pre-check
  doesn't consult a split-horizon/internal resolver that returns the wrong TXT.
- **`acme.json` permissions.** Traefik creates the storage files `0600`; the directory is
  bind-mounted and gitignored. Back it up — it holds the cert private keys.
- **Wildcard anchor is on whoami in COMPOSE-1.** If whoami is removed before COMPOSE-2
  re-anchors the wildcard onto the dashboard router, the wildcard stops renewing. COMPOSE-2
  moves the anchor to permanent infra.
- **`frontend` must exist first.** `docker network create frontend` (or Ansible) before any
  stack comes up, or Compose errors on the missing external network.

---

## 13. Testing & acceptance

This phase is configuration, not application code — there is no unit-test layer to
TDD. Verification is static validation plus a documented live smoke test.

**Automatable locally now (no real token/DNS needed):**

- `docker compose -f traefik/compose.yml config` and `… -f whoami/compose.yml config` parse
  and resolve (with a dummy env) — schema valid, flags well-formed.
- `.gitignore` excludes `.env`, `secrets.sops.yaml`, and `traefik/data/certs/*`.

**Live smoke test (needs the real scoped CF token + `v2e.sh` on Cloudflare — run on a Docker
host, e.g. via `make`):**

1. `make bootstrap`; fill `.env` + token; `make up` (staging).
2. `make logs` shows the `staging` resolver complete a DNS-01 order with no errors.
3. `curl -vk https://whoami.int.v2e.sh` (host resolves via `/etc/hosts` if needed) → whoami
   body; certificate issuer is **(STAGING) Let's Encrypt**.
4. `curl -I http://whoami.int.v2e.sh` → `301` to `https://`.
5. `make prod`; re-check `curl -v https://whoami.int.v2e.sh` (no `-k`) → succeeds with a real
   Let's Encrypt issuer.

**Acceptance (from MASTER-PLAN §5):** Traefik pulls a valid wildcard LE cert
(staging→prod); `whoami` reachable over HTTPS with the redirect working.

---

## 14. Handoff to ANS-3

COMPOSE-1 defines the contract ANS-3 fulfils on the real host:

- **Var contract:** `DOMAIN`, `ACME_EMAIL`, `CERT_RESOLVER`, `CF_DNS_API_TOKEN` (§7),
  mirrored as keys in the SOPS-managed `group_vars`.
- **Network:** ANS-3 creates `frontend` (`community.docker.docker_network`) before deploy.
- **Deploy:** `community.docker.docker_compose_v2` per stack, with the environment supplied
  from the sops-decrypted vars (equivalent to `sops exec-env`). No host `.env` required;
  optionally template one (`.env.j2` → `0600`) for inspectability.
- **Secrets:** the same single SOPS file (D-1) the operator authored locally before
  `terraform apply`.

---

## 15. Sources

- Traefik latest stable `v3.7.6`, whoami `v1.11.0` — verified 2026-06-30 via the GitHub
  releases API.
- Christian Lempa Boilerplates v2 Traefik template (single-file command-flag style, env/
  secret handling) — `github.com/ChristianLempa/boilerplates` → `library/compose/traefik`.
- MASTER-PLAN.md §5 (COMPOSE-1), §3 (secrets flow), Appendix B (Cloudflare DNS-01 / TLS).
