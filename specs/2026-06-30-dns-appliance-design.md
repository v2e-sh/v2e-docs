# DNS-1 — Internal DNS Appliance (Technitium) + `*.int.v2e.sh` — Design Spec

**Date:** 2026-06-30
**Phase:** DNS-1 (new) — cross-layer: Packer (reuse) · Terraform · Ansible · Compose
**Repos:** `v2e-tf`, `v2e-ansible`, `v2e-compose` · **Depends on:** Packer Debian template (9002),
COMPOSE-1/2 (Traefik + auth). **Consumed by:** ANS-4 (Tailscale delivers it remotely).
**Status:** rough design — brainstorm + planning done ahead; **implementation in a dedicated conversation.**

> **Scope note:** this phase spans four layers by nature (a DNS appliance is infra, not an app).
> It is intentionally its own phase/conversation. Packer needs no new work — it reuses the
> existing lightweight-Debian template (9002).

---

## 1. Goal

Give every Traefik-fronted dashboard a real name at **`*.int.v2e.sh`**, resolved by a
self-hosted internal resolver, so browsing the lab stops relying on `/etc/hosts`. DNS supplies
the **name** (`*.int.v2e.sh → 10.1.2.10`, the services node running Traefik); **Tailscale
(ANS-4)** supplies the **route** for remote clients; on-lab nodes get it via DHCP.

---

## 2. Locked decisions

1. **Dedicated minimal-Debian VM** for DNS — not a container on the services node. Foundational
   DNS should not ride the app-stack's Docker churn. Spec: **1 vCPU / 512 MB / 8 GB**, from the
   existing Packer Debian template (9002).
2. **Technitium DNS Server**, installed **native (systemd)** — no Docker on a single-purpose
   appliance — configured **via its HTTP API** (idempotent, IaC-friendly). Chosen over Pi-hole/
   AdGuard for real authoritative zones + API + query logging.
3. **Dedicated internal subdomain `int.v2e.sh`** (split by subdomain, not split-horizon on the
   apex). `lab.v2e.sh` is already the Cloudflare tunnel host — `int.` avoids that collision.
4. **`INTERNAL_DOMAIN` is one config variable** (`int.v2e.sh`), single source of truth across
   Compose (`${INTERNAL_DOMAIN}`) and Ansible (`internal_domain`). No hardcoded internal-domain
   strings.
5. **Transport is Tailscale (ANS-4)** for remote browsing; this phase only provides the names +
   documents the Tailscale hook. On-lab clients via VyOS DHCP.
6. **Node placement:** infra IP **`10.1.0.53`** on the mgmt VLAN (100); VyOS routes/firewalls
   `53/udp+tcp` from the node VLANs to it.
7. **Public zone stays on Cloudflare.** Technitium is authoritative *only* for `int.v2e.sh` and
   forwards everything else upstream — so `v2e.sh`/`lab.v2e.sh` and ACME are unaffected.

---

## 3. Architecture & layer responsibilities

```
                          upstream (Cloudflare/Quad9, DoT)
                                     ▲ forwards everything not int.v2e.sh
   lab node ──DNS :53──►  Technitium (dns node 10.1.0.53)
   (resolver via DHCP)      authoritative zone int.v2e.sh:  *.int.v2e.sh → 10.1.2.10
                                     │
   your mac ──Tailscale split-DNS (ANS-4)──► same resolver ─► subnet route 10.1.2.0/24 ─► Traefik
                                                                                              │
                              https://traefik.int.v2e.sh  (wildcard *.int.v2e.sh LE cert, auth)
```

- **Packer** — reuse Debian template 9002. No change.
- **Terraform (`v2e-tf`)** — (a) a `dns` node (VMID `node_vmid_base + 4`, Debian template,
  mgmt VLAN, `10.1.0.53`); (b) a VyOS firewall rule allowing `53/udp+tcp` from control/services/
  agent VLANs → `10.1.0.53`; (c) VyOS DHCP **option 6** = `10.1.0.53` for the node VLANs.
- **Ansible (`v2e-ansible`)** — a `technitium` role: install Technitium (native systemd),
  then drive its HTTP API to create the `${internal_domain}` zone, the wildcard
  `*.${internal_domain} → 10.1.2.10` record, upstream forwarders (DoT), query logging on, and
  (optional) block lists. Idempotent via the API. `internal_domain` is a group_var.
