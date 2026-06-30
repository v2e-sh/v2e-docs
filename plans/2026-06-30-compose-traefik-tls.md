# COMPOSE-1 — Traefik + Cloudflare DNS-01 + Wildcard HTTPS — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up Traefik v3 in `v2e-compose` issuing a wildcard `*.v2e.sh` Let's Encrypt cert over the Cloudflare DNS-01 challenge, redirecting HTTP→HTTPS, with a `whoami` test service — staging CA first, one-variable flip to production.

**Architecture:** Per-stack folders (each service its own Compose project) sharing one external `frontend` network. Traefik's static config is CLI command flags; two ACME resolvers (`staging`/`production`) each with their own storage file; the active one is chosen per-router by `${CERT_RESOLVER}`. Config (`DOMAIN`/`ACME_EMAIL`/`CERT_RESOLVER`) lives in a plaintext root `.env`; the one secret (`CF_DNS_API_TOKEN`) lives in a root `secrets.sops.yaml`. A `Makefile` is the local front door; Ansible (ANS-3) is the automated one.

**Tech Stack:** Docker Compose v2, Traefik v3.7.6, `traefik/whoami:v1.11.0`, Let's Encrypt (ACME DNS-01), Cloudflare DNS, SOPS + age, GNU Make.

**Source spec:** `v2e-docs/specs/2026-06-30-compose-traefik-tls-design.md`

## Global Constraints

- **Repo / branch:** all work in `v2e-compose`, branch `feat/compose-traefik-tls`.
- **Image pins (exact tags):** `traefik:v3.7.6`, `traefik/whoami:v1.11.0`.
- **Domain:** `v2e.sh`. Wildcard cert is `v2e.sh` + `*.v2e.sh`.
- **CF token scopes:** Zone:Read + DNS:Edit (scoped, not the global key). Sourced from SOPS.
- **Staging first:** `CERT_RESOLVER=staging` is the default; flip to `production` only after staging issues cleanly.
- **Variable contract (declared as `${VAR}` in compose; source is pluggable):** `DOMAIN`, `ACME_EMAIL`, `CERT_RESOLVER` (config, `.env`); `CF_DNS_API_TOKEN` (secret, SOPS).
- **Security baseline applied now:** `no-new-privileges:true` on every service; `/var/run/docker.sock` mounted `:ro`; `--providers.docker.exposedByDefault=false`.
- **Deferred (do NOT add now — documented backlog only):** `cap_drop`/read-only rootfs, `docker-socket-proxy`, image digest pins, HSTS, file-based TLS options, the Traefik dashboard, the `backend` network.
- **Git:** commit per task; short messages; **no Claude attribution trailers**.
- **No host `.env` in automation:** the host gets values via `sops exec-env`; `.env` is a local convenience only.

---

## File Structure

| File | Responsibility | Task |
|---|---|---|
| `.gitignore` | Keep `.env`, `secrets.sops.yaml`, and all cert material out of git | 1 |
| `traefik/data/certs/.gitkeep` | Hold the (gitignored) ACME storage dir on a fresh clone | 1 |
| (remove) `compose.yml` | Delete the experimental Dockge+Homepage root stack | 1 |
| `.env.example` | Non-secret config contract + local seed | 2 |
| `secrets.sops.yaml.example` | Secret contract (CF token), pre-encryption template | 2 |
| `.sops.yaml` | SOPS creation rule (path + age recipient) | 2 |
| `traefik/compose.yml` | Traefik static config (flags), security, ACME resolvers, redirect, middleware | 3 |
| `whoami/compose.yml` | whoami router + wildcard anchor + security | 4 |
| `Makefile` | Local bootstrap, deploy, staging↔prod flip, config validation | 5 |
| `README.md` | Runbook: layout, secrets/config, network bootstrap, deploy, gotchas, handoff | 6 |

Repository layout after implementation:

```
v2e-compose/
├── Makefile
├── .gitignore
├── .sops.yaml
├── .env.example
├── secrets.sops.yaml.example
├── README.md
├── traefik/
│   ├── compose.yml
│   └── data/certs/.gitkeep
└── whoami/
    └── compose.yml
```

---

## Task 1: Branch, scaffolding, `.gitignore`, remove experimental stack

