# v2e — Master Plan (Consolidated)

**Single source of truth.** Merges the implementation roadmap, the open-source tooling
audit, and the integration analysis into one document, with every decision settled so
far reflected as the current plan. Each phase is a branch / its own Claude conversation,
stopped at a testable point and merged to `main`.

**Date:** 2026-06-30
**Status:** living document — update the phase checkboxes and decision log as work lands.

> **Consolidation note (2026-06-30):** This revision folds in three additions on top of
> the original plan, grounded in a deep review of the current repos:
> 1. **Killswitch re-homed** — the existing `killswitch` + `vyos-hardening` operational
>    playbooks are explicitly preserved through the ANS restructure (ANS-1) instead of
>    being lost in the refactor.
> 2. **ANS-6 — Agent-action audit logging** — a new phase giving an immutable record of
>    what the AI agent does (the accountability layer for "let the AI explore fully").
> 3. **Containment decision resolved** — keep the killswitch **and** today's
>    control→router access as-is; audit logging (ANS-6) is the added safeguard. See §9.

---

## 0. The system in brief

A self-hosted DevOps lab on a single Proxmox host, reachable from anywhere, where AI
agents can run it. Built as code across five repos and a four-layer stack:

```
Packer (image)      → reproducible base templates (VyOS, Ubuntu, Debian, Parrot)
   ↓
Terraform (infra)   → VMs, networking, bootstrap identity (v2e + ansible users)
   ↓
Ansible (config)    → health checks, hardening, Docker, services
   ↓
Compose (apps)      → Traefik + TLS, auth, Semaphore, Dockge, monitoring
```

**Repos:** `v2e-packer` (images), `v2e-tf` (OpenTofu/HCL), `v2e-ansible` (config),
`v2e-compose` (stacks), `v2e-docs` (runbooks).

---

## 1. Locked decisions

- **Division of labour.** Packer = base images. Terraform = infra + bootstrap identity
  only (`v2e`/`ansible` users, router's `vyos` user). Ansible = everything after.
- **Full automation.** `terraform apply` → control's cloud-init clones `v2e-ansible`
  `main` → `ansible-playbook site.yml` → health checks → config. One lever.
- **No playbook ref pinning.** Cloud-init clones `main` (tested before merge, always
  deployable). No `ansible_repo_ref`. Keep `ansible_version` pinned.
- **Ansible structure.** `site.yml` statically imports `01-bootstrap` → `02-services` →
  `03-applications`; health checks gate the rest (fail-fast).
- **Don't reinvent.** Host plumbing uses Galaxy roles (`geerlingguy.docker`,
  `devsec.hardening`, `artis3n.tailscale`, `community.sops`). App layer stays
  compose-native in `v2e-compose`.
- **HTTPS everywhere.** Traefik terminates TLS via Let's Encrypt + Cloudflare DNS-01 →
  wildcard certs, zero inbound ports.
- **Hardening won't break the workload.** `devsec.hardening` with `sftp_enabled: true`
  (Ansible file copy) and `net.ipv4.ip_forward=1` preserved (Docker networking).
- **D-1 — Secrets: SOPS + age.** One encrypted file read directly by Terraform, Ansible,
  and Compose. Every user supplies their own locally, like `terraform.tfvars`.
  (Replaces Ansible Vault.)
- **D-5 — Packer bake/boot/config split.** Bake only static prerequisites into images
  (`qemu-guest-agent`, `sops` + `age` binaries, cloud-init, baseline packages). Keep
  dev-tools and hardening in Ansible — dev-tools so the toolset stays flexible per node;
  hardening because `os_hardening` clamps the sysctl Docker needs and a bad bake breaks
  every clone.
- **Q3 — Auth: TinyAuth.** Start with TinyAuth (stateless cookie, no session backend, no
  Valkey). Behind a Traefik forward-auth middleware so it's swappable to Authelia +
  Valkey later with no change to the rest of the stack.
- **Containment model (resolved 2026-06-30).** Keep the `killswitch` and today's
  control→router SSH access **as-is** (status quo). Agent accountability comes from the
  new **ANS-6** agent-action audit logging, not from retiring keys. Decommission of the
  current root AI identities is **not** scheduled — Phase I's MCP model runs alongside
  them. See §9.

---

## 2. Open decisions

- **Q1 — Agent egress.** Deny-by-default + allowlist (recommended for an AI-agent node —
  zero-trust egress) vs keep open and relabel "internal-isolation-only." Decide at TF-3.
  (Backstops Phase I.)
- **Q2 — Reboot handling.** Confirm `package_reboot_if_required: false` + Ansible-managed
  reboots. Largely eased by Packer (current image = less first-boot churn). Decide at TF-3.
- **Q4 — State backend.** Local + protect the workdir vs encrypted remote backend.
  Investigate the missing encryption warning first.
- **Q5 — Tailscale node scope.** Which node(s) join the tailnet and whether `--ssh` is
  wanted (ANS-4).
- **D-2 — Tailscale vs Headscale.** Hosted Tailscale (free tier) now; Headscale (OSS,
  self-hosted control plane) if you want full sovereignty later.
- **D-3 — Packer under BSL.** Accept BSL for internal/shareable use (recommended) vs
  pure-OSS pipeline (libguestfs/mkosi, more friction). Decide at Phase 0/E.
- **D-4 — Cloudflare Tunnel vs Pangolin.** Keep Cloudflare (optional) vs self-hosted OSS
  Pangolin. Relevant only at COMPOSE-4.
- **Remote access to control.** Leaning RDP/remote over the Tailscale/WireGuard mesh (no
  exposed port, nothing extra to host) rather than exposing Rustdesk publicly. Deferred —
  decide when needed.

---

## 3. Secrets flow (SOPS + age)

