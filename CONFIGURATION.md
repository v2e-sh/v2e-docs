# v2e — Configuration Reference

Every variable a deploy can set, in one place: what it does, its default, and how to
obtain or generate the value. The walk-through that consumes these is
[`RUNBOOK.md`](RUNBOOK.md).

Configuration lives in four places:

| File | Repo | Holds | Secret? |
|---|---|---|---|
| `terraform.tfvars` | v2e-tf (gitignored) | Infrastructure: Proxmox, network, sizing, users, tunnel | Partly (tokens, passwords) |
| `secrets.sops.yaml` | v2e-tf (age-encrypted) | App secrets, shipped to control → Ansible → compose `.env` | Yes (encrypted at rest) |
| `inventory/group_vars/*.yml` | v2e-ansible (committed) | App-layer choices: domain, stacks, hardening knobs | No |
| `config.env` | v2e-templates (committed) | Template build: storage, bridge, VMIDs, image URLs | No |

Owner's rule: **every configurable thing is a variable** — if you find yourself
editing a template or compose file to change a value, that's a bug; file it.

---

## 1. terraform.tfvars — infrastructure

Only the first group is required. Everything else has a sane default; set it only to
deviate. (Source of truth: `v2e-tf/variables.tf`; a commented skeleton is
`terraform.tfvars.example`.)

### 1.1 Required

| Variable | Example | How to obtain |
|---|---|---|
| `proxmox_endpoint` | `"https://192.168.1.10:8006"` | Your PVE host's web UI address |
| `proxmox_api_token` | `"terraform@pve!iac=<uuid>"` | *Datacenter → Permissions → API Tokens* |
| `node_name` | `"pve"` | The PVE node to deploy on (top of the PVE UI tree) |
| `workstation_public_key` | `"ssh-ed25519 AAAA… you@mac"` | `cat ~/.ssh/id_ed25519.pub` |
| `sudo_password` | strong string | `openssl rand -base64 12` — the `v2e` user's **sudo** password (SSH stays key-only). Weak/known values fail validation |
| `wan_address` | `"192.168.1.50/24"` | A **free static IP** on the LAN behind the WAN bridge (default `"dhcp"` works but makes the SSH target unpredictable) |
| `wan_gateway` | `"192.168.1.1"` | That LAN's gateway — whatever NATs the WAN bridge to the internet. Ignored when `wan_address = "dhcp"` |

### 1.2 Templates

Defaults are the production VMIDs; point at staging `9900–9903` if you haven't
promoted the builds (see RUNBOOK Step 1).

| Variable | Default | Consumed by |
|---|---|---|
| `vyos_template_id` | `9000` | router |
| `ubuntu_template_id` | `9001` | services |
| `debian_template_id` | `9002` | agent, infra |
| `parrot_template_id` | `9003` | control (UEFI desktop image) |

### 1.3 Proxmox plumbing

| Variable | Default | Notes |
|---|---|---|
| `proxmox_insecure` | `false` | Skip PVE API TLS verification; set `true` in tfvars for a self-signed cert (as this lab does) |
| `proxmox_ssh_username` | `"root"` | User for the snippet upload over SSH |
| `datastore_id` | `"local-lvm"` | VM disks |
| `snippet_datastore_id` | `"local"` | Must have the `snippets` content type enabled |
| `wan_bridge` | `"vmbr0"` | Bridge with internet access (VyOS WAN) |
| `lan_bridge` | `"vmbr1"` | VLAN-aware trunk bridge |

### 1.4 Addressing & DNS

| Variable | Default | Notes |
|---|---|---|
| `network_prefix` | `"10.1"` | Subnets become `10.1.<vlan-index>.0/24` |
| `subnet_mask` | `24` | Per-subnet prefix length |
| `lan_supernet` | `"10.1.0.0/16"` | Masquerade/source-NAT scope — keep consistent with `network_prefix` |
| `node_host_octet` | `10` | Every node's host octet (`10.1.1.10`, `10.1.2.10`, …) |
| `name_servers` | `["1.1.1.1", "9.9.9.9"]` | Upstream resolvers baked into router + nodes |

### 1.5 Sizing & VMIDs

