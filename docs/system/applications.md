# Application estate

Every user-facing service in the lab runs as a Docker Compose stack. This page maps
each stack to what it does, where it lives, how you reach it, and whether it is gated
behind authentication.

The stacks are split across two nodes:

- **`services`** (`10.1.2.10`, VLAN 102) — the web app estate. Everything here sits
  behind **Traefik** and, for the sensitive apps, **TinyAuth**. Each app is published as
  `https://<app>.int.v2e.sh` with a production Let's Encrypt **wildcard** cert.
- **`infra`** (`10.1.0.10`, mgmt VLAN 100) — foundational appliances that need the host
  network (DNS, remote-desktop relay, an optional VPN exit). These are **not** fronted by
  Traefik and do **not** live under `int.v2e.sh`; they are reached directly on the LAN or
  over Tailscale.

!!! note "One source of truth"
    Everything on this page is read from `v2e-compose/*/compose.yml`. Hostnames come from
    `INTERNAL_DOMAIN=int.v2e.sh`; the public zone is `DOMAIN=v2e.sh` (ACME + email only).

## At a glance

| App | URL | Node | Purpose | Gated? |
|---|---|---|---|---|
| Traefik dashboard | `traefik.int.v2e.sh` | services | Reverse proxy / TLS terminator; dashboard | **TinyAuth** |
| TinyAuth | `tinyauth.int.v2e.sh` | services | Forward-auth provider (the gate itself) | Public login page |
| whoami | `whoami.int.v2e.sh` | services | Request-echo / connectivity probe | **Open** |
| Semaphore | `semaphore.int.v2e.sh` | services | Ansible/task runner UI (+ Postgres) | **TinyAuth** |
| Arcane | `arcane.int.v2e.sh` | services | Docker stack manager (Dockge-style) | **TinyAuth** |
| Grafana | `grafana.int.v2e.sh` | services | Metrics/logs dashboards | **TinyAuth** |
| Uptime Kuma | `uptime.int.v2e.sh` | services | Uptime/status monitoring | **TinyAuth** |
| Prometheus | *(internal only)* | services | Metrics TSDB — proxied via Grafana | No route |
| Loki | *(internal only)* | services | Log store — proxied via Grafana | No route |
| Alloy | *(internal only)* | services | Log/metric collector → Loki | No route |
| node-exporter | *(host network)* | services | Host metrics for Prometheus | No route |
| cAdvisor | *(internal only)* | services | Container metrics for Prometheus | No route |
| Technitium | `infra:5380` (HTTP) | infra | Internal DNS server + admin console | LAN/tailnet only |
| RustDesk (hbbs/hbbr) | *(host ports)* | infra | Self-hosted remote-desktop relay | Tailnet only |
| Mullvad exit | *(Tailscale exit node)* | infra | Optional Mullvad-tunnelled exit node | Tailnet only |

!!! info "What \"gated\" means"
    A gated app carries the `auth@docker` middleware, which is TinyAuth's forward-auth. A
    request to it is redirected to the TinyAuth login before it ever reaches the app. Open
    apps carry only `secure-headers@docker`. Apps with no Traefik route are unreachable
    over HTTPS at all — they are internal to the Docker network, or exposed on the LAN/
    tailnet outside the proxy.

## How the pieces relate

```mermaid
flowchart TD
    user([Browser / client])

    subgraph services["services node (10.1.2.10)"]
        traefik["Traefik<br/>:80 / :443"]
        tinyauth["TinyAuth<br/>forward-auth"]
        whoami["whoami<br/>(open)"]
        semaphore["Semaphore + Postgres"]
        arcane["Arcane"]
        grafana["Grafana"]
        uptime["Uptime Kuma"]
        prom["Prometheus"]
        loki["Loki"]
        alloy["Alloy"]
        cadvisor["cAdvisor"]
        nodeexp["node-exporter"]
    end

    subgraph infra["infra node (10.1.0.10)"]
        technitium["Technitium DNS"]
        rustdesk["RustDesk relay<br/>hbbs / hbbr"]
        mullvad["Mullvad exit<br/>gluetun + tailscale"]
    end

    user -->|https *.int.v2e.sh| traefik
    traefik -.->|forward-auth| tinyauth
    traefik --> whoami
    traefik -->|gated| semaphore
    traefik -->|gated| arcane
    traefik -->|gated| grafana
    traefik -->|gated| uptime

    grafana --> prom
    grafana --> loki
    prom --> cadvisor
    prom --> nodeexp
    prom -->|:8082 metrics| traefik
    alloy --> loki

    user -->|DNS :53 / console :5380| technitium
    user -->|tailnet| rustdesk
    user -->|exit node| mullvad
```

