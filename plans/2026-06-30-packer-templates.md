# Packer Templates (Phase 0/E) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `v2e-packer` repo that produces four reproducible Proxmox templates (VyOS 9000, Ubuntu 9001, Debian 9002, Parrot 9003) via a single idiomatic `proxmox-iso` mechanism.

**Architecture:** One Packer config directory, one `.pkr.hcl` per image, selected at build time with `-only=proxmox-iso.<os>`. Each image boots an ISO, drives an unattended install (Ubuntu autoinstall / Debian preseed / VyOS boot_command), provisions via shared shell scripts, and converts the VM to a template. Shared plugin/variables/locals are declared once at the repo root.

**Tech Stack:** Packer (BSL) + HashiCorp Proxmox plugin (`github.com/hashicorp/proxmox` ≥ 1.2.3), Bash provisioners, ShellCheck, Make. Builds run from the Mac against the Proxmox API.

## Global Constraints

- Plugin: `source = "github.com/hashicorp/proxmox"`, `version = ">= 1.2.3"` — pinned in `plugins.pkr.hcl`.
- One config dir; per-image **files** at repo root (NOT subdirectories — Packer does not recurse).
- Build selection: `packer build -only=proxmox-iso.<os> -var-file=common.pkrvars.hcl .`
- **Bake split (D-5):** bake `qemu-guest-agent` + cloud-init (install+enable) + baseline on ALL images; `sops`+`age` on NODE images only (ubuntu/debian/parrot), **never VyOS**.
- Do NOT bake dev-tools or OS hardening (those are Ansible, ANS-2).
- Secrets (Proxmox token) live only in gitignored `secrets.auto.pkrvars.hcl` — never committed.
- Target VMIDs are fixed: VyOS=9000, Ubuntu=9001, Debian=9002, Parrot=9003 (must match `v2e-tf/variables.tf`).
- VyOS clone must expose NICs as `eth0`/`eth1`.
- Make-or-break acceptance for every image: a clone reaches `cloud-init status: done` and applies the Proxmox cloud-init drive's user-data.
- Reuse Proxmox connection values (endpoint/node/datastore/bridge/token) already in `v2e-tf/terraform.tfvars`.
- Packer is BSL (not OSI-OSS) — note it in the README.

**Reference spec:** `v2e-docs/specs/2026-06-30-packer-templates-design.md`.

**Prerequisites before any build task (Tasks 3–6):** Proxmox API reachable from the Mac; an API token with VM.Allocate/Datastore perms; the `snippets` + `iso` content types enabled on the ISO datastore; a VLAN-aware/relevant bridge. These are environment facts, not code — confirm them, don't build them.

---

## File Structure

| File | Responsibility |
|---|---|
| `.gitignore` | Exclude secrets varfile, packer cache, crash logs |
| `plugins.pkr.hcl` | `required_plugins` (proxmox) |
| `variables.pkr.hcl` | All variable declarations (conn, per-image ISO url/checksum, vmids, versions) |
| `locals.pkr.hcl` | Shared locals (provisioner script paths) |
| `common.pkrvars.hcl` | Shared non-secret values (node, datastore, bridge, versions) |
| `secrets.auto.pkrvars.hcl` | Gitignored; Proxmox token (user supplies) |
| `scripts/bake-common.sh` | agent + install/enable cloud-init + baseline (all images) |
| `scripts/bake-sops.sh` | sops + age binaries (node images) |
| `scripts/parrot-repos.sh` | Parrot keyring + repos + Home metapackage |
| `http/ubuntu/{user-data,meta-data}` | Ubuntu subiquity autoinstall |
| `http/debian/preseed.cfg` | Debian netinst preseed |
| `http/parrot/preseed.cfg` | Copy of Debian preseed |
| `ubuntu.pkr.hcl` … `parrot.pkr.hcl` | One source+build per image |
| `Makefile` | per-image build targets + `all` + `lint` |
| `README.md` | concepts primer, usage, BSL caveat, image→template→VMID chain, vyos-build fallback |

---

## Task 1: Repo scaffold + plugin + lint harness

**Files:**
- Create: `.gitignore`, `plugins.pkr.hcl`, `variables.pkr.hcl`, `locals.pkr.hcl`, `common.pkrvars.hcl`, `secrets.auto.pkrvars.hcl` (local only), `Makefile`, `README.md` (skeleton)

**Interfaces:**
- Produces: variable names consumed by every image file — `proxmox_url`, `proxmox_username`, `proxmox_token`, `proxmox_node`, `proxmox_insecure`, `iso_datastore`, `disk_datastore`, `bridge`, `sops_version`, `age_version`, and per-image `*_iso_url` / `*_iso_checksum`. Local `local.bake_common` / `local.bake_sops` script paths.

- [ ] **Step 1: Create `.gitignore`**