| Variable | Default | Notes |
|---|---|---|
| `vyos_vmid` / `node_vmid_base` | `310` / `310` | Router = 310; control/services/agent/infra = base+1…+4 |
| `vyos_cores` / `vyos_memory` / `vyos_disk_size` | `1` / `1024` / `10` | Disk must match the VyOS template (10G) |
| `node_cores` / `node_memory` / `node_disk_size` | `2` / `2048` / `20` | services + agent. **`node_memory = 4096` recommended** — 2048 is OOM-tight under the full app estate |
| `control_cores` / `control_memory` / `control_disk_size` | `4` / `8192` / `64` | ParrotOS desktop needs the headroom |
| `infra_cores` / `infra_memory` / `infra_disk_size` | `2` / `2048` / `20` | Technitium + RustDesk are light |
| `router_boot_wait` | `"120s"` | Fixed delay before node creation (not a health check) — bump on slow hosts |

### 1.6 Users & access

| Variable | Default | Notes |
|---|---|---|
| `cluster_user` | `"v2e"` | Human login on every node; mesh hub on control |
| `ansible_user` | `"ansible"` | Automation account — locked password, NOPASSWD sudo, internal-only |
| `root_password` | `""` | Optional break-glass console/`su` password; empty = locked. SSH root is disabled regardless |
| `router_password_hash` | `""` | Optional VyOS console password, **pre-hashed**: `openssl passwd -6 '<pw>'` with OpenSSL 3.x (macOS LibreSSL can't; plaintext here = unusable). Empty = key-only |
| `control_ssh_wan_port` | `2201` | WAN port DNAT-forwarded to control:22 |
| `authorize_workstation_on_all_nodes` | `false` | `true` puts your key on services/agent/infra too (default: reach them via control) |
| `ansible_vault_password` | `""` | Only if you use Ansible Vault vars (e.g. AI API keys, §3.4); seeded to control as `~ansible/.vault_pass` |

### 1.7 Firewall

Policy is default-deny, defined in `cloud-init/vyos-router.yaml.tftpl` and applied at
router build (changes need a router `-replace`).

| Variable | Default | Notes |
|---|---|---|
| `firewall_enabled` | `true` | `false` disables the whole firewall |
| `trusted_mgmt_sources` | `[]` | CIDRs allowed to SSH the router over the WAN; empty = manage via control |
| `agent_egress_restricted` | `true` | Deny-by-default internet egress for the agent node |
| `agent_egress_allow_tcp_ports` | `[80, 443]` | TCP ports the agent may reach when restricted (DNS + NTP always allowed) |

### 1.8 Ansible bootstrap (control's first boot)

| Variable | Default | Notes |
|---|---|---|
| `ansible_repo_url` | `https://github.com/v2e-sh/v2e-ansible` | Point at your fork if you changed group_vars (§3). Empty disables the bootstrap |
| `ansible_repo_ref` | `""` | Branch/tag to deploy; empty = default branch |
| `ansible_version` | `"12.3.0"` | Pin the pipx-installed Ansible (default 12.3.0 → ansible-core 2.19.5); set empty for latest (not reproducible) |
| `ansible_playbook` / `ansible_inventory` | `"site.yml"` / `"inventory/hosts.ini"` | Rarely changed |

### 1.9 SOPS shipping (set both or neither)

| Variable | Default | Notes |
|---|---|---|
| `sops_secrets_file` | `""` | Path to the encrypted secrets (§2) → control `~ansible/ansible/group_vars/all.sops.yaml`; also decrypted at first boot and passed to `ansible-playbook` as `-e @~/.v2e-secrets.yml` |
| `sops_age_key_file` | `""` | Path to the age **private** key → control `~/.config/sops/age/keys.txt`. Read into tf state — encrypt state (§1.10) |

### 1.10 Cloudflare tunnel (optional; set the first three together or none)

| Variable | Default | Notes |
|---|---|---|
| `cloudflare_api_token` | `""` | Scopes: *Account → Cloudflare Tunnel:Edit*, *Zone → DNS:Edit + Zone:Read* — **not** the DNS-01 token (§2) |
| `cloudflare_account_id` / `cloudflare_zone_id` | `""` | Cloudflare dashboard → account / zone overview |
| `tunnel_name` | `"v2e-control-ssh"` | Zero Trust dashboard name |
| `tunnel_hostname` / `tunnel_dns_name` | `"lab.v2e.sh"` / `"lab"` | Public host + its CNAME label |
| `cloudflare_access_emails` | `[]` | Non-empty = OTP gate (Cloudflare Access) before SSH key auth |

**State encryption** is not a variable — it's the `TF_ENCRYPTION` env var (RUNBOOK
Step 4a). Without it, state holds the mesh keys, both tokens, `sudo_password`, and
the age key in plaintext.

