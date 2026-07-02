# Solidify the base — hardening plan (2026-07-02)

Goal: take the working lab from "runs until it silently doesn't" to "recoverable,
self-reporting, and drift-resistant". Ordered so each tier protects the one below it.
Status legend: ✅ done · 🔨 in a PR · 📋 planned · ❓ needs a decision.

---

## Tier 0 — finish the tailnet cutover
- ✅ Rename `mullvad-2` → `mullvad` in the console.
- 📋 De-ephemeralize `control`: it still shows the *Ephemeral* badge (force-reauth
  kept the old node-record properties). Fix = one `tailscale logout` + fresh
  `tailscale up` with the tagged key. Low urgency; only bites after a long control outage.
- 🔨 Tailnet ACL policy into git (see "policy-as-code" below).

## Tier 1 — don't lose it (backup/DR — the biggest current gap)
- 📋 **Proxmox vzdump schedule** — daily whole-VM backups of services+infra to local
  storage. Fastest restore path; do this before the fancier file-level backups.
- 📋 **TrueNAS backup role** (HANDOVER backlog H) — ansible systemd-timer: `pg_dump`
  (semaphore) + tar of {technitium zone, acme.json, grafana, uptime-kuma} **plus the
  named docker volumes the original plan missed** (`prometheus-data`, `loki-data`) →
  a TrueNAS share. Include a written `acme.json` restore path (re-issuing certs through
  the lab's DNS interception is painful — restoring beats re-issuing).

## Tier 2 — know when it breaks (alerting + probes)
- 🔨 **Alert-rules pack** (v2e-compose): cert-expiry (<21d warn / <7d crit), disk <15%,
  memory <10%, internal-DNS-down, target-down. Grafana-provisioned, git-tracked.
- 🔨 **Synthetic probes** via `blackbox-exporter` (declarative, not Uptime-Kuma click-ops):
  TLS cert expiry + Technitium DNS resolution today; tailnet node/exit-node liveness
  **gated** on services joining the tailnet (same prerequisite as cross-node metrics).
- ❓ **Notification routing** — see "Apprise?" below.
- ❓ **Cross-node host metrics** (backlog G) — see "How cross-node would work" below.

## Tier 3 — don't get owned
- 🔨 **Alloy log scrubbing** — redact `password=`/`token=`/bearer/`tskey-` before logs
  persist in Loki.
- 📋 **Scoped Cloudflare token (H1)** — see "The CF token" below.
- 📋 **Arcane behind docker-socket-proxy (H2)** — see "Docker security" below.
- 📋 **Encrypt tofu state (C2)** — `TF_ENCRYPTION`, once the final env is set (owner's call).
- 📋 **Restrict the Proxmox snippets datastore** — see "The snippets secrets" below.

## Tier 4 — keep it honest (drift resistance)
- 🔨 **Renovate major-bump guard** — majors now require an explicit tick on the
  Dependency Dashboard before a PR opens (`dependencyDashboardApproval`). No silent
  major merges past the pin-exact policy.
- 📋 **Scheduled `tofu plan` drift check** — nightly Semaphore job, alerts on exit-code 2
  (GitHub CI can't reach Proxmox; the on-control runner can).

---

## Answers to the open questions

### Does the tailnet policy hold personal data?
Effectively no. The policy file (`tagOwners` / `autoApprovers` / `grants` / `ssh`) is
network topology, not PII: tag names, the `10.1.0.0/16` route, and `autogroup:*` refs.
The only identity-ish string is the tailnet **owner handle** (the GitHub-derived
`undefined6799.github` and the `autogroup:admin` that resolves to it) — an account
identifier, not a secret, and already visible in every machine row. So committing the
policy to git leaks nothing that isn't already public in the repo. **Policy-as-code is
safe and worth doing**: today the ACL lives only in the console, unversioned — a fat-finger
has no undo and no audit trail. Plan: keep the canonical copy in `v2e-tf` (or a small
`gitops-acls` workflow using Tailscale's GitHub Action + an API key in SOPS) so changes
are reviewed and revertable like everything else.

### Apprise — should we adopt it now? (yes)
**Recommended.** Right now the alert rules evaluate but route nowhere — a firing alert
sits in Grafana's UI. Apprise is the clean fix: one notification hub that fans out to
80+ targets (ntfy, Telegram, Discord, email, Gotify…) from a single URL list. Two ways
to wire it, pick one:
- **Grafana → Apprise (recommended):** run the `caronc/apprise` API container, add a
  Grafana contact point of type *webhook* pointing at it, set it as the default notification
  policy. All six rules then deliver. ~15 MiB container, one SOPS secret (the target URL).
- **Uptime Kuma → Apprise:** Kuma speaks Apprise natively, but Kuma monitors are
  click-ops (no file provisioning), which fights the IaC ethos — that's exactly why the
  probes above went to blackbox instead. Use Kuma's Apprise only if you also want Kuma's
  status page.
Suggested first target: **ntfy** (self-hostable later, push to your phone, no account).
This is a small dedicated PR once you name the channel.

### How would cross-node host metrics work? (backlog G)
Today only the **services** host is scraped; control/home/infra/agent host metrics are
dark because the VyOS default-deny firewall isolates the spokes and blocks `:9100`.
Two options:
- **(a) Recommended — scrape over the tailnet.** Run `node-exporter` on each node
  (already trivial via ansible), put the **services** host on the tailnet, and have
  Prometheus scrape the `100.x` addresses. No new VyOS holes, no widening of the VLAN
  boundary — the tailnet *is* the cross-node fabric. Bonus: this same step unblocks the
  **tailnet health probes** (blackbox ICMP to control/home/mullvad) — they're gated on
  exactly this. One change, two features.
- **(b) Not recommended — punch VyOS rules** services→{control,home,agent}:9100. Needs a
  router rebuild and widens the security boundary for a metrics convenience.
Decision needed: approve (a). Then it's an ansible change (node-exporter role already
exists) + a `tailscale` join for services + uncommenting the gated Prometheus jobs.

### "The CF token" — what did I mean, is there a better place? (H1)
Traefik does ACME DNS-01 for the `*.int.v2e.sh` wildcard via Cloudflare, using
`CF_DNS_API_TOKEN` in its container env. The problem isn't *where* it lives (env-from-SOPS
is fine) — it's the token's **scope**: today it's the same broad token used for the
Cloudflare **tunnel**, carrying Tunnel + Zone + DNS-edit rights across the account. If that
one container's env leaks, an attacker gets your tunnel and full DNS control. **Fix = least
privilege:** mint a *dedicated* API token scoped to `Zone:DNS:Edit` + `Zone:Read` for the
`v2e.sh` zone only, put it in SOPS as its own key, point Traefik at it, and rotate the broad
one. Same storage, far smaller blast radius. (Longer term, backlog I moves off Cloudflare
entirely to certbot + a privacy DNS provider — but the scoped token is the cheap win now.)

### Docker security — can we improve it? (H2 + more)
Yes, concretely:
- **Arcane's `docker.sock` is mounted read-write** = root-equivalent on the services host.
  Front it with **docker-socket-proxy** (Tecnativa): a tiny sidecar that exposes only the
  API endpoints Arcane needs (containers/images/networks list + the actions you allow) and
  blocks the dangerous ones (`POST /containers/create` with host mounts, `/exec`, etc.).
  Upstream Arcane already ships an example. This is the single biggest container-security
  win and its own PR. ⚠️ Deploy-with-care: too tight a proxy ACL breaks Arcane's management
  UI, so it needs a converge + a functionality check, not a blind apply.
- **Alloy also mounts the socket** — already read-only (`:ro`), good; the socket-proxy
  pattern could cover it too.
- **Explicit `user:` / cap-drop** on the stateless services (traefik, alloy, blackbox) —
  smaller blast radius on escape. Per-service work (volume ownership, traefik's `:443`
  bind needs `cap_net_bind_service`), so a gradual follow-up, not one PR.
- Already good: `no-new-privileges` everywhere, mem/cpu caps, non-root images for the
  stateful services, read-only socket on Alloy.

### "The snippets secrets" — how would we fix it? (novel finding)
`v2e-tf` renders cloud-init via `node.yaml.tftpl` into **Proxmox snippets** on the host's
storage, and those snippets contain the Cloudflare token, SSH keys, the vault password,
and the SOPS age key — **base64-encoded, not encrypted**. Anyone with Proxmox
datastore/console access can read them (sibling of C2, the plaintext tofu state). Two fixes,
increasing effort:
- **Cheap:** restrict the snippets datastore — tighten the Proxmox storage ACL/permissions
  so only the deploy identity can read it, and treat the snippets dir as key material
  (document it, rotate after deploy). Reduces exposure to anyone-with-console.
- **Proper:** stop embedding secrets in cloud-init. First-boot fetches them from control
  over SSH/SOPS (control already holds the age key and runs ansible), so the Proxmox
  storage only ever holds a bootstrap public key, not the secrets themselves. Bigger change
  to the bootstrap flow; schedule with the C2/state-encryption work at final-env setup.

---

## What's shipping in this batch of PRs
1. **v2e-compose** — alert-rules pack + blackbox probes + Alloy log scrubbing (this doc's
   Tier 2/3 🔨 items).
2. **v2e-tf / v2e-ansible / v2e-compose / v2e-templates** — Renovate major-bump guard.
3. This doc.

Everything else above is 📋/❓ and waits for a decision or a follow-up PR.
