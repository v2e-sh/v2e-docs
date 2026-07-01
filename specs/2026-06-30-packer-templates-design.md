# Phase 0/E — Packer Templates — Design Spec

**Date:** 2026-06-30
**Phase:** 0/E (Packer) — see `MASTER-PLAN.md` §5
**Branch:** `feat/packer-templates`
**Repo:** `v2e-packer` (new) · **Upstream of:** TF-1
**Status:** design approved — ready for implementation planning (writing-plans).

---

## 1. Goal

Produce the four Proxmox VM **templates** that `v2e-tf` clones, as reproducible
code instead of hand-run scripts. Packer codifies build processes that already work
manually today (the `vyos-build` cloud-init kit and the §0a Parrot `virt-customize`
bake); the value is making them declarative, version-pinned, and shareable.

Templates and the VMIDs `v2e-tf` expects (from `v2e-tf/variables.tf`):

| Template | VMID | Consumed by | Source style |
|---|---|---|---|
| VyOS | 9000 | router | proxmox-iso + boot_command |
| Ubuntu Server LTS | 9001 | control + services | proxmox-iso + autoinstall |
| Debian (lightweight) | 9002 | agent | proxmox-iso + preseed |
| Parrot OS Home | 9003 | control (parrot-control-workstation branch) | proxmox-iso (Debian netinst preseed) + Parrot repos |

**All four images use the single `proxmox-iso` builder** (one idiomatic mechanism, no
non-Packer `qm importdisk` bootstrap). Each boots an ISO, drives an unattended install,
provisions, and templates. ISO + autoinstall was chosen over cloud-image + `proxmox-clone`
because the latter needs a two-step bootstrap partly outside Packer.

---

## 2. Locked decisions (this phase)

- **One repo, per-OS templates.** Heterogeneous build methods (three of them) make a
  single combined template a tangle; per-OS templates build/lint/fail in isolation while
  sharing variables + a common bake script.
- **VyOS via `proxmox-iso` end-to-end** (user choice), with a documented **`vyos-build`
  fallback** if interactive-install automation proves too brittle.
- **All four images in scope**, including Parrot now (does not block the VyOS → TF-1
  critical path).
- **Parrot built from a Debian netinst install + Parrot repos** (not the official desktop
  qcow2). Parrot Home is Debian-based; do a `proxmox-iso` Debian netinst install with the
  **same preseed as 9002**, then add `parrot-archive-keyring` + Parrot repos + a Parrot
  Home/desktop metapackage via a `shell` provisioner. This keeps Parrot on the same
  `proxmox-iso` mechanism, gives it working cloud-init, and eliminates the desktop-image
  OOM and the `virt-customize` workaround. There is no official cloud-init-ready Parrot
  Proxmox image — the official QCOW2 is the full desktop disk.
- **Packer BSL accepted** (D-3) — fine for internal/shareable use; documented in README.
  No mature OSS fork exists.
- **Builds run locally (Mac → Proxmox API)** — `proxmox-iso` needs only Proxmox API
  reachability. CI/build-node deferred to a later follow-up. Reuse the existing Proxmox
  connection details from `v2e-tf/terraform.tfvars`.
- **All images via `proxmox-iso` + unattended install** (Ubuntu autoinstall, Debian
  preseed, VyOS boot_command). Chosen over cloud-image + `proxmox-clone` because that path
  is two-step and partly outside Packer (`qm importdisk` bootstrap). Cost accepted:
  authoring autoinstall/preseed files + an explicit "install + enable cloud-init for the
  NoCloud datasource" bake step so the installed system consumes Proxmox's cloud-init drive
  at runtime (the same make-or-break check VyOS has).
- **Bake/boot/config split (D-5).** Bake only static prerequisites: `qemu-guest-agent`,
  cloud-init, baseline packages on **all** images; `sops` + `age` on **node images only**.
  **Do not** bake dev-tools or hardening — those stay in Ansible (dev-tools for per-node
  flexibility; hardening because `os_hardening` clamps the sysctl Docker needs, and a bad
  bake breaks every clone).
- **VyOS does NOT get `sops` + `age`.** The router is managed by Ansible over `network_cli`
  (`vyos.vyos`); `community.sops` decrypts controller-side on `control` and pushes config.
  VyOS never decrypts anything and runs no Compose, so it gets the lean bake (cloud-init +
  agent + baseline). The "sops must exist before Ansible runs" rationale applies to
  `control`, not the router.

---

## 3. Repository structure

**Single config directory, one file per image** (Packer evaluates all `*.pkr.hcl` in a
directory together and does NOT recurse into subdirs; a single dir + `-only=<source>`
selection keeps the plugin and variable declarations DRY):