**Files:**
- Create branch: `feat/compose-traefik-tls`
- Create: `.gitignore`
- Create: `traefik/data/certs/.gitkeep`
- Delete: `compose.yml` (experimental Dockge+Homepage stack)

**Interfaces:**
- Consumes: nothing.
- Produces: the directory skeleton (`traefik/data/certs/`, `whoami/` implied) and the ignore rules every later task relies on; the clean slate (old `compose.yml` gone).

- [ ] **Step 1: Create the branch**

```bash
cd v2e-compose
git checkout -b feat/compose-traefik-tls
```

- [ ] **Step 2: Remove the experimental stack and create the skeleton**

```bash
git rm compose.yml
mkdir -p traefik/data/certs whoami
touch traefik/data/certs/.gitkeep
```

- [ ] **Step 3: Write `.gitignore`**

```gitignore
# Local non-secret config (commit .env.example instead)
.env

# The real encrypted secrets file (commit secrets.sops.yaml.example instead).
# SOPS files are safe to commit, but we follow the v2e "supply your own locally" model.
secrets.sops.yaml

# ACME storage + any cert material — contains cert private keys. Keep only the dir.
traefik/data/certs/*
!traefik/data/certs/.gitkeep

.DS_Store
```

- [ ] **Step 4: Verify the ignore rules actually ignore secrets (this is the test)**

```bash
touch .env traefik/data/certs/acme.json
git check-ignore -v .env traefik/data/certs/acme.json traefik/data/certs/.gitkeep; echo "exit=$?"
```
Expected: `.env` and `traefik/data/certs/acme.json` print as ignored; `.gitkeep` prints **nothing** (it is NOT ignored). `git check-ignore` exits `0` because at least one path matched. Confirm `.gitkeep` is absent from the output.

```bash
rm .env traefik/data/certs/acme.json
git status --short
```
Expected: shows `D compose.yml`, `A` (or `??`) for `.gitignore` and `traefik/data/certs/.gitkeep` — and **no** `.env`/`acme.json`.

- [ ] **Step 5: Commit**

```bash
git add .gitignore traefik/data/certs/.gitkeep
git commit -m "compose: scaffold COMPOSE-1 structure, drop experimental stack"
```

---

## Task 2: Config & secret templates

**Files:**
- Create: `.env.example`
- Create: `secrets.sops.yaml.example`
- Create: `.sops.yaml`

**Interfaces:**
- Consumes: the skeleton from Task 1.
- Produces: the variable contract — `DOMAIN`, `ACME_EMAIL`, `CERT_RESOLVER` (config), `CF_DNS_API_TOKEN` (secret) — consumed by Tasks 3, 4, 5 and mirrored by ANS-3.

- [ ] **Step 1: Write `.env.example`**

```bash
# Non-secret config for the v2e-compose stacks. Copy to .env and edit (or let Ansible
# render it from group_vars in the automated flow). The secret (CF_DNS_API_TOKEN) lives
# in secrets.sops.yaml, NOT here.
DOMAIN=v2e.sh
ACME_EMAIL=you@example.com
CERT_RESOLVER=staging          # flip to `production` once staging issues cleanly
```

- [ ] **Step 2: Write `secrets.sops.yaml.example`**

```yaml
# Scoped Cloudflare API token — Zone:Read + DNS:Edit (NOT the global key).
# Encrypt to produce the real file:
#   sops --encrypt secrets.sops.yaml.example > secrets.sops.yaml
# then `sops secrets.sops.yaml` to insert the real token.
CF_DNS_API_TOKEN: "<your-scoped-cloudflare-token>"
```

- [ ] **Step 3: Write `.sops.yaml`**

```yaml
# SOPS creation rule for v2e-compose secrets.
# Replace the age recipient with YOUR age PUBLIC key (starts with `age1`).
# Public keys are safe to commit. Get yours with:
#   age-keygen -y ~/.config/sops/age/keys.txt
creation_rules:
  - path_regex: secrets\.sops\.ya?ml$
    age: "age1replacewithyourpublickeyxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

- [ ] **Step 4: Verify the contract is complete (the test)**

```bash
grep -Eq '^DOMAIN=' .env.example \
  && grep -Eq '^ACME_EMAIL=' .env.example \
  && grep -Eq '^CERT_RESOLVER=' .env.example \
  && grep -Eq '^CF_DNS_API_TOKEN:' secrets.sops.yaml.example \
  && grep -Eq 'path_regex:.*secrets' .sops.yaml \
  && echo "contract OK"
