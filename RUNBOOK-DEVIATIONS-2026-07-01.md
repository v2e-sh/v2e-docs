# v2e — First-Run Deviations & Fixes (2026-07-01)

Companion to **`RUNBOOK.md`**. This is the field log from the first *real* end-to-end
deploy of the full stack onto live hardware — everything that the runbook does **not**
say (or quietly assumes), plus the fix or workaround applied for each. Read the runbook
first; this only records the deltas.

> **Scope.** Documentation only. Nothing here has been folded back into `RUNBOOK.md`,
> the TF, Ansible, or the templates yet — several fixes are still on branches or were
> applied by hand during the run. Where a deviation is now the intended mainline design
> (Parrot control), it is flagged as such.

> **Real addresses used this run** (the runbook uses `192.168.1.x` placeholders — these
> are the concrete values behind them here):
> `mgmt net 10.10.10.0/24` · `PVE host 10.10.10.62` · `WAN net 10.11.10.0/24` ·
> `control WAN 10.11.10.25` (VyOS DNAT `:2201 → control:22`). The internal `10.1.x.x`
> addresses are unchanged from the runbook.

---

## TL;DR of what bit us

| # | Where | What bit us | Fix |
|---|---|---|---|
| 1 | Step 0 / mac | `sops` + `age` not installed (runbook assumes them) | `brew install sops age` |
| 2 | Step 2 / mac | macOS ships LibreSSL — no `openssl passwd -6` for the router hash | brew OpenSSL 3.x, or skip it (we went **key-only**) |
| 3 | Step 3a / mac | No Docker → can't run `tinyauth user create` | `htpasswd -bnBC 10` **and** rewrite `$2y$` → `$2a$` (Go bcrypt rejects `$2y$`) |
| 4 | Step 0 / host | No templates on the PVE host at all; `make` absent on Proxmox | build on the host with `bash build-*.sh`; point tfvars at staging `9900–9903` |
| 5 | Mainline | control is now **ParrotOS** (UEFI), not Ubuntu | new `parrot_template_id` + per-node `bios/machine/efi_disk` + `control_*` sizing |
| 6 | Parrot | Parrot 7.2 "echo" codename 404s cloudflared repo **and breaks all apt** | `any` suite + strip Parrot's `cloudflared.list` + make the connector best-effort |
| 7 | Access | mac can't route to the `10.11.10.x` WAN | jump through the PVE host: `ssh -J root@10.10.10.62 -p 2201 v2e@10.11.10.25` |
| 8 | Deploy | cloud-init clones the repo's **default branch, no ref** | added `ansible_repo_ref` TF var + `git clone --branch` |
| 9 | **OPEN** | `compose_stack` SOPS assert fails in the unattended multi-phase run | pass decrypted secrets as `ansible-playbook -e @<file>` extra-vars (in progress) |

**Bottom line:** infra + convergence are solid; the only thing not green is the
`compose_stack` app containers, blocked on #9.

---

# Part 1 — Prerequisites & tooling (Step 0 gaps)

## 1.1 `sops` + `age` were not installed on the mac

**Runbook says:** Step 0's `brew install opentofu git sops age` implies these are already
present (Step 3 uses `sops`/`age-keygen` with no further mention).

**What happened:** Fresh mac — neither binary existed, so Step 3 (`age-keygen`, `sops
--encrypt`) failed immediately.

**Fix:**
```bash
brew install sops age
```
Nothing else in Step 3 changes once these are on `PATH`.

## 1.2 macOS LibreSSL has no `openssl passwd -6` (router hash)

**Runbook says:** Step 2 generates the optional VyOS console password with
`ROUTER_HASH=$(openssl passwd -6 "$ROUTER_PW")`, and Appendix E notes this needs
OpenSSL 3.x. The stock macOS `openssl` is **LibreSSL**, which has no `-6` (sha-512crypt)
mode — the command errors out.

**What happened:** `openssl passwd -6` failed on the built-in LibreSSL.