```
local machine:                         Terraform:                control cloud-init:               Ansible / Compose (first boot):
  age-keygen → keys.txt (private)  ┌─►  sops_age_key_file   ─►  write ~/.config/sops/age/keys.txt   age private key decrypts:
  sops --encrypt secrets.sops.yaml │    (age private key)        (0600, ansible-owned)               • Ansible: community.sops vars plugin
  (encrypted to age PUBLIC key)    └─►  sops_secrets_file    ─►  write group_vars/all.yml (0600)      • Compose: sops exec-env '… up -d'
                                        (encrypted file)                                              • Terraform (if needed): carlpett/sops
```

- **Asymmetric, no password.** The age public key encrypts; the private key decrypts.
  User keeps the private key safe and passes it to Terraform like any credential.
- `sops`/`age` binaries are baked into the Packer image (D-5) so they exist before
  Ansible runs.
- **Multiple recipients allowed** — the user's key and a backup/CI key can both decrypt
  the same file (resilience; lets CI validate).
- **Plaintext never touches the repo or Terraform state** — only the path does.

> **Hygiene note:** the existing `v2e-tf/terraform.tfstate` currently holds plaintext
> mesh private keys and any tokens from earlier applies. Treat those secrets as **burned**
> and rotate them on the first clean rebuild under the SOPS model. Also: losing the age
> **private** key = losing every secret — keep a second recipient (backup/CI key) so the
> file is always recoverable.

---

## 4. Execution order

```
Phase 0/E (Packer) ─────────────────────────────────┐ feeds templates
  VyOS-src · Ubuntu · Debian · Parrot                │ bakes: agent, sops+age
                                                     ▼
                                        TF-1 ──► TF-2 (SOPS) ──► TF-3
                                           │
                                           └──► ANS-1 ──► ANS-2 ──► ANS-4
                                                            │  └──► ANS-6 (agent audit)
                            COMPOSE-1 ──► ANS-3 ◄──────────┘
                               │
                               ▼
                          COMPOSE-2 (TinyAuth) ──► COMPOSE-3 ──► COMPOSE-4

cross-cutting from Phase 0:  pre-commit · gitleaks · Trivy · Checkov · Renovate · SOPS
additive after core:         Phase G (monitoring) · Phase H (GHCR) · Phase I (agent/MCP)
                             ANS-6 (agent audit) ships to Phase G's Loki sink
operational (tag-gated):     killswitch · vyos-hardening · patch  (standalone playbooks)
DOCS-1/2: parallel; finalize last
```

**Critical path:** Phase 0 → TF-1 → ANS-1 → ANS-2 → COMPOSE-1 → ANS-3.

---

## 5. Phases

> Legend: `[ ]` not started · `[~]` in progress · `[x]` merged to main.

### `[x]` Phase 0/E — Packer templates

**Branch:** `feat/packer-templates` · **Repo:** `v2e-packer` (new) · **Upstream of:** TF-1

**Goal:** reproducible, shareable Proxmox templates as code. Packer files are ~50–100
lines of HCL — point at an ISO URL, run provisioners, output a template. Far easier to
share/version than a built image.

**Scope:** four images — VyOS LTS built from source, Ubuntu Server LTS, lightweight
Debian, Parrot OS. Bake: `qemu-guest-agent`, `sops` + `age`, cloud-init, baseline
packages. Do not bake dev-tools or hardening (D-5). Use the `bpg/packer-proxmox` plugin;
pin ISO versions as variables; lint provisioners with ShellCheck and any Dockerfiles with
Hadolint.

**License note:** Packer is BSL (not OSI-OSS); fine for internal/shareable use — document
it. Pure-OSS fallback: libguestfs `virt-builder`/`virt-customize` or mkosi.

**Acceptance:** `packer build` outputs each template; a cloned VM boots with cloud-init +
agent working and `sops`/`age` present.

**Kickoff prompt**

> New `v2e-packer` repo, branch `feat/packer-templates`. Implement Phase 0/E from the v2e
> master plan: Packer templates on Proxmox (via `bpg/packer-proxmox`) for VyOS LTS built
> from source, Ubuntu Server LTS, lightweight Debian, and Parrot OS. Bake
> `qemu-guest-agent`, `sops`+`age`, cloud-init, and baseline packages — but NOT dev-tools
> or hardening. Pin ISO versions as variables; add ShellCheck/Hadolint lint. Start by
> confirming the VyOS-from-source flow. Note Packer's BSL status in the README.

---

### `[x]` TF-1 — Critical fixes + simplifications

**Branch:** `fix/tf-critical` · **Repo:** `v2e-tf` · **Depends on:** Phase 0 templates

**Tasks:** (1) change the cloud-init Ansible clone to plain `git clone` of `main`; remove
`ansible_repo_ref`. (2) `jsonencode()` the sudo password and VyOS `password_hash` in
cloud-init templates. (3) make `ssh_to_vyos` output conditional on
`firewall`/`trusted_mgmt_sources`; fix the README "Access" snippet to use the `vyos` user
via control. (4) remove `extra_vyos_commands`/`extra_packages`. (5) align template-ID
variables to Packer outputs (document the image→template→VMID chain). Keep
`ansible_version`.

**Acceptance:** `tofu fmt`/`validate` pass; clean apply; control boots, clones `main`,
bootstrap runs; `ssh -p 2201 v2e@<wan>` reaches control and `ssh vyos` from control
reaches the router; a special-char password no longer corrupts cloud-init.

**Kickoff prompt**

