# v2e — Handover / State of the Lab (2026-07-06)

> **2026-07-06:** the auth layer migrated from TinyAuth to **Authelia 4.39.20** — Authelia
> is now the sole SSO gate (forward-auth for whoami/traefik/uptime/semaphore, native OIDC
> for Grafana and Arcane). References below reflect that change.

Single source of truth for picking the project back up. Everything below is **live and
verified** unless marked otherwise. Secrets are **not** in this file — real values live in
`v2e-tf/.deploy-creds.txt` (local, gitignored). **All current secrets are throwaway/test —
rotate everything at real launch** (age key, Cloudflare token, every password).

---

## 1. Current status — what's running

Five VMs on the Proxmox host `pve` (`10.10.10.62`), all up:

| VMID | Node | OS | vCPU/RAM | IP (VLAN) | Role |
|---|---|---|---|---|---|
| 310 | router | VyOS | 1 / 1G | 10.1.x.1 gateways | Router, DNAT, default-deny firewall |
| 311 | control | ParrotOS (UEFI desktop) | 4 / 8G | 10.1.1.10 (101) | Mesh hub, Ansible controller, Tailscale subnet router, RustDesk target |
| 312 | services | Ubuntu | 4 / 4G | 10.1.2.10 (102) | Docker app estate |
| 313 | agent | Debian | 4 / 4G | 10.1.3.10 (103) | AI agent node (restricted egress) |
| 314 | infra | Debian | 2 / 2G | 10.1.0.10 (100/mgmt) | Technitium DNS + RustDesk relay (Docker) |