**Fix / decision:** the router hash is **optional** (empty = key-only console fallback;
`router_password_hash` defaults to `""` and both mesh keys already reach the router).
We **skipped it — key-only**, leaving:
```hcl
router_password_hash = ""   # key-only; no break-glass console password this run
```
If you do want the break-glass hash, install a real OpenSSL and call it by full path so
you don't get LibreSSL back:
```bash
brew install openssl@3
ROUTER_HASH=$("$(brew --prefix openssl@3)/bin/openssl" passwd -6 "$ROUTER_PW")
```

## 1.3 No Docker on the mac → TinyAuth bcrypt by hand (`$2y$` → `$2a$`)

**Runbook says:** Step 3a mints the TinyAuth user with
`docker run --rm -it ghcr.io/steveiliop56/tinyauth:v5.0.7 user create`, producing a
`user:$2a$10$...` string.

**What happened:** Docker isn't installed on the mac, so the container path was a
non-starter. The obvious substitute — `htpasswd -bnBC 10` (Apache's `htpasswd` ships
with macOS) — emits a **`$2y$`**-prefixed bcrypt hash. TinyAuth (Go, using
`golang.org/x/crypto/bcrypt`) **rejects the `$2y$` prefix**; it only accepts `$2a$`/`$2b$`.
A `$2y$` hash silently fails auth.

**Fix:** generate with `htpasswd` and rewrite the version byte:
```bash
# ---- EDIT THESE ----
TA_USER='admin'
TA_PASS='your-strong-password'

TINYAUTH_USER=$(htpasswd -bnBC 10 "$TA_USER" "$TA_PASS" | sed 's/\$2y\$/\$2a\$/')
echo "$TINYAUTH_USER"      # admin:$2a$10$....  <- paste into secrets.yaml (Step 3b)
```
The resulting `user:$2a$...` drops straight into `tinyauth_auth_users` in the SOPS file;
everything downstream is identical to the runbook.

> **Why it matters:** the failure mode is silent — the container comes up, the hash
> "looks" like bcrypt, and login just never succeeds. The `$2y$`→`$2a$` byte is the whole
> bug. (The `$2y$` vs `$2a$` distinction is a historical crypt-vendor tag, not a different
> algorithm, so the rewrite is safe.)

## 1.4 No VM templates existed on the PVE host — build them first, `make` absent

**Runbook says:** Step 0 "Build the VM templates" reads as if templates are a quick
`make ubuntu debian` and the staging-vs-prod VMID note is the only snag. It assumes a
working build toolchain on the host.

**What happened:** the PVE host (`10.10.10.62`) had **no templates at all** — the first
`tofu apply` would clone from VMIDs that don't exist. Building them surfaced a second
problem: **`make` is not installed on Proxmox** (minimal Debian; no `build-essential`),
so `make ubuntu` / `make parrot` just error with `command not found`.

**Fix:** clone `v2e-templates` **on the host** and call the build scripts directly — the
`Makefile` targets are thin wrappers around them anyway (`ubuntu: bash build-ubuntu.sh`,
etc.), so nothing is lost:
```bash
# On the PVE host (10.10.10.62):
git clone https://github.com/v2e-sh/v2e-templates.git && cd v2e-templates
$EDITOR config.env                 # STORAGE/BRIDGE + the *_VMID / *_URL / *_SHA256
bash build-vyos.sh                 # or vyos/build-from-source.sh first (see runbook)
bash build-ubuntu.sh
bash build-debian.sh
bash build-parrot.sh               # control is Parrot now — see Part 2
```
Templates land at the **staging** VMIDs baked into `config.env`:

| Image | Staging VMID | Consumed by |
|---|---|---|
| VyOS | **9900** | router |
| Ubuntu 24.04 | **9901** | services (was: control too) |
| Debian 13 | **9902** | agent |
| ParrotOS Home 7.2 | **9903** | control |

`terraform.tfvars` must therefore point every `*_template_id` at these (the TF defaults
are the prod `9000–9003` range):
```hcl
vyos_template_id   = 9900
ubuntu_template_id = 9901
debian_template_id = 9902
parrot_template_id = 9903     # new var — see 2.1
```

