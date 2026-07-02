# v2e — Privacy-Hardening Roadmap (2026-07-02)

From a fan-out privacy audit across seven surfaces (DNS, TLS/CA, egress/exit, public
surface, telemetry, logs/data-at-rest, third-party dependencies). Ordered by
privacy-gain per effort. Phase tags: **now** (no new infra), **with-vps** (needs the
planned VPS), **later**.

The theme: **shrink the third-party footprint** — every external service (Cloudflare,
Let's Encrypt/CT, Tailscale coordination, GitHub, Docker Hub, public resolvers) sees
some metadata; each item below removes or self-hosts one.

---

## Quick wins (now, high gain / low effort)

- **Scope the Cloudflare DNS-01 token** to `Zone:DNS:Edit+Read` on `v2e.sh` only; rotate
  out the current broad Tunnel+DNS+Zone token (HANDOVER §5 H1).
- **Fix the agent DNS firewall rule** — agent is blocked from Technitium (VyOS rules
  34-35) and silently falls back to plaintext `1.1.1.1`/`9.9.9.9`. Permit
  `agent → 10.1.0.10:53`.
- **Change upstream resolvers** off Cloudflare/Quad9 → Mullvad DNS (`194.242.2.2` /
  `194.242.2.4`) or a self-hosted Unbound on `home`.
- **Disable browser DoH** on the control desktop (else it bypasses Technitium split-DNS
  and queries Cloudflare directly for internal names).
- **WHOIS privacy** on the `v2e.sh` registration.
- **Move the age keys** (`keys.txt`, `keys-backup.txt`) out of the repo tree to airgapped
  storage; **enable `TF_ENCRYPTION`** for tfstate (both HANDOVER §5).
- **Disable Uptime-Kuma telemetry** (`UPTIME_KUMA_DISABLE_STATS=true`).
- **Trim log retention** (Loki 14→7d, Prometheus 15→7d).

---

## Phase: now (no new infrastructure)

| Gain | Effort | Step |
|---|---|---|
| high | med | **Implement the full Technitium role** (API: forwarders, **DoT** `forwarderProtocol=Tls` to `…:853`, recursion ACL `UseSpecifiedNetworks`, query-log policy). Closes the DNS-privacy + open-resolver gaps at once. |
| high | med | **Alloy log scrubbing** — regex-relabel to drop `password=`/`key=`/`token=` lines and redact internal IPs before shipping to Loki (alloy ships *all* container logs today). |
| high | low | **Agent DNS firewall fix** (quick win above). |
| high | med | **Encrypt Traefik's `acme.json`** (wildcard private key) at the filesystem level. |
| med | low | **Scoped Cloudflare token** + rotate the broad one. |
| med | low | **Mullvad/Unbound upstream resolvers** instead of Cloudflare/Quad9. |
| med | med | **docker-socket-proxy** in front of Arcane (HANDOVER §5 H2). |
| high | high | **Evaluate making the GitHub repos private** (or publish a redacted mirror) — they currently expose the whole architecture: IP plan, firewall rules, `int.v2e.sh` names, DNAT port, usernames. Biggest single opsec item. |
| med | low | **Drop the Cloudflare tunnel** (`lab.v2e.sh`) — SSH already works via the `:2201` DNAT and Tailscale; the tunnel proxies SSH metadata through Cloudflare. |

## Phase: with-vps

| Gain | Effort | Step |
|---|---|---|
| high | med | **Mullvad exit node** on the VPS (the `mullvad-exit` stack, already built — moves off `home`). |
| high | high | **Replace the Cloudflare tunnel** with self-hosted remote access on the VPS (WireGuard / Pangolin / Tailscale-Funnel-via-Headscale); flip `lab.v2e.sh` to DNS-only. |
| high | med | **Headscale** (self-hosted Tailscale control plane) on the VPS — removes Tailscale-the-SaaS from seeing the tailnet graph; point nodes via `--login-server`. |
| med | med | **Private container registry** (Harbor/`registry:2`) mirroring Docker Hub/GHCR — stops per-pull metadata to the public registries. |
| med | med | **Encrypted backups to TrueNAS** — age-encrypt (separate backup key) before send; retention + tested restore. Pairs with backlog H. |
| med | high | **Self-host git** (Forgejo/Gitea) on the VPS; keep GitHub as a mirror-only pull URL. |

## Phase: later

| Gain | Effort | Step |
|---|---|---|
| high | high | **step-ca internal CA** for `*.int.v2e.sh` — issue internal certs locally, **no Certificate Transparency exposure** of internal hostnames. Distribute the root to client trust stores. This is the endgame of the "off Cloudflare / off public CT" move. |
| high | high | **Agent egress kill-switch** — systemd unit that cuts agent WAN if the exit/Tailscale drops (pairs with the restricted-egress design). |
| med | high | **Encrypt service/DB volumes at rest** (LUKS/ZFS) before production. |

---

## How this ties to your stated moves

- **Off Cloudflare → certbot** = the "now" scoped-token + drop-tunnel steps, then the
  with-vps self-hosted-remote-access + step-ca endgame (internal certs stop leaking to
  public CT logs — the deepest privacy win of the migration).
- **TrueNAS redundant backups** = backlog H + the with-vps encrypted-backup step.
- **Mullvad exit** = built (`mullvad-exit`), moves to the VPS.
- **Most impactful single item that needs a decision:** whether the public GitHub repos
  should stay public — they currently document the entire lab.

Full per-surface findings are in the audit run; this is the deduped, prioritized view.
