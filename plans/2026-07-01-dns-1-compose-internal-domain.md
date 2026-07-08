# DNS-1 (slice 3/3) — `v2e-compose` `INTERNAL_DOMAIN` + `.int` routers — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Introduce a single `INTERNAL_DOMAIN` config variable (default `int.v2e.sh`), move the dashboard/auth/whoami routers onto `*.${INTERNAL_DOMAIN}`, extend the Traefik wildcard cert to also request `*.${INTERNAL_DOMAIN}`, and add a `dns.${INTERNAL_DOMAIN}` router to the Technitium console (`10.1.0.53:5380`) behind `auth@docker`.

**Architecture:** `INTERNAL_DOMAIN` (+ `DNS_ADMIN_ADDR`) join `.env`. The three existing routers swap `${DOMAIN}` → `${INTERNAL_DOMAIN}` (a mechanical variable rename). The dashboard router's wildcard anchor gains a second `domains[1]` entry so Cloudflare DNS-01 issues both `*.${DOMAIN}` (kept for the public tunnel) and `*.${INTERNAL_DOMAIN}`. The Technitium console is a non-container backend, so Traefik gains a **file provider**; because the file provider does not expand env vars, the dynamic router file is a template rendered by `make` via `envsubst`. The ACME `1.1.1.1` resolver pin is untouched, so cert issuance ignores the internal resolver.

**Tech Stack:** Docker Compose v2, Traefik v3.7.6 (docker + file providers), Let's Encrypt DNS-01 (Cloudflare), SOPS + age, GNU Make + `envsubst`, GitHub Actions.

**Source spec:** `v2e-docs/specs/2026-06-30-dns-appliance-design.md` (§4, §6, §12) + MASTER-PLAN DNS-1.

**Environment note:** Docker / SOPS are NOT available in the implementation environment. In-env verification uses `ruby -ryaml` (YAML parse), `grep` on the raw files, `envsubst` (available) to render + parse the dynamic template, and `make -n`. The `docker compose config` schema check and the live cert/auth/routing flow run in CI / on the Docker host.

## Global Constraints

- **Repo / branch:** `v2e-compose`, branch `feat/compose-internal-domain`. Lands via **PR** (the `main` ruleset requires PRs). This is **PR #3** of three; it references the tf-provisioned `10.1.0.53`, so land it after the tf + ansible PRs (or at least after tf).
- **One variable, no hardcoding:** every internal hostname is `*.${INTERNAL_DOMAIN}`. Default `INTERNAL_DOMAIN=int.v2e.sh`. The only place a literal internal host may appear is the rendered `traefik/dynamic/dns.yml` (generated, gitignored) — its source template uses `${INTERNAL_DOMAIN}`.
- **Router moves (exact):** `traefik.${DOMAIN}`→`traefik.${INTERNAL_DOMAIN}`, `tinyauth.${DOMAIN}`→`tinyauth.${INTERNAL_DOMAIN}` (incl. `TINYAUTH_APPURL`), `whoami.${DOMAIN}`→`whoami.${INTERNAL_DOMAIN}`. New `dns.${INTERNAL_DOMAIN}`.
- **Cert anchor:** keep `domains[0] = ${DOMAIN}` / `*.${DOMAIN}` (public + future tunnel) AND add `domains[1] = ${INTERNAL_DOMAIN}` / `*.${INTERNAL_DOMAIN}`. Both live on the `dashboard` router (the existing wildcard anchor).
- **Keep the ACME resolver pin:** `--certificatesResolvers.*.acme.dnsChallenge.resolvers=1.1.1.1:53,1.0.0.1:53` stays on both staging + production resolvers — do NOT change or remove it.
- **DNS router serves the wildcard from the store:** the `dns` file-provider router uses `tls: {}` (like `whoami`/`tinyauth`) — it does NOT own its own `certResolver`/`domains` (the dashboard router requests the `*.${INTERNAL_DOMAIN}` wildcard; the store serves it by SNI). This avoids a duplicate ACME order.
- **`DNS_ADMIN_ADDR`** default `10.1.0.53:5380` — the Technitium console. The `dns` service load-balances to `http://${DNS_ADMIN_ADDR}`.
- **`${VAR}` guards on every subcommand:** `down`/`logs`/`validate` must pre-set values for every referenced var so Compose interpolation doesn't warn/fail (the COMPOSE-1 lesson) — now including `INTERNAL_DOMAIN` and `DNS_ADMIN_ADDR`.
- **Git:** commit per task; short messages, project style; **no AI attribution trailers**.