---

# Part 2 — ParrotOS control node (now mainline)

The biggest structural change vs the runbook: **control is ParrotOS, not Ubuntu.** This
is intentional (control is the operator workbench), but it is not yet reflected in
`RUNBOOK.md` or the committed TF, and it drags in a chain of UEFI + apt + boot-timing
consequences.

## 2.1 control switched to ParrotOS (UEFI) — required TF changes

**Runbook / current TF says:** `network.tf` builds **both** control and services from
`ubuntu_template_id`, agent from `debian_template_id`, and all three share the generic
`node_cores` / `node_memory` / `node_disk_size` and the default SeaBIOS/i440fx machine
type. There is no notion of a per-node BIOS/machine or an EFI disk.

**What happened:** ParrotOS Home is a **desktop** image — UEFI-only, btrfs root,
NetworkManager — so cloning it under the default SeaBIOS/i440fx profile does not boot.
It needs OVMF + q35 + an EFI vars disk, and it wants more RAM than a headless node.

**Fix — TF changes applied:**
- New variable **`parrot_template_id`** (staging `9903`), and `local.nodes.control` now
  uses `template_id = var.parrot_template_id` (services stays on `ubuntu_template_id`).
- Per-node **`bios`/`machine`/`efi_disk`** so control alone gets UEFI:
  `bios = "ovmf"`, `machine = "q35"`, plus an `efi_disk` block. The Ubuntu/Debian nodes
  keep the SeaBIOS default.
- Control-specific sizing — **`control_cores` / `control_memory` / `control_disk_size`**,
  with `control_memory = 8192` (8 GiB) so the desktop image has headroom. The other
  nodes keep `node_cores` / `node_memory` / `node_disk_size`.

> **Net effect for tfvars:** in addition to `parrot_template_id = 9903`, control is now
> sized independently. Everything else in Steps 2–3 is unchanged.

## 2.2 Parrot's "echo" codename breaks the cloudflared apt repo — and *all* apt

**Runbook says:** nothing — the cloudflared connector "just installs" at first boot
(Appendix A). The node cloud-init writes the apt source with the guest's own codename:
```
deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] \
    https://pkg.cloudflare.com/cloudflared $VERSION_CODENAME main
```

**What happened:** ParrotOS 7.2 "echo" is **Debian 13 / trixie**-based, but its
`VERSION_CODENAME` is **`echo`** — a name `pkg.cloudflare.com` has never heard of. So:
1. The cloudflared apt repo **404s** for suite `echo`.
2. Worse, a bad/mis-codenamed `cloudflared.list` (including the one **Parrot ships
   pre-installed**) makes **every** `apt update` fail — Debian 13's apt uses the strict
   `sqv` signature verifier, so one unresolvable/unsigned list entry aborts the whole
   run. That takes down the baseline OS + hardening steps that depend on apt, not just
   cloudflared.

**Fix — three independent guards, because the tunnel must never break the core stack:**
1. **cloud-init uses Cloudflare's distro-agnostic `any` suite** instead of the codename:
   ```
   deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] \
       https://pkg.cloudflare.com/cloudflared any main
   ```
   `any` resolves on Parrot/trixie where `echo` does not.
2. **The template build (`build-parrot.sh`) removes the Parrot-shipped
   `/etc/apt/sources.list.d/cloudflared.list`** so it can't poison apt before cloud-init
   even runs.
3. **The connector install is best-effort in `runcmd`** (`... || true`) — the tunnel is
   **optional** (SSH-over-router already works, see 3.1), so a cloudflared hiccup must
   never fail the deploy.

> **Rule of thumb established this run:** anything cloudflared-related on control is
> decoupled and non-fatal. The tunnel is a convenience path, not a dependency of the
> stack.

## 2.3 Parrot's NetworkManager brings eth0 up late → boot-time fetches are unreliable

**Runbook says:** nothing — on the cloud-ready Ubuntu/Debian images the NIC is up before
cloud-init's package stage, so the keyring `curl` (fetching `cloudflare-main.gpg`) always
succeeds.

