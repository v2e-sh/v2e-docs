# COMPOSE-2 — Auth (TinyAuth forward-auth) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a TinyAuth forward-auth layer to the v2e-compose stack — bring the Traefik dashboard up on `traefik.v2e.sh` behind login, protect it, leave `whoami` public, and move the wildcard anchor onto the dashboard router — kept swappable to Authelia later.

**Architecture:** A new per-stack `tinyauth/` project runs TinyAuth (login UI on a public `tinyauth.v2e.sh` router) and defines a backend-neutral forward-auth middleware named `auth`. The Traefik dashboard is enabled and routed on `traefik.v2e.sh` behind `auth@docker,secure-headers@docker`, carrying the wildcard `tls.domains` anchor (moved off `whoami`). `whoami` and the login page serve the wildcard from the store via plain `tls=true`. Secrets (`TINYAUTH_SECRET`, `TINYAUTH_AUTH_USERS`) join the one SOPS file.

**Tech Stack:** Docker Compose v2, Traefik v3.7.6, TinyAuth v5.0.7, Let's Encrypt (DNS-01, from COMPOSE-1), SOPS + age, GNU Make, GitHub Actions.

**Source spec:** `v2e-docs/specs/2026-06-30-compose-auth-design.md`

**Environment note:** Docker is NOT available in the implementation environment. Per-file verification uses `ruby -ryaml` (YAML parse) + `grep` on the raw files + `make -n`. The `docker compose config` schema check and the live auth flow run in CI / on a Docker host.

> **Correction (post-final-review):** `TINYAUTH_SECRET` is **not** a TinyAuth v5 variable and
> was removed from the shipped stack (commit `2de2637`) — v5 auto-manages the session key. The
> `TINYAUTH_SECRET` references in the Global Constraints and Tasks 1/2/5 below are **superseded**;
> the valid v5 env is `TINYAUTH_APPURL` + `TINYAUTH_AUTH_USERS` only.

## Global Constraints

- **Repo / branch:** `v2e-compose`, branch `feat/compose-auth`. Lands via **PR** (the `main` ruleset requires PRs) — do NOT merge to `main` locally.
- **Image pin (exact):** `ghcr.io/steveiliop56/tinyauth:v5.0.7`. (Traefik stays `traefik:v3.7.6`, whoami `traefik/whoami:v1.11.0`.)
- **Middleware name:** the forward-auth middleware is **`auth`** (not `tinyauth`) — protected routers reference `auth@docker`.
- **Protected vs public:** the Traefik **dashboard** is protected (`auth@docker,secure-headers@docker`); **`whoami` and the TinyAuth login page are public** (no `auth` middleware — the login page cannot gate itself).
- **Wildcard anchor** (`tls.certResolver` + `tls.domains` main=`${DOMAIN}` sans=`*.${DOMAIN}`) lives on the **dashboard router**; `whoami` and the login router use plain `tls=true`.
- **Secrets:** `TINYAUTH_SECRET` (32 chars) + `TINYAUTH_AUTH_USERS` (`user:bcrypt`) in `secrets.sops.yaml`, reached via `sops exec-env`. `TINYAUTH_APPURL=https://tinyauth.${DOMAIN}` is derived — no new `.env` var.
- **HSTS stays OFF** (dashboard validated on staging cert first).
- **`down`/`logs`/`validate`** must pre-set dummy values for every `${VAR:?}` guard (Compose evaluates guards on every subcommand — the COMPOSE-1 lesson).
- **Git:** commit per task; short messages; **no Claude attribution trailers**.

---

## File Structure