---

## File Structure

| File | Change | Responsibility | Task |
|---|---|---|---|
| `.env.example` | modify | Add `INTERNAL_DOMAIN`, `DNS_ADMIN_ADDR` | 1 |
| `traefik/compose.yml` | modify | File provider + mount; cert `domains[1]`; dashboard rule → INTERNAL_DOMAIN | 2 |
| `traefik/dns.yml.tmpl` | create | Dynamic router/service template (envsubst source) | 3 |
| `.gitignore` | modify | Ignore generated `traefik/dynamic/` | 3 |
| `tinyauth/compose.yml` | modify | Router rule + `TINYAUTH_APPURL` → INTERNAL_DOMAIN | 4 |
| `whoami/compose.yml` | modify | Router rule → INTERNAL_DOMAIN | 4 |
| `Makefile` | modify | Render dynamic file; up/prod/down/logs/validate carry new vars | 5 |
| `.github/workflows/ci.yml` | modify | Dummy env incl. new vars; validate rendered dynamic file | 6 |
| `README.md` | modify | INTERNAL_DOMAIN contract, dns router, file provider, cert | 7 |

---

## Task 1: Add `INTERNAL_DOMAIN` + `DNS_ADMIN_ADDR` to config

**Files:**
- Branch: `feat/compose-internal-domain` (off `main`)
- Modify: `.env.example`

**Interfaces:**
- Produces: env vars `INTERNAL_DOMAIN` (default `int.v2e.sh`) and `DNS_ADMIN_ADDR` (default `10.1.0.53:5380`), consumed by Tasks 2–6.

- [ ] **Step 1: Confirm the branch**

Run: `git -C ~/v2e-environment/v2e-compose branch --show-current`
Expected: `feat/compose-internal-domain` (else `git checkout -b feat/compose-internal-domain`).

- [ ] **Step 2: Extend `.env.example`**

Overwrite `.env.example` with:

```env
# Non-secret config for the v2e-compose stacks. Copy to .env and edit (or let Ansible
# render it from group_vars in the automated flow). The secret (CF_DNS_API_TOKEN) lives
# in secrets.sops.yaml, NOT here.
DOMAIN=v2e.sh
# Internal-only domain resolved by the self-hosted Technitium resolver (DNS-1).
# All lab dashboards live at *.${INTERNAL_DOMAIN}; the public DOMAIN stays for the
# Cloudflare tunnel (lab.v2e.sh) + public zone.
INTERNAL_DOMAIN=int.v2e.sh
# Technitium admin console (the dns node) — target of the dns.${INTERNAL_DOMAIN} router.
DNS_ADMIN_ADDR=10.1.0.53:5380
ACME_EMAIL=you@example.com
CERT_RESOLVER=staging          # flip to `production` once staging issues cleanly
```

- [ ] **Step 3: Verify**

Run: `grep -nE 'INTERNAL_DOMAIN|DNS_ADMIN_ADDR' ~/v2e-environment/v2e-compose/.env.example`
Expected: both present with the defaults above.

- [ ] **Step 4: Commit**

```bash
cd ~/v2e-environment/v2e-compose
git add .env.example
git commit -m "compose: add INTERNAL_DOMAIN + DNS_ADMIN_ADDR config"
```

---

## Task 2: Traefik — file provider, dashboard on INTERNAL_DOMAIN, dual wildcard

**Files:**
- Modify: `traefik/compose.yml`

**Interfaces:**
- Consumes: `${INTERNAL_DOMAIN}`, `${DOMAIN}`, `${CERT_RESOLVER}`.
- Produces: a Traefik file provider watching `/etc/traefik/dynamic`; the dashboard router on `traefik.${INTERNAL_DOMAIN}`; a wildcard cert covering both domains. Consumed by Task 3 (dynamic file lands in the mounted dir).

- [ ] **Step 1: Add the file provider to the `command:` list**

In `traefik/compose.yml`, after the docker provider lines (line 22, `--providers.docker.network=frontend`), add:

```yaml
      - --providers.file.directory=/etc/traefik/dynamic
      - --providers.file.watch=true
```

- [ ] **Step 2: Mount the dynamic config dir**

In the `volumes:` block (lines 39–41), add:

```yaml
      - ./traefik/dynamic:/etc/traefik/dynamic:ro
```