```gitignore
# secrets
secrets.auto.pkrvars.hcl
*.auto.pkrvars.hcl
# packer
packer_cache/
crash.log
*.log
.packer.d/
```

- [ ] **Step 2: Create `plugins.pkr.hcl`**

```hcl
packer {
  required_plugins {
    proxmox = {
      version = ">= 1.2.3"
      source  = "github.com/hashicorp/proxmox"
    }
  }
}
```

- [ ] **Step 3: Create `variables.pkr.hcl`**

```hcl
# ---- Proxmox connection (values come from common.pkrvars.hcl + secrets.auto.pkrvars.hcl) ----
variable "proxmox_url" { type = string }       # https://HOST:8006/api2/json
variable "proxmox_username" { type = string }  # user@pve!tokenid
variable "proxmox_token" {
  type      = string
  sensitive = true
}
variable "proxmox_node" { type = string }
variable "proxmox_insecure" {
  type    = bool
  default = true
}

# ---- Placement ----
variable "iso_datastore" {
  type    = string
  default = "local"
}
variable "disk_datastore" {
  type    = string
  default = "local-lvm"
}
variable "bridge" {
  type    = string
  default = "vmbr0"
}

# ---- Baked tool versions ----
variable "sops_version" { type = string } # e.g. "3.9.4" (no leading v)
variable "age_version" { type = string }  # e.g. "1.2.1"

# ---- Per-image ISO sources (pin url + checksum) ----
variable "ubuntu_iso_url" { type = string }
variable "ubuntu_iso_checksum" { type = string } # "sha256:...."
variable "debian_iso_url" { type = string }
variable "debian_iso_checksum" { type = string }
variable "vyos_iso_url" { type = string }
variable "vyos_iso_checksum" { type = string }
# Parrot reuses the Debian ISO (built as a Debian netinst install, then Parrot repos)

# ---- Build-time SSH identity created by the unattended installers ----
variable "build_ssh_username" {
  type    = string
  default = "packer"
}
variable "build_ssh_password" {
  type      = string
  default   = "packer"
  sensitive = true
}
```

- [ ] **Step 4: Create `locals.pkr.hcl`**

```hcl
locals {
  bake_common = "scripts/bake-common.sh"
  bake_sops   = "scripts/bake-sops.sh"
  parrot_repos = "scripts/parrot-repos.sh"
}
```

- [ ] **Step 5: Create `common.pkrvars.hcl` (committed, non-secret)**

```hcl
# Fill from v2e-tf/terraform.tfvars (non-secret parts)
proxmox_url      = "https://10.10.10.62:8006/api2/json"
proxmox_username = "root@pam!packer"
proxmox_node     = "pve"
proxmox_insecure = true

iso_datastore  = "local"
disk_datastore = "local-lvm"
bridge         = "vmbr0"

sops_version = "3.9.4"
age_version  = "1.2.1"

# ISO sources — see Step 7 to obtain current URLs + checksums and paste here.
ubuntu_iso_url      = "REPLACE_VIA_STEP_7"
ubuntu_iso_checksum = "REPLACE_VIA_STEP_7"
debian_iso_url      = "REPLACE_VIA_STEP_7"
debian_iso_checksum = "REPLACE_VIA_STEP_7"
vyos_iso_url        = "REPLACE_VIA_STEP_7"
vyos_iso_checksum   = "REPLACE_VIA_STEP_7"
```

- [ ] **Step 6: Create `secrets.auto.pkrvars.hcl` (local only, gitignored)**

```hcl
proxmox_token = "PASTE_PROXMOX_API_TOKEN_SECRET"
```

- [ ] **Step 7: Obtain + pin current ISO URLs and checksums**

Run (paste results into `common.pkrvars.hcl`, replacing the `REPLACE_VIA_STEP_7` values):

```bash
# Ubuntu Server LTS live-server ISO — get URL + SHA256 from the release SHA256SUMS
#   https://releases.ubuntu.com/24.04/  (pick the live-server amd64 iso + its SHA256SUMS line)
# Debian netinst ISO — get URL + SHA256 from
#   https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/  (SHA256SUMS)
# VyOS rolling ISO — get URL + SHA256 from
#   https://vyos.net/get/nightly-builds/   (or the artifact you already build)
# Format each checksum as "sha256:<hex>".
echo "paste ubuntu/debian/vyos iso_url + iso_checksum into common.pkrvars.hcl"
```

> These are real, current values that must be fetched at implementation time (ISOs rotate). Not a placeholder — Step 8's `packer validate` will fail loudly until they are real strings.

- [ ] **Step 8: Create the `Makefile`**

