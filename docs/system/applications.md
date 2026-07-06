# Application estate

Every user-facing service in the lab runs as a Docker Compose stack. This page maps each
stack to what it does, where it lives, how it is reached, and whether it is gated behind
authentication. It is the reference for the running application surface, grounded in the
compose definitions under `v2e-compose/`.

The estate is split across two nodes, and the split is deliberate:

- The `services` node (`10.1.2.10`, VLAN 102) carries the web application estate.
  Everything here sits behind Traefik and Authelia — either as forward-auth or as a native
  OIDC identity provider, depending on the application. Each
  application is published as `https://<app>.int.v2e.sh` with a Let's Encrypt wildcard
  certificate. The enabled stacks are listed in `v2e-ansible` `group_vars/services.yml`.
- The `infra` node (`10.1.0.10`, management VLAN 100) carries foundational appliances that
  require the host network — DNS, a remote-desktop relay, and an optional VPN exit. These
  are not fronted by Traefik and do not live under `int.v2e.sh`; they are reached directly
  on the LAN or over Tailscale. The enabled stacks are listed in `group_vars/infra.yml`.

Hostnames derive from `INTERNAL_DOMAIN=int.v2e.sh`; the public zone `DOMAIN=v2e.sh` is used
for ACME registration and email only.

## At a glance

| Application | URL | Node | Purpose | Gated? |
|---|---|---|---|---|
| Traefik dashboard | `traefik.int.v2e.sh` | `services` | Reverse proxy / TLS terminator; dashboard | Authelia (forward-auth) |
| Authelia | `auth.int.v2e.sh` | `services` | SSO provider — forward-auth + OIDC IdP (the gate itself) | Public portal |
| whoami | `whoami.int.v2e.sh` | `services` | Request-echo / connectivity probe | Authelia (forward-auth) |
| Semaphore | `semaphore.int.v2e.sh` | `services` | Ansible/task runner UI (+ Postgres) | Authelia (forward-auth) |
| Arcane | `arcane.int.v2e.sh` | `services` | Docker stack manager | Authelia (native OIDC) |
| Grafana | `grafana.int.v2e.sh` | `services` | Metrics/logs dashboards | Authelia (native OIDC) |
| Uptime Kuma | `uptime.int.v2e.sh` | `services` | Uptime/status monitoring | Authelia (forward-auth) |
| Prometheus | *(internal only)* | `services` | Metrics store; surfaced through Grafana | No route |
| Loki | *(internal only)* | `services` | Log store; surfaced through Grafana | No route |
| Alloy | *(internal only)* | `services` | Log/metric collector to Loki | No route |
| node-exporter | *(host network)* | `services` | Host metrics for Prometheus | No route |
| cAdvisor | *(internal only)* | `services` | Container metrics for Prometheus | No route |
| Technitium | `infra:5380` (HTTP) | `infra` | Internal DNS server + admin console | LAN / tailnet only |
| RustDesk (hbbs/hbbr) | *(host ports)* | `infra` | Self-hosted remote-desktop relay | Tailnet only |
| Mullvad exit | *(Tailscale exit node)* | `infra` | Mullvad-tunnelled exit node | Tailnet only |

!!! info "What gated means"
    There are two gate styles. Most gated applications carry the `authelia@docker`
    middleware — Authelia's forward-auth — and an unauthenticated request is redirected to
    the Authelia portal at `auth.int.v2e.sh` before it ever reaches the application. Grafana
    and Arcane instead carry only `secure-headers@docker` at the proxy and are gated
    in-application by native Authelia OIDC (no forward-auth, to avoid double-gating).
    Applications with no Traefik route are unreachable over HTTPS: they are internal to a
    Docker network, or exposed on the LAN or tailnet outside the proxy.

## How the pieces relate