| File | Change | Responsibility | Task |
|---|---|---|---|
| `secrets.sops.yaml.example` | modify | Add `TINYAUTH_SECRET` + `TINYAUTH_AUTH_USERS` (+ generation comments) | 1 |
| `tinyauth/compose.yml` | create | TinyAuth service, public login router, the `auth` middleware | 2 |
| `traefik/compose.yml` | modify | `--api.dashboard=true`; dashboard router (protected) + wildcard anchor | 3 |
| `whoami/compose.yml` | modify | Swap anchor labels for `tls=true`; stays public | 4 |
| `Makefile` | modify | up/prod/down/validate include the tinyauth stack | 5 |
| `.github/workflows/ci.yml` | modify | Validate `tinyauth/compose.yml` too | 5 |
| `README.md` | modify | Auth section: create user, protected vs public, dashboard, SSO | 6 |

---

## Task 1: Branch + extend the secret contract

**Files:**
- Branch: `feat/compose-auth` (off `main`)
- Modify: `secrets.sops.yaml.example`

**Interfaces:**
- Consumes: the COMPOSE-1 `secrets.sops.yaml.example` (has `CF_DNS_API_TOKEN`).
- Produces: the `TINYAUTH_SECRET` + `TINYAUTH_AUTH_USERS` secret contract consumed by Tasks 2 and 5.

- [ ] **Step 1: Confirm the branch (created by the controller)**

Run: `git branch --show-current`
Expected: `feat/compose-auth`. (If not, `git checkout -b feat/compose-auth`.)

- [ ] **Step 2: Overwrite `secrets.sops.yaml.example` with the extended contract**

```yaml
# Shared secrets for the v2e-compose stacks. Encrypt to produce the real file:
#   sops --encrypt secrets.sops.yaml.example > secrets.sops.yaml
# then `sops secrets.sops.yaml` to fill in the real values.

# --- Traefik / Cloudflare DNS-01 (COMPOSE-1) ---
# Scoped Cloudflare API token — Zone:Read + DNS:Edit (NOT the global key).
CF_DNS_API_TOKEN: "<your-scoped-cloudflare-token>"

# --- TinyAuth (COMPOSE-2) ---
# Cookie signing/encryption key — exactly 32 chars:  openssl rand -hex 16
TINYAUTH_SECRET: "<32-char-random-secret>"
# One or more users, comma-separated:  user:bcrypt[:totp]
#   docker run --rm -it ghcr.io/steveiliop56/tinyauth:v5.0.7 user create
TINYAUTH_AUTH_USERS: "<user:bcrypt-hash>"
```

- [ ] **Step 3: Verify the contract (the test)**

```bash
grep -Eq '^CF_DNS_API_TOKEN:' secrets.sops.yaml.example \
  && grep -Eq '^TINYAUTH_SECRET:' secrets.sops.yaml.example \
  && grep -Eq '^TINYAUTH_AUTH_USERS:' secrets.sops.yaml.example \
  && ruby -ryaml -e 'YAML.load_file("secrets.sops.yaml.example"); puts "yaml OK"' \
  && echo "secret contract OK"
```
Expected: `yaml OK` then `secret contract OK`.

- [ ] **Step 4: Commit**

```bash
git add secrets.sops.yaml.example
git commit -m "compose: add tinyauth secret contract (secret + users)"
```

---

## Task 2: TinyAuth stack (`tinyauth/compose.yml`)

**Files:**
- Create: `tinyauth/compose.yml`