> `v2e-tf`, branch `fix/tf-critical`. Implement TF-1 from the v2e master plan: (1) clone
> `main` (drop `ansible_repo_ref`); (2) `jsonencode()` the sudo password + VyOS
> `password_hash`; (3) conditional `ssh_to_vyos` output + README fix to `vyos` via
> control; (4) remove `extra_vyos_commands`/`extra_packages`; (5) align template IDs to
> Packer outputs. Show diffs + test plan.

---

### `[x]` TF-2 — SOPS secrets

> **Status:** implemented + validated on `feat/tf-critical-sops` → **PR #2** (bundled
> with the TF-1 completion: jsonencode passwords, clone ansible `main`, remove
> `extra_*`, `stop_on_destroy`). Merged to `main`.

**Branch:** `feat/tf-sops-secrets` · **Depends on:** TF-1

**Tasks:** add `sops_secrets_file` (path to a locally sops-encrypted file) and
`sops_age_key_file` (path to the age private key); blank = feature off. In control's
cloud-init: write the encrypted file to `group_vars/all.yml` (0600, `ansible:ansible`)
and the age key to `~/.config/sops/age/keys.txt` (0600, ansible-owned). Document the user
workflow: `age-keygen`, `sops --encrypt`, pass both paths in `terraform.tfvars`.

**Acceptance:** a test SOPS file round-trips — present + decryptable on control via the
baked binaries; blank var = byte-identical to TF-1.

**Kickoff prompt**

> `v2e-tf`, branch `feat/tf-sops-secrets`. Implement TF-2 from the v2e master plan: add
> `sops_secrets_file` + `sops_age_key_file` variables; cloud-init writes the encrypted
> file to `group_vars/all.yml` and the age key to `~/.config/sops/age/keys.txt` (both
> 0600, ansible-owned); gate blank=off; document the encrypt-and-supply workflow. Confirm
> `community.sops` decrypts on first boot.

---

### `[x]` TF-3 — Firewall + state hardening

> **Status:** implemented + validated on `fix/tf-hardening-state` → **PR #3** (stacked
> on PR #2). Done: ICMP scoped to LAN; agent egress deny-default + allowlist (Q1 =
> deny-default); `package_reboot_if_required=false` (Q2); state-encryption warning via
> an `external` TF_ENCRYPTION probe + `check` (Q4 = keep local + TF_ENCRYPTION). The
> default-deny firewall itself already landed on `main` earlier. Agent allowlist ships
> as a starter set (DNS/NTP/80/443) — refine as needed. Merged.

**Branch:** `fix/tf-hardening-state` · **Depends on:** TF-1

**Tasks:** (1) debug why the state-encryption warning doesn't fire; make plan/apply emit
it when unset. (2) scope the VyOS input ICMP rule to the LAN interface. (3) agent egress
per Q1 (recommend deny-by-default + allowlist). (4) set
`package_reboot_if_required: false` (Packer partly mitigates the reboot race — Q2). (5)
document local-state secret sensitivity + optional encrypted backend.

**Acceptance:** agent reaches the internet per policy; control reaches services/agent;
router doesn't answer WAN ICMP; the warning fires; a first-boot kernel update no longer
hangs apply.

**Kickoff prompt**

> `v2e-tf`, branch `fix/tf-hardening-state`. Implement TF-3 from the v2e master plan: fix
> the state-encryption warning; scope router ICMP to LAN; agent egress per Q1
> (deny-by-default + allowlist — ask me for the list); `package_reboot_if_required: false`;
> document local-state sensitivity. Show diffs + test plan.

---

### `[ ]` ANS-1 — Structure + skeleton

**Branch:** `refactor/ansible-structure` · **Repo:** `v2e-ansible` · **Depends on:** TF-1

**Tasks:** `site.yml` imports `01-bootstrap`/`02-services`/`03-applications` via
`import_playbook`. `requirements.yml` pins `geerlingguy.docker` (7.x), `community.docker`
(>=3), `devsec.hardening` (10.x), `artis3n.tailscale`, `community.sops`. `ansible.cfg`:
roles/collections paths, SOPS vars plugin enabled, SFTP-compatible transfer, host-key
handling. `inventory/hosts.ini`: groups `control`/`services`/`agent`, keep `all:!vyos`.
Scaffold `health_check` and `dev-tools` roles.