**services stacks** (all healthy, behind Traefik + **Authelia 4.39.20**, on `int.v2e.sh`,
**production** Let's Encrypt wildcard cert): traefik, authelia, whoami, semaphore (+postgres),
arcane (+ docker-socket-proxy), observability (prometheus, grafana, loki, alloy, uptime-kuma,
node-exporter, cadvisor).

**infra stacks**: technitium (DNS, `int.v2e.sh` zone), rustdesk (hbbs/hbbr relay).

App URLs (internal, trusted cert): `https://<app>.int.v2e.sh`. Auth portal is
`https://auth.int.v2e.sh` (admin user `admin`). `whoami`, `uptime`, `semaphore` and `traefik`
are gated by Authelia forward-auth (the `authelia@docker` middleware — whoami is no longer open,
an unauthenticated hit now redirects `302` to the portal); `grafana` and `arcane` sign in via
native Authelia OIDC (their routers carry only `secure-headers@docker`, and pin
`auth.int.v2e.sh → 10.1.2.10` via `extra_hosts` for the OIDC backchannel). Groups drive roles:
`admins` / `arcane-admins` / `grafana-admins` / `grafana-editors`.

---

## 2. Access — how to reach it (no AI needed)

- **Proxmox host**: `ssh root@10.10.10.62` (passwordless from the workstation). Web UI `https://10.10.10.62:8006`.
- **Control (SSH), two paths**:
  - via PVE jump: `ssh -J root@10.10.10.62 -p 2201 v2e@10.11.10.25`
  - via Cloudflare tunnel: `ssh -o ProxyCommand="cloudflared access ssh --hostname lab.v2e.sh" v2e@lab.v2e.sh`
  - `v2e` login/sudo password = `sudo_password` (tfvars / `.deploy-creds.txt`). SSH is key-only.
- **Mesh from control** (keys live only on control): `ssh services|agent|vyos` as `v2e`; `sudo -iu ansible` then same for the automation mesh (NOPASSWD sudo).
- **Browser apps from the mac**: mac is on the Tailnet; control is a subnet router advertising
  `10.1.0.0/16`, and Tailscale split-DNS sends `int.v2e.sh` → Technitium (`10.1.0.10`). If the
  mac stops resolving `int.v2e.sh`, the reliable fix is a native resolver file:
  `echo 'nameserver 10.1.0.10' | sudo tee /etc/resolver/int.v2e.sh && sudo killall -HUP mDNSResponder`.
  **Caveat**: while Mullvad VPN is connected it hijacks DNS and this breaks — disconnect Mullvad, or use the Tailscale Mullvad exit-node add-on (see backlog).
- **RustDesk to the control desktop**: RustDesk client → **Direct IP Access** → `100.90.7.86`
  (control's Tailscale IP), password in `.deploy-creds.txt`. **Do not use the ID** (`285510431`) —
  that hits RustDesk's public server which now requires login; Direct-IP over Tailscale uses no
  server. Codec H264 / Quality Best for good quality (see §6 gotchas).

---

## 3. Repos — all on `main`, current

`github.com/v2e-sh/{v2e-tf, v2e-ansible, v2e-compose, v2e-templates, v2e-docs}` (public).

- **v2e-tf** — OpenTofu: router + 4 nodes, cloud-init, VyOS firewall. Deploy entry point.
- **v2e-ansible** — first-boot config: `site.yml` imports `01-bootstrap` → `02-services` →
  `03-applications` → `04-infra` → `05-tailscale` → `06-control-desktop`. Control clones + runs
  this at first boot.
- **v2e-compose** — docker stacks (traefik/authelia/whoami/semaphore/arcane/observability/technitium/rustdesk).
- **v2e-templates** — Proxmox image build (run on the PVE host; `bash build-{ubuntu,debian,vyos,parrot}.sh`; staging VMIDs 9900-9903).
- **v2e-docs** — this file, `RUNBOOK.md`, `CONFIGURATION.md`, `MASTER-PLAN.md`, `specs/`, `plans/`, `RUNBOOK-DEVIATIONS-2026-07-01.md`.

Deploy flow: build templates (host) → set variables → `tofu apply` (v2e-tf) → control auto-runs
v2e-ansible → stacks come up. Walk-through: `RUNBOOK.md`; every variable: `CONFIGURATION.md`.

---

## 4. Outstanding: live config NOT yet in IaC (drift — top priority)

These were configured **by hand this session** and must become ansible roles + variables (owner's
rule: *every configurable thing = a variable, auto-applied*).

> **STATUS 2026-07-02: DONE — merged (v2e-ansible #13 + #14) and validated live.**
> Full clean rebuild converges unattended (cloud-init done, errors: []); re-converge
> on existing nodes also proven (#14 fixed the git-ownership/Parrot-tailscale/
> broken-conditional re-run bugs). Tailscale is JOINED with a reusable key in SOPS
> (control advertises 10.1.0.0/16; infra on the tailnet). The route + split-DNS
> `int.v2e.sh → 10.1.0.10` are now approved and live in the Tailscale admin console, and
> `rustdesk_client_version` is pinned to `1.4.8` + sha256 in
> `inventory/group_vars/control.yml`. Docs are browsable
> at <https://v2e-sh.github.io/v2e-docs/>. The original mapping (implemented):

| Live thing | Proposed | Secrets (SOPS) | Non-secret (group_vars) |
|---|---|---|---|
| **Tailscale** (control+infra: `tailscale up`, subnet routes, `--ssh`) — ANS-4, not implemented; `artis3n.tailscale` is pinned+vendored but unreferenced | `roles/tailscale` (wraps artis3n) + `playbooks/05-tailscale.yml` | `tailscale_authkey` (use a **reusable/tagged** key, not ephemeral) | `tailscale_advertise_routes`, `tailscale_advertise_exit_node`, `tailscale_args` per node |
| **Technitium** (Docker on infra, zone `int.v2e.sh` + wildcard→services + node A records, admin pw) — set via API by hand | reuse `compose_stack` via `group_vars/infra.yml` + a `technitium_zone` API task | `technitium_admin_password` | `internal_domain`, `technitium_wildcard_target: 10.1.2.10`, forwarders |
| **RustDesk client on control** (.deb install, unattended pw, direct-IP) | `roles/rustdesk_client` | `rustdesk_unattended_password`, `rustdesk_key` | relay host, direct-IP toggle |
| **Control desktop** (lightdm autologin=v2e, KDE compositor off, resolv.conf→Technitium, /etc/hosts shim) | `roles/control_desktop` | — | `desktop_autologin_user`, `resolv_nameservers`, compositor toggle |

**env.j2 secret rendering — DONE**: `roles/compose_stack/templates/env.j2` now renders
`TECHNITIUM_ADMIN_PASSWORD` / `NAME_SERVERS` (lines 49–50, password with the
`| replace('$','$$')` escape). The TinyAuth secrets were removed in the SSO migration and
replaced with the `AUTHELIA_*` set (`AUTHELIA_SESSION_SECRET`,
`AUTHELIA_STORAGE_ENCRYPTION_KEY`, `AUTHELIA_RESET_PASSWORD_JWT_SECRET`,
`AUTHELIA_OIDC_HMAC_SECRET`) plus the plaintext OIDC client secrets for the RP apps
(`ARCANE_OIDC_CLIENT_SECRET`, `GRAFANA_OIDC_CLIENT_SECRET`). Each secret block is now scoped
by `{% if '<stack>' in compose_stack_stacks %}`, so a host only renders the secrets for the
stacks it runs.

---

## 5. Security findings (audit 2026-07-02, prioritized)

- **C1 — age key + creds un-gitignored** — ✅ FIXED (v2e-tf #14). Age key (`keys.txt`), backup, and
  `.deploy-creds.txt` are now gitignored. Still: move the age key out of the repo tree and rotate at launch.
- **C2 — OpenTofu state is plaintext** and contains the mesh SSH keys, Cloudflare token, sudo
  password, and the age key. State is gitignored but any copy = full compromise. **Fix: enable
  `TF_ENCRYPTION`** (RUNBOOK Appendix B) before real launch; treat `.tfstate*` as key material.
- **H1 — Cloudflare DNS-01 token is the reused broad tunnel token** (Tunnel+DNS+Zone), and it lives
  in Traefik's `.env` on services. **Fix: mint a dedicated Zone:DNS:Edit+Read token for `v2e.sh` only.**
- **H2 — Arcane `docker.sock` RW** — ✅ FIXED (v2e-compose #34). Arcane no longer mounts the
  socket; it reaches Docker via `DOCKER_HOST=tcp://docker-socket-proxy:2375` (tecnativa
  `docker-socket-proxy:v0.4.2`, socket mounted `:ro`, `cap_drop: ALL`, default-deny API allowlist,
  on an internal no-egress network).
- **H3 — Arcane local `admin`/`password`** — superseded by the SSO migration (v2e-compose #34):
  Arcane now signs in via native Authelia OIDC (`arcane-admins → role_admin`). Residual: the local
  login is kept ENABLED as break-glass — disable/rotate it in the UI once OIDC-only is proven.
- **M1** two always-on SSH ingress paths to control (DNAT 2201 + tunnel), Access OTP off by default —
  pick one / gate behind `trusted_mgmt_sources`. **M2** ephemeral Tailscale key (drops on reboot) →
  ANS-4 with reusable key. **M3** ParrotOS baked `user` account still on control (console/RustDesk
  surface) → `userdel`/lock in the image build. **M4** Technitium console `:5380` is plaintext HTTP
  (contained to control subnet + tailnet). **M5** ansible mesh key = root everywhere + router (protect
  the hub + state).
- Verified good: default-deny firewall + agent egress allowlist, `no-new-privileges` everywhere,
  key-only SSH, correct SOPS `$$`-escaping, encrypted `secrets.sops.yaml`, `acme.json` 0600, no keys baked into images.

---

## 6. Key gotchas (hard-won this session — don't relearn)

- **docker compose interpolates `$` in env-file values** → bcrypt hashes / tokens mangle. `env.j2`
  `$$`-escapes every secret. Any new secret with `$` needs it.
- **Lab WAN path intercepts `:53`** with a caching resolver → ACME DNS-01 self-checks self-poison.
  Traefik uses `propagation.delayBeforeChecks=60s` + `disableChecks=true` (let LE validate). Never
  pin `resolvers=1.1.1.1`.
- **RustDesk on Linux needs an active X11 session** — control uses lightdm autologin=v2e +
  `autologin-session=plasmax11`; set `--password` as root in the session context.
- **Semaphore remote runner**: needs `private_key_file` in config + `is_default:true` (untagged
  tasks only dispatch to default runners). Match runner binary to server version (`docker cp` from the image).
- **Postgres 18 image** mounts `/var/lib/postgresql` (parent), not `.../data`.
- **Parrot codename `echo`** isn't served by cloudflared's apt repo → use `any` suite; cloudflared
  install is best-effort in runcmd so it can't break the node.
- **Tailscale split-DNS** on the mac is flaky to auto-apply and **Mullvad hijacks DNS** when connected.
- Reaching control after a rebuild: `ssh-keygen -R lab.v2e.sh` (host key changes).
- See `RUNBOOK-DEVIATIONS-2026-07-01.md` and the `memory/` notes for the full detail.

---

## 7. Backlog / future work (prioritized)

**A. Codify live config into roles+variables** (§4) — ✅ DONE (merged + live). The reusable
`tailscale_authkey` is in SOPS, `rustdesk_client_version` + sha256 are pinned
(`inventory/group_vars/control.yml`), and the route/split-DNS are approved in the Tailscale
console. A clean rebuild reproduces everything.

**B. Security hardening for launch**: C2 (encrypt state), H1 (dedicated DNS-01 token), H2 (arcane
socket-proxy), H3 (arcane seeded creds), rotate all secrets + age key, M1-M5.

**A2. Code-review sweep 2026-07-02 — DONE (merged + live)**: 43 findings triaged/implemented/
adversarially-verified. Landed: tf plan-time validation (v2e-tf #15), technitium API retries +
per-group secret scoping + locale/NM fixes (v2e-ansible #18), mem/cpu limits + healthchecks on
every stack (v2e-compose #12, deployed — all containers healthy), template `.part` cleanup + a
`set -e` bug (v2e-templates #4). Rejected wrong ones (ai-identities re-scope, loki schema-date
edit, postgres mount). **Two escalated, need YOUR call (below G/H).**

**G. Cross-node host metrics** (review compose#2): Prometheus (on services) can't scrape
node-exporter on the isolated spokes — the VyOS firewall isolates spokes by design. Options:
(a) **[recommended]** scrape over the tailnet — node-exporter on `home`+`control` (both on the
tailnet), put services on the tailnet too, scrape 100.x IPs → no VLAN holes; (b) narrow VyOS
rules services→{home,agent,control}:9100 → needs a router rebuild + widens the boundary. `agent`
is lowest priority (restricted AI node). Today: services host + all containers ARE monitored
(node-exporter + cadvisor + app metrics); only other VMs' host metrics are dark.

**H. Backup/DR** (review compose#6): **target = the user's TrueNAS** (redundant databases + files).
Plan: an ansible systemd-timer role → pg_dump (semaphore) + tar of {technitium zone, arcane data,
acme.json, grafana, uptime-kuma, **Authelia** (`authelia_authelia-data` volume — the OIDC/consent/
user SQLite state — plus `users_database.yml`)} → push to a TrueNAS share (NFS/SMB mount or
`restic`/`rsync` to a TrueNAS dataset). Authelia is now the single IdP, so losing its state = an
SSO outage for every gated app — it must be in scope. Add TrueNAS as an offsite/redundant copy
(3-2-1). Do once any lab state stops being throwaway; wire the TrueNAS mount as a variable.

**I. De-Cloudflare / privacy migration** (user direction, needs the VPS first): move off Cloudflare
toward certbot to shrink the third-party footprint. Steps once a VPS exists: run ACME via certbot
(HTTP-01 on the VPS, or DNS-01 with a non-Cloudflare privacy DNS provider); retire the Cloudflare
tunnel + broad DNS token; consider a self-hosted internal CA (step-ca) for `*.int.v2e.sh` so
internal hostnames stop leaking into public CT logs. See the privacy audit output for the full
roadmap. Related: the Mullvad exit (below) and headscale (self-hosted Tailscale control plane).

**J. Mullvad exit node**: scaffold merged (v2e-compose `mullvad-exit`, gluetun + tailscale sidecar);
Mullvad WireGuard creds now in SOPS (`mullvad_wireguard_private_key`/`_addresses` — throwaway, from
the AT key generated 2026-07-02, rotate at prod). Enable = uncomment `mullvad-exit` in
group_vars/infra.yml + refresh control's secrets + converge + approve "mullvad" in the TS console.
Moves to a VPS later (stack is portable via .env).

**C. Access polish**: reusable/tagged Tailscale key (survives reboot); decide exit-node + Mullvad
(paid Tailscale-Mullvad exit, or DIY Mullvad on an exit node — see the exit-node discussion); the
mac `/etc/resolver/int.v2e.sh` or a Tailscale exit node so no per-service config is needed.

**D. Docs (prod-ready)**:
- **D1** ✅ DONE — `RUNBOOK.md` rewritten straightforward (prereqs → variables → deploy → verify → access), 5-node reality, specs cross-referenced, manual post-deploy steps called out (Step 8).
- **D2** ✅ DONE — `CONFIGURATION.md`: every variable (tfvars, SOPS, group_vars, config.env) with how-to-generate + copy-paste `secrets.yaml` template.
- **D3** reconcile the stale `plans/2026-07-01-dns-1-ansible-technitium.md` (describes a `dns` node @ `.53` native — reality is Docker on `infra` @ `.10`).
- **D4** docs PR #2 (full-stack-runbook) superseded by the D1 rewrite — closed. PR #3 (master-plan-reconcile) still open: review vs current reality, merge or close.

**E. Remaining MASTER-PLAN phases**: COMPOSE-4 (Cloudflare tunnel) — 2FA is now an Authelia
`two_factor` policy toggle (currently `one_factor`, deferred by design), no longer a TinyAuth/
tunnel deliverable; Phase G alerting rules are merged + live (six rules in
`observability/config/grafana-alerting.yaml`) but route nowhere — remaining work is
contact-point/notification delivery; Phase H (GHCR images), agent/MCP (Phase I), DR/backup (DOCS-2).

**F. Cleanup**: prune stale git branches (scope grown post-SSO — ~16 merged SSO feature branches
in v2e-compose plus stale remotes in v2e-tf (4), v2e-ansible (2), v2e-docs (2); verify merged
first with `gh pr list --state merged`); ParrotOS baked `user` removal; browser-DoH breaks
`int.v2e.sh` on the control desktop (disable DoH or set browser to system DNS).

---

## 8. Where the credentials are

`v2e-tf/.deploy-creds.txt` (local, gitignored) holds: v2e sudo/login, root, the Authelia `admin`
portal password (+ the `authelia_*` session/storage/JWT/HMAC secrets, RS256 issuer key and OIDC
client secrets live in SOPS `secrets.sops.yaml`), Grafana, Semaphore, Technitium admin, RustDesk
unattended, and the Cloudflare/Proxmox tokens are in
`v2e-tf/terraform.tfvars`. SOPS secrets are in `v2e-tf/secrets.sops.yaml` (decrypt:
`SOPS_AGE_KEY_FILE=keys.txt sops -d secrets.sops.yaml`). **Rotate all of it for production.**
</content>