```
v2e-packer/
  README.md                 # concepts primer (builder/provisioner/bake-vs-fry), usage, prerequisites, BSL caveat, image→template→VMID chain
  .gitignore                # *.auto.pkrvars.hcl with secrets, packer_cache/, build artifacts
  plugins.pkr.hcl           # required_plugins → github.com/hashicorp/proxmox
  variables.pkr.hcl         # variable DECLARATIONS (proxmox conn, per-image ISO urls/checksums, version pins, VMIDs)
  locals.pkr.hcl            # shared locals (provisioner script lists, common source fields)
  common.pkrvars.hcl        # shared non-secret values (node, datastore, bridge, versions)
  secrets.auto.pkrvars.hcl  # NOT committed — proxmox token (gitignored; user supplies)
  ubuntu.pkr.hcl            # source proxmox-iso.ubuntu + build (autoinstall)
  debian.pkr.hcl            # source proxmox-iso.debian + build (preseed)
  vyos.pkr.hcl              # source proxmox-iso.vyos + build (boot_command; lean bake — NO sops/age)
  parrot.pkr.hcl            # source proxmox-iso.parrot + build (debian preseed + parrot-repos)
  http/
    ubuntu/   {user-data, meta-data}   # subiquity autoinstall
    debian/   preseed.cfg              # Debian netinst preseed
    parrot/   preseed.cfg              # copy of debian preseed
  scripts/
    bake-common.sh          # agent + install/enable cloud-init + baseline (ALL images, idempotent)
    bake-sops.sh            # sops + age binaries (NODE images only: ubuntu, debian, parrot)
    parrot-repos.sh         # parrot-archive-keyring + repos + Home/desktop metapackage
    vyos-install.expect     # optional: drive `install image` if boot_command is insufficient
  Makefile                  # make ubuntu|debian|vyos|parrot → packer build -only=proxmox-iso.<os> . ; make all ; make lint
```

**Shared, not duplicated:** Proxmox connection variables, `bake-common.sh`, version pins.
**Separate per image:** the builder/build mechanism, so each is independent.

---

## 4. Shared components

### 4.1 Variables (`variables.pkr.hcl`)
- Proxmox connection: `proxmox_url`, `proxmox_username`, `proxmox_token` (sensitive),
  `proxmox_node`, `proxmox_insecure_skip_tls_verify`.
- Placement: `datastore_id` (VM disk, e.g. `local-lvm`), `iso_datastore` (e.g. `local`),
  `bridge` (e.g. `vmbr0`).
- Per-image: ISO/cloud-image URL + checksum, target `vmid`, `template_name`, disk size.
- Version pins: `sops_version`, `age_version`, image/ISO versions — all as variables so
  bumps are one-line.