**Existing-role migration map (don't lose work in the refactor):**

| Current role | Disposition in new structure |
|---|---|
| `baseline` | Folded into `01-bootstrap` (drop `qemu-guest-agent` install — now baked by Packer). |
| `deb-hardening-basic` | **Replaced** by `devsec.hardening` in `01-bootstrap` (retire the cloud-init `60-v2e.conf` drop-in). |
| `docker` (vendored geerlingguy) | **Replaced** by pinned `geerlingguy.docker` in `02-services` (ANS-3). |
| `patch` | Kept as a **standalone tag-gated** operational playbook (`--tags patch`); reboot policy per Q2. |
| `vyos-hardening-basic` | Kept as a **standalone tag-gated** vyos playbook (`--tags vyos`); not part of `site.yml`. |
| `killswitch` | **Re-homed, not removed** — preserved as a standalone operational playbook (`killswitch.yml`, tags `cut`/`allow`/`cut-hard`) targeting the `vyos` group. This is the agent-VLAN containment backstop (see §1 containment decision). |
| `ai-identities` / `ai-workbench` | Retained as-is under `03-applications` for now; Phase I (MCP) layers on top. No decommission scheduled. |

**Acceptance:** syntax-check + `ansible-lint` pass; requirements install; dry-run works;
`killswitch.yml` and the vyos-hardening playbook still resolve and run standalone after
the restructure.

**Kickoff prompt**

> `v2e-ansible`, branch `refactor/ansible-structure`. Implement ANS-1 from the v2e master
> plan: split `site.yml` into `import_playbook` of bootstrap/services/applications;
> `requirements.yml` (geerlingguy.docker, community.docker, devsec.hardening,
> artis3n.tailscale, community.sops); `ansible.cfg` with SOPS vars plugin + SFTP +
> host-key handling; `inventory/hosts.ini` (control/services/agent, keep `all:!vyos`);
> scaffold `health_check` + `dev-tools` roles. **Apply the existing-role migration map
> from the plan — in particular, preserve `killswitch.yml` and the vyos-hardening playbook
> as standalone tag-gated operational playbooks; do not delete them.** Keep idempotent +
> lint-clean.

---

### `[ ]` ANS-2 — Bootstrap playbook

**Branch:** `feat/ansible-bootstrap` · **Depends on:** ANS-1

**Tasks:** `health_check` role — `wait_for_connection` across the mesh, disk/memory
asserts, inter-VLAN reachability, fail-fast. `devsec.hardening` with `sftp_enabled: true`,
`net.ipv4.ip_forward=1` preserved, `AllowUsers`/`PermitRootLogin`/`PasswordAuthentication`
set to match intent, and retire the cloud-init `60-v2e.conf` drop-in (devsec owns
`sshd_config`). `dev-tools` role — ripgrep, fd, fzf, bat, eza, jq, yq, tmux, lazygit,
zoxide, delta, hyperfine, tldr, dust, duf, btop, ncdu (control; optionally other nodes).
`terminal-polish` role — Ghostty, Starship, Zsh on control. Verify `sops`/`age` present
(baked). Idempotent.

**Acceptance:** Molecule converge + idempotence pass; after hardening Ansible can still
copy files (SFTP) and a container still reaches another container + the internet
(forwarding intact); no SSH lockout.

**Kickoff prompt**

> `v2e-ansible`, branch `feat/ansible-bootstrap`. Implement ANS-2 from the v2e master
> plan: `health_check` role (mesh `wait_for_connection`, disk/mem asserts, inter-VLAN
> check, fail-fast); `devsec.hardening` with `sftp_enabled: true`, `ip_forward=1`
> preserved, `AllowUsers`/root/password matching intent, retiring the cloud-init drop-in;
> `dev-tools` role (rg/fd/fzf/bat/eza/jq/yq/tmux/lazygit/zoxide/delta/hyperfine/tldr/dust/duf/btop/ncdu);
> `terminal-polish` (Ghostty/Starship/Zsh). Add Molecule with idempotence + "file copy
> works" + "container forwarding works" checks.

---

### `[ ]` ANS-3 — Services: Docker + compose deploy

**Branch:** `feat/ansible-services-docker` · **Depends on:** ANS-2 and COMPOSE-1

**Tasks:** `geerlingguy.docker` (engine, ansible user in docker group, `daemon.json` with
log rotation + live-restore). `community.docker` `docker_compose_v2` to deploy the
`v2e-compose` stack. Feed secrets via `sops exec-env` (CF DNS token, Tailscale authkey,
service passwords).

**Acceptance:** Docker running; compose stack converges healthy; Traefik issues a valid
cert; re-run idempotent.

**Kickoff prompt**

> `v2e-ansible`, branch `feat/ansible-services-docker`. Implement ANS-3 from the v2e
> master plan: `geerlingguy.docker` (ansible in docker group, `daemon.json` log rotation +
> live-restore), then `community.docker` `docker_compose_v2` to deploy the `v2e-compose`
> stack, feeding secrets via `sops exec-env`. Idempotent. Requires `feat/compose-traefik-tls`
> merged.

---

### `[ ]` ANS-4 — Tailscale

**Branch:** `feat/ansible-tailscale` · **Depends on:** ANS-2

**Tasks:** `artis3n.tailscale` on the chosen node(s); `tailscale_authkey` from SOPS; tags;
optional `--ssh`. (This is also where mesh-based remote access to control would live — see
the "Remote access" open decision.)

**Acceptance:** node joins the tailnet; re-run idempotent; authkey never logged.

**Kickoff prompt**

> `v2e-ansible`, branch `feat/ansible-tailscale`. Implement ANS-4 from the v2e master
> plan: `artis3n.tailscale` for the node(s) per Q5, authkey from SOPS, tags, optional
> `--ssh`. Confirm idempotence and redacted authkey.

---

### `[ ]` ANS-5 — Molecule + CI

**Branch:** `feat/ansible-molecule-ci` · **Depends on:** ANS-2

**Tasks:** Molecule scenarios per custom role (start with `health_check`): converge →
verify → idempotence. GitHub Actions: `ansible-lint` + `molecule test` + Trivy + Checkov +
gitleaks.

**Kickoff prompt**

> `v2e-ansible`, branch `feat/ansible-molecule-ci`. Implement ANS-5 from the v2e master
> plan: Molecule (converge/verify/idempotence) for custom roles starting with
> `health_check`, and GitHub Actions running `ansible-lint` + molecule + Trivy + Checkov +
> gitleaks.

---

### `[ ]` ANS-6 — Agent-action audit logging *(added 2026-06-30)*

**Branch:** `feat/ansible-agent-audit` · **Repo:** `v2e-ansible` · **Depends on:** ANS-2
(Loki sink from Phase G; degrades to local + journald if G isn't up yet)

**Goal:** an immutable, append-only record of *what the AI agent actually does* on the
agent node — the accountability layer that makes "let the AI explore fully" safe to
operate, given the containment decision keeps the killswitch + router keys as-is rather
than locking the agent down further.

**Scope:** an `agent-audit` role applied to the `agent` node (optionally all linux nodes):

- **auditd** with a ruleset capturing process exec (`execve`), writes under sensitive
  paths (`/etc`, infra repos, SSH config), `sudo`/privilege use, and outbound connection
  attempts. Make the rules **immutable** (`-e 2`) so a live session can't quietly disable
  them until reboot.
- **Session capture** for the agent identities (`claude`/`codex` and any MCP service
  account) — e.g. `tlog` session recording, or a `script`/auditd-based approach — so both
  interactive and `agent-run`/MCP-driven sessions are replayable.
- **Off-box shipping** of audit + session logs to the Phase G Loki sink via promtail
  (append-only), with local copies rotated. The point of off-box shipping is that a
  compromised agent cannot trivially erase its own trail.
- **Tie-ins:** this is the trail referenced by Phase I (unprivileged MCP) and surfaced in
  Grafana (Phase G). Until Phase G lands, log locally + to journald and wire the Loki
  sink when monitoring is up.

**Acceptance:** a command run via `agent-run` (or an MCP action) appears in `auditd` and,
when Phase G is up, in Loki within seconds; auditd rules are immutable until reboot; a
session is replayable; shipping survives local log deletion. Molecule idempotence passes.

**Kickoff prompt**

> `v2e-ansible`, branch `feat/ansible-agent-audit`. Implement ANS-6 from the v2e master
> plan: an `agent-audit` role on the agent node giving an immutable agent-action trail —
> auditd ruleset (execve, sensitive-path writes, sudo, outbound conns) made immutable
> (`-e 2`); session recording for the agent identities (tlog or script/auditd); ship audit
> + session logs off-box append-only to Loki via promtail (fall back to local+journald
> until Phase G is up). Verify a command shows up in the trail and that local deletion
> doesn't lose the shipped copy. Add Molecule idempotence.

---

### `[x]` COMPOSE-1 — Traefik + Cloudflare DNS-01 + wildcard HTTPS

**Branch:** `feat/compose-traefik-tls` · **Repo:** `v2e-compose`

**Tasks:** Traefik v3 — `web(:80)`→`websecure(:443)` redirect;
`certificatesResolvers.cloudflare.acme.dnsChallenge`; persist `acme.json` (0600). Scoped
`CF_DNS_API_TOKEN` (Zone:Read + DNS:Edit, from SOPS). Wildcard `*.<domain>` via
`tls.domains`. `no-new-privileges`, read-only `docker.sock` (socket-proxy as a
follow-up), explicit `proxy`/`internal` networks. `whoami` test service. Use LE staging
first.

**Gotcha:** gray-cloud internal service DNS records (orange-cloud shows Cloudflare's cert,
not LE). The tunnel hostname stays proxied — different record.

**Acceptance:** Traefik pulls a valid wildcard LE cert (staging→prod); `whoami` reachable
over HTTPS with redirect working.

**Kickoff prompt**

> `v2e-compose`, branch `feat/compose-traefik-tls`. Implement COMPOSE-1 from the v2e
> master plan: Traefik v3 with HTTP→HTTPS redirect, LE via Cloudflare DNS-01 using a
> scoped `CF_DNS_API_TOKEN` (from SOPS), wildcard `*.<domain>` via `tls.domains`,
> `no-new-privileges`, read-only `docker.sock`, explicit `proxy`/`internal` networks,
> `whoami` test. Start with LE staging. Note the gray-cloud caveat in the README.

---

### `[x]` COMPOSE-2 — Auth (TinyAuth)

**Branch:** `feat/compose-auth` · **Depends on:** COMPOSE-1

**Tasks:** TinyAuth (Q3) as a Traefik forward-auth middleware; protect the dashboard +
selected services; leave public ones open; secrets from SOPS. Keep the middleware layer
clean so swapping to Authelia + Valkey later is a service swap, not a rebuild.

**Acceptance:** a protected route prompts for auth and only passes valid users;
unprotected routes unaffected.

**Kickoff prompt**

> `v2e-compose`, branch `feat/compose-auth`. Implement COMPOSE-2 from the v2e master plan:
> TinyAuth as a Traefik forward-auth middleware protecting the dashboard + selected
> services, secrets from SOPS. Keep it swappable to Authelia+Valkey later.

---

### `[ ]` DNS-1 — Internal DNS appliance (Technitium) + `*.int.v2e.sh` *(added 2026-06-30)*

**Repos:** `v2e-tf` + `v2e-ansible` + `v2e-compose` (cross-layer) · **Depends on:** Packer Debian
template (9002), COMPOSE-2 · **Pairs with:** ANS-4 (Tailscale delivers it remotely). Full design:
`v2e-docs/specs/2026-06-30-dns-appliance-design.md`.

**Goal:** every Traefik-fronted dashboard reachable by name at `*.int.v2e.sh`, resolved by a
self-hosted internal resolver. DNS supplies the name (`*.int.v2e.sh → 10.1.2.10`, the services
node running Traefik); Tailscale (ANS-4) supplies the route for remote clients; on-lab nodes via
VyOS DHCP.

**Tasks (by layer):**
- **TF:** add a `dns` node (VMID `node_vmid_base + 4`, Debian template, mgmt VLAN, `10.1.0.53`,
  1 vCPU / 512 MB / 8 GB); VyOS firewall allow `53/udp+tcp` from control/services/agent VLANs →
  `10.1.0.53`; VyOS DHCP option 6 = `10.1.0.53` for the node VLANs.
- **Ansible:** a `technitium` role — install native (systemd), configure via the HTTP API
  (authoritative `int.v2e.sh` zone, `*.int.v2e.sh → 10.1.2.10`, DoT forwarders, recursion
  restricted to lab VLANs, query logging on). `internal_domain` group_var; API creds from SOPS.
- **Compose:** introduce `INTERNAL_DOMAIN` (one config var, default `int.v2e.sh`); move the
  COMPOSE-1/2 router host rules to `${INTERNAL_DOMAIN}`; extend the wildcard anchor to also
  request `*.${INTERNAL_DOMAIN}`; add a `dns.${INTERNAL_DOMAIN}` router → the Technitium console
  (`10.1.0.53:5380`) behind `auth@docker`.

**Decisions:** dedicated minimal-Debian VM (foundational DNS off the app-stack Docker); Technitium
native systemd, API-configured (chosen over Pi-hole/AdGuard for real zones + API + logging);
dedicated subdomain `int.` (`lab.` is the CF tunnel); **`INTERNAL_DOMAIN` is one config variable**
shared by Compose + Ansible — no hardcoded internal-domain strings; ACME stays safe via COMPOSE-1's
`1.1.1.1` challenge-resolver pin. Feeds **Q1**/ANS-6 (agent DNS visibility); **Q5** (Tailscale
scope) must include the DNS node's tailnet reachability.

**Acceptance:** `dig traefik.int.v2e.sh @10.1.0.53` → `10.1.2.10`; upstream names still resolve;
recursion refused from non-lab sources; `https://traefik.int.v2e.sh` loads behind auth with a valid
`*.int.v2e.sh` LE cert; public `lab.v2e.sh` / `v2e.sh` unaffected; `technitium` role idempotent.

**Kickoff prompt**

> New dedicated conversation. Implement DNS-1 from the v2e master plan (full design:
> `v2e-docs/specs/2026-06-30-dns-appliance-design.md`): a dedicated minimal-Debian DNS node
> (`10.1.0.53`, mgmt VLAN) running **Technitium** native (systemd), configured via its HTTP API,
> authoritative for `int.v2e.sh` with `*.int.v2e.sh → 10.1.2.10`, forwarding the rest upstream
> over DoT. Split across `v2e-tf` (node + VyOS firewall/DHCP option 6), `v2e-ansible` (`technitium`
> role), and `v2e-compose` (introduce `INTERNAL_DOMAIN`, move the dashboards onto `*.int.v2e.sh`,
> extend the wildcard cert, add a `dns.int.v2e.sh` admin router behind `auth@docker`). Remote
> delivery is ANS-4 (Tailscale split-DNS + subnet route) — document the hook, out of scope here.
> Keep the ACME `1.1.1.1` pin. Brainstorm each repo's slice, then plan + implement.

---

### `[ ]` COMPOSE-3 — Semaphore + Dockge

**Branch:** `feat/compose-semaphore-dockge` · **Depends on:** COMPOSE-1, COMPOSE-2

**Tasks:** Semaphore on the services node (Ansible UI + REST API; Postgres on the internal
network) behind Traefik + TinyAuth. Dockge (compose manager) behind Traefik + TinyAuth.
Secrets from SOPS.

**Acceptance:** Semaphore reachable over HTTPS and can run a test playbook; Dockge manages
stacks; both gated.

**Kickoff prompt**

> `v2e-compose`, branch `feat/compose-semaphore-dockge`. Implement COMPOSE-3 from the v2e
> master plan: Semaphore (Postgres on internal net) + Dockge behind Traefik + TinyAuth,
> secrets from SOPS. Verify Semaphore runs a test playbook over HTTPS.

---

### `[ ]` COMPOSE-4 — Cloudflare tunnel + 2FA (later)

**Branch:** `feat/compose-cloudflare-tunnel-2fa` · **Depends on:** COMPOSE-3

Deferred remote-access path — a Cloudflare tunnel for the services node with Zero Trust /
Access 2FA. Pangolin is the OSS self-hosted alternative (D-4). Design the 2FA/Access
policy in its own conversation.

---

### `[ ]` DOCS-1 — First-time runbook

**Branch:** `docs/first-time-runbook` · **Repo:** `v2e-docs`

Sectioned guide: Prerequisites (Packer build, templates, VLAN-aware bridge, API token),
Proxmox, Networking, VyOS/WAN, Access, Cloudflare (optional), SOPS secrets flow
(`age-keygen` + encrypt). `terraform.tfvars.example` stays the full reference; each
section explains its variables with a copy-paste snippet. Add Mermaid diagrams + MkDocs to
render.

**Kickoff prompt**

> `v2e-docs`, branch `docs/first-time-runbook`. Write the first-time runbook per the v2e
> master plan: sectioned (Prereqs incl. Packer, Proxmox, Networking, VyOS/WAN, Access,
> Cloudflare optional, SOPS secrets flow), each section explaining its tfvars variables
> with a snippet, plus a full deploy walkthrough, Mermaid diagrams, and MkDocs.

---

### `[ ]` DOCS-2 — Day-two / troubleshooting / DR

**Branch:** `docs/day2-troubleshooting-dr` · **Depends on:** DOCS-1

Troubleshooting ("bootstrap didn't run" → cloud-init logs; "node can't reach router" →
firewall; "Traefik cert wrong" → gray-cloud/staging), day-two ops (add a node, change
VLANs, update via Semaphore), DR (state backup, age-key rotation, snapshots), and the
bake/boot/config split reference.

> DR remains **documentation-only** for now (no implementation phase scheduled). The
> age-key escrow / rotation runbook is the highest-leverage piece — under SOPS, a lost
> private key loses every secret.

**Kickoff prompt**

> `v2e-docs`, branch `docs/day2-troubleshooting-dr`. Write the day-two/troubleshooting/DR
> runbook per the v2e master plan.

---

### `[ ]` Phase F — CI/quality (cross-cutting, starts at Phase 0)

**Branch:** `feat/ci-quality` · **Repos:** all

**Tasks:** pre-commit in every repo: `tofu fmt`, `tflint`, `yamllint`, `ansible-lint`,
ShellCheck, Hadolint, gitleaks. GitHub Actions mirroring those + Trivy + Checkov. Renovate
across repos (Galaxy roles, OpenTofu providers, Docker tags, Action versions). (SOPS is
core, not part of F.)

**Kickoff prompt**

> Across the v2e repos, branch `feat/ci-quality`. Implement Phase F from the v2e master
> plan: shared pre-commit (`tofu fmt`, `tflint`, `yamllint`, `ansible-lint`, ShellCheck,
> Hadolint, gitleaks), GitHub Actions running those + Trivy + Checkov, and a Renovate
> config for Galaxy roles / OpenTofu providers / Docker tags / Actions.

---

### `[ ]` Phase G — Monitoring/observability (+ G.2 policy)

**Branch:** `feat/compose-observability` · **Repo:** `v2e-compose` · **Depends on:**
COMPOSE-1/2

**Tasks:** Prometheus, Grafana, Loki + promtail, Alertmanager, Uptime Kuma, Dozzle behind
Traefik + auth; one test alert. The Loki sink here is also the destination for **ANS-6**
agent-audit logs — add a Grafana view for the agent-action trail. G.2 (optional): OPA with
basic policies (resource limits, approved registries). CrowdSec can land here.

**Kickoff prompt**

> `v2e-compose`, branch `feat/compose-observability`. Implement Phase G from the v2e
> master plan: Prometheus + Grafana + Loki/promtail + Alertmanager + Uptime Kuma + Dozzle
> behind Traefik + auth, scraping Traefik/node/Docker, with one test alert. Wire the
> ANS-6 agent-audit logs into Loki and add a Grafana view for them. Optionally add OPA
> basic policies (G.2) and CrowdSec.

---

### `[ ]` Phase H — GHCR images

**Branch:** `feat/ghcr-images` · **Repo:** `v2e-compose` (+ Actions) · **Depends on:** F

GitHub Actions to build + push custom v2e images to GHCR on tag; Renovate tracks base
images. Harbor is the OSS self-hosted alternative for later.

**Kickoff prompt**

> `v2e-compose`, branch `feat/ghcr-images`. Implement Phase H from the v2e master plan: a
> GitHub Actions workflow building/pushing custom images to GHCR on tag, with Renovate
> tracking base images. Note Harbor as the OSS self-hosted option.

---

### `[ ]` Phase I — Agent + MCP (ground idea, own discussion)

**Branch:** `feat/agent-mcp` · **Depends on:** services stack stable

**Ground idea:** run the AI agent on the agent node via MCP servers so it works
unprivileged — no root, no raw `docker.sock`. Candidate servers (MIT): Filesystem, Git,
GitHub, Docker, PostgreSQL, Cloudflare, GitHub Actions. The agent-VLAN deny-by-default
egress allowlist (Q1) is the backstop, and **ANS-6 provides the audit trail** for whatever
the agent does. Note: per the containment decision, the existing root `claude`/`codex`
identities remain in place alongside this MCP model — flag any tension between "unprivileged
MCP" and "leftover root accounts" in the threat model. Flesh out in a dedicated
conversation: scope, per-MCP permission boundaries, agent auth, egress interaction.

**Kickoff prompt**

> New dedicated conversation, branch `feat/agent-mcp`. Design Phase I from the v2e master
> plan: run the agent on the agent node via MCP servers (Filesystem, Git, GitHub, Docker,
> PostgreSQL, Cloudflare, GitHub Actions) so it operates unprivileged — no root, no raw
> `docker.sock`. Start with a threat model + a minimal first set of MCPs, the per-MCP
> permission boundaries, and how this combines with the Q1 egress allowlist and the ANS-6
> audit trail. Call out the tension with the still-present root `claude`/`codex` accounts.

---

## 6. Tooling & open-source audit

Verdict legend: ✅ in stack · 🔵 optional/later · ⏭️ skip · 🔁 swap to OSS alternative.
Three relicensings shaped these choices: HashiCorp→BSL (2023, hit Vault & Packer),
Redis→SSPL/AGPL (2024–25), and Tailscale's closed control plane.

| Category | Tool → decision | License / note |
|---|---|---|
| **IaC** | OpenTofu ✅ · Ansible ✅ · Cloud-Init ✅ | OpenTofu MPL (chosen over BSL Terraform). |
| | Packer ✅ (caveat) | BSL — no mature fork; fine internally, document it. Fallback: libguestfs/mkosi. |
| **Containers** | Docker Engine ✅ · Compose v2 ✅ | Apache-2.0. Docker Desktop not needed on Linux. |
| | K8s/Helm/Kustomize ⏭️ | OSS; defer until scale-out. |
| **VCS/CI** | Git ✅ · GitHub Actions ✅ (free tier) · pre-commit ✅ · Renovate ✅ | OSS alt to Actions: Forgejo/Woodpecker. |
| **Secrets** | SOPS ✅ + age ✅ (D-1) | MPL / BSD. One file, three consumers. |
| | OpenBao 🔵 · Vault 🔁→OpenBao · Bitwarden SM ⏭️ | OpenBao = OSS Vault fork, only if dynamic secrets needed later. |
| **Networking** | Traefik ✅ · Cloudflared ✅ (optional) · Tailscale ✅ (free) | Traefik MIT. |
| | Headscale 🔵 · Pangolin 🔵 · Caddy ⏭️ · WireGuard (under Tailscale) | Headscale/NetBird = OSS self-host; Pangolin = OSS CF-tunnel alt. |
| **Monitoring (G)** | Prometheus ✅ · Grafana ✅ · Loki ✅ · Alertmanager ✅ · Uptime Kuma ✅ · Dozzle ✅ | All OSS (Grafana/Loki AGPL). |
| **Security** | Fail2ban ✅ · CrowdSec ✅ · Trivy ✅ · Gitleaks ✅ · Checkov ✅ · OPA 🔵 (G.2) | All OSS. |
| **AI dev** | Claude Code ✅ (primary) · Aider ✅ (refactor companion) | Aider Apache-2.0. Claude designs bulk edits → outputs Aider commands. |
| | Continue 🔵 · OpenHands 🔵 · Copilot/Cursor/Windsurf ⏭️ | Continue/OpenHands = OSS alts. |
| **CLI tools** | dev-tools Ansible role ✅ | All MIT/Apache. rg/fd/fzf/bat/eza/jq/yq/tmux/lazygit/zoxide/delta/hyperfine/tldr/dust/duf/btop/ncdu. |
| **API** | Bruno ✅ · curl ✅ · HTTPie ✅ · Postman 🔁→Bruno | Bruno MIT, offline, git-native. |
| **Docs** | Mermaid ✅ · MkDocs ✅ · draw.io/Docusaurus/PlantUML 🔵 | All OSS. |
| **Editors** | VS Code + Claude Code ✅ · Neovim 🔵 · Zed/Cursor/Windsurf ⏭️ | VSCodium = pure-OSS build. |
| **Terminal** | Ghostty ✅ · Starship ✅ · Zsh ✅ | All OSS. |
| **Testing** | Molecule/ansible-lint/yamllint ✅ · tflint ✅ · ShellCheck ✅ · Hadolint ✅ · Terratest 🔵 · pytest 🔵 | All OSS; pytest if Python enters. |
| **Registries (H)** | GHCR ✅ (free) · Harbor 🔵 · Docker Hub/Nexus/Artifactory ⏭️ | Harbor = CNCF OSS. |
| **Observability** | Alertmanager ✅ (G) · Jaeger/Tempo/OTel ⏭️ | OSS; tracing deferred. |
| **Databases** | PostgreSQL ✅ · Valkey 🔁 (if a cache is needed) · SQLite 🔵 | Valkey = BSD Redis fork (only if Authelia/cache). |
| **Platforms** | Proxmox VE ✅ · Incus/LXD 🔵 · ESXi ⏭️ | Proxmox AGPL. |

---

## 7. Appendix A — Galaxy roles & gotchas

| Need | Role/collection | Gotcha |
|---|---|---|
| Docker engine | `geerlingguy.docker` (7.x) | Standard; pair with `community.docker`. |
| Deploy compose | `community.docker` (>=3) → `docker_compose_v2` | Supported compose-from-Ansible path. |
| OS + SSH hardening | `devsec.hardening` (10.x) | Set `sftp_enabled: true` (else Ansible copy breaks). Preserve `ip_forward=1` (else Docker networking breaks). Owns `sshd_config` — retire the cloud-init drop-in. |
| Tailscale | `artis3n.tailscale` | Idempotent; authkey/OAuth; host daemon → Ansible, not compose. |
| SOPS decryption | `community.sops` | Vars plugin auto-decrypts `*.sops.yaml`; needs `sops`+`age` binaries (baked) + `SOPS_AGE_KEY_FILE`. |

---

## 8. Appendix B — Cloudflare DNS-01 / TLS

- **Token:** scoped, Zone:Read + DNS:Edit (All Zones) — not the global key. From SOPS.
- **Why DNS-01:** wildcard certs with no inbound ports — fits the outbound-only posture.
- **Wildcard:** one `*.<domain>` cert reused via `tls.domains`.
- **Gray-cloud internal records:** orange-cloud shows Cloudflare's cert, not LE. Tunnel
  hostname stays proxied (different record).
- **Staging first:** use the LE staging CA while testing, then flip to prod.

---

## 9. Appendix C — Decision log

**Resolved:**
- **D-1** — SOPS + age (replaces Ansible Vault).
- **D-5** — Packer bake split: bake `sops`/`age`/agent, keep dev-tools + hardening in
  Ansible.
- **Q3** — TinyAuth (swappable to Authelia + Valkey).
- **Containment model (2026-06-30)** — keep the `killswitch` **and** today's control→router
  SSH access as-is; do **not** retire control's router-capable keys and do **not** schedule
  a root-AI decommission. The added safeguard is **ANS-6 agent-action audit logging**, which
  gives an off-box, immutable trail of agent activity. Trade-off accepted: the killswitch
  stays reachable from a compromised control, but the audit trail provides after-the-fact
  accountability and Q1 deny-by-default egress provides the network backstop.

**Open:**
- **Q1** — egress allowlist · **Q2** — reboot (eased by Packer) · **Q4** — state backend ·
  **Q5** — Tailscale scope · **D-2** — Headscale · **D-3** — Packer-BSL accept · **D-4** —
  Pangolin · remote access (leaning Tailscale/WireGuard mesh over exposing Rustdesk).

---

## 10. Consolidation changelog

- **2026-06-30** — Consolidated the roadmap, tooling audit, and integration analysis into
  this single document. Added **ANS-6** (agent-action audit logging); added the
  existing-role **migration map** to ANS-1 and explicitly re-homed the `killswitch` /
  `vyos-hardening` / `patch` operational playbooks; resolved the **containment decision**
  (status quo + audit logging); added the **state-secret hygiene** note (§3) flagging the
  current plaintext `terraform.tfstate` secrets as burn-and-rotate. DR remains doc-only
  (DOCS-2); no root-AI decommission phase scheduled, per decision.

---

### Sources for licensing facts (§6)

HashiCorp BSL / OpenTofu / OpenBao: thenewstack.io, techtarget.com, scalr.com (2023–2026).
Redis/Valkey: redis.io, infoq.com, percona.com (2025–2026). Tailscale/Headscale:
tailscale.com/opensource, github.com/juanfont/headscale (2025–2026). Verify current
versions before pinning.