**What happened:** Parrot uses **NetworkManager**, which brings `eth0` up anywhere from
**~15 to ~90 s** into boot (highly variable). Any network fetch that cloud-init runs
early — notably the cloudflared **keyring download** — races that window and fails
intermittently. The existing 3× / 5 s retry loop in the node template is not enough of a
margin on a cold Parrot boot.

**Fix:** same decoupling as 2.2 — the keyring fetch and connector install are treated as
best-effort and are not on the critical path. Because the tunnel is optional, a missed
fetch on a slow-NIC boot degrades gracefully (no tunnel) instead of aborting the deploy;
a later manual/Ansible run picks it up once the network is settled.

---

# Part 3 — Access & connectivity

## 3.1 The mac can't route to the WAN — jump through the PVE host

**Runbook says:** Step 6 connects straight to control with
`ssh -p 2201 v2e@"$WAN_IP"`, assuming the mac can reach the VyOS WAN address.

**What happened:** the mac lives on the **`10.10.10.x` management** network and has **no
route to the `10.11.10.x` WAN** subnet, so `ssh -p 2201 v2e@10.11.10.25` just times out.
The PVE host, however, is dual-homed (it's `10.10.10.62` on mgmt and hosts the WAN
bridge), so it can reach the WAN.

**Fix — jump through the PVE host:**
```bash
# host key changes every rebuild — clear the stale one first (runbook Step 6):
ssh-keygen -R '[10.11.10.25]:2201'
ssh -J root@10.10.10.62 -p 2201 v2e@10.11.10.25      # v2e@control, via VyOS DNAT :2201 → :22
```
The `-p 2201` applies to the final hop (the WAN DNAT); the jump host is reached on
plain `:22`. Passwordless SSH to `root@10.10.10.62` is already required by the runbook
(Step 0 uploads snippets over it), so no new trust is needed.

**Alternative (once cloudflared is up):** the Cloudflare tunnel from Appendix A gives a
WAN-independent path with no jump host:
```bash
ssh -o ProxyCommand="cloudflared access ssh --hostname lab.v2e.sh" v2e@lab.v2e.sh
```
Given 2.2/2.3, treat this as the *second* option until the tunnel is confirmed live.

## 3.2 `v2e` sudo needs `sudo_password`; `ansible` is internal-only

**Runbook says:** implied in the connection reference, but worth stating plainly after a
real login: from control, `sudo` as **`v2e`** prompts for the **`sudo_password`** you set
in Step 2 (SSH is key-only, but sudo is not passwordless for the human account). The
**`ansible`** automation account is **internal-only** — it can't log in from outside and
has NOPASSWD sudo everywhere; use it via `sudo -iu ansible` from control, not as an SSH
target from the mac.

---

# Part 4 — Ansible / deploy mechanism

## 4.1 origin/main must carry the full phased stack; added `ansible_repo_ref`

**Runbook says:** Step 5 states control "clones `v2e-ansible`, installs its Galaxy roles,
and runs `site.yml`." Cloud-init does a **plain `git clone ${ansible_repo_url}` with no
ref**, so it always gets the repo's **default branch (`main`)**.

**What happened / requirement:** for the unattended path to work, `origin/main` of
`v2e-ansible` must already be the **full phased stack** — `site.yml` importing
`01-bootstrap` → `02-services` → `03-applications`, with the operational playbooks
(`patch`, `vyos-hardening`, `killswitch`, …) moved out to `playbooks/ops/` and **not**
imported by `site.yml`. During bring-up the work was on feature branches, and there was
no way to tell control to deploy a **non-main** branch for testing without merging to
main first.

**Fix:** added an **`ansible_repo_ref`** TF variable and changed the cloud-init clone to
`git clone --branch <ref>` so a specific branch/tag can be deployed:
```hcl
ansible_repo_url = "https://github.com/v2e-sh/v2e-ansible"
ansible_repo_ref = "main"        # or a feature branch during bring-up; blank = default branch
```
This let us iterate on a branch, verify convergence, then consolidate to `main`. The
consolidated `main` now contains the full phased `site.yml`.