```
Expected: `contract OK`.

- [ ] **Step 5: Commit**

```bash
git add .env.example secrets.sops.yaml.example .sops.yaml
git commit -m "compose: env/secret templates + sops creation rule"
```

---

## Task 3: Traefik stack

**Files:**
- Create: `traefik/compose.yml`

**Interfaces:**
- Consumes: `${DOMAIN}` (unused here), `${ACME_EMAIL}`, `${CF_DNS_API_TOKEN}` from the env; the external `frontend` network.
- Produces: the `traefik` service; the `web`→`websecure` redirect; the `staging` and `production` cert resolvers (referenced by name in Task 4); the `secure-headers@docker` middleware (referenced in Task 4); storage at `/certs/acme-staging.json` and `/certs/acme.json`.

- [ ] **Step 1: Verify the file does not yet exist / does not validate (the failing test)**

```bash
DOMAIN=example.com ACME_EMAIL=a@b.c CERT_RESOLVER=staging CF_DNS_API_TOKEN=dummy \
  docker compose -f traefik/compose.yml config >/dev/null; echo "exit=$?"
```
Expected: non-zero exit — `no configuration file provided: not found` (the file does not exist yet).

- [ ] **Step 2: Write `traefik/compose.yml`**

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
      # The one secret. Sourced from the process env: `sops exec-env` (automation/`make`)
      # or a local export. NOT placed in .env. lego reads it for the DNS-01 challenge.
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
      # Reusable security-headers middleware. HSTS deliberately omitted until prod cert
      # (HSTS + a staging cert makes the untrusted-cert error un-bypassable).
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

- [ ] **Step 3: Validate the config parses and interpolates (test passes)**

```bash
DOMAIN=example.com ACME_EMAIL=a@b.c CERT_RESOLVER=staging CF_DNS_API_TOKEN=dummy \
  docker compose -f traefik/compose.yml config >/dev/null && echo "traefik config OK"
```
Expected: `traefik config OK` (exit 0). Note: the `external: true` network and `${CF_DNS_API_TOKEN:?...}` guard are why a dummy token must be present for `config` to succeed.

- [ ] **Step 4: Assert the security-critical settings are present (test)**

```bash
DOMAIN=example.com ACME_EMAIL=a@b.c CERT_RESOLVER=staging CF_DNS_API_TOKEN=dummy \
  docker compose -f traefik/compose.yml config > /tmp/traefik.rendered.yml
grep -q 'no-new-privileges:true'                  /tmp/traefik.rendered.yml \
  && grep -q '/var/run/docker.sock:/var/run/docker.sock:ro' /tmp/traefik.rendered.yml \
  && grep -q 'exposedByDefault=false'             /tmp/traefik.rendered.yml \
  && grep -q 'redirections.entryPoint.scheme=https' /tmp/traefik.rendered.yml \
  && grep -q 'staging.acme.caServer=https://acme-staging' /tmp/traefik.rendered.yml \
  && grep -q 'production.acme.caServer=https://acme-v02'  /tmp/traefik.rendered.yml \
  && grep -q 'traefik:v3.7.6'                      /tmp/traefik.rendered.yml \
  && echo "traefik hardening+acme assertions OK"
rm -f /tmp/traefik.rendered.yml
```
Expected: `traefik hardening+acme assertions OK`.

- [ ] **Step 5: Commit**

```bash
git add traefik/compose.yml
git commit -m "compose: traefik stack (v3 + cloudflare dns-01 + http->https)"
```

---

## Task 4: whoami test stack

**Files:**
- Create: `whoami/compose.yml`

**Interfaces:**
- Consumes: `${DOMAIN}`, `${CERT_RESOLVER}` from the env; the external `frontend` network; the `staging`/`production` resolvers and `secure-headers@docker` middleware from Task 3 (resolved by Traefik's Docker provider across stacks).
- Produces: the `whoami` router that anchors the wildcard `v2e.sh` + `*.v2e.sh`.

- [ ] **Step 1: Verify the file does not yet validate (failing test)**

```bash
DOMAIN=example.com CERT_RESOLVER=staging \
  docker compose -f whoami/compose.yml config >/dev/null; echo "exit=$?"