- [ ] **Step 3: Move the dashboard router to INTERNAL_DOMAIN and add the second wildcard**

Replace the dashboard label block (lines 49–56) with:

```yaml
      # Dashboard router — PROTECTED, on the internal domain, and it carries the
      # wildcard anchor requesting BOTH domains (public DOMAIN + INTERNAL_DOMAIN).
      - traefik.http.routers.dashboard.rule=Host(`traefik.${INTERNAL_DOMAIN}`)
      - traefik.http.routers.dashboard.entryPoints=websecure
      - traefik.http.routers.dashboard.service=api@internal
      - traefik.http.routers.dashboard.middlewares=auth@docker,secure-headers@docker
      - traefik.http.routers.dashboard.tls.certResolver=${CERT_RESOLVER:-staging}
      - traefik.http.routers.dashboard.tls.domains[0].main=${DOMAIN}
      - traefik.http.routers.dashboard.tls.domains[0].sans=*.${DOMAIN}
      - traefik.http.routers.dashboard.tls.domains[1].main=${INTERNAL_DOMAIN}
      - traefik.http.routers.dashboard.tls.domains[1].sans=*.${INTERNAL_DOMAIN}
```

Leave the ACME resolver lines (27–38, incl. the `1.1.1.1:53,1.0.0.1:53` pin) and the `secure-headers` middleware untouched.

- [ ] **Step 4: Parse-check the YAML**

Run:
```bash
cd ~/v2e-environment/v2e-compose
ruby -ryaml -e 'YAML.load_file("traefik/compose.yml"); puts "traefik/compose.yml parses"'
```
Expected: `traefik/compose.yml parses`.

Run: `grep -nE 'providers.file|domains\[1\]|traefik.\$\{INTERNAL_DOMAIN\}|1.1.1.1:53' traefik/compose.yml`
Expected: file provider (2 lines), `domains[1]` main+sans, dashboard rule on `${INTERNAL_DOMAIN}`, and the ACME pin still present.

- [ ] **Step 5: Commit**

```bash
git add traefik/compose.yml
git commit -m "compose: traefik file provider + dashboard on internal domain + dual wildcard"
```

---

## Task 3: Dynamic router template for the DNS console

**Files:**
- Create: `traefik/dns.yml.tmpl`
- Modify: `.gitignore`

**Interfaces:**
- Consumes: `${INTERNAL_DOMAIN}`, `${DNS_ADMIN_ADDR}` (substituted by `envsubst` in Task 5).
- Produces: `traefik/dns.yml.tmpl` → rendered to `traefik/dynamic/dns.yml` (gitignored). Defines router `dns` → service `dns` (external backend `http://${DNS_ADMIN_ADDR}`), behind `auth@docker,secure-headers@docker`, TLS from the store.

- [ ] **Step 1: Create `traefik/dns.yml.tmpl`**

```yaml
# SOURCE TEMPLATE — rendered to traefik/dynamic/dns.yml by `make` (envsubst).
# Do not edit the generated file. The file provider is used because the Technitium
# console (${DNS_ADMIN_ADDR}) is a non-container backend. TLS is served from the
# shared *.${INTERNAL_DOMAIN} wildcard in the cert store (requested by the
# dashboard router), so this router owns no certResolver.
http:
  routers:
    dns:
      rule: "Host(`dns.${INTERNAL_DOMAIN}`)"
      entryPoints:
        - websecure
      service: dns
      middlewares:
        - auth@docker
        - secure-headers@docker
      tls: {}
  services:
    dns:
      loadBalancer:
        servers:
          - url: "http://${DNS_ADMIN_ADDR}"
```

- [ ] **Step 2: Ignore the generated dir**

Add to `.gitignore`:

```
# Traefik dynamic config is generated from traefik/dns.yml.tmpl by `make`.
traefik/dynamic/
```

- [ ] **Step 3: Render + parse-check the template (envsubst is available in-env)**