## 4.2 RESOLVED — `compose_stack` SOPS assert fails in the unattended multi-phase run

**Resolved same day** — the extra-vars fix below is implemented in v2e-tf `main`
(cloud-init decrypts to `~/.v2e-secrets.yml` and appends `-e @$HOME/.v2e-secrets.yml` to
the `ansible-playbook` invocation) and validated live: the full stack deployed and the
assert passed. Kept for the record:

**Runbook says:** Step 3 / Appendix E treat the SOPS secrets as a solved problem — if
`cf_dns_api_token` and `tinyauth_auth_users` are present (lowercase) and `sops -d`
decrypts, the services phase proceeds. `compose_stack` asserts both keys are non-empty:
```
compose_stack needs cf_dns_api_token and tinyauth_auth_users from the SOPS
group_vars/all.sops.yaml ... One or both are missing/empty
```

**What happened:** during the **full unattended multi-phase** run
(`01→02→03` in one `site.yml`), that assert **fails on the services node** — both vars
come back **empty** by `compose_stack` time. This is not a decryption or lowercase-key
problem:
- The `community.sops` **demand-mode** group_vars resolve empty at that point.
- Even swapping in **plaintext** group_vars comes back empty — so it isn't SOPS itself.
- Root cause: **`geerlingguy.docker` issues `meta: reset_connection`** (it adds `ansible`
  to the `docker` group), and in the multi-phase run that connection reset **clears the
  host's resolved vars**, so `compose_stack` (which runs right after) sees them empty.
- **Manual / isolated re-runs of phase 02 resolve the vars fine** — the failure only
  shows in the single unattended `site.yml` pass.

**Partial mitigation already in tree (not sufficient):** `playbooks/02-services.yml`
pins the two secrets as host facts in `pre_tasks` *before* the roles run:
```yaml
pre_tasks:
  - name: Pin SOPS secrets as host facts
    ansible.builtin.set_fact:
      cf_dns_api_token: "{{ cf_dns_api_token }}"
      tinyauth_auth_users: "{{ tinyauth_auth_users }}"
    no_log: true
```
This survives *some* re-resolution but does **not** hold across the full unattended run —
the vars are already empty by the time `set_fact` reads them in that path.

**Robust fix (under implementation):** stop relying on re-resolvable vars entirely — pass
the **decrypted** secrets as Ansible **extra-vars**, which have the **highest precedence**
and are immune to connection resets / re-resolution:
```bash
# decrypt once, then run site.yml with -e @<file> (extra-vars win over everything):
sudo -iu ansible bash -lc '
  cd ~/ansible &&
  sops -d group_vars/all.sops.yaml > /dev/shm/secrets.decrypted.yml &&
  ansible-playbook -i inventory/hosts.ini site.yml -e @/dev/shm/secrets.decrypted.yml ;
  shred -u /dev/shm/secrets.decrypted.yml'
```
Wiring this into control's first boot (decrypt → `-e @…`) shipped in v2e-tf `main`
(`cloud-init/node.yaml.tftpl`).

---

## 4.3 Traefik never obtains the wildcard cert — the lab's WAN path intercepts `:53`

**Runbook says:** with the DNS-01 token in the SOPS file, Traefik obtains the staging
wildcard cert on first boot (Step 6 check 5); Appendix E points at token scope or egress
if it doesn't.

**What happened:** every issuance attempt failed, in two distinct ways that together
took most of a day to untangle:

1. **Not the token.** The reused tunnel-scoped token creates
   `_acme-challenge.v2e.sh` TXT records that appear on **both authoritative NS and
   1.1.1.1 within ~10 s** (verified with a live probe). A dedicated DNS-01 token is
   least-privilege hygiene, not a functional requirement.