```
Expected: non-zero — `no configuration file provided: not found`.

- [ ] **Step 2: Write `whoami/compose.yml`**

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

- [ ] **Step 3: Validate config parses + interpolates (test passes)**

```bash
DOMAIN=v2e.sh CERT_RESOLVER=staging \
  docker compose -f whoami/compose.yml config >/dev/null && echo "whoami config OK"
```
Expected: `whoami config OK`.

- [ ] **Step 4: Assert the router, wildcard anchor, and security are present (test)**

```bash
DOMAIN=v2e.sh CERT_RESOLVER=staging \
  docker compose -f whoami/compose.yml config > /tmp/whoami.rendered.yml
grep -q 'Host(`whoami.v2e.sh`)'        /tmp/whoami.rendered.yml \
  && grep -q 'tls.domains\[0\].main=v2e.sh' /tmp/whoami.rendered.yml \
  && grep -q 'tls.domains\[0\].sans=\*.v2e.sh' /tmp/whoami.rendered.yml \
  && grep -q 'tls.certResolver=staging'  /tmp/whoami.rendered.yml \
  && grep -q 'no-new-privileges:true'    /tmp/whoami.rendered.yml \
  && grep -q 'traefik/whoami:v1.11.0'    /tmp/whoami.rendered.yml \
  && echo "whoami router+anchor assertions OK"
rm -f /tmp/whoami.rendered.yml
```
Expected: `whoami router+anchor assertions OK`.

- [ ] **Step 5: Verify the prod resolver selection flips via env override (test)**

```bash
DOMAIN=v2e.sh CERT_RESOLVER=production \
  docker compose -f whoami/compose.yml config | grep -q 'tls.certResolver=production' \
  && echo "prod flip OK"
```
Expected: `prod flip OK` (confirms `${CERT_RESOLVER}` drives the resolver).

- [ ] **Step 6: Commit**

```bash
git add whoami/compose.yml
git commit -m "compose: whoami test stack + wildcard anchor"
```

---

## Task 5: Makefile

**Files:**
- Create: `Makefile`

**Interfaces:**
- Consumes: both compose files (Tasks 3, 4); `.env.example`/`secrets.sops.yaml.example` (Task 2); the `${...}` contract.
- Produces: `bootstrap`, `up`, `prod`, `down`, `logs`, `validate` targets — the local operational surface.

- [ ] **Step 1: Write `Makefile` (use TAB indentation for recipe lines)**

```makefile
# v2e-compose — local bootstrap & operations for the Traefik + TLS stack.
# LOCAL/standalone front door only. The automated lab path is
# `terraform apply` -> Ansible -> ANS-3 (docker_compose_v2); make is not involved there.

SHELL    := /bin/bash
COMPOSE  := docker compose
ENVFILE  := .env
SECRETS  := secrets.sops.yaml
TRAEFIK  := -f traefik/compose.yml
WHOAMI   := -f whoami/compose.yml

.DEFAULT_GOAL := help
.PHONY: help bootstrap up prod down logs validate

help:
	@echo "targets: bootstrap | up | prod | down | logs | validate"

bootstrap:
	@docker network inspect frontend >/dev/null 2>&1 || docker network create frontend
	@[ -f $(ENVFILE) ] || { cp .env.example $(ENVFILE); echo "created $(ENVFILE) — edit DOMAIN / ACME_EMAIL"; }
	@mkdir -p traefik/data/certs
	@if [ ! -f $(SECRETS) ]; then \
		if [ -z "$$SOPS_AGE_KEY_FILE" ] && [ ! -f "$$HOME/.config/sops/age/keys.txt" ]; then \
			echo "No age key found. Run age-keygen first (see README)."; exit 1; \
		fi; \
		sops --encrypt secrets.sops.yaml.example > $(SECRETS); \
		echo "created $(SECRETS) — opening editor to set the real token"; \
		sops $(SECRETS); \
	fi
	@echo "bootstrap done. Next: edit .env, then 'make up'"

up:
	sops exec-env $(SECRETS) '$(COMPOSE) --env-file $(ENVFILE) $(TRAEFIK) up -d'
	$(COMPOSE) --env-file $(ENVFILE) $(WHOAMI) up -d