```mermaid
flowchart TD
    user([Browser / client])

    subgraph services["services node (10.1.2.10)"]
        traefik["Traefik<br/>:80 / :443"]
        authelia["Authelia<br/>forward-auth + OIDC"]
        whoami["whoami"]
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
    traefik -.->|forward-auth| authelia
    traefik -->|gated| whoami
    traefik -->|gated| semaphore
    traefik --> arcane
    traefik --> grafana
    traefik -->|gated| uptime
    arcane -.->|OIDC backchannel| authelia
    grafana -.->|OIDC backchannel| authelia

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

## The applications

### Traefik — the front door

Traefik `v3.7.6` terminates TLS on `:443` (with a `:80` to `:443` redirect) and routes by
`Host()` header to every stack on the `frontend` Docker network. It obtains a wildcard
`*.int.v2e.sh` certificate over the ACME DNS-01 challenge via Cloudflare, and carries two
resolvers — `staging` and `production` — selected by `CERT_RESOLVER`.

- The dashboard is published at `traefik.int.v2e.sh` and is gated (`authelia@docker`). Its
  router also anchors the wildcard certificate through `tls.domains[0].main` plus the `*`
  SAN.
- A reusable `secure-headers` middleware (frame-deny, no-sniff, XSS filter, referrer
  policy) is applied to every router.
- Prometheus metrics are exposed on a dedicated container-internal entrypoint `:8082`,
  scraped over the `frontend` network and never published.

!!! warning "DNS-01 and the lab's DNS interceptor"
    Both resolvers set `propagation.delayBeforeChecks=60s` and `disableChecks=true`. The
    lab's WAN path runs a caching DNS interceptor that negative-caches the
    `_acme-challenge` lookup, poisoning any on-node self-check. Traefik sleeps 60s after
    the Cloudflare write and lets Let's Encrypt validate from its own resolvers. Do not pin
    `resolvers=1.1.1.1` here — it falls into the same negative-cache trap.

### Authelia — the gate

Authelia `4.39.20` (`docker.io/authelia/authelia`) is the lab's single SSO gate, replacing
TinyAuth in the 2026-07-06 migration. It plays two roles:

- **Forward-auth** for whoami, the Traefik dashboard, Uptime Kuma, and Semaphore. It
  publishes the `authelia` middleware
  (`forwardauth.address=http://authelia:9091/api/authz/forward-auth`,
  `trustForwardHeader=true`) that those applications reference as `authelia@docker`. On
  success it injects the same `Remote-User`, `Remote-Groups`, `Remote-Email`, and
  `Remote-Name` headers TinyAuth did, so downstream consumers are unchanged.
- **OIDC identity provider** for Grafana and Arcane, which authenticate against Authelia
  directly rather than through forward-auth (see those sections).

Its own portal at `auth.int.v2e.sh` is public by design — a forward-auth service cannot gate
its own login, so that router carries only `secure-headers@docker`. The admin user (`admin`)
and any others are defined in `users_database.yml` with an argon2id hash rendered from SOPS
(`authelia_admin_password_hash`); the RS256 OIDC issuer key and per-client secret hashes are
inlined into a `configuration.yml` rendered out-of-band to `/opt/authelia-secrets` (outside
the compose tree, so the root-running Arcane container cannot read the signing key). Groups
drive roles: `admins`, `arcane-admins`, `grafana-admins`, `grafana-editors`. Sessions are
in-memory with a `one_factor` policy and a 256m `mem_limit` (argon2id is memory-hard) — an
accepted interim posture.

### whoami — the connectivity probe

whoami `v1.11.0` (`traefik/whoami`) echoes the incoming HTTP request, used to confirm
routing, TLS, and connectivity end to end. It was the first application migrated to
`authelia@docker` (the SSO canary), so it is now gated rather than open — an unauthenticated
request redirects `302` to the Authelia portal. It is a scratch image with no shell, so it
carries no container healthcheck; liveness is covered by Traefik's router state and Uptime Kuma.

### Semaphore — automation UI

Semaphore `v2.18.14` (`semaphoreui/semaphore`) runs with a dedicated `postgres:18.4-alpine`
database and provides the Ansible/task runner web UI. It is gated with Authelia forward-auth
in front of Semaphore's own login, giving two layers of authentication.

- Postgres sits on an `internal` network reachable only by Semaphore.
- Semaphore also publishes `:3000` on the LAN so its remote runner — which lives on the
  `control` node as the `ansible` user, next to the mesh SSH keys — can poll the server
  outbound. The runner cannot come through Traefik, since Authelia forward-auth would reject
  its API calls with 401, so it reaches the LAN-published `:3000` directly. The VyOS firewall
  permits the `control` to `services` path; the public hostname stays gated.
- The first-boot admin is bootstrapped as `admin`. `SEMAPHORE_ACCESS_KEY_ENCRYPTION`
  (from SOPS) encrypts stored credentials; rotating it invalidates every stored key.

### Arcane — Docker manager

Arcane `v2.3.0` (`ghcr.io/getarcaneapp/manager`) is a UI to view and manage the running
compose stacks, pointed at the deployed projects via `PROJECTS_DIRECTORY=/opt/v2e-compose`.
It signs in via native Authelia OIDC (`arcane-admins → role_admin`); forward-auth was dropped
from its router to avoid double-gating the OIDC redirect. The local login is kept enabled as
break-glass until OIDC-only is proven.

!!! warning "Privileged by design"
    Arcane reaches the Docker API through a tecnativa `docker-socket-proxy:v0.4.2` sidecar
    (`DOCKER_HOST=tcp://docker-socket-proxy:2375`) rather than bind-mounting the raw socket:
    the proxy mounts the socket read-only, drops all capabilities, enforces a default-deny
    API allowlist, and sits on an internal no-egress network. Arcane still runs with
    `cgroup: host` so it can detect its own container.

### Observability — Grafana, Prometheus, Loki, and exporters

The observability stack is defined in a single `observability/compose.yml`. Only two of its
services get Traefik routes; everything else is internal and surfaced through Grafana.