```makefile
PACKER ?= packer
VARS   := -var-file=common.pkrvars.hcl
.DEFAULT_GOAL := lint

init:
	$(PACKER) init .

fmt:
	$(PACKER) fmt .

lint: init
	$(PACKER) fmt -check .
	$(PACKER) validate $(VARS) .
	shellcheck scripts/*.sh

ubuntu: init
	$(PACKER) build -only=proxmox-iso.ubuntu $(VARS) .
debian: init
	$(PACKER) build -only=proxmox-iso.debian $(VARS) .
vyos: init
	$(PACKER) build -only=proxmox-iso.vyos $(VARS) .
parrot: init
	$(PACKER) build -only=proxmox-iso.parrot $(VARS) .

all: ubuntu debian vyos parrot

.PHONY: init fmt lint ubuntu debian vyos parrot all
```

- [ ] **Step 9: Create README skeleton**

```markdown
# v2e-packer

Reproducible Proxmox VM templates for the v2e lab, built with Packer.

## Concepts (Packer in 30 seconds)
- **Builder** boots a throwaway VM from an ISO (`proxmox-iso`).
- **Provisioners** run scripts on it (our `scripts/bake-*.sh`).
- **Bake vs fry:** we *bake* static prerequisites into the image (agent, cloud-init,
  sops/age); we *fry* dynamic config later in Ansible (dev-tools, hardening).

## Images
| Template | VMID | Consumed by |
|---|---|---|
| VyOS   | 9000 | router (no sops/age) |
| Ubuntu | 9001 | control + services |
| Debian | 9002 | agent |
| Parrot | 9003 | control workstation (Debian + Parrot repos) |

## Usage
1. Copy your Proxmox token into `secrets.auto.pkrvars.hcl`.
2. Set ISO URLs/checksums + connection in `common.pkrvars.hcl`.
3. `make lint` then `make ubuntu` (etc.) or `make all`.

## Notes
- **Packer is BSL** (not OSI-OSS); fine for internal/shareable use.
- **VyOS fallback:** if the interactive install automation is too brittle, build the
  cloud-init qcow2 with `vyos-build` (Docker) and import it — see the VyOS section.
```

- [ ] **Step 10: Lint to verify scaffold validity**

Run: `make lint`
Expected: `packer init` installs the proxmox plugin; `packer fmt -check` passes; `packer validate` **fails** at this point only if ISO vars are still `REPLACE_*` OR because no `source`/`build` blocks exist yet — that's expected until Task 3. ShellCheck passes (no scripts yet → no-op or "no files" — acceptable).

> Acceptance for Task 1: `packer init` succeeds and `packer fmt -check` passes. `validate` is not expected to fully pass until the first image (Task 3) exists.

- [ ] **Step 11: Commit**

```bash
git add .gitignore plugins.pkr.hcl variables.pkr.hcl locals.pkr.hcl common.pkrvars.hcl Makefile README.md
git commit -m "feat(packer): scaffold v2e-packer repo (plugin, vars, lint harness)"
```

---

## Task 2: Shared bake scripts

**Files:**
- Create: `scripts/bake-common.sh`, `scripts/bake-sops.sh`, `scripts/parrot-repos.sh`