prod:
	sops exec-env $(SECRETS) 'CERT_RESOLVER=production $(COMPOSE) --env-file $(ENVFILE) $(TRAEFIK) up -d'
	CERT_RESOLVER=production $(COMPOSE) --env-file $(ENVFILE) $(WHOAMI) up -d

down:
	-$(COMPOSE) $(WHOAMI) down
	-sops exec-env $(SECRETS) '$(COMPOSE) $(TRAEFIK) down'

logs:
	$(COMPOSE) $(TRAEFIK) logs -f traefik

validate:
	@DOMAIN=example.com ACME_EMAIL=a@b.c CERT_RESOLVER=staging CF_DNS_API_TOKEN=dummy \
		$(COMPOSE) $(TRAEFIK) config >/dev/null && echo "traefik/compose.yml OK"
	@DOMAIN=example.com CERT_RESOLVER=staging \
		$(COMPOSE) $(WHOAMI) config >/dev/null && echo "whoami/compose.yml OK"
```

- [ ] **Step 2: Verify targets resolve and `validate` works (the test)**

```bash
make help
make validate
```
Expected: `help` lists the targets; `validate` prints `traefik/compose.yml OK` and `whoami/compose.yml OK`.

- [ ] **Step 3: Verify `up` expands to the right command without running it (test)**

```bash
make -n up
```
Expected: prints a line containing `sops exec-env secrets.sops.yaml '... --env-file .env -f traefik/compose.yml up -d'` and a `... -f whoami/compose.yml up -d` line. Nothing is executed.

- [ ] **Step 4: Commit**

```bash
git add Makefile
git commit -m "compose: makefile (bootstrap + deploy + staging/prod flip + validate)"
```

---

## Task 6: README / runbook

**Files:**
- Create: `README.md`

**Interfaces:**
- Consumes: everything from Tasks 1–5.
- Produces: the operator runbook and the ANS-3 handoff documentation.

- [ ] **Step 1: Write `README.md`**

````markdown
# v2e-compose

The application layer of the v2e lab: Docker Compose stacks fronted by **Traefik v3**,
which terminates TLS using **Let's Encrypt** wildcard certs obtained over the **Cloudflare
DNS-01** challenge (no inbound ports). COMPOSE-1 ships Traefik + a `whoami` test service.

## Layout

Each service is its own Compose project in its own folder, all sharing one external
`frontend` network:

```
traefik/compose.yml         Traefik (edge, TLS, routing)
traefik/data/certs/         ACME storage (acme-staging.json / acme.json), 0600, gitignored
whoami/compose.yml          test service proving the cert path
.env / .env.example         non-secret config (DOMAIN, ACME_EMAIL, CERT_RESOLVER)
secrets.sops.yaml(.example) the one secret (CF_DNS_API_TOKEN), SOPS-encrypted
Makefile                    local bootstrap + deploy
```

**Networks — two tiers.** `frontend` = edge (Traefik ↔ routed services). `backend`
(`internal: true`, no egress) is the data tier for stateful services and arrives in
COMPOSE-3 (Semaphore ↔ Postgres); COMPOSE-1 only needs `frontend`.

## Prerequisites

- Docker Engine + Compose v2.
- `sops` and `age` (baked into the v2e images; `brew install sops age` locally).
- A domain on Cloudflare (here `v2e.sh`) and a **scoped** API token: **Zone:Read +
  DNS:Edit**. Not the global key.
- An age keypair (created once for the whole v2e SOPS setup — D-1). Check:
  `echo $SOPS_AGE_KEY_FILE` or `ls ~/.config/sops/age/keys.txt`. If absent:
  `age-keygen -o ~/.config/sops/age/keys.txt`, then put your **public** key
  (`age-keygen -y ~/.config/sops/age/keys.txt`) into `.sops.yaml`.

## First run (local / standalone)

```bash
make bootstrap          # creates the frontend network, .env, certs dir, and secrets.sops.yaml
$EDITOR .env            # set DOMAIN + ACME_EMAIL (CERT_RESOLVER stays 'staging' for now)
make up                 # deploys traefik + whoami against the STAGING CA
make logs               # watch the ACME order complete
```

Test (point the host at your Docker host if there's no public record):

```bash
echo "127.0.0.1 whoami.v2e.sh" | sudo tee -a /etc/hosts   # local test only
curl -vk https://whoami.v2e.sh     # whoami body; issuer = (STAGING) Let's Encrypt
curl -I  http://whoami.v2e.sh      # 301 -> https
```

When staging looks good, flip to the real CA:

```bash
make prod               # CERT_RESOLVER=production; production resolver, separate acme.json
curl -v https://whoami.v2e.sh      # no -k; issuer = Let's Encrypt
```

Rollback is instant — `CERT_RESOLVER=staging` again (the two resolvers keep separate
storage, so neither flip wipes the other).

## How variables reach the container

The compose files declare `${DOMAIN}`, `${ACME_EMAIL}`, `${CERT_RESOLVER}`, and
`${CF_DNS_API_TOKEN}`. Compose interpolates these from the **process environment**, so the
source is pluggable:

- **Local:** `.env` provides the config; `sops exec-env secrets.sops.yaml '…'` (wrapped by
  `make up`) adds the token. Only `CF_DNS_API_TOKEN` is injected *into* the Traefik
  container (via `environment:`); the rest become flag/label text at launch.
- **Automated lab (ANS-3):** you set all values **once, locally, before `terraform
  apply`**, in the SOPS-managed group_vars (the same D-1 file that carries the token).
  Terraform ships it; Ansible decrypts it; ANS-3 deploys under `sops exec-env`. **No `.env`
  is copied to the host** — the values land directly in the container environment.

## Cloudflare / DNS-01 notes

- **Wildcard:** one `v2e.sh` + `*.v2e.sh` cert (anchored on the whoami router in COMPOSE-1;
  re-anchored onto the authenticated dashboard router in COMPOSE-2).
- **Gray-cloud:** any *public* A/AAAA record for an internal service must be **DNS-only
  (gray cloud)** — orange-cloud serves Cloudflare's edge cert, not LE, and can't reach a
  private IP. The `lab.v2e.sh` tunnel record stays proxied (different record).
- DNS-01 needs **no** inbound ports and no public record for the cert itself.
- The propagation pre-check is pinned to `1.1.1.1:53,1.0.0.1:53` to avoid split-horizon DNS.
- **Staging first** — production LE allows only 5 duplicate certs/week; the resolver split
  means staging mistakes never burn that quota.

## Security posture (COMPOSE-1) and the hardening backlog

Applied now: `no-new-privileges` on every service; read-only `docker.sock`;
`exposedByDefault=false`; exact image tags; non-HSTS security headers.

**Caveat — `:ro` on `docker.sock` is largely cosmetic:** it makes the socket *file*
read-only but does not stop Docker API writes over it. Real protection is the socket-proxy
below.

**COMPOSE-1.x hardening (deferred, all additive):**
- `tecnativa/docker-socket-proxy` (Traefik talks to a read-only proxy; the real socket
  never touches Traefik) — removes the caveat above.
- `cap_drop: ALL` (+ `NET_BIND_SERVICE` for Traefik), read-only root FS + `tmpfs`.
- Image digest pins (on top of the current tags).
- **HSTS** on the `secure-headers` middleware — only after a production cert is active
  (HSTS + a staging cert makes the cert warning un-bypassable).

## How the lab deploys this (ANS-3)

ANS-3 creates the `frontend` network, then runs `community.docker.docker_compose_v2` per
stack with the environment supplied from the sops-decrypted group_vars (no host `.env`
required). The variable contract above is mirrored as group_vars keys.
````

- [ ] **Step 2: Verify the runbook covers the required topics (the test)**

```bash
grep -q 'DNS-only (gray cloud)'  README.md \
  && grep -q 'Staging first'      README.md \
  && grep -q 'make prod'          README.md \
  && grep -q 'docker network'     README.md 2>/dev/null; \