> Reuse the Proxmox endpoint/token/node/datastore values already in
> `v2e-tf/terraform.tfvars`. Keep secrets out of committed varfiles — put the token in the
> gitignored `secrets.auto.pkrvars.hcl` (or pass `-var`/env). This phase does **not** need
> SOPS itself (that's TF-2); it only **bakes the sops/age binaries** so they exist before
> Ansible runs.

### 4.2 Bake scripts
Idempotent shell, run as `shell` provisioners. Split so VyOS can skip sops/age:

**`scripts/bake-common.sh`** — runs on **all** images (Ubuntu, Debian, VyOS, Parrot):
- `qemu-guest-agent` (enabled) — required so Packer/Proxmox can read the VM's IP
- **install + enable cloud-init** and configure the NoCloud/ConfigDrive datasource so the
  installed system consumes Proxmox's cloud-init drive at runtime. Since every image is now
  ISO-installed (no pre-wired cloud image), this is a real step on **all** of them, not just
  VyOS — and it is the make-or-break check.
- baseline packages (curl, ca-certificates, gnupg, etc. — keep minimal)

**`scripts/bake-sops.sh`** — runs on **node images only** (Ubuntu, Debian, Parrot), **not
VyOS**:
- `sops` + `age` binaries (pinned versions, verified checksum; from GitHub releases —
  distro packages lag)

**Seal** (end of every build): truncate machine-id, remove host SSH keys,
`cloud-init clean --logs`, clear shell history — so clones get fresh identity.

### 4.3 `Makefile`
- `make ubuntu|debian|vyos|parrot` → `packer init .` + `packer build
  -only=proxmox-iso.<os> -var-file=common.pkrvars.hcl .`.
- `make all` → ubuntu, debian, vyos, parrot in order.
- `make lint` → `packer fmt -check .`, `packer validate -var-file=common.pkrvars.hcl .`,
  ShellCheck `scripts/`, Hadolint any Dockerfile.

### 4.4 Plugin
Use the **HashiCorp Packer Proxmox plugin**, `proxmox-iso` source — `source =
"github.com/hashicorp/proxmox"` in `required_plugins`. **Correct the master plan's
`bpg/packer-proxmox` reference** — bpg publishes the *Terraform* provider, not the Packer
plugin. Pin the plugin version during implementation. Packer serves the autoinstall/preseed
files to the booting VM via its built-in HTTP server (`http_directory` / the `{{ .HTTPIP }}`
+ `{{ .HTTPPort }}` template vars in `boot_command`).

---

## 5. Per-image design

### 5.1 Ubuntu (9001) — build FIRST
**Method:** `proxmox-iso` + autoinstall. Source = Ubuntu Server LTS **live-server ISO**
(URL + checksum pinned). `boot_command` points the installer at Packer's HTTP-served
`http/user-data` (subiquity autoinstall). After install + reboot, SSH in and run
`bake-common.sh` + `bake-sops.sh`, seal, convert to template 9001.

**Why first:** proves the Proxmox plugin, connection vars, autoinstall HTTP serving, and
`bake-common.sh` on the most-documented path before touching VyOS.

**Acceptance:** template 9001 exists; a clone boots, `cloud-init status` is `done`,
consumes Proxmox cloud-init drive user-data, and `qemu-guest-agent` + `sops` + `age` are
present.

### 5.2 Debian (9002)
**Method:** `proxmox-iso` + preseed. Source = Debian **netinst ISO** (URL + checksum
pinned); `boot_command` points the installer at Packer's HTTP-served `http/preseed.cfg`.
After install + reboot, run `bake-common.sh` + `bake-sops.sh`, seal, template 9002. Reuses
the bake scripts verbatim. **The same preseed is reused by Parrot (9003).**

**Acceptance:** same as Ubuntu, for 9002.

### 5.3 VyOS (9000) — the risky one
**Method (chosen):** `proxmox-iso` end-to-end.
1. Boot the VyOS ISO in a Proxmox build VM.
2. Drive the interactive `install image` via `boot_command` (VNC keystroke injection);
   default creds `vyos`/`vyos`. (If `boot_command` alone is insufficient, fall back to an
   `expect` script over SSH — `scripts/vyos-install.expect`.)
3. After install + reboot, SSH in and **install + enable cloud-init** — this is the step
   that fixes the documented trap: *the stock VyOS ISO ships no cloud-init, so an
   ISO-install image silently ignores user-data.*
4. Run `bake-common.sh` (agent + cloud-init + baseline — **no sops/age**), seal, convert
   to template 9000.

**Clean NIC naming:** a cloud-init-built VyOS image clones with `eth0`/`eth1` (no `hw-id`
pin); preserve that so the `v2e-tf` interface-name assumptions hold (`vyos_wan_iface=eth0`,
`vyos_lan_iface=eth1`).

**Documented fallback:** if interactive automation is too brittle, wrap the existing
`vyos-build` flow (kit at `~/Documents/vyos/cloudinit-image/`) — run `vyos-build` (Docker)
to produce the cloud-init qcow2, then import + template it. This needs Docker on the build
host; note it in the README as the alternate path.

**Acceptance:** template 9000 exists; a clone boots, **cloud-init runs and applies
user-data** (the make-or-break check), NICs are `eth0`/`eth1`, `qemu-guest-agent` running.
No sops/age expected on the router.

### 5.4 Parrot (9003) — Debian netinst install + Parrot repos
**Method:** `proxmox-iso` Debian netinst install using the **same preseed as 9002**, then
convert Debian → Parrot Home in a `shell` provisioner. No official Parrot desktop qcow2, no
`virt-customize`, no no-boot hack — same `proxmox-iso` mechanism as the others, with working
cloud-init inherited from the Debian install.

Steps:
1. `proxmox-iso` boots the **Debian netinst ISO** with the 9002 preseed, into a build VM
   sized with enough RAM/disk for a desktop install (the old OOM came from booting Parrot's
   *desktop* qcow2 with too little RAM; here we control the build VM's resources).
2. `bake-common.sh` + `bake-sops.sh` (agent, cloud-init, baseline, sops/age).
3. `parrot-repos.sh` — install `parrot-archive-keyring`, add Parrot's APT repos, `apt
   update`, install the Parrot Home/desktop metapackage(s) + tools. Pin the repo suite
   explicitly (don't rely on Debian's codename mapping). Keep the cloudflared-repo
   `bookworm` pin lesson in mind if cloudflared is added later.
4. Ensure the cloud-init network renderer works with the resulting desktop (Parrot Home
   uses NetworkManager — set the NoCloud/NetworkManager renderer so the cloned NIC comes
   up). Seal, convert to template 9003.

**Exact metapackage set + repo/keyring steps are an open item (§10)** — confirm against
Parrot's "install on Debian" docs during implementation.

**Acceptance:** template 9003 exists; a clone boots cleanly (no manual network, no OOM),
cloud-init applies user-data, the Parrot desktop + tools are present, and `qemu-guest-agent`
+ `sops` + `age` are present.

---

## 6. Build order & critical path

```
ubuntu (9001) ──► debian (9002) ──► parrot (9003)   [parrot reuses the Debian preseed + repos]
        │
        ▼
vyos (9000)  ──────────────────────────► TF-1 (critical path)
```

Build Ubuntu first to prove the `proxmox-iso` + autoinstall + HTTP-serving + bake machinery
on the best-documented path; Debian reuses it via preseed; Parrot reuses the Debian preseed
and layers Parrot repos. Critical path to unblock TF-1 = **VyOS template** (the hardest
`boot_command`). Parrot is off the critical path.

---

## 7. Risks & mitigations

| Risk | Mitigation |
|---|---|
| VyOS `install image` resists `boot_command` automation | `expect`-over-SSH variant; ultimately the `vyos-build` Docker fallback. |
| VyOS image ends up without working cloud-init | Explicit install+enable step in 5.3; acceptance gate verifies a clone consumes user-data. |
| Parrot first-boot OOM (old desktop-qcow2 path) | Eliminated — Parrot is a Debian netinst install; size the build VM's RAM/disk for the desktop install. |
| Wrong Parrot metapackage/repo set → broken or non-faithful Home spin | Follow Parrot's official "install on Debian" docs; pin repo suite + keyring; verify desktop + tools on a clone (§10). |
| Autoinstall/preseed `boot_command` timing or syntax flaky | Iterate against Packer's HTTP-served files; well-documented for Ubuntu subiquity / Debian preseed; verify install completes unattended. |
| ISO-installed system doesn't consume Proxmox cloud-init drive | `bake-common.sh` installs + enables cloud-init with the NoCloud/ConfigDrive datasource; acceptance gate verifies a clone applies user-data. |
| Wrong/missing Packer Proxmox plugin name | Pin `source = "github.com/hashicorp/proxmox"` + version in `required_plugins` (§4.4). |
| Secrets (Proxmox token) leaking into git | Gitignored `secrets.auto.pkrvars.hcl`; `.gitignore` + gitleaks (Phase F). |
| ISO URLs / checksums drift | Pin version + checksum as variables; Renovate tracks later (Phase F/H). |

---

## 8. Out of scope (explicit)

- Dev-tools, OS/SSH hardening (Ansible — ANS-2).
- SOPS *usage*/secret material (TF-2 / Ansible) — this phase only bakes the binaries.
- CI for the Packer repo (Phase F adds pre-commit + Actions; build-node deferred).
- GHCR/registry work (Phase H).

---

## 9. Acceptance (phase-level)

1. `make all` produces templates 9000/9001/9002/9003 on the Proxmox host.
2. For **each** template, a cloned VM:
   - boots unattended,
   - reaches `cloud-init status: done` and **applies user-data**,
   - has `qemu-guest-agent` running.
   - **Node templates only** (Ubuntu/Debian/Parrot) have `sops` + `age` on `PATH`; VyOS
     does **not** (by design).
3. VyOS clone shows `eth0`/`eth1`. Parrot clone has the desktop + Parrot tools.
4. `make lint` passes (fmt, validate, ShellCheck, Hadolint).
5. README documents prerequisites, the image→template→VMID chain, the BSL caveat, and the
   `vyos-build` fallback.

---

## 10. Open items to resolve during planning/implementation

- Pin the HashiCorp Packer Proxmox plugin version (`source = "github.com/hashicorp/proxmox"`).
- Pinned **ISO** versions + checksums for the three sources: Ubuntu Server LTS live-server
  ISO, Debian netinst ISO (also the Parrot base), VyOS rolling ISO.
- Ubuntu autoinstall `user-data` and Debian `preseed.cfg` content (minimal: user, SSH,
  disk, locale) + the matching `boot_command` for each installer.
- Parrot: exact `parrot-archive-keyring` + repo suite + Home/desktop metapackage set, per
  Parrot's official "install on Debian" docs; plus the NetworkManager/NoCloud renderer
  setting for the cloned NIC.
- VyOS `boot_command` keystroke sequence vs `expect` script — decide after a first manual
  install transcript.