- **Compose (`v2e-compose`)** — introduce `INTERNAL_DOMAIN` and move the dashboards onto it:
  extend the wildcard anchor to also request `*.${INTERNAL_DOMAIN}`; switch the COMPOSE-1/2
  router host rules to `${INTERNAL_DOMAIN}`; add a `dns.${INTERNAL_DOMAIN}` router → the DNS
  node's Technitium console (`10.1.0.53:5380`) behind `auth@docker`.
- **ANS-4 (Tailscale, separate phase)** — split-DNS (`int.v2e.sh` → `10.1.0.53`) + subnet route
  (`10.1.2.0/24`, and the DNS node) so remote clients resolve + reach the dashboards.

---

## 4. The `INTERNAL_DOMAIN` config contract

| Consumer | How it reads the value |
|---|---|
| Compose | `.env` / group_vars → `${INTERNAL_DOMAIN}` in every router/anchor/APPURL label |
| Ansible (Technitium) | group_var `internal_domain` → the authoritative zone name + wildcard record |
| Traefik cert | wildcard anchor `tls.domains` gains `main=${INTERNAL_DOMAIN}`, `sans=*.${INTERNAL_DOMAIN}` |

Default `INTERNAL_DOMAIN=int.v2e.sh`. Changing it (or the whole domain) is a one-line edit in
the config channel; nothing hardcoded.

**Compose router rename (COMPOSE-1/2 → this phase):**
`traefik.${DOMAIN}` → `traefik.${INTERNAL_DOMAIN}`, likewise `tinyauth.` and `whoami.`; new
`dns.${INTERNAL_DOMAIN}`. Because it's a variable swap, the change is mechanical and low-risk.

---

## 5. Technitium configuration (Ansible via API)