Run:
```bash
cd ~/v2e-environment/v2e-compose
mkdir -p traefik/dynamic
INTERNAL_DOMAIN=int.v2e.sh DNS_ADMIN_ADDR=10.1.0.53:5380 \
  envsubst '$INTERNAL_DOMAIN $DNS_ADMIN_ADDR' < traefik/dns.yml.tmpl > traefik/dynamic/dns.yml
ruby -ryaml -e 'd=YAML.load_file("traefik/dynamic/dns.yml"); \
  raise "bad rule"  unless d.dig("http","routers","dns","rule")   == "Host(`dns.int.v2e.sh`)"; \
  raise "bad url"   unless d.dig("http","services","dns","loadBalancer","servers",0,"url") == "http://10.1.0.53:5380"; \
  puts "dns.yml renders + parses"'
```
Expected: `dns.yml renders + parses`. (Note the `envsubst` allow-list `'$INTERNAL_DOMAIN $DNS_ADMIN_ADDR'` — it stops `envsubst` from eating Traefik's own `${...}`-free content; there is none here, but the allow-list is the safe habit.)

- [ ] **Step 4: Commit** (the `.tmpl` is tracked; the rendered file is ignored)

```bash
git add traefik/dns.yml.tmpl .gitignore
git status --short   # confirm traefik/dynamic/dns.yml is NOT staged
git commit -m "compose: dynamic file-provider router for the dns console"
```

---

## Task 4: Move tinyauth + whoami onto INTERNAL_DOMAIN

**Files:**
- Modify: `tinyauth/compose.yml`
- Modify: `whoami/compose.yml`

**Interfaces:**
- Consumes: `${INTERNAL_DOMAIN}`.
- Produces: the login page at `tinyauth.${INTERNAL_DOMAIN}` (with matching `TINYAUTH_APPURL`) and `whoami.${INTERNAL_DOMAIN}`. The `auth` forward-auth middleware definition is unchanged (still `auth@docker`).

- [ ] **Step 1: Update `tinyauth/compose.yml`**

Change line 10 and line 15:

```yaml
      - TINYAUTH_APPURL=https://tinyauth.${INTERNAL_DOMAIN}
```

```yaml
      - traefik.http.routers.tinyauth.rule=Host(`tinyauth.${INTERNAL_DOMAIN}`)
```

Leave the `auth` middleware (lines 20–22) and the loadbalancer port unchanged. (TinyAuth v5 derives its cookie domain from `TINYAUTH_APPURL`, so moving the app URL to `int.v2e.sh` gives SSO across `*.int.v2e.sh` automatically — no extra var.)

- [ ] **Step 2: Update `whoami/compose.yml`**

Change line 11:

```yaml
      - traefik.http.routers.whoami.rule=Host(`whoami.${INTERNAL_DOMAIN}`)
```

- [ ] **Step 3: Parse-check both**

Run:
```bash
cd ~/v2e-environment/v2e-compose
ruby -ryaml -e 'YAML.load_file("tinyauth/compose.yml"); YAML.load_file("whoami/compose.yml"); puts "ok"'
grep -nE '\$\{INTERNAL_DOMAIN\}' tinyauth/compose.yml whoami/compose.yml
```
Expected: `ok`; `INTERNAL_DOMAIN` appears in the APPURL + tinyauth rule + whoami rule. Confirm no `${DOMAIN}` remains in router rules:
`grep -n 'Host(`.*\${DOMAIN}`)' tinyauth/compose.yml whoami/compose.yml traefik/compose.yml` → **no matches**.

- [ ] **Step 4: Commit**

```bash
git add tinyauth/compose.yml whoami/compose.yml
git commit -m "compose: move tinyauth + whoami routers onto internal domain"
```

---

## Task 5: Makefile — render the dynamic file; carry new vars

**Files:**
- Modify: `Makefile`

**Interfaces:**
- Consumes: `.env` (`INTERNAL_DOMAIN`, `DNS_ADMIN_ADDR`).
- Produces: a `dynamic` target that renders `traefik/dynamic/dns.yml`; `up`/`prod` depend on it; `down`/`logs`/`validate` carry the new vars.

- [ ] **Step 1: Add the render target and wire it into up/prod**

In `Makefile`, update `.PHONY` and add a `dynamic` target; make `up`/`prod` depend on it. Replace lines 14–42 region as follows.

Update the `.PHONY` line (14):

```makefile
.PHONY: help bootstrap dynamic up prod down logs validate
```

Add the `dynamic` target after `bootstrap` (before `up`):

```makefile
# Render the Traefik file-provider config from its template (env substitution —
# the file provider does not expand env vars itself). Sourced from .env.
dynamic:
	@mkdir -p traefik/dynamic
	@set -a; . ./$(ENVFILE); set +a; \
		: "$${INTERNAL_DOMAIN:?set INTERNAL_DOMAIN in .env}" "$${DNS_ADMIN_ADDR:?set DNS_ADMIN_ADDR in .env}"; \
		envsubst '$$INTERNAL_DOMAIN $$DNS_ADMIN_ADDR' < traefik/dns.yml.tmpl > traefik/dynamic/dns.yml
	@echo "rendered traefik/dynamic/dns.yml"
```

Make `up` and `prod` depend on `dynamic`:

```makefile
up: dynamic
	sops exec-env $(SECRETS) '$(COMPOSE) --env-file $(ENVFILE) $(TRAEFIK) up -d'
	sops exec-env $(SECRETS) '$(COMPOSE) --env-file $(ENVFILE) $(TINYAUTH) up -d'
	$(COMPOSE) --env-file $(ENVFILE) $(WHOAMI) up -d

prod: dynamic
	sops exec-env $(SECRETS) 'CERT_RESOLVER=production $(COMPOSE) --env-file $(ENVFILE) $(TRAEFIK) up -d'
	sops exec-env $(SECRETS) '$(COMPOSE) --env-file $(ENVFILE) $(TINYAUTH) up -d'
	CERT_RESOLVER=production $(COMPOSE) --env-file $(ENVFILE) $(WHOAMI) up -d
```

- [ ] **Step 2: Carry the new vars in down/logs/validate**

Replace `down`, `logs`, `validate` (lines 44–58) with:

```makefile
down:
	-INTERNAL_DOMAIN=unused $(COMPOSE) $(WHOAMI) down
	-INTERNAL_DOMAIN=unused TINYAUTH_AUTH_USERS=unused $(COMPOSE) $(TINYAUTH) down
	-INTERNAL_DOMAIN=unused DNS_ADMIN_ADDR=unused CF_DNS_API_TOKEN=unused $(COMPOSE) $(TRAEFIK) down

logs:
	INTERNAL_DOMAIN=unused DNS_ADMIN_ADDR=unused CF_DNS_API_TOKEN=unused $(COMPOSE) $(TRAEFIK) logs -f traefik

validate: dynamic
	@DOMAIN=example.com INTERNAL_DOMAIN=int.example.com DNS_ADMIN_ADDR=10.0.0.1:5380 \
		ACME_EMAIL=a@b.c CERT_RESOLVER=staging CF_DNS_API_TOKEN=dummy \
		$(COMPOSE) $(TRAEFIK) config >/dev/null && echo "traefik/compose.yml OK"
	@DOMAIN=example.com INTERNAL_DOMAIN=int.example.com TINYAUTH_AUTH_USERS=dummy \
		$(COMPOSE) $(TINYAUTH) config >/dev/null && echo "tinyauth/compose.yml OK"
	@DOMAIN=example.com INTERNAL_DOMAIN=int.example.com CERT_RESOLVER=staging \
		$(COMPOSE) $(WHOAMI) config >/dev/null && echo "whoami/compose.yml OK"
	@ruby -ryaml -e 'YAML.load_file("traefik/dynamic/dns.yml")' && echo "traefik/dynamic/dns.yml OK"
```

- [ ] **Step 3: Dry-run the Makefile logic (no Docker needed)**

Run:
```bash
cd ~/v2e-environment/v2e-compose
[ -f .env ] || cp .env.example .env
make dynamic
ruby -ryaml -e 'YAML.load_file("traefik/dynamic/dns.yml"); puts "rendered via make OK"'
make -n up | head -5   # shows `dynamic` runs before the compose up lines
```
Expected: `rendered traefik/dynamic/dns.yml`, `rendered via make OK`, and `make -n up` shows the render step.

- [ ] **Step 4: Commit**

```bash
git add Makefile
git commit -m "compose: render dynamic dns router + carry internal-domain vars"
```

---

## Task 6: CI — validate with the new vars + the rendered file

**Files:**
- Modify: `.github/workflows/ci.yml`

**Interfaces:**
- Consumes: the compose files + `traefik/dns.yml.tmpl`.
- Produces: CI that validates all stacks with `INTERNAL_DOMAIN`/`DNS_ADMIN_ADDR` set and checks the rendered dynamic file is valid YAML.

- [ ] **Step 1: Update the `compose` job**

Replace the `run:` block in the `compose` job with:

```yaml
      - run: |
          DOMAIN=example.com INTERNAL_DOMAIN=int.example.com DNS_ADMIN_ADDR=10.0.0.1:5380 \
          ACME_EMAIL=a@b.c CERT_RESOLVER=staging CF_DNS_API_TOKEN=dummy \
            docker compose -f traefik/compose.yml config -q
          DOMAIN=example.com INTERNAL_DOMAIN=int.example.com TINYAUTH_AUTH_USERS=dummy \
            docker compose -f tinyauth/compose.yml config -q
          DOMAIN=example.com INTERNAL_DOMAIN=int.example.com CERT_RESOLVER=staging \
            docker compose -f whoami/compose.yml config -q
      - name: Render + validate the file-provider dynamic config
        run: |
          mkdir -p traefik/dynamic
          INTERNAL_DOMAIN=int.example.com DNS_ADMIN_ADDR=10.0.0.1:5380 \
            envsubst '$INTERNAL_DOMAIN $DNS_ADMIN_ADDR' < traefik/dns.yml.tmpl > traefik/dynamic/dns.yml
          python3 -c 'import yaml,sys; yaml.safe_load(open("traefik/dynamic/dns.yml")); print("dns.yml OK")'
```

(`envsubst` ships with the `gettext-base` package, present on `ubuntu-latest`. Leave the `secret-scan` job unchanged.)

- [ ] **Step 2: Lint the workflow YAML**

Run:
```bash
cd ~/v2e-environment/v2e-compose
ruby -ryaml -e 'YAML.load_file(".github/workflows/ci.yml"); puts "ci.yml parses"'
```
Expected: `ci.yml parses`.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "compose: ci validates internal-domain stacks + rendered dns router"
```

---

## Task 7: README

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add an INTERNAL_DOMAIN / DNS section**

Document in `README.md`:

- the `INTERNAL_DOMAIN` contract (one var, default `int.v2e.sh`) — all dashboards live at `*.${INTERNAL_DOMAIN}`; `DOMAIN` stays for the public zone + Cloudflare tunnel (`lab.v2e.sh`);
- the wildcard cert now requests both `*.${DOMAIN}` and `*.${INTERNAL_DOMAIN}` (Cloudflare DNS-01 issues `*.int.v2e.sh` because `int.v2e.sh` is under the CF-managed zone);
- **the `1.1.1.1` ACME resolver pin stays** — cert issuance checks the public authoritative answer, never the internal resolver;
- `dns.${INTERNAL_DOMAIN}` → the Technitium console (`${DNS_ADMIN_ADDR}`, default `10.1.0.53:5380`) behind `auth@docker`, served via the Traefik **file provider** (rendered from `traefik/dns.yml.tmpl` by `make` because the file provider doesn't expand env vars);
- resolution: on-lab via VyOS DHCP option 6; remotely via Tailscale split-DNS (ANS-4) — out of scope here.

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "compose: document INTERNAL_DOMAIN + the dns console router"
```

---

## Final acceptance (runs on the Docker host / CI — out of the implementation env)

- [ ] `make validate` prints OK for all three stacks + the rendered `dns.yml`.
- [ ] After `make prod` on the services host: `https://traefik.int.v2e.sh` loads behind auth with a valid `*.int.v2e.sh` Let's Encrypt cert; `https://dns.int.v2e.sh` shows the Technitium console (auth-gated) via the file-provider route to `10.1.0.53:5380`.
- [ ] Public `lab.v2e.sh` tunnel + `v2e.sh` resolution unaffected; Traefik still issues certs (ACME uses `1.1.1.1`).
- [ ] `grep -rn 'Host(`.*\${DOMAIN}`)' */compose.yml` → no matches (all routers moved).

## PR

Open a PR from `feat/compose-internal-domain` → `main`. Title: `compose: INTERNAL_DOMAIN + internal dashboards + dns console router (DNS-1)`. Note it depends on the tf node (`10.1.0.53`) and pairs with the ansible resolver.

## Self-review checklist

- Spec coverage: `INTERNAL_DOMAIN` var (§4) ✓ T1; router moves (§4) ✓ T2/T4; dual wildcard cert (§6) ✓ T2; `dns.` console router to `10.1.0.53:5380` behind auth (§5, §3) ✓ T3/T5; `1.1.1.1` pin kept (§6) ✓ constraint/T2; file-provider env wrinkle (§12.5) ✓ T3/T5.
- No `${DOMAIN}` left in any router `Host()` rule ✓ T4 grep.
- Generated `traefik/dynamic/dns.yml` gitignored; only the `.tmpl` tracked ✓ T3.
- ACME resolver pin untouched ✓.