## The apps

### Traefik — the front door

`traefik:v3.7.6`. Terminates TLS on `:443` (with a `:80 → :443` redirect) and routes by
`Host()` header to every stack on the `frontend` Docker network. It obtains a **wildcard
`*.int.v2e.sh`** certificate over the ACME **DNS-01** challenge via Cloudflare, and carries
two resolvers — `staging` and `production` — selected by `CERT_RESOLVER`.

- Dashboard at `traefik.int.v2e.sh`, **gated** (`auth@docker`) — it also anchors the
  wildcard cert (`tls.domains[0].main` + `*` SAN).
- A reusable `secure-headers` middleware (frame-deny, nosniff, XSS filter, referrer
  policy) is applied to every router.
- Prometheus metrics are exposed on a dedicated container-internal entrypoint `:8082`,
  scraped over the `frontend` network and never published.

!!! warning "DNS-01 and the lab's DNS interceptor"
    Both resolvers set `propagation.delayBeforeChecks=60s` + `disableChecks=true`. The
    lab's WAN path runs a caching DNS interceptor that negative-caches the
    `_acme-challenge` lookup, poisoning any on-node self-check. Traefik sleeps 60s after
    the Cloudflare write and lets Let's Encrypt validate from its own resolvers. Never pin
    `resolvers=1.1.1.1` here.

### TinyAuth — the gate

`ghcr.io/steveiliop56/tinyauth:v5.0.7`. A backend-neutral **forward-auth** provider. It
publishes the `auth` middleware
(`forwardauth.address=http://tinyauth:3000/api/auth/traefik`) that every gated app
references as `auth@docker`. On success it injects `Remote-User` / `Remote-Groups` /
`Remote-Email` / `Remote-Name` headers.

Its own login page at `tinyauth.int.v2e.sh` is **public by design** — a forward-auth
service cannot gate its own login, so that router carries only `secure-headers`. Users are
defined in `TINYAUTH_AUTH_USERS` (SOPS).

### whoami — the open probe

`traefik/whoami:v1.11.0`. Echoes the incoming HTTP request. The one app with **no auth**
(only `secure-headers`), used to confirm routing, TLS, and connectivity end to end. It is
a scratch image with no shell, so it has no container healthcheck — liveness is covered by
Traefik's router state and Uptime Kuma.

### Semaphore — automation UI

`semaphoreui/semaphore:v2.18.14` with a dedicated `postgres:18.4-alpine`. The Ansible/task
runner web UI. **Gated** with TinyAuth in front of Semaphore's own login (defense in
depth).

- Postgres sits on an `internal` network reachable only by Semaphore.
- Semaphore also publishes `:3000` on the **LAN** so its **remote runner** — which lives
  on the `control` node as the `ansible` user, next to the mesh SSH keys — can poll the
  server outbound. The runner cannot come through Traefik (TinyAuth would 401 its API
  calls), so it reaches the LAN-published `:3000` directly. The VyOS forward rule
  (rule 30) permits the whole `control → services` subnet path — it is **not** scoped
  to port 3000 — and Semaphore publishes `:3000` on the services LAN for the runner.
- First-boot admin is bootstrapped as `admin`; `SEMAPHORE_ACCESS_KEY_ENCRYPTION` (SOPS)
  encrypts stored credentials.

### Arcane — Docker manager

`ghcr.io/getarcaneapp/manager:v2.3.0`. A Dockge-style UI to view and manage the running
compose stacks (`PROJECTS_DIRECTORY=/opt/v2e-compose`). **Gated** with TinyAuth in front
of its own login.