- **Zone:** authoritative `int.v2e.sh` (primary).
- **Records:** `*.int.v2e.sh` A → `10.1.2.10`; `int.v2e.sh` A → `10.1.2.10` (apex, optional).
- **Forwarders:** `1.1.1.1` / `9.9.9.9` over **DoT** for everything else.
- **Recursion:** allowed only from the lab VLANs (not open — it's not a public resolver).
- **Logging:** query logging on (feeds the Q1 egress / ANS-6 audit story; agent DNS visibility).
- **Admin console:** `:5380`, reached via Traefik at `dns.int.v2e.sh` behind auth (one of the
  dashboards). The API user/password is a **SOPS secret** (group_vars).

---

## 6. Certificates

Traefik's wildcard anchor (the COMPOSE-2 dashboard router) requests **both** wildcards:
`v2e.sh` + `*.v2e.sh` (existing, for public/tunnel) **and** `*.int.v2e.sh` (new). Cloudflare
DNS-01 issues `*.int.v2e.sh` because `int.v2e.sh` is under the CF-managed zone. COMPOSE-1's
`--dnschallenge.resolvers=1.1.1.1` pin means cert issuance checks the **public** authoritative
answer, not the internal resolver — so the split-horizon resolver never interferes with ACME.

---

## 7. Client pointing

- **Lab nodes:** VyOS DHCP option 6 → `10.1.0.53`. Every node resolves `*.int.v2e.sh` internally
  and forwards the rest. (Alternative: Ansible `resolv.conf`/`systemd-resolved` per node — DHCP
  preferred, one place.)
- **Remote (you):** **ANS-4** — Tailscale split-DNS routes `int.v2e.sh` to `10.1.0.53` and a
  subnet route exposes `10.1.2.0/24`; the mac browses `traefik.int.v2e.sh` over the tailnet. This
  phase does not implement Tailscale; it documents the required split-DNS + route.

---

## 8. Testing & acceptance

- **From a lab node:** `dig traefik.int.v2e.sh @10.1.0.53` → `10.1.2.10`; `dig google.com
  @10.1.0.53` → resolves (forwarded); recursion from a non-lab source is refused.
- **Browser (on-lan, or remote via Tailscale once ANS-4 lands):** `https://traefik.int.v2e.sh`
  loads behind auth with a valid **`*.int.v2e.sh`** Let's Encrypt cert; `dns.int.v2e.sh` shows
  the Technitium console (auth-gated).
- **Regression:** public `lab.v2e.sh` tunnel + `v2e.sh` resolution unaffected; Traefik still
  issues certs (ACME uses `1.1.1.1`).
- **Idempotence:** re-running the `technitium` role via the API changes nothing.

---

## 9. Gotchas

- **`lab.v2e.sh` is taken** (CF tunnel) — internal subdomain is `int.`, never `lab.`.
- **Host `:53` is not a problem here** — the dedicated appliance owns port 53; the systemd-
  resolved conflict that a container-on-services would hit is avoided by construction.
- **Don't make it an open resolver** — restrict recursion/queries to the lab VLANs.
- **ACME depends on the `1.1.1.1` pin** already set in COMPOSE-1 — keep it.
- **Session/agent egress:** routing the agent node's DNS through Technitium gives query
  visibility (Q1/ANS-6) but also means the agent's `deny-by-default` egress must permit
  `10.1.0.53:53` — coordinate with TF-3 / Q1.

---

## 10. Master-plan placement

Add **DNS-1** as a new phase spanning `v2e-tf` (node + VyOS firewall/DHCP), `v2e-ansible`
(`technitium` role), and `v2e-compose` (`INTERNAL_DOMAIN` + `.int` routers + cert). It slots
after COMPOSE-2 and pairs with **ANS-4** (which delivers it remotely). Decision **Q5** (Tailscale
node scope) should include the DNS node's reachability.

---

## 12. Implementation decisions (resolved during planning, 2026-07-01)

Resolved against the actual repo state before implementation. Three PRs land in order
`v2e-tf` → `v2e-ansible` → `v2e-compose` (each repo has a require-PR ruleset on `main`).

1. **Client pointing = DHCP option 6 only.** The server nodes take static DNS
   (`1.1.1.1/9.9.9.9`) from cloud-init, which overrides DHCP, and there is no VyOS DHCP
   server today. Add a VyOS DHCP server handing `option 6 = 10.1.0.53` to the node VLANs
   (the spec item, and the "one place" for future lab-attached clients) but **do not**
   repoint the static server nodes — that avoids a first-boot / ACME chicken-and-egg on the
   DNS node. Remote browsing stays on Tailscale (ANS-4). `dig @10.1.0.53` satisfies
   acceptance regardless of DHCP.
2. **TF per-node overrides (wrinkle).** All nodes currently share global `node_cores/
   node_memory/node_disk_size` and one `node_host_octet` (which would place `dns` at
   `10.1.0.10`). Extend the `local.nodes` schema with *optional* per-node `host_octet`,
   `cores`, `memory`, `disk` that fall back to the existing globals, so `dns` gets
   `host_octet=53` + `1 vCPU / 512 MB / 8 GB` while control/services/agent stay
   byte-identical to today.
3. **Technitium install = official `install.sh`**, guarded by a `creates`/systemd status
   check for idempotence; API configured idempotently with `ansible.builtin.uri`
   (GET-before-POST), admin password bootstrapped off the default and stored in SOPS
   (`group_vars/dns/secrets.sops.yaml`) — matching the master-plan SOPS direction.
4. **Compose external backend = Traefik file provider.** `dns.${INTERNAL_DOMAIN}` routes to
   the non-container console at `10.1.0.53:5380` via a file-provider dynamic config, since
   Traefik currently runs the docker provider only.
5. **File-provider env substitution (wrinkle).** Traefik's file provider does not expand env
   vars, so a static dynamic file would hardcode the internal domain and break the
   single-`INTERNAL_DOMAIN`-variable contract. Keep `traefik/dynamic/dns.yml.tmpl` and have
   the Makefile `envsubst` it to `dns.yml` (using `INTERNAL_DOMAIN` + `DNS_ADMIN_ADDR`)
   before `up`/`prod`. The router references `auth@docker` + `secure-headers@docker`
   cross-provider.

---

## 11. Sources

- Technitium DNS Server — native install + HTTP API + authoritative zones/forwarders/logging;
  footprint ~256–512 MB RAM. technitium.com / github.com/TechnitiumSoftware/DnsServer.
- Topology from `v2e-tf/network.tf` (VLANs 100–103, nodes `.10`, services `10.1.2.10`).
- MASTER-PLAN.md §5 (COMPOSE-2, ANS-4), §2 (Q1 egress, Q5 Tailscale scope), Appendix B (DNS-01).