**Interfaces:**
- Consumes: `${DOMAIN}`, `${TINYAUTH_SECRET}`, `${TINYAUTH_AUTH_USERS}`; the external `frontend` network; the `secure-headers@docker` middleware (from `traefik/compose.yml`).
- Produces: the **`auth@docker`** forward-auth middleware (referenced by Task 3's dashboard router and by COMPOSE-3 services); the public login router `Host(tinyauth.${DOMAIN})`.

- [ ] **Step 1: Verify the file does not yet validate (failing test)**

```bash
DOMAIN=example.com TINYAUTH_SECRET=x TINYAUTH_AUTH_USERS=y \
  docker compose -f tinyauth/compose.yml config >/dev/null 2>&1; echo "exit=$?"
```
Expected: non-zero (file absent). (Docker may also be absent — either way it does not pass yet.)

- [ ] **Step 2: Write `tinyauth/compose.yml`**

```yaml
services:
  tinyauth:
    image: ghcr.io/steveiliop56/tinyauth:v5.0.7
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - frontend
    environment:
      - TINYAUTH_APPURL=https://tinyauth.${DOMAIN}
      - TINYAUTH_SECRET=${TINYAUTH_SECRET:?set via `make up` (sops exec-env)}
      - TINYAUTH_AUTH_USERS=${TINYAUTH_AUTH_USERS:?set via `make up` (sops exec-env)}
    labels:
      - traefik.enable=true
      # Public login router — NO `auth` middleware (it cannot gate its own login).
      - traefik.http.routers.tinyauth.rule=Host(`tinyauth.${DOMAIN}`)
      - traefik.http.routers.tinyauth.entryPoints=websecure
      - traefik.http.routers.tinyauth.tls=true
      - traefik.http.routers.tinyauth.middlewares=secure-headers@docker
      - traefik.http.services.tinyauth.loadbalancer.server.port=3000
      # Backend-neutral forward-auth middleware (named `auth`, not `tinyauth`).
      - traefik.http.middlewares.auth.forwardauth.address=http://tinyauth:3000/api/auth/traefik
      - traefik.http.middlewares.auth.forwardauth.authResponseHeaders=Remote-User,Remote-Groups,Remote-Email,Remote-Name

networks:
  frontend:
    external: true
```

- [ ] **Step 3: YAML-parse (test passes)**

```bash
ruby -ryaml -e 'YAML.load_file("tinyauth/compose.yml"); puts "yaml OK"'
```
Expected: `yaml OK`.

- [ ] **Step 4: Assert the critical settings (grep the raw file with fixed strings)**

```bash
grep -qF 'ghcr.io/steveiliop56/tinyauth:v5.0.7' tinyauth/compose.yml \
  && grep -qF 'no-new-privileges:true'          tinyauth/compose.yml \
  && grep -qF 'Host(`tinyauth.${DOMAIN}`)'      tinyauth/compose.yml \
  && grep -qF 'middlewares.auth.forwardauth.address=http://tinyauth:3000/api/auth/traefik' tinyauth/compose.yml \
  && grep -qF 'middlewares.auth.forwardauth.authResponseHeaders=Remote-User' tinyauth/compose.yml \
  && grep -qF 'loadbalancer.server.port=3000'   tinyauth/compose.yml \
  && echo "tinyauth assertions OK"
# The login router must NOT carry the auth middleware (would infinite-redirect):
grep -qF 'routers.tinyauth.middlewares=secure-headers@docker' tinyauth/compose.yml \
  && ! grep -qF 'routers.tinyauth.middlewares=auth' tinyauth/compose.yml \
  && echo "login-page-public OK"
```
Expected: `tinyauth assertions OK` then `login-page-public OK`.

- [ ] **Step 5: Commit**

```bash
git add tinyauth/compose.yml
git commit -m "compose: tinyauth stack (login router + auth forward-auth middleware)"
```

---

## Task 3: Traefik dashboard + wildcard anchor (`traefik/compose.yml`)

**Files:**
- Modify: `traefik/compose.yml`

**Interfaces:**
- Consumes: `${DOMAIN}`, `${CERT_RESOLVER}`; the `auth@docker` middleware (Task 2) and the existing `secure-headers@docker`.
- Produces: the dashboard router `Host(traefik.${DOMAIN})` (protected) that now holds the wildcard anchor.

- [ ] **Step 1: Overwrite `traefik/compose.yml` with the dashboard added**

(Full file — the two changes vs COMPOSE-1 are the `--api.dashboard=true` command flag and the `dashboard` router labels carrying the moved wildcard anchor.)

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
      - CF_DNS_API_TOKEN=${CF_DNS_API_TOKEN:?set via `make up` (sops exec-env) or a local export}
    command:
      - --global.checkNewVersion=false
      - --global.sendAnonymousUsage=false
      - --log.level=INFO
      - --ping=true
      - --api.dashboard=true
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
      # Reusable security-headers middleware (HSTS omitted until prod cert).
      - traefik.http.middlewares.secure-headers.headers.frameDeny=true
      - traefik.http.middlewares.secure-headers.headers.contentTypeNosniff=true
      - traefik.http.middlewares.secure-headers.headers.browserXssFilter=true
      - traefik.http.middlewares.secure-headers.headers.referrerPolicy=strict-origin-when-cross-origin
      # Dashboard router — PROTECTED, and it now carries the wildcard anchor (moved off whoami).
      - traefik.http.routers.dashboard.rule=Host(`traefik.${DOMAIN}`)
      - traefik.http.routers.dashboard.entryPoints=websecure
      - traefik.http.routers.dashboard.service=api@internal
      - traefik.http.routers.dashboard.middlewares=auth@docker,secure-headers@docker
      - traefik.http.routers.dashboard.tls.certResolver=${CERT_RESOLVER:-staging}
      - traefik.http.routers.dashboard.tls.domains[0].main=${DOMAIN}
      - traefik.http.routers.dashboard.tls.domains[0].sans=*.${DOMAIN}
    healthcheck:
      test: ["CMD", "traefik", "healthcheck", "--ping"]
      interval: 10s
      timeout: 5s
      retries: 3

networks:
  frontend:
    external: true
```

- [ ] **Step 2: YAML-parse (test)**

```bash
ruby -ryaml -e 'YAML.load_file("traefik/compose.yml"); puts "yaml OK"'
```
Expected: `yaml OK`.

- [ ] **Step 3: Assert the dashboard + anchor + unchanged baseline (test)**

```bash
grep -qF '--api.dashboard=true'                                   traefik/compose.yml \
  && grep -qF 'routers.dashboard.rule=Host(`traefik.${DOMAIN}`)'  traefik/compose.yml \
  && grep -qF 'routers.dashboard.service=api@internal'            traefik/compose.yml \
  && grep -qF 'routers.dashboard.middlewares=auth@docker,secure-headers@docker' traefik/compose.yml \
  && grep -qF 'routers.dashboard.tls.domains[0].sans=*.${DOMAIN}' traefik/compose.yml \
  && grep -qF 'no-new-privileges:true'                            traefik/compose.yml \
  && grep -qF '/var/run/docker.sock:/var/run/docker.sock:ro'      traefik/compose.yml \
  && echo "traefik dashboard+anchor OK"
# HSTS must still be absent:
! grep -qiF 'stsSeconds' traefik/compose.yml && ! grep -qiF 'Strict-Transport-Security' traefik/compose.yml \
  && echo "HSTS still off OK"
```
Expected: `traefik dashboard+anchor OK` then `HSTS still off OK`.

- [ ] **Step 4: Commit**

```bash
git add traefik/compose.yml
git commit -m "compose: enable dashboard behind auth + move wildcard anchor to it"
```

---

## Task 4: whoami — anchor → `tls=true` (`whoami/compose.yml`)

**Files:**
- Modify: `whoami/compose.yml`

**Interfaces:**
- Consumes: `${DOMAIN}`; the wildcard cert from the store (now anchored on the dashboard router, Task 3).
- Produces: a public `whoami.${DOMAIN}` route serving the wildcard via `tls=true`.

- [ ] **Step 1: Overwrite `whoami/compose.yml`**

(Full file — the wildcard anchor labels (`tls.certResolver`, `tls.domains[0].*`) are replaced by a single `tls=true`; `secure-headers` and public status are unchanged; no `auth` middleware.)

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
      - traefik.http.routers.whoami.tls=true
      - traefik.http.routers.whoami.middlewares=secure-headers@docker
      - traefik.http.services.whoami.loadbalancer.server.port=80

networks:
  frontend:
    external: true
```

- [ ] **Step 2: YAML-parse (test)**

```bash
ruby -ryaml -e 'YAML.load_file("whoami/compose.yml"); puts "yaml OK"'
```
Expected: `yaml OK`.

- [ ] **Step 3: Assert anchor removed, still public, still TLS (test)**

```bash
grep -qF 'routers.whoami.tls=true'          whoami/compose.yml \
  && grep -qF 'Host(`whoami.${DOMAIN}`)'    whoami/compose.yml \
  && grep -qF 'middlewares=secure-headers@docker' whoami/compose.yml \
  && echo "whoami tls+public OK"
# The wildcard anchor must be GONE from whoami (moved to dashboard), and NO auth middleware:
! grep -qF 'tls.domains' whoami/compose.yml && ! grep -qF 'certResolver' whoami/compose.yml \
  && ! grep -qF 'auth@docker' whoami/compose.yml \
  && echo "whoami anchor-removed + still-public OK"
```
Expected: `whoami tls+public OK` then `whoami anchor-removed + still-public OK`.

- [ ] **Step 4: Commit**

```bash
git add whoami/compose.yml
git commit -m "compose: whoami serves wildcard via tls=true (anchor moved to dashboard)"
```

---

## Task 5: Makefile + CI — include the tinyauth stack

**Files:**
- Modify: `Makefile`
- Modify: `.github/workflows/ci.yml`

**Interfaces:**
- Consumes: the three stacks + the secret contract.
- Produces: `up`/`prod`/`down`/`validate` covering tinyauth; CI validating tinyauth.

- [ ] **Step 1: Overwrite `Makefile` (TAB-indented recipes)**

```makefile
# v2e-compose — local bootstrap & operations for the Traefik + TLS + auth stack.
# LOCAL/standalone front door only. The automated lab path is
# `terraform apply` -> Ansible -> ANS-3 (docker_compose_v2); make is not involved there.

SHELL    := /bin/bash
COMPOSE  := docker compose
ENVFILE  := .env
SECRETS  := secrets.sops.yaml
TRAEFIK  := -f traefik/compose.yml
TINYAUTH := -f tinyauth/compose.yml
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
		echo "created $(SECRETS) — set CF_DNS_API_TOKEN, TINYAUTH_SECRET (openssl rand -hex 16),"; \
		echo "  and TINYAUTH_AUTH_USERS (docker run --rm -it ghcr.io/steveiliop56/tinyauth:v5.0.7 user create)"; \
		sops $(SECRETS); \
	fi
	@echo "bootstrap done. Next: edit .env, then 'make up'"

up:
	sops exec-env $(SECRETS) '$(COMPOSE) --env-file $(ENVFILE) $(TRAEFIK) up -d'
	sops exec-env $(SECRETS) '$(COMPOSE) --env-file $(ENVFILE) $(TINYAUTH) up -d'
	$(COMPOSE) --env-file $(ENVFILE) $(WHOAMI) up -d

prod:
	sops exec-env $(SECRETS) 'CERT_RESOLVER=production $(COMPOSE) --env-file $(ENVFILE) $(TRAEFIK) up -d'
	sops exec-env $(SECRETS) '$(COMPOSE) --env-file $(ENVFILE) $(TINYAUTH) up -d'
	CERT_RESOLVER=production $(COMPOSE) --env-file $(ENVFILE) $(WHOAMI) up -d

down:
	-$(COMPOSE) $(WHOAMI) down
	-TINYAUTH_SECRET=unused TINYAUTH_AUTH_USERS=unused $(COMPOSE) $(TINYAUTH) down
	-CF_DNS_API_TOKEN=unused $(COMPOSE) $(TRAEFIK) down

logs:
	CF_DNS_API_TOKEN=unused $(COMPOSE) $(TRAEFIK) logs -f traefik

validate:
	@DOMAIN=example.com ACME_EMAIL=a@b.c CERT_RESOLVER=staging CF_DNS_API_TOKEN=dummy \
		$(COMPOSE) $(TRAEFIK) config >/dev/null && echo "traefik/compose.yml OK"
	@DOMAIN=example.com TINYAUTH_SECRET=dummy TINYAUTH_AUTH_USERS=dummy \
		$(COMPOSE) $(TINYAUTH) config >/dev/null && echo "tinyauth/compose.yml OK"
	@DOMAIN=example.com CERT_RESOLVER=staging \
		$(COMPOSE) $(WHOAMI) config >/dev/null && echo "whoami/compose.yml OK"
```

- [ ] **Step 2: Overwrite `.github/workflows/ci.yml`**

```yaml
name: ci
on: [pull_request, push]
jobs:
  compose:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # Per-stack compose files (no root compose.yml). Dummy env satisfies the
      # ${VAR}/${VAR:?} interpolation; `config -q` validates without a daemon.
      - run: |
          DOMAIN=example.com ACME_EMAIL=a@b.c CERT_RESOLVER=staging CF_DNS_API_TOKEN=dummy \
            docker compose -f traefik/compose.yml config -q
          DOMAIN=example.com TINYAUTH_SECRET=dummy TINYAUTH_AUTH_USERS=dummy \
            docker compose -f tinyauth/compose.yml config -q
          DOMAIN=example.com CERT_RESOLVER=staging \
            docker compose -f whoami/compose.yml config -q
```

- [ ] **Step 3: Verify Makefile expands correctly + CI YAML parses (test)**

```bash
make help
make -n up | grep -qF '-f tinyauth/compose.yml up -d' && echo "up includes tinyauth"
make -n down | grep -qF 'TINYAUTH_SECRET=unused TINYAUTH_AUTH_USERS=unused docker compose -f tinyauth/compose.yml down' && echo "down guards tinyauth"
ruby -ryaml -e 'YAML.load_file(".github/workflows/ci.yml"); puts "ci yaml OK"'
grep -qF 'tinyauth/compose.yml config -q' .github/workflows/ci.yml && echo "ci validates tinyauth"
```
Expected: `up includes tinyauth`, `down guards tinyauth`, `ci yaml OK`, `ci validates tinyauth`.

- [ ] **Step 4: Commit**

```bash
git add Makefile .github/workflows/ci.yml
git commit -m "compose: make + ci cover the tinyauth stack"
```

---

## Task 6: README — auth section

**Files:**
- Modify: `README.md`

**Interfaces:**
- Consumes: everything from Tasks 1–5.
- Produces: operator docs for the auth layer.

- [ ] **Step 1: Insert an "## Authentication (TinyAuth)" section into `README.md`**

Add this section (place it after the "First run" section; leave the rest of the README intact):

```markdown
## Authentication (TinyAuth)

COMPOSE-2 puts a TinyAuth forward-auth layer in front of protected services.

- **Protected:** the Traefik dashboard — `https://traefik.v2e.sh` (behind login).
- **Public:** `whoami.v2e.sh` and the TinyAuth login page `tinyauth.v2e.sh`.
- **SSO:** the session cookie is set on `.v2e.sh`, so one login covers every protected
  `*.v2e.sh` subdomain (COMPOSE-3's services inherit it).

### Create the signing secret + a user

Both live in `secrets.sops.yaml` (edit with `sops secrets.sops.yaml`):

```bash
openssl rand -hex 16                                                   # -> TINYAUTH_SECRET (32 chars)
docker run --rm -it ghcr.io/steveiliop56/tinyauth:v5.0.7 user create   # -> TINYAUTH_AUTH_USERS (user:bcrypt)
```

`make up` deploys tinyauth alongside traefik/whoami. Then:

- `https://traefik.v2e.sh` → redirected to the TinyAuth login → valid creds → dashboard.
- `https://whoami.v2e.sh` → loads with no prompt (public).

### Protecting another service

Add one label to its router: `traefik.http.routers.<name>.middlewares=auth@docker`
(chain with `secure-headers@docker` as needed). The login page must **never** carry
`auth` — it cannot gate itself.

### Swapping to Authelia later

The middleware is named `auth` (not `tinyauth`) on purpose: replace the tinyauth stack with
Authelia (+ Valkey), define a middleware also named `auth` pointing at Authelia's verify
endpoint, and every protected router (`middlewares=auth@docker`) keeps working unchanged.
```

- [ ] **Step 2: Verify coverage (test)**

```bash
grep -q 'Authentication (TinyAuth)' README.md \
  && grep -q 'traefik.v2e.sh'       README.md \
  && grep -q 'auth@docker'          README.md \
  && grep -q 'user create'          README.md \
  && grep -q 'Swapping to Authelia' README.md \
  && grep -q 'never.*carry'         README.md 2>/dev/null; \
grep -q 'openssl rand -hex 16'      README.md \
  && echo "README auth coverage OK"
```
Expected: `README auth coverage OK`.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "compose: readme — tinyauth auth section"
```

---

## Task 7: Open the PR

**Files:** none (integration).

- [ ] **Step 1: Push the branch**

```bash
git push -u origin feat/compose-auth
```

- [ ] **Step 2: Open the PR** (CI runs the per-stack `config -q`, now including tinyauth)

```bash
gh pr create --title "COMPOSE-2: auth (TinyAuth forward-auth)" \
  --body "Adds TinyAuth forward-auth: dashboard behind login on traefik.v2e.sh, whoami public, wildcard anchor moved to the dashboard router. Live auth flow to be validated on a Docker host. See v2e-docs spec/plan 2026-06-30-compose-auth."
```

- [ ] **Step 3: Report the PR URL and CI status.** No merge here — review + merge is the user's call.

---

## Self-Review

**1. Spec coverage** (spec → task):
- §3.1 local user/pass, TOTP deferred → Task 1 (`user:bcrypt`, no totp field required). ✓
- §3.2 dashboard protected / whoami public → Task 3 (`auth@docker` on dashboard), Task 4 (no auth on whoami). ✓
- §3.3 middleware named `auth` → Task 2 defines `auth`; Task 3 references `auth@docker`. ✓
- §3.4 wildcard anchor → dashboard; whoami/login `tls=true` → Tasks 2, 3, 4. ✓
- §3.5 pin `tinyauth:v5.0.7` → Tasks 1, 2, 5 (asserted). ✓
- §3.6 secrets via SOPS → Tasks 1, 5. ✓
- §3.7 PR workflow → Task 7. ✓
- §4 file surface → all tasks match. ✓
- §5 flow (public login, protected dashboard, SSO) → Tasks 2/3; README §6. ✓
- §6 swap path → Task 2 (`auth` name + `Remote-*` headers); README §6. ✓
- §7 secret contract + derived APPURL → Tasks 1, 2. ✓
- §8 Makefile/CI (incl. down/logs dummy guards) → Task 5. ✓
- §9 testing → per-task YAML+grep; CI config; Task 7 PR; live on host. ✓
- §10 gotchas (public login host, HSTS off) → Task 2 (login-page-public assert), Task 3 (HSTS-off assert); README. ✓
- §11 handoff → README swap/protect section. ✓

No gaps.

**2. Placeholder scan:** `<...>` tokens appear only inside `secrets.sops.yaml.example` content (per-user values, with generation commands) — not plan placeholders. Every code step shows full file content; every test step has exact commands + expected output. No TBD/TODO.

**3. Type/name consistency:** middleware `auth`/`auth@docker`, `secure-headers@docker`, hosts `traefik.${DOMAIN}`/`tinyauth.${DOMAIN}`/`whoami.${DOMAIN}`, vars `TINYAUTH_SECRET`/`TINYAUTH_AUTH_USERS`/`TINYAUTH_APPURL`/`DOMAIN`/`CERT_RESOLVER`, image `ghcr.io/steveiliop56/tinyauth:v5.0.7`, port `3000`, forward-auth path `/api/auth/traefik` — all used identically across Tasks 1–7 and the Makefile/CI. ✓