!!! warning "Privileged by design"
    Arcane mounts the Docker socket **read-write** (root-equivalent on the `services`
    host) and runs with `cgroup: host`. Its first-boot login is `admin`/`password` — change
    it immediately. Hardening options (docker-socket-proxy, seeded creds) are tracked in
    the security backlog.

### Observability — Grafana, Prometheus, Loki, and friends

A single stack (Phase G). Only two services get Traefik routes; everything else is
internal and surfaced **through Grafana**:

- **Grafana** `grafana:13.0.3` → `grafana.int.v2e.sh`, **gated**. Sign-up disabled;
  admin password from SOPS.
- **Uptime Kuma** `louislam/uptime-kuma:2.4.0-slim` → `uptime.int.v2e.sh`, **gated**. Also
  probes the `infra` appliances (Technitium, RustDesk) over the tailnet.
- **Prometheus** `prom/prometheus:v3.13.0` — metrics TSDB; scrapes cAdvisor,
  node-exporter, and Traefik's `:8082`. No route.
- **Loki** `grafana/loki:3.7.3` + **Alloy** `grafana/alloy:v1.17.1` — log store and
  collector (Alloy tails the Docker socket → Loki). No routes.
- **node-exporter** `v1.11.1` (host network / host PID) and **cAdvisor** `v0.60.3` — host
  and container metrics. No routes.

!!! note "4 GB budget"
    The `services` node is a 4 GB VM, so every container in this stack is memory-capped
    (Loki/Alloy also set `GOMEMLIMIT`), keeping the whole observability stack under a
    ~1.2 GiB hard cap.

### Technitium — internal DNS

`technitium/dns-server:15.2.0` on the **infra** node, **host network**. Owns the node's
`:53` (tcp+udp) and serves its admin console on `:5380`. It is the authoritative resolver
for the `int.v2e.sh` zone (wildcard → `services`, plus per-node A records) and forwards
everything else upstream (`NAME_SERVERS`, default `1.1.1.1, 9.9.9.9`).

It is **not** behind Traefik and not under `int.v2e.sh`. Reach the console directly at
`http://<infra-ip>:5380` from the LAN subnet or over Tailscale.

!!! warning "Plaintext console"
    The `:5380` console is HTTP only, contained to the control subnet + tailnet. Host
    networking is required so per-client DNS rules see real client source IPs.

### RustDesk relay — remote desktop

`rustdesk/rustdesk-server:1.1.15`, two containers — `hbbs` (rendezvous/ID) and `hbbr`
(relay) — on the **infra** node, **host network** (RustDesk spreads across tcp/udp
`21114-21119` + `21116/udp`). **Internal / Tailscale-only**: no WAN DNAT. Clients reach the
`control` desktop via Direct-IP over Tailscale, not through RustDesk's public server. No
web UI, no Traefik route.

### Mullvad exit — optional VPN egress

A **toggleable** stack on the **infra** node — currently **enabled** in
`group_vars/infra.yml` (`compose_stack_stacks` includes `mullvad-exit`).
`qmcgaw/gluetun:v3.41.1` brings up a Mullvad WireGuard tunnel; a `tailscale/tailscale`
sidecar **shares gluetun's network namespace** and advertises itself as an exit node named
`mullvad`. Any tailnet device selecting it egresses through Mullvad.

It is namespace-isolated — Mullvad's WireGuard never touches the host routing table, so it
runs safely beside the DNS server and RustDesk relay. There is no web UI; it appears only
in the Tailscale exit-node picker. The stack is portable and intended to move to a VPS
later.

## Auth model in one line

```mermaid
flowchart LR
    req([Request to *.int.v2e.sh]) --> t{Traefik router}
    t -->|whoami| open[secure-headers only → app]
    t -->|tinyauth| login[public login page]
    t -->|traefik / semaphore / arcane<br/>grafana / uptime| gate{auth@docker}
    gate -->|no session| login
    gate -->|valid session| app2[secure-headers → app's own login]
```

Gated apps stack **two** layers: TinyAuth forward-auth, then the application's own login
(Semaphore, Arcane, Grafana all keep their native auth). whoami is deliberately open, and
the `infra` appliances sit entirely outside this HTTP path.