- Grafana `13.0.3` is published at `grafana.int.v2e.sh`. Its router carries only
  `secure-headers@docker`; sign-in is native Authelia OIDC (generic OAuth, group→role
  mapping: `grafana-admins → GrafanaAdmin`, `grafana-editors → Editor`). The local admin
  password (SOPS `GRAFANA_ADMIN_PASSWORD`) remains as break-glass.
- Uptime Kuma `2.4.0-slim` (`louislam/uptime-kuma`) is published at `uptime.int.v2e.sh` and
  gated by Authelia forward-auth. It also probes the `infra` appliances over the tailnet.
- Prometheus `v3.13.0` is the metrics store; it scrapes cAdvisor, node-exporter, and
  Traefik's `:8082`. No route.
- Loki `3.7.3` and Alloy `v1.17.1` are the log store and collector; Alloy tails the Docker
  socket and ships to Loki. No routes.
- node-exporter `v1.11.1` (host network, host PID) and cAdvisor `v0.60.3` provide host and
  container metrics. No routes.

!!! note "Memory budget"
    The `services` node is a 4 GB VM, so every container in this stack is memory-capped
    (Loki and Alloy also set `GOMEMLIMIT` to force GC ahead of the OOM killer), keeping the
    whole observability stack under a roughly 1.2 GiB hard cap.

### Technitium — internal DNS

Technitium `15.2.0` (`technitium/dns-server`) runs on the `infra` node using the host
network. It owns the node's `:53` (TCP and UDP) and serves its admin console on `:5380`. It
is the authoritative resolver for the `int.v2e.sh` zone (wildcard to `services`, plus
per-node A records) and forwards everything else upstream (`NAME_SERVERS`, default
`1.1.1.1, 9.9.9.9`).

It is not behind Traefik and not under `int.v2e.sh`. Reach the console directly at
`http://<infra-ip>:5380` from the LAN subnet or over Tailscale.

!!! warning "Plaintext console"
    The `:5380` console is HTTP only, contained to the control subnet and tailnet. Host
    networking is required so per-client DNS rules see real client source IPs; port
    publishing would rewrite them.

### RustDesk relay — remote desktop

RustDesk server `1.1.15` (`rustdesk/rustdesk-server`) runs as two containers on the `infra`
node using the host network — `hbbs` (rendezvous/ID) and `hbbr` (relay). RustDesk spreads
across TCP/UDP `21114-21119` plus `21116/udp`, and `hbbs` must see real client source
addresses, so host networking is required. The relay is internal and Tailscale-only: there
is no WAN DNAT. Clients reach the `control` desktop via direct IP over Tailscale rather than
through a public RustDesk server. There is no web UI and no Traefik route.

### Mullvad exit — VPN egress

The Mullvad exit is a portable stack on the `infra` node, enabled in `group_vars/infra.yml`.
gluetun `v3.41.1` (`qmcgaw/gluetun`) brings up a Mullvad WireGuard tunnel, and a Tailscale
`v1.98.4` sidecar shares gluetun's network namespace and advertises itself as an exit node
named `mullvad`. Any tailnet device selecting it egresses through Mullvad.

The stack is namespace-isolated so Mullvad's WireGuard never touches the host routing table,
letting it run safely beside the Technitium DNS server and RustDesk relay. There is no web UI;
the node appears only in the Tailscale exit-node picker, and the compose file is identical to a
standalone VPS deployment given the same environment. See
[Tailscale, exit nodes & DNS](tailscale-dns.md) for the gluetun kill-switch, the Tailscale
CGNAT allowlist, and the full namespace-isolation model.

## Authentication model

```mermaid
flowchart LR
    req([Request to *.int.v2e.sh]) --> t{Traefik router}
    t -->|authelia portal| portal["public portal<br/>(auth.int.v2e.sh)"]
    t -->|"whoami / traefik<br/>semaphore / uptime"| gate{"authelia@docker"}
    gate -->|no session| portal
    gate -->|valid session| app2["secure-headers → app<br/>(semaphore keeps a 2nd login)"]
    t -->|grafana / arcane| oidc["secure-headers → app<br/>(native OIDC to Authelia)"]
    oidc -.->|OIDC backchannel| portal
```

Two gate classes now exist. **Forward-auth** (`authelia@docker`) fronts whoami, the Traefik
dashboard, Uptime Kuma, and Semaphore — of these, only Semaphore keeps a native second login
behind the SSO gate. **Native OIDC** covers Grafana and Arcane: their routers carry only
`secure-headers@docker` and they authenticate against Authelia in-application (one SSO
identity, no double login). whoami is no longer open — it was the SSO-2 canary — and the
`infra` appliances sit entirely outside this HTTP path.

## Related

- [Network, VLANs & firewall](networking.md) — VLAN layout, the VyOS firewall, and the Traefik ingress path.
- [Observability & alerting](observability.md) — the metrics and logging pipeline behind Grafana.
- [Tailscale, exit nodes & DNS](tailscale-dns.md) — the tailnet, Technitium, and exit-node routing.
- [Secrets & SOPS flow](secrets.md) — how SOPS renders the credentials these stacks consume.