grep -q 'socket-proxy'           README.md \
  && grep -q 'HSTS'               README.md \
  && grep -q 'sops exec-env'      README.md \
  && grep -q 'Zone:Read + DNS:Edit' README.md \
  && echo "README coverage OK"
```
Expected: `README coverage OK`.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "compose: COMPOSE-1 readme/runbook"
```

---

## Task 7: Live acceptance smoke test (run on a Docker host with the real token)

This task is the phase's acceptance gate. It is **not automatable in CI** — it needs the
real scoped Cloudflare token and `v2e.sh` on Cloudflare. Run it on a Docker host (your
machine or the lab services node). No code is written here; it is a verification procedure
whose output is the acceptance evidence.

**Files:** none (verification only).

**Interfaces:** consumes the whole stack (Tasks 1–6).

- [ ] **Step 1: Bootstrap and bring up staging**

```bash
make bootstrap          # fill .env (real DOMAIN/ACME_EMAIL) and the real token when prompted
make up
```

- [ ] **Step 2: Confirm the staging cert issued**

```bash
make logs               # look for a completed ACME order on the `staging` resolver, no errors
```
Expected: log lines showing the DNS-01 challenge for `v2e.sh` succeeding on the staging CA; no `unable to obtain ACME certificate` errors.

- [ ] **Step 3: Confirm HTTPS + redirect + staging issuer**