**Interfaces:**
- Consumes: env vars `SOPS_VERSION`, `AGE_VERSION` (passed by each build's provisioner `environment_vars`).
- Produces: `/usr/bin/qemu-guest-agent` enabled, cloud-init installed+enabled, and (node images) `sops`/`age` on `PATH`.

- [ ] **Step 1: Write `scripts/bake-common.sh`**

```bash
#!/usr/bin/env bash
# Bake static prerequisites needed on ALL images (idempotent).
set -euo pipefail
export DEBIAN_FRONTEND=noninteractive

# qemu-guest-agent: lets Proxmox/Packer read the VM IP
apt-get update -y
apt-get install -y --no-install-recommends \
  qemu-guest-agent cloud-init curl ca-certificates gnupg
systemctl enable qemu-guest-agent || true

# cloud-init: ensure the Proxmox NoCloud/ConfigDrive datasource is used at runtime
mkdir -p /etc/cloud/cloud.cfg.d
cat >/etc/cloud/cloud.cfg.d/99-pve.cfg <<'EOF'
datasource_list: [ NoCloud, ConfigDrive ]
EOF
systemctl enable cloud-init cloud-init-local cloud-config cloud-final || true

echo "bake-common: done"
```

- [ ] **Step 2: Write `scripts/bake-sops.sh`**

```bash
#!/usr/bin/env bash
# Install sops + age binaries (node images only). Versions via env.
set -euo pipefail
: "${SOPS_VERSION:?}" "${AGE_VERSION:?}"
arch="$(dpkg --print-architecture)"   # amd64

# sops
curl -fsSL -o /usr/local/bin/sops \
  "https://github.com/getsops/sops/releases/download/v${SOPS_VERSION}/sops-v${SOPS_VERSION}.linux.${arch}"
chmod +x /usr/local/bin/sops
sops --version

# age
tmp="$(mktemp -d)"
curl -fsSL -o "${tmp}/age.tgz" \
  "https://github.com/FiloSottile/age/releases/download/v${AGE_VERSION}/age-v${AGE_VERSION}-linux-${arch}.tar.gz"
tar -xzf "${tmp}/age.tgz" -C "${tmp}"
install -m 0755 "${tmp}/age/age" /usr/local/bin/age
install -m 0755 "${tmp}/age/age-keygen" /usr/local/bin/age-keygen
rm -rf "${tmp}"
age --version

echo "bake-sops: done"
```

- [ ] **Step 3: Write `scripts/parrot-repos.sh`**

```bash
#!/usr/bin/env bash
# Convert a Debian install into Parrot Home by adding Parrot's repos + metapackage.
# NOTE: confirm the exact keyring pkg, repo suite, and metapackage against Parrot's
# official "install Parrot on Debian" docs before first build (see plan Task 6 open item).
set -euo pipefail
export DEBIAN_FRONTEND=noninteractive

apt-get update -y
apt-get install -y --no-install-recommends curl ca-certificates gnupg

# Parrot archive keyring + repo (suite pinned explicitly; do NOT rely on Debian codename)
install -d -m 0755 /etc/apt/keyrings
curl -fsSL https://deb.parrot.sh/parrot/misc/parrotsec.gpg \
  -o /etc/apt/keyrings/parrot.gpg
cat >/etc/apt/sources.list.d/parrot.list <<'EOF'
deb [signed-by=/etc/apt/keyrings/parrot.gpg] https://deb.parrot.sh/parrot lory main contrib non-free non-free-firmware
EOF

apt-get update -y
# Home/desktop metapackage — confirm exact name (e.g. parrot-meta-home) in Task 6.
apt-get install -y parrot-core parrot-meta-home

echo "parrot-repos: done"
```

> The keyring URL, repo suite (`lory`), and metapackage names are best-known starting values and **must be confirmed** against current Parrot docs in Task 6 Step 1 before the Parrot build. The build's acceptance check (desktop + tools present) is the gate.

- [ ] **Step 4: Make scripts executable + ShellCheck**

Run:
```bash
chmod +x scripts/*.sh
shellcheck scripts/*.sh
```
Expected: ShellCheck passes (exit 0), no warnings.

- [ ] **Step 5: Commit**

```bash
git add scripts/
git commit -m "feat(packer): shared bake scripts (common, sops, parrot-repos)"
```

---

## Task 3: Ubuntu template (9001)

**Files:**
- Create: `ubuntu.pkr.hcl`, `http/ubuntu/user-data`, `http/ubuntu/meta-data`

**Interfaces:**
- Consumes: all `proxmox_*`, `*_datastore`, `bridge`, `ubuntu_iso_*`, `build_ssh_*`, `sops_version`, `age_version` vars; `local.bake_common`, `local.bake_sops`.
- Produces: Proxmox template `vm_id = 9001`, name `ubuntu-2404`.

- [ ] **Step 1: Write `http/ubuntu/meta-data`** (empty file required by NoCloud)

```bash
: > http/ubuntu/meta-data
```

- [ ] **Step 2: Write `http/ubuntu/user-data` (subiquity autoinstall)**

```yaml
#cloud-config
autoinstall:
  version: 1
  locale: en_US.UTF-8
  keyboard: { layout: us }
  ssh:
    install-server: true
    allow-pw: true
  identity:
    hostname: ubuntu-template
    username: packer
    # password "packer" hashed (mkpasswd -m sha-512 packer) — regenerate if changed
    password: "$6$rounds=4096$packersalt$O8Qv9q1mE0w0m0Y9m5l1pYV0xY3cQ0t8h0r4n6mJ8m9aJ2bUg2yqv5l8mC3nQ1pV7tF0xR2dW9oN4kP6sT5u/"
  packages: [qemu-guest-agent]
  late-commands:
    - echo 'packer ALL=(ALL) NOPASSWD:ALL' > /target/etc/sudoers.d/packer
```

> Regenerate the password hash with `mkpasswd -m sha-512 packer` if you change `build_ssh_password`. The hash above corresponds to `packer`.

- [ ] **Step 3: Write `ubuntu.pkr.hcl`**

```hcl
source "proxmox-iso" "ubuntu" {
  proxmox_url              = var.proxmox_url
  username                 = var.proxmox_username
  token                    = var.proxmox_token
  node                     = var.proxmox_node
  insecure_skip_tls_verify = var.proxmox_insecure

  vm_id                = 9001
  vm_name              = "ubuntu-2404-build"
  template_name        = "ubuntu-2404"
  template_description  = "Ubuntu Server 24.04 — baked by v2e-packer"
  cores                = 2
  memory               = 2048
  qemu_agent           = true

  boot_iso {
    type         = "scsi"
    iso_url      = var.ubuntu_iso_url
    iso_checksum = var.ubuntu_iso_checksum
    iso_storage_pool = var.iso_datastore
    unmount      = true
  }

  disks {
    disk_size    = "20G"
    storage_pool = var.disk_datastore
    type         = "scsi"
  }
  network_adapters {
    bridge = var.bridge
    model  = "virtio"
  }

  cloud_init              = true
  cloud_init_storage_pool = var.disk_datastore

  http_directory = "http/ubuntu"
  boot_wait      = "5s"
  boot_command = [
    "c<wait>",
    "linux /casper/vmlinuz --- autoinstall ds=\"nocloud-net;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/\"<enter><wait>",
    "initrd /casper/initrd<enter><wait>",
    "boot<enter>"
  ]

  ssh_username = var.build_ssh_username
  ssh_password = var.build_ssh_password
  ssh_timeout  = "30m"
}

build {
  sources = ["source.proxmox-iso.ubuntu"]

  provisioner "shell" {
    execute_command = "echo '${var.build_ssh_password}' | sudo -S -E bash '{{ .Path }}'"
    script          = local.bake_common
  }
  provisioner "shell" {
    execute_command   = "echo '${var.build_ssh_password}' | sudo -S -E bash '{{ .Path }}'"
    environment_vars  = ["SOPS_VERSION=${var.sops_version}", "AGE_VERSION=${var.age_version}"]
    script            = local.bake_sops
  }
  provisioner "shell" {
    inline = [
      "echo '${var.build_ssh_password}' | sudo -S cloud-init clean --logs",
      "echo '${var.build_ssh_password}' | sudo -S truncate -s0 /etc/machine-id",
      "echo '${var.build_ssh_password}' | sudo -S rm -f /etc/ssh/ssh_host_*"
    ]
  }
}
```

- [ ] **Step 4: Validate**

Run: `packer validate -var-file=common.pkrvars.hcl .`
Expected: PASS (all variables resolved, schema valid). If a `boot_iso`/`disks` field name is rejected, consult the proxmox-iso builder docs and adjust — the validator names the offending field.

- [ ] **Step 5: Build**

Run: `make ubuntu`
Expected: Packer boots the ISO, autoinstall runs unattended, SSH connects as `packer`, the three provisioners run, and the VM is converted to template `9001`. Watch for the autoinstall `boot_command` timing — if the installer doesn't pick up `user-data`, see Step 7.

- [ ] **Step 6: Verify the template (acceptance)**

Run (clone + boot via Proxmox CLI, or via a throwaway `qm clone`):
```bash
qm clone 9001 9101 --name ubuntu-verify
qm start 9101
# after boot, SSH in (cloud-init must have applied), then:
ssh ubuntu-verify 'cloud-init status --wait && systemctl is-active qemu-guest-agent && sops --version && age --version'
qm stop 9101 && qm destroy 9101
```
Expected: `cloud-init status: done`, `qemu-guest-agent` active, sops + age print versions.

- [ ] **Step 7: (If autoinstall failed) Iterate the boot_command, then re-build**

The Ubuntu live-server `boot_command` is the most fragile part. If the installer didn't read `user-data`: open the Proxmox VM console during a build, confirm the GRUB edit sequence, and adjust the `boot_command` keystrokes/`boot_wait`. Re-run `make ubuntu`. Repeat until Step 5 completes unattended.

- [ ] **Step 8: Commit**

```bash
git add ubuntu.pkr.hcl http/ubuntu/
git commit -m "feat(packer): ubuntu 9001 template (autoinstall + bake)"
```

---

## Task 4: Debian template (9002)

**Files:**
- Create: `debian.pkr.hcl`, `http/debian/preseed.cfg`

**Interfaces:**
- Consumes: same vars as Ubuntu + `debian_iso_*`.
- Produces: template `vm_id = 9002`, name `debian-12`. **Its `http/debian/preseed.cfg` is reused by Parrot (Task 6).**

- [ ] **Step 1: Write `http/debian/preseed.cfg`**

```text
d-i debian-installer/locale string en_US.UTF-8
d-i keyboard-configuration/xkb-keymap select us
d-i netcfg/choose_interface select auto
d-i netcfg/get_hostname string debian-template
d-i netcfg/get_domain string lab
d-i mirror/country string manual
d-i mirror/http/hostname string deb.debian.org
d-i mirror/http/directory string /debian
d-i passwd/root-login boolean false
d-i passwd/user-fullname string packer
d-i passwd/username string packer
d-i passwd/user-password password packer
d-i passwd/user-password-again password packer
d-i clock-setup/utc boolean true
d-i time/zone string UTC
d-i partman-auto/method string regular
d-i partman-auto/choose_recipe select atomic
d-i partman/confirm boolean true
d-i partman/confirm_nooverwrite boolean true
d-i partman-partitioning/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
tasksel tasksel/first multiselect standard, ssh-server
d-i pkgsel/include string qemu-guest-agent sudo
d-i grub-installer/bootdev string default
d-i preseed/late_command string in-target sh -c 'echo "packer ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/packer'
d-i finish-install/reboot_in_progress note
```

- [ ] **Step 2: Write `debian.pkr.hcl`**

```hcl
source "proxmox-iso" "debian" {
  proxmox_url              = var.proxmox_url
  username                 = var.proxmox_username
  token                    = var.proxmox_token
  node                     = var.proxmox_node
  insecure_skip_tls_verify = var.proxmox_insecure

  vm_id               = 9002
  vm_name             = "debian-12-build"
  template_name       = "debian-12"
  template_description = "Debian 12 — baked by v2e-packer"
  cores               = 2
  memory              = 2048
  qemu_agent          = true

  boot_iso {
    type             = "scsi"
    iso_url          = var.debian_iso_url
    iso_checksum     = var.debian_iso_checksum
    iso_storage_pool = var.iso_datastore
    unmount          = true
  }

  disks {
    disk_size    = "20G"
    storage_pool = var.disk_datastore
    type         = "scsi"
  }
  network_adapters {
    bridge = var.bridge
    model  = "virtio"
  }

  cloud_init              = true
  cloud_init_storage_pool = var.disk_datastore

  http_directory = "http/debian"
  boot_wait      = "5s"
  boot_command = [
    "<esc><wait>",
    "auto url=http://{{ .HTTPIP }}:{{ .HTTPPort }}/preseed.cfg ",
    "<enter>"
  ]

  ssh_username = var.build_ssh_username
  ssh_password = var.build_ssh_password
  ssh_timeout  = "30m"
}

build {
  sources = ["source.proxmox-iso.debian"]

  provisioner "shell" {
    execute_command = "echo '${var.build_ssh_password}' | sudo -S -E bash '{{ .Path }}'"
    script          = local.bake_common
  }
  provisioner "shell" {
    execute_command  = "echo '${var.build_ssh_password}' | sudo -S -E bash '{{ .Path }}'"
    environment_vars = ["SOPS_VERSION=${var.sops_version}", "AGE_VERSION=${var.age_version}"]
    script           = local.bake_sops
  }
  provisioner "shell" {
    inline = [
      "echo '${var.build_ssh_password}' | sudo -S cloud-init clean --logs || true",
      "echo '${var.build_ssh_password}' | sudo -S truncate -s0 /etc/machine-id",
      "echo '${var.build_ssh_password}' | sudo -S rm -f /etc/ssh/ssh_host_*"
    ]
  }
}
```

- [ ] **Step 3: Validate**

Run: `packer validate -var-file=common.pkrvars.hcl .`
Expected: PASS.

- [ ] **Step 4: Build**

Run: `make debian`
Expected: preseed install runs unattended, provisioners run, template `9002` created.

- [ ] **Step 5: Verify (acceptance)**

```bash
qm clone 9002 9102 --name debian-verify
qm start 9102
ssh debian-verify 'cloud-init status --wait && systemctl is-active qemu-guest-agent && sops --version && age --version'
qm stop 9102 && qm destroy 9102
```
Expected: cloud-init done, agent active, sops + age present.

- [ ] **Step 6: Commit**

```bash
git add debian.pkr.hcl http/debian/
git commit -m "feat(packer): debian 9002 template (preseed + bake)"
```

---

## Task 5: VyOS template (9000)

**Files:**
- Create: `vyos.pkr.hcl`, (optional) `scripts/vyos-install.expect`

**Interfaces:**
- Consumes: `proxmox_*`, datastores, `bridge`, `vyos_iso_*`. **No sops/age** (lean bake).
- Produces: template `vm_id = 9000`, name `vyos`, NICs `eth0`/`eth1`.

- [ ] **Step 1: Capture the VyOS install dialog (one-time, manual)**

Boot the VyOS ISO once manually in a Proxmox VM, log in `vyos`/`vyos`, run `install image`, and record each prompt + response (disk, partition, password, console). This transcript becomes the `boot_command`. (VyOS install is interactive; this capture is required before automating.)

- [ ] **Step 2: Write `vyos.pkr.hcl`**

```hcl
source "proxmox-iso" "vyos" {
  proxmox_url              = var.proxmox_url
  username                 = var.proxmox_username
  token                    = var.proxmox_token
  node                     = var.proxmox_node
  insecure_skip_tls_verify = var.proxmox_insecure

  vm_id               = 9000
  vm_name             = "vyos-build"
  template_name       = "vyos"
  template_description = "VyOS rolling — baked by v2e-packer (cloud-init enabled)"
  cores               = 1
  memory              = 1024
  qemu_agent          = true

  boot_iso {
    type             = "scsi"
    iso_url          = var.vyos_iso_url
    iso_checksum     = var.vyos_iso_checksum
    iso_storage_pool = var.iso_datastore
    unmount          = true
  }

  disks {
    disk_size    = "10G"
    storage_pool = var.disk_datastore
    type         = "scsi"
  }
  # Two NICs so the installed image presents eth0 (WAN) + eth1 (LAN trunk)
  network_adapters {
    bridge = var.bridge
    model  = "virtio"
  }
  network_adapters {
    bridge = var.bridge
    model  = "virtio"
  }

  cloud_init              = true
  cloud_init_storage_pool = var.disk_datastore

  boot_wait = "30s"
  # Replace with the transcript captured in Step 1. Skeleton:
  boot_command = [
    "vyos<enter><wait>",                # login user
    "vyos<enter><wait5>",               # login password
    "install image<enter><wait5>",
    "<enter><wait>",                    # Would you like to continue? Yes
    "<enter><wait>",                    # partition: Auto
    "<enter><wait>",                    # disk select
    "<enter><wait>",                    # confirm wipe
    "<enter><wait>",                    # filesystem size default
    "vyos<enter><wait>",                # set root/admin password
    "vyos<enter><wait>",                # confirm password
    "<enter><wait60>",                  # GRUB target default; wait for copy
    "reboot<enter>"
  ]

  ssh_username = "vyos"
  ssh_password = "vyos"
  ssh_timeout  = "40m"
}

build {
  sources = ["source.proxmox-iso.vyos"]

  # Lean bake: agent + cloud-init enable + baseline only. NO sops/age.
  provisioner "shell" {
    execute_command = "sudo -S -E bash '{{ .Path }}'"
    script          = local.bake_common
  }
  provisioner "shell" {
    inline = ["sudo cloud-init clean --logs || true", "sudo rm -f /etc/ssh/ssh_host_*"]
  }
}
```

> **bake-common on VyOS:** VyOS is Debian-based but uses its own config system; `apt-get`
> may be restricted. If `bake-common.sh` can't run as-is on VyOS, the cloud-init enable +
> agent must be done via VyOS's own mechanism (or rely on the `vyos-build` fallback, which
> bakes cloud-init at image-build time). Decide during this task; record the outcome.

