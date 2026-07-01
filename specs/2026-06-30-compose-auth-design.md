# COMPOSE-2 — Auth (TinyAuth forward-auth) — Design Spec

**Date:** 2026-06-30
**Phase:** COMPOSE-2 — see `MASTER-PLAN.md` §5
**Branch:** `feat/compose-auth` · **Integration:** feature branch → **PR** (the `main` ruleset requires PRs)
**Repo:** `v2e-compose` · **Depends on:** COMPOSE-1 (merged) · **Followed by:** COMPOSE-3 (Semaphore + Dockge)
**Status:** design approved — ready for implementation planning (writing-plans).

---

## 1. Goal

Add a **TinyAuth forward-auth** layer to the shipped stack: bring the Traefik **dashboard**
up on `traefik.v2e.sh` **behind authentication**, protect it, leave `whoami` **public** as
the untouched-open-route proof, and keep the middleware layer **backend-neutral** so a later
swap to Authelia + Valkey is a service swap, not a rebuild. This is the master plan's **Q3**
decision (TinyAuth: stateless cookie, no session backend) realised as a Traefik middleware.

---

## 2. Scope

**In scope:**

- A `tinyauth/` stack: the TinyAuth service (login UI on a **public** `tinyauth.v2e.sh`
  router) and the forward-auth middleware definition.
- Bring up the Traefik **dashboard** (`--api.dashboard=true`) on `traefik.v2e.sh`, behind
  the auth middleware + the existing `secure-headers` middleware.
- **Move the wildcard `tls.domains` anchor** from the `whoami` router onto the (permanent)
  dashboard router.
- **Local username/password** auth (bcrypt), user + signing secret from SOPS.
- `whoami` stays **public** (no auth middleware) — the acceptance "unprotected routes
  unaffected" proof.
- Extend `Makefile` + CI + README for the third stack.

**Out of scope / deferred:**

- **TOTP / 2FA** — TinyAuth supports it as an optional per-user field; deferred (addable
  later with no structural change). Broader Access/2FA is COMPOSE-4.
- **OAuth / OIDC** providers — TinyAuth supports them; deferred (env-only addition later).
- **Authelia + Valkey** — the swap target; the middleware naming (§5) keeps it a clean swap.
- The COMPOSE-1.x hardening backlog (socket-proxy, `cap_drop`/read-only rootfs, digest pins,
  HSTS-on-prod) — still deferred; **HSTS stays off** (the dashboard is tested on the staging
  cert first, same footgun as COMPOSE-1).
- Protecting additional services — none exist yet; Semaphore/Dockge arrive in COMPOSE-3 and
  inherit the middleware + SSO cookie for free.

---

## 3. Locked decisions

1. **Local username/password**, TOTP deferred (user decision). One admin user to start;
   more are comma-separated additions to `TINYAUTH_AUTH_USERS`.
2. **Dashboard protected, `whoami` public** (user decision) — directly satisfies the
   acceptance test (one protected route + one open route).
3. **Backend-neutral middleware named `auth`** (not `tinyauth`) — protected routers reference
   `auth@docker`; swapping the backend later is a middleware-address change, not a per-router
   edit. See §5.
4. **Wildcard anchor moves to the dashboard router** — permanent infra, so cert lifecycle no
   longer depends on the `whoami` test service (as flagged in the COMPOSE-1 spec). `whoami`'s
   router becomes plain `tls=true` and serves the wildcard from the store by SNI.
5. **Pin** `ghcr.io/steveiliop56/tinyauth:v5.0.7` (repo is now `tinyauthapp/tinyauth`; image
   path unchanged; v5 is OpenID-certified). Renovate tracks bumps.
6. **Secrets stay in the one SOPS file** (D-1), passed via `sops exec-env` — which also
   sidesteps the bcrypt-`$` escaping problem (Compose does not re-interpolate a substituted
   env value).
7. **PR workflow** — COMPOSE-2 lands via `feat/compose-auth` → PR (the `main` ruleset
   requires PRs; the CI validates the compose files on the PR).