```bash
curl -vk https://whoami.v2e.sh 2>&1 | grep -E 'issuer|Hostname:'
curl -sI http://whoami.v2e.sh | head -1
```
Expected: issuer contains `(STAGING) Let's Encrypt`; whoami body shows its `Hostname:`; the HTTP call returns `HTTP/1.1 301 Moved Permanently`.

- [ ] **Step 4: Flip to production and confirm a real cert**

```bash
make prod
sleep 30
curl -v https://whoami.v2e.sh 2>&1 | grep -E 'issuer|SSL certificate verify ok'
```
Expected: issuer is `Let's Encrypt` (no STAGING); `curl` (without `-k`) reports
`SSL certificate verify ok`.

- [ ] **Step 5: Record acceptance**

Acceptance (MASTER-PLAN §5) met when: Traefik pulled a valid wildcard LE cert
(staging→prod) and `whoami` is reachable over HTTPS with the redirect working. Note the
result in the PR/branch description. No commit (verification only).

---

## Self-Review

**1. Spec coverage** (spec → task):
- §3.1 replace experimental stack → Task 1 (`git rm compose.yml`). ✓
- §3.2 per-stack folders → Tasks 1, 3, 4. ✓
- §3.3 external `frontend` → declared in Tasks 3/4; created in Task 5 `bootstrap`; README. ✓
- §3.4 command flags → Task 3. ✓
- §3.5 two resolvers → Task 3; flip proven in Task 4 Step 5 + Task 5 `prod`. ✓
- §3.6 frontend/backend naming → Tasks 3/4 (`frontend`); README (`backend` deferred). ✓
- §3.7 config/secret split → Task 2. ✓
- §3.8 no dashboard → Task 3 (no `--api*` flags). ✓
- §3.9 pins `v3.7.6`/`v1.11.0` → Tasks 3/4 + asserted in Steps 4. ✓
- §3.10 cert storage `traefik/data/certs/` → Task 1 (.gitkeep) + Task 3 (mount). ✓
- §4 layout + `.gitignore` → Task 1. ✓
- §5 networks/flows → Tasks 3/4 + README. ✓
- §6 stacks → Tasks 3/4. ✓
- §7 contract → Task 2; asserted in Task 2 Step 4. ✓
- §8 two channels → Task 2 (`.env.example`) + README + Makefile. ✓
- §9 Makefile → Task 5. ✓
- §10 staging→prod → Task 5 `prod` + README + Task 7. ✓
- §11 hardening (now vs backlog) → Tasks 3/4 (now) + README (backlog). ✓
- §12 gotchas → README; asserted in Task 6 Step 2. ✓
- §13 testing → per-task `docker compose config` + grep asserts; Task 7 live. ✓
- §14 ANS-3 handoff → README. ✓

No gaps.

**2. Placeholder scan:** The only `<...>`/`age1replace...` tokens are inside `.example`
and `.sops.yaml` template *file contents* (legitimate per-user values, with explicit
"replace with yours" instructions) — not plan placeholders. No TBD/TODO; every code step
shows complete content; every test step has an exact command + expected output.

**3. Type/name consistency:** resolver names `staging`/`production`, middleware
`secure-headers`/`secure-headers@docker`, network `frontend`, storage paths
`/certs/acme-staging.json` + `/certs/acme.json`, and the var names `DOMAIN`/`ACME_EMAIL`/
`CERT_RESOLVER`/`CF_DNS_API_TOKEN` are used identically across Tasks 2–7 and the Makefile.
Image tags `traefik:v3.7.6` and `traefik/whoami:v1.11.0` match everywhere. ✓