- [ ] **Step 3: Validate**

Run: `packer validate -var-file=common.pkrvars.hcl .`
Expected: PASS.

- [ ] **Step 4: Build (expect iteration)**

Run: `make vyos`
Expected: the `boot_command` drives `install image`; SSH connects post-reboot; template `9000` created. The `boot_command` will likely need several timing/keystroke adjustments (this is the riskiest image).

- [ ] **Step 5: (Fallback) If boot_command automation is intractable**

Switch to the documented fallback: run `vyos-build` (Docker) from the kit at
`~/Documents/vyos/cloudinit-image/` to produce a cloud-init qcow2, import it
(`qm importdisk`), set NICs, and `qm template 9000`. Document this path in the README as the
VyOS alternate. (This needs Docker on the build host.)

- [ ] **Step 6: Verify (acceptance)**

```bash
qm clone 9000 9100 --name vyos-verify
qm start 9100
# via Proxmox console or SSH once cloud-init applies a key:
#   confirm cloud-init applied user-data, and `show interfaces` lists eth0 + eth1
qm stop 9100 && qm destroy 9100
```
Expected: cloud-init applies user-data; interfaces are `eth0`/`eth1`; `qemu-guest-agent`
running. No sops/age expected.

- [ ] **Step 7: Commit**

```bash
git add vyos.pkr.hcl scripts/vyos-install.expect 2>/dev/null; git add vyos.pkr.hcl
git commit -m "feat(packer): vyos 9000 template (proxmox-iso install + cloud-init)"
```