---

## 2. secrets.sops.yaml — app secrets

One flat, age-encrypted YAML file. Keys are **lowercase** (Ansible asserts on the
exact names); `env.j2` renders them into the compose `.env` as the UPPER_CASE
equivalents, `$$`-escaping every value so bcrypt hashes and tokens survive docker
compose interpolation. Edit later with `sops secrets.sops.yaml`.

| Key | Required | Generate / obtain | Consumed by |
|---|---|---|---|
| `cf_dns_api_token` | **Yes** | Cloudflare token, scopes *Zone:Read + DNS:Edit* on your zone only | Traefik ACME DNS-01 |
| `authelia_session_secret` | **Yes** | `openssl rand -base64 48` | Authelia session encryption |
| `authelia_storage_encryption_key` | **Yes** | `openssl rand -base64 48` | Authelia storage encryption |
| `authelia_reset_password_jwt_secret` | **Yes** | `openssl rand -base64 48` | Authelia password-reset JWTs |
| `authelia_oidc_hmac_secret` | **Yes** | `openssl rand -base64 48` | Authelia OIDC token HMAC |
| `authelia_oidc_issuer_key` | **Yes** | RS256 private key PEM (`openssl genrsa 4096`) | Authelia OIDC token signing |
| `authelia_admin_password_hash` | **Yes** | argon2id hash (`authelia crypto hash generate argon2`) | Authelia admin login |
| `authelia_oidc_arcane_client_secret` / `_hash` | for arcane | `openssl rand -base64 32` + its pbkdf2 hash | Arcane OIDC (RP plaintext + Authelia-side hash) |
| `authelia_oidc_grafana_client_secret` / `_hash` | for observability | `openssl rand -base64 32` + its pbkdf2 hash | Grafana OIDC (RP plaintext + Authelia-side hash) |
| `semaphore_db_pass` | for semaphore | `openssl rand -base64 18` | Postgres + Semaphore |
| `semaphore_admin_password` | for semaphore | `openssl rand -base64 15` | Semaphore UI admin (first boot) |
| `semaphore_access_key_encryption` | for semaphore | `openssl rand -base64 32` — **rotating it invalidates every stored key/credential** | Semaphore key store |
| `semaphore_runner_reg_token` | for remote runner | `openssl rand -base64 24` | Runner registration (control-node runner) |
| `arcane_encryption_key` | for arcane | `openssl rand -base64 32` | Arcane data encryption |
| `arcane_jwt_secret` | for arcane | `openssl rand -base64 32` | Arcane sessions |
| `grafana_admin_password` | for observability | `openssl rand -base64 15` | Grafana `admin` login |
| `technitium_admin_password` | for infra DNS | `openssl rand -base64 15` | Technitium console `:5380` + zone API |
| `tailscale_authkey` | for tailnet access | Tailscale admin console → *Settings → Keys* — mint a **reusable** key (an ephemeral one drops the node on reboot). Empty = the tailscale role skips with a warning | control + infra `tailscale up` |
| `rustdesk_unattended_password` | for RustDesk | `openssl rand -base64 15` | Unattended access to the control desktop |
| `vaultwarden_admin_token` | for vaultwarden | `openssl rand -base64 32` (or an argon2 hash) | Vaultwarden `/admin` panel token |
| `mullvad_wireguard_private_key` / `_addresses` | for mullvad-exit | From your Mullvad WireGuard config | mullvad-exit stack (gluetun) |

Copy-paste template (RUNBOOK Step 3 wraps this with generation + encryption):