---

## 4. Files changed

```
v2e-compose/
├── tinyauth/
│   └── compose.yml               # NEW — TinyAuth service + login router + `auth` middleware
├── traefik/compose.yml           # MODIFY — +--api.dashboard, dashboard router, move wildcard anchor here
├── whoami/compose.yml            # MODIFY — swap the anchor labels (certResolver+domains) for tls=true; KEEP Host + secure-headers; stays public (no auth)
├── secrets.sops.yaml.example     # MODIFY — add TINYAUTH_SECRET + TINYAUTH_AUTH_USERS
├── Makefile                      # MODIFY — up/prod/down/logs/validate include tinyauth; bootstrap adds secret+user
├── .github/workflows/ci.yml      # MODIFY — validate tinyauth/compose.yml too
└── README.md                     # MODIFY — auth section: create a user, protected vs public, dashboard access
```

No new top-level config files; `.env` is unchanged (`TINYAUTH_APPURL` is derived from the
existing `DOMAIN`).

---

## 5. Architecture & request flow

```
tinyauth  (frontend, :3000)
  ├─ public router  Host(tinyauth.v2e.sh) · websecure · tls   → login UI (NOT behind auth)
  └─ defines middleware  auth@docker
        forwardauth.address=http://tinyauth:3000/api/auth/traefik
        forwardauth.authResponseHeaders=Remote-User,Remote-Groups,Remote-Email,Remote-Name

traefik dashboard  Host(traefik.v2e.sh) · websecure · service=api@internal
        middlewares=auth@docker,secure-headers@docker
        tls.certResolver=${CERT_RESOLVER} · tls.domains: main=${DOMAIN}, sans=*.${DOMAIN}   ← wildcard anchor

whoami   Host(whoami.v2e.sh) · websecure · tls=true · secure-headers · NO auth middleware   ← public
```

**Protected flow:** request `traefik.v2e.sh` → Traefik → `auth` middleware → TinyAuth
`/api/auth/traefik`. No valid cookie → 302 to `tinyauth.v2e.sh` login → user signs in →
TinyAuth sets the session cookie on the **parent domain `.v2e.sh`** → back to the dashboard.

**SSO:** because the cookie is on `.v2e.sh`, one login covers **every** protected
`*.v2e.sh` subdomain — COMPOSE-3's Semaphore/Dockge inherit it by just adding
`middlewares=auth@docker`.

**Public flow:** `whoami.v2e.sh` has no auth middleware → served directly, no prompt.

**Login page is public by necessity** — `tinyauth.v2e.sh` carries no `auth` middleware (it
cannot gate its own login).

---

## 6. Backend-neutral middleware (swap path)

The forward-auth middleware is defined on the TinyAuth container's labels but **named
`auth`**, carrying TinyAuth's `Remote-*` response headers. Protected routers reference
`auth@docker`. To swap to Authelia + Valkey later:

1. Replace the `tinyauth/` stack with `authelia/` (+ `valkey` on the `backend` network from
   COMPOSE-3).
2. Define a middleware **also named `auth`** on the Authelia container, pointing
   `forwardauth.address` at Authelia's `/api/verify?rd=…` endpoint (Authelia uses the same
   `Remote-*` header convention).
3. Protected routers (`middlewares=auth@docker`) are **unchanged**.

That is the "service swap, not a rebuild" the master plan requires.

---

## 7. Secrets & config contract (extends COMPOSE-1)

| Variable | Kind | Home | Used by |
|---|---|---|---|
| `TINYAUTH_SECRET` | **secret** | `secrets.sops.yaml` | TinyAuth cookie signing/encryption (32 chars) |
| `TINYAUTH_AUTH_USERS` | **secret** | `secrets.sops.yaml` | `user:bcrypt` (comma-separated for many) |
| `TINYAUTH_APPURL` | config (derived) | compose (`https://tinyauth.${DOMAIN}`) | TinyAuth base/login URL |