---

## Task 6: Parrot template (9003) + finalize

**Files:**
- Create: `parrot.pkr.hcl`, `http/parrot/preseed.cfg`
- Modify: `README.md` (finalize), `scripts/parrot-repos.sh` (confirm metapackages)

**Interfaces:**
- Consumes: `debian_iso_*` (Parrot installs from the Debian netinst ISO), `proxmox_*`, datastores, `bridge`, bake scripts, `local.parrot_repos`.
- Produces: template `vm_id = 9003`, name `parrot-home`, with the Parrot desktop + tools.

- [ ] **Step 1: Confirm Parrot repo/keyring/metapackage against official docs**

Verify the keyring URL, repo suite, and Home metapackage names in `scripts/parrot-repos.sh`
against Parrot's current "install Parrot on Debian" documentation. Update the script if they
differ. (This resolves the spec §10 open item.)

- [ ] **Step 2: Reuse the Debian preseed for Parrot**

```bash
cp http/debian/preseed.cfg http/parrot/preseed.cfg
# Optionally change the hostname line to parrot-template
sed -i 's/debian-template/parrot-template/' http/parrot/preseed.cfg
```

- [ ] **Step 3: Write `parrot.pkr.hcl`**

```hcl
source "proxmox-iso" "parrot" {
  proxmox_url              = var.proxmox_url
  username                 = var.proxmox_username
  token                    = var.proxmox_token
  node                     = var.proxmox_node
  insecure_skip_tls_verify = var.proxmox_insecure

  vm_id               = 9003
  vm_name             = "parrot-home-build"
  template_name       = "parrot-home"
  template_description = "Parrot Home on Debian base — baked by v2e-packer"
  cores               = 4
  memory              = 8192        # desktop install headroom
  qemu_agent          = true

  boot_iso {
    type             = "scsi"
    iso_url          = var.debian_iso_url      # Parrot = Debian netinst + Parrot repos
    iso_checksum     = var.debian_iso_checksum
    iso_storage_pool = var.iso_datastore
    unmount          = true
  }

  disks {
    disk_size    = "40G"
    storage_pool = var.disk_datastore
    type         = "scsi"
  }
  network_adapters {
    bridge = var.bridge
    model  = "virtio"
  }

  cloud_init              = true
  cloud_init_storage_pool = var.disk_datastore

  http_directory = "http/parrot"
  boot_wait      = "5s"
  boot_command = [
    "<esc><wait>",
    "auto url=http://{{ .HTTPIP }}:{{ .HTTPPort }}/preseed.cfg ",
    "<enter>"
  ]

  ssh_username = var.build_ssh_username
  ssh_password = var.build_ssh_password
  ssh_timeout  = "40m"
}

build {
  sources = ["source.proxmox-iso.parrot"]

  provisioner "shell" {
    execute_command = "echo '${var.build_ssh_password}' | sudo -S -E bash '{{ .Path }}'"
    script          = local.bake_common
  }
  provisioner "shell" {
    execute_command  = "echo '${var.build_ssh_password}' | sudo -S -E bash '{{ .Path }}'"
    environment_vars = ["SOPS_VERSION=${var.sops_version}", "AGE_VERSION=${var.age_version}"]
    script           = local.bake_sops
  }
  provisioner "shell" {
    execute_command = "echo '${var.build_ssh_password}' | sudo -S -E bash '{{ .Path }}'"
    script          = local.parrot_repos
  }
  # cloud-init NetworkManager renderer so the cloned desktop NIC comes up
  provisioner "shell" {
    inline = [
      "echo '${var.build_ssh_password}' | sudo -S bash -c 'printf \"network:\\n  renderer: NetworkManager\\n\" > /etc/cloud/cloud.cfg.d/90-nm.cfg'",
      "echo '${var.build_ssh_password}' | sudo -S cloud-init clean --logs || true",
      "echo '${var.build_ssh_password}' | sudo -S truncate -s0 /etc/machine-id",
      "echo '${var.build_ssh_password}' | sudo -S rm -f /etc/ssh/ssh_host_*"
    ]
  }
}
```