```yaml
cf_dns_api_token: "<scoped cloudflare token — Zone:Read + DNS:Edit>"
authelia_session_secret: "<openssl rand -base64 48>"
authelia_storage_encryption_key: "<openssl rand -base64 48>"
authelia_reset_password_jwt_secret: "<openssl rand -base64 48>"
authelia_oidc_hmac_secret: "<openssl rand -base64 48>"
authelia_oidc_issuer_key: "<RS256 private key PEM — openssl genrsa 4096>"
authelia_admin_password_hash: "<argon2id hash — authelia crypto hash generate argon2>"
authelia_oidc_arcane_client_secret: "<openssl rand -base64 32>"
authelia_oidc_arcane_client_secret_hash: "<pbkdf2 hash of the arcane secret>"
authelia_oidc_grafana_client_secret: "<openssl rand -base64 32>"
authelia_oidc_grafana_client_secret_hash: "<pbkdf2 hash of the grafana secret>"
semaphore_db_pass: "<openssl rand -base64 18>"
semaphore_admin_password: "<openssl rand -base64 15>"
semaphore_access_key_encryption: "<openssl rand -base64 32>"
semaphore_runner_reg_token: "<openssl rand -base64 24>"
arcane_encryption_key: "<openssl rand -base64 32>"
arcane_jwt_secret: "<openssl rand -base64 32>"
grafana_admin_password: "<openssl rand -base64 15>"
technitium_admin_password: "<openssl rand -base64 15>"
tailscale_authkey: "<reusable key from the Tailscale admin console, or empty>"
rustdesk_unattended_password: "<openssl rand -base64 15>"
vaultwarden_admin_token: "<openssl rand -base64 32>"
mullvad_wireguard_private_key: "<from your Mullvad WireGuard config>"
mullvad_wireguard_addresses: "<from your Mullvad WireGuard config>"
```

Encrypt to **two** age recipients — control's key and an offline backup key
(`sops --encrypt --age "<pub1>,<pub2>" secrets.yaml > secrets.sops.yaml`); under SOPS
a lost private key loses every secret.

Not in SOPS: **Arcane's login** seeds as `admin`/`password` on first boot — change it
in the UI immediately (codifying it is security backlog H3).

---

## 3. v2e-ansible — group_vars

These are **committed** values — change them in a fork and point `ansible_repo_url`
at it. Files live in `inventory/group_vars/`.

### 3.1 services.yml — the app estate

| Variable | Current value | Notes |
|---|---|---|
| `compose_stack_domain` | `"v2e.sh"` | **Your public zone on Cloudflare** — change this |
| `compose_stack_internal_domain` | `"int.v2e.sh"` | Internal-only app domain (wildcard cert is issued for this) |
| `compose_stack_acme_email` | `"admin@v2e.sh"` | Let's Encrypt account email — change this |
| `compose_stack_cert_resolver` | `"production"` | `"staging"` while testing (LE rate limits are real) |
| `compose_stack_stacks` | traefik, authelia, whoami, semaphore, arcane, observability, vaultwarden | Which v2e-compose stacks deploy on services |
| `compose_stack_repo_url` / `compose_stack_dir` | v2e-compose.git / `/opt/v2e-compose` | Fork-and-point as with ansible |
| `ssh_allow_users` | `"v2e ansible"` | sshd AllowUsers on this node |

### 3.2 infra.yml — DNS + relay appliance

| Variable | Current value | Notes |
|---|---|---|
| `compose_stack_stacks` | technitium, rustdesk, mullvad-exit | Host-network stacks; `compose_stack_network: ""` skips the Traefik bridge |
| `compose_stack_required_secrets` | `[technitium_admin_password, mullvad_wireguard_private_key, mullvad_wireguard_addresses, tailscale_authkey]` | Per-group fail-fast assert (services keeps the traefik + Authelia secrets) |
| `infra_upstream_nameservers` | `[1.1.1.1, 9.9.9.9]` | Node resolv.conf **and** Technitium's forwarders (`technitium_forwarders` derives from it) |
| `technitium_zone_name` | `"int.v2e.sh"` | The internal zone the API role creates |
| `technitium_zone_wildcard_target` | `10.1.2.10` | Apex + `*.int.<domain>` → services/Traefik |
| `technitium_zone_records` | infra/control/services/agent → their IPs | Per-node A records inside the zone |
| `tailscale_node_ssh` / `tailscale_node_advertise_routes` | `true` / `""` | infra joins the tailnet plainly |

### 3.3 control.yml — tailnet + desktop

| Variable | Current value | Notes |
|---|---|---|
| `tailscale_node_advertise_routes` | `"10.1.0.0/16"` | control is the subnet router. **Route approval + split-DNS (`int.<domain>` → `10.1.0.10`) stay manual in the Tailscale admin console** |
| `tailscale_node_ssh` | `true` | Tailscale SSH to control from the tailnet |
| `control_desktop_autologin_user` / `_session` | `v2e` / `plasmax11` | RustDesk needs a live X11 session |
| `control_desktop_disable_compositing` | `true` | Remote-desktop smoothness |
| `control_desktop_nameservers` | `[10.1.0.10]` | Desktop browser resolves `int.<domain>` via Technitium |
| `control_desktop_hosts_entries` | `{}` | Optional /etc/hosts shim (managed block; empty removes it) |
| `rustdesk_client_version` / `_deb_sha256` | `"1.4.8"` / `"c426f0a…7d5f547e"` | Pinned exact release; empty would skip the install with a warning |