2. **The lab's WAN path runs a caching transparent DNS interceptor.** Queries from the
   services node to `sergi.ns.cloudflare.com` are answered by a recursive cache
   (`rd ra`, no `aa` flag), not the real authoritative server. Verified live from two
   vantage points at once: the mac watched both real authoritative NS **serve the ACME
   TXT** for the full window while the services node got **NXDOMAIN from "sergi"** the
   entire time.
3. **Every attempt self-poisons.** lego's zone lookup queries `_acme-challenge.v2e.sh`
   ~1 s *before* creating the TXT; the interceptor negative-caches that NXDOMAIN for the
   SOA min TTL (**1800 s**), so every subsequent on-node propagation check — whatever
   resolver it uses — is poisoned. Local checks can *never* pass on this network.
4. The original config compounded it (`resolvers=1.1.1.1` pin → same negative-cache
   trap), and a plain `disablePropagationCheck` swap raced Let's Encrypt's validation
   (~8 s) against the ~10 s API→authoritative propagation.

**The fix (v2e-compose `main`, `traefik/compose.yml`):** skip local checks entirely and
enforce a fixed wait — lego sleeps, then hands straight to the ACME server, which
validates from *its* resolvers (unaffected by lab DNS):

```yaml
- --certificatesResolvers.staging.acme.dnsChallenge.propagation.delayBeforeChecks=60s
- --certificatesResolvers.staging.acme.dnsChallenge.propagation.disableChecks=true
```

lego v5 logs the intended behavior explicitly: *"the active propagation check is
disabled, waiting for the propagation instead (wait=60000)"* — then:
`Certificates obtained for domains [v2e.sh *.v2e.sh]` (staging CA, wildcard SAN).

> **Infra follow-up:** find and fix whatever intercepts UDP/53 on the
> `services → VyOS → PVE-NAT → home-LAN` path (PVE `dnsmasq`? router DNS hijack?).
> The DNS-1 phase (internal Technitium resolver) changes lab DNS anyway — fold the
> investigation in there.

---

# Part 5 — Status at time of writing (2026-07-01)

| Layer | State |
|---|---|
| Infrastructure (router + 3 nodes) | **Solid** — builds clean, Parrot control boots (UEFI) |
| Ansible convergence | **Solid** — control (Parrot) + agent `failed=0` |
| Cloudflare tunnel | **Active** (best-effort install held up) |
| Docker on services | **Installed** (geerlingguy.docker) |
| SSH-over-router (jump host) | **Works** (3.1) |
| App containers (traefik / tinyauth / whoami) | **Up + healthy** (4.2 extra-vars fix merged) |
| Wildcard HTTPS (staging CA) | **Issued** — `*.v2e.sh` + `v2e.sh`, TinyAuth gate verified (4.3) |
| Production cert flip (Step 7) | **Pending** — branch `v2e-ansible feat/production-certs` ready |

**One-line status:** full stack green on the staging CA; the Step 7 production flip is
the only remaining runbook step, staged and awaiting sign-off.

---

# Appendix — deviations mapped to runbook sections

| Runbook location | Deviation here |
|---|---|
| Step 0 · "Your mac" | 1.1 (`sops`/`age`), 1.3 (Docker → `htpasswd`) |
| Step 0 · "Build the VM templates" | 1.4 (`make` absent → `bash build-*.sh`, staging `9900–9903`) |
| Step 2 · router hash | 1.2 (LibreSSL, key-only) |
| Step 2 · template IDs / node model | 2.1 (`parrot_template_id`, UEFI, `control_*` sizing) |
| Step 3a · TinyAuth user | 1.3 (`$2y$` → `$2a$`) |
| Step 5 · clone + run Ansible | 4.1 (`ansible_repo_ref`), 4.2 (SOPS extra-vars, resolved) |
| Step 6 · connect to control | 3.1 (jump host), 3.2 (`v2e` sudo vs `ansible`) |
| Step 6 check 5 / Appendix E · cert never issues | 4.3 (`:53` interception → `PropagationWait`, not the token) |
| Step 7 · production flip | Part 5 (staged on `feat/production-certs`) |
| Appendix A · Cloudflare tunnel | 2.2, 2.3 (Parrot codename / NetworkManager timing) |