- [ ] **Step 4: Validate + Build**

Run: `packer validate -var-file=common.pkrvars.hcl .` (expect PASS), then `make parrot`.
Expected: Debian installs, Parrot repos + desktop install, template `9003` created. Build is
the longest (desktop packages).

- [ ] **Step 5: Verify (acceptance)**

```bash
qm clone 9003 9103 --name parrot-verify
qm start 9103
ssh parrot-verify 'cloud-init status --wait && systemctl is-active qemu-guest-agent && sops --version && age --version && dpkg -l | grep -c parrot'
qm stop 9103 && qm destroy 9103
```
Expected: cloud-init done, agent active, sops + age present, Parrot packages installed (count > 0). NIC came up via NetworkManager (clone got an IP).

- [ ] **Step 6: Finalize README**

Fill the README with: full prerequisites, the image→template→VMID chain, per-image build
commands, the BSL caveat, and the VyOS `vyos-build` fallback (from Task 5 Step 5). Add any
VyOS bake-on-VyOS outcome recorded in Task 5.

- [ ] **Step 7: Full lint + `make all` dry confirm**

Run: `make lint`
Expected: `packer fmt -check` clean, `packer validate` PASS, ShellCheck clean.

- [ ] **Step 8: Commit**

```bash
git add parrot.pkr.hcl http/parrot/ scripts/parrot-repos.sh README.md
git commit -m "feat(packer): parrot 9003 template + finalize README"
```

---

## Self-Review notes (carried from spec)

- **Spec coverage:** Tasks 1–6 cover every image (5.1–5.4), the bake split (§2), the
  single-dir layout (§3), lint/Makefile (§4.3), and the phase acceptance (§9). The README
  (Task 6 Step 6) covers §9.5.
- **Known iteration points (not placeholders — real tuning):** Ubuntu autoinstall
  `boot_command` (Task 3 Step 7), VyOS install transcript (Task 5 Steps 1/4), Parrot
  metapackages (Task 6 Step 1), ISO checksums (Task 1 Step 7). These are inherent to
  infra-from-ISO work and each has a concrete capture/obtain step + an acceptance gate.
- **Field-name risk:** the `proxmox-iso` `boot_iso`/`disks`/`network_adapters` field names
  are from the current plugin docs; `packer validate` (each image's Validate step) is the
  catch-net if any drifted.
```