### 3.4 Other committed group_vars

| File | Variable | Value | Why |
|---|---|---|---|
| linux.yml | `sysctl_overwrite: {net.ipv4.ip_forward: 1}` | — | devsec hardening sets 0, which breaks Docker |
| linux.yml | `ssh_client_alive_count` | `2` | 600s idle timeout (interval 300 × 2) |
| control.yml | `ssh_allow_users` | `"v2e"` | Control accepts only the human account |
| agent.yml / infra.yml | `ssh_allow_users` | `"v2e ansible"` | |

### 3.5 Role defaults worth knowing (override in group_vars)

| Role | Variable | Default | Meaning |
|---|---|---|---|
| health_check | `health_check_min_disk_percent` / `_min_mem_mb` / `_connect_timeout` | `15` / `512` / `30` | Fail-fast gate before any change |
| baseline | `baseline_timezone` | `"UTC"` | |
| baseline | `baseline_journald_max_use` | `"200M"` | |
| baseline | `baseline_unattended_auto_reboot` | `false` | Security-only unattended-upgrades, no auto-reboot |
| fail2ban | `fail2ban_enabled` | `true` | sshd jail, journald backend |
| compose_stack | `compose_stack_cert_resolver` | `"staging"` | services.yml overrides to production |
| ai_identities | `ai_identities_roster` | `[claude, codex]` | AI accounts (ops/agents.yml, not auto-run) |
| ai_workbench | `ai_workbench_node_major` | `"22"` | NodeSource major for the agent workbench |
| patch (ops) | `patch_upgrade` / `patch_autoremove` | `"dist"` / `true` | Never reboots the controller |
| killswitch (ops) | `killswitch_state` | `"cut"` | `cut` \| `allow` \| `cut-hard`; tag-gated, persists via `killswitch_save` |

### 3.6 Vault-encrypted extras (optional)

`ai_workbench` reads `vault_anthropic_api_key` / `vault_openai_api_key` from an
Ansible-Vault-encrypted vars file (pair with `ansible_vault_password` in §1.6).
Empty/absent = the workbench skips API auth.

Galaxy dependencies are pinned exact in `requirements.yml`
(geerlingguy.docker 7.9.0, artis3n.tailscale 5.0.1, community.docker 5.2.0,
devsec.hardening 10.6.0, community.sops 2.3.0). Renovate bumps them — don't loosen
to ranges.

---

## 4. v2e-templates — config.env

Edited on the PVE host before building (RUNBOOK Step 1). All builds re-source this
file; CLI overrides are ignored.

| Variable | Default | Notes |
|---|---|---|
| `STORAGE` | `local-lvm` | Must support cloud-init disks |
| `BRIDGE` | `vmbr0` | Bridge with internet (used during verify boots) |
| `VYOS_VMID` / `UBUNTU_VMID` / `DEBIAN_VMID` / `PARROT_VMID` | `9900`–`9903` | Staging IDs; set to `9000`–`9003` to build straight to prod |
| `*_URL` / `*_SHA256` | pinned per image | An upstream point-release fails the checksum on purpose — update both together |
| `VYOS_BUILDER_IP` / `BUILD_GW` | — | Only for the build-from-source VyOS path |

---

## 5. v2e-compose standalone (.env) — reference only

In the deployed lab, Ansible renders each stack's `.env` from §2/§3 — there is
nothing to configure in v2e-compose itself. For running a stack standalone (dev):
`make bootstrap`, edit `.env` (`DOMAIN`, `INTERNAL_DOMAIN`, `ACME_EMAIL`,
`CERT_RESOLVER`), put secrets in the repo-local `secrets.sops.yaml`, then `make up`
(wraps `sops exec-env`). `make validate` type-checks every compose file with dummy
values.

---

## 6. Known gaps

- **Manual by nature** (tailnet-side, no node API): approving control's advertised
  route and setting split-DNS `int.<domain>` → `10.1.0.10` in the Tailscale admin
  console.
- **Values still to pin**: a **reusable** `tailscale_authkey` (the SOPS key is
  present but empty until minted); the live `/etc/hosts` shim entries (variable
  defaults to empty — capture before enabling).
- Live-lab throwaway credentials are recorded in `v2e-tf/.deploy-creds.txt` (local,
  gitignored) — rotate everything at real launch.