- `TINYAUTH_SECRET`: `openssl rand -hex 16` (32 chars).
- `TINYAUTH_AUTH_USERS`: `docker run --rm -it ghcr.io/steveiliop56/tinyauth:v5.0.7 user create` →
  paste the `user:bcrypt` output.
- Both reach the container via `sops exec-env` (bcrypt `$` safe — not re-interpolated).
- **Impl-time check:** confirm the exact secret var name in v5's prefixed scheme
  (`TINYAUTH_SECRET`); the docs confirm `TINYAUTH_APPURL` + `TINYAUTH_AUTH_USERS`.

`secrets.sops.yaml.example` gains both keys with placeholders + the generation commands in
comments.

---

## 8. Makefile & CI changes

- **`up`/`prod`:** add a third deploy — the `tinyauth` stack under `sops exec-env` (it needs
  `TINYAUTH_SECRET`/`TINYAUTH_AUTH_USERS`); `whoami` still needs no secret.
- **`down`/`logs`:** extend to the `tinyauth` stack. TinyAuth's `${TINYAUTH_*:?}` guards get
  the same dummy-value treatment as `traefik` so `down`/`logs`/`config` don't error on a
  missing secret (the COMPOSE-1 final-review lesson).
- **`validate` + CI:** add `docker compose -f tinyauth/compose.yml config -q` with dummy
  `TINYAUTH_*` values.
- **`bootstrap`:** after seeding `secrets.sops.yaml`, prompt to add `TINYAUTH_SECRET`
  (`openssl rand -hex 16`) and a `TINYAUTH_AUTH_USERS` entry (via the `user create` command).

---

## 9. Testing & acceptance

**Static (local + CI):** `docker compose config` for all **three** stacks (traefik, tinyauth,
whoami) with dummy env; `make validate` and the CI workflow both cover tinyauth.

**Live (run on a Docker host with the real token + user):**
1. `traefik.v2e.sh` → redirected to the TinyAuth login → **wrong** creds rejected → **valid**
   creds → dashboard loads.
2. `whoami.v2e.sh` → loads with **no** prompt (public).
3. A second protected `*.v2e.sh` (or re-hitting the dashboard in a new tab) → **no** re-login
   (the `.v2e.sh` SSO cookie).

**Acceptance (MASTER-PLAN §5):** a protected route prompts for auth and only passes valid
users; unprotected routes unaffected.

---

## 10. Gotchas (→ README)

- **The login host is public.** `tinyauth.v2e.sh` must NOT carry the `auth` middleware, or
  login becomes an infinite redirect.
- **Cookie parent-domain / SSO** depends on all services sharing `*.v2e.sh`. A bare DDNS
  host without real subdomains breaks the shared cookie (TinyAuth doc caveat).
- **HSTS still off** — the dashboard is validated on the staging cert first; HSTS would make
  the staging warning un-bypassable (COMPOSE-1 §11). HSTS ships with the prod-cert hardening.
- **Wildcard anchor moved** — if the dashboard router is ever removed, re-home the anchor;
  do not leave `*.v2e.sh` un-anchored.
- **bcrypt escaping** is a non-issue *only* because the hash arrives via `sops exec-env`; if
  ever hardcoded in a compose file it needs `$$`.

---

## 11. Handoff to COMPOSE-3

COMPOSE-3 (Semaphore + Dockge) protects each new service by adding one label —
`middlewares=auth@docker` — and inherits the `.v2e.sh` SSO cookie. Semaphore's Postgres lands
on the (then-introduced) `backend` `internal: true` network. No auth rework needed.

---

## 12. Sources

- TinyAuth v5 docs (getting-started + advanced), image `ghcr.io/steveiliop56/tinyauth:v5`,
  forward-auth address `http://tinyauth:3000/api/auth/traefik`, `TINYAUTH_APPURL` /
  `TINYAUTH_AUTH_USERS`, parent-domain SSO cookie — tinyauth.app/docs, github.com/tinyauthapp/tinyauth (verified 2026-06-30).
- MASTER-PLAN.md §1 (Q3 TinyAuth), §5 (COMPOSE-2), §9 (decision log).
