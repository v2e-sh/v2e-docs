# DNS-1 (slice 2/3) — `v2e-ansible` `technitium` role — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A `technitium` role that installs Technitium DNS Server natively (systemd) on the `dns` node and drives its HTTP API idempotently to serve `int.v2e.sh` — `*.int.v2e.sh` + apex A → `10.1.2.10`, DoT forwarders, recursion restricted to the lab VLANs, query logging on — with the admin password kept in SOPS.

**Architecture:** The `dns` node is added to the inventory as its own group and folded into `[linux]` so it gets the existing bootstrap (health_check → baseline → devsec hardening → fail2ban). A new `technitium` role (install + API config) runs from a new phase playbook `playbooks/02-dns.yml`, imported into `site.yml` right after bootstrap (DNS is foundational). Config is applied via `ansible.builtin.uri` with GET-before-POST so re-runs are no-ops. The admin password is bootstrapped off Technitium's default `admin/admin` on first run and thereafter read from `group_vars/dns/secrets.sops.yaml` (the `community.sops` vars plugin is already enabled in `ansible.cfg`).

**Tech Stack:** Ansible core 2.21, `ansible.builtin` (`uri`, `get_url`, `command`, `stat`, `systemd_service`), `community.sops` vars plugin, SOPS + age, Technitium DNS Server (native install script), Debian 13.

**Source spec:** `v2e-docs/specs/2026-06-30-dns-appliance-design.md` (§2, §5, §12).

**Environment note:** There is no `dns` host and no Docker/Molecule in the implementation environment, so live API calls and Molecule cannot run here. In-env verification uses `ansible-lint`, `ansible-playbook --syntax-check`, and reading the files. Idempotence is **designed in** (install guarded by a `stat`; every API write preceded by a GET and gated with `when`/`changed_when`) and confirmed on the real node by running `site.yml` twice (second run: `changed=0` for the role). Molecule/CI is out of scope here (owned by ANS-5).

## Global Constraints

- **Repo / branch:** `v2e-ansible`, branch `feat/ansible-technitium`. Lands via **PR** (the `main` ruleset requires PRs). This is **PR #2** of three; apply **after** the v2e-tf PR so `10.1.0.53` exists and is reachable from control.
- **House style (exact):** FQCN modules only (`ansible.builtin.*`, `community.*`, `ansible.posix.*`); `defaults/main.yml` declares every variable; descriptive task names with `# --- section ---` comment breaks; no tags inside role tasks (tags live in playbooks); `no_log: true` on any task carrying the admin password or token.
- **Single source of truth:** the internal domain is the group_var `internal_domain` (default `int.v2e.sh`) — the zone name and wildcard record derive from it; never hardcode `int.v2e.sh` in role tasks.
- **Wildcard target:** `*.${internal_domain}` and the apex both A → `10.1.2.10` (the services node running Traefik), from the var `technitium_wildcard_target`.
- **Forwarders over DoT:** `1.1.1.1` / `9.9.9.9`, `forwarderProtocol=Tls`.
- **Recursion:** `UseSpecifiedNetworks`, allowed only from the lab subnets (`10.1.0.0/24`, `10.1.1.0/24`, `10.1.2.0/24`, `10.1.3.0/24`) — NOT an open resolver.
- **Secrets:** the admin password lives in `inventory/group_vars/dns/secrets.sops.yaml` as `vault_technitium_admin_password` (keeping the repo's existing `vault_`-prefix convention); the role reads it via `| default('')`. Never commit the plaintext; ship a `.example`.
- **API is called on the target** (`http://127.0.0.1:5380`) — `uri` runs on the `dns` node against localhost, so the console port never needs to be exposed to the controller.
- **Git:** commit per task; short messages, project style; **no AI attribution trailers**.

> **Implementer note on the Technitium HTTP API:** the endpoints below (`/api/user/login`, `/api/user/changePassword`, `/api/zones/create`, `/api/zones/records/add`, `/api/zones/list`, `/api/zones/records/get`, `/api/settings/set`, `/api/settings/get`) are stable across Technitium v11–v13 and return JSON `{"status":"ok", ...}`. Before finalizing Task 5/6, `curl` one endpoint on the running node (or check `https://github.com/TechnitiumSoftware/DnsServer/blob/master/APIDOCS.md`) to confirm the exact parameter names for the installed version, and adjust the query params if the pinned version differs. The GET-before-POST structure does not change.

---

## File Structure

| File | Change | Responsibility | Task |
|---|---|---|---|
| `inventory/hosts.ini` | modify | Add `[dns]` group; add `dns` under `[linux:children]` | 1 |
| `inventory/group_vars/dns/main.yml` | create | `internal_domain`, Technitium settings, `ssh_allow_users` | 1 |
| `.sops.yaml` | create | SOPS creation rule for `group_vars/**/secrets.sops.yaml` | 2 |
| `inventory/group_vars/dns/secrets.sops.yaml.example` | create | `vault_technitium_admin_password` template | 2 |
| `roles/technitium/defaults/main.yml` | create | All role variables | 3 |
| `roles/technitium/README.md` | create | What it does, vars, security notes | 3 |
| `roles/technitium/handlers/main.yml` | create | Restart Technitium | 3 |
| `roles/technitium/tasks/main.yml` | create | Install (guarded) + include API config | 4 |
| `roles/technitium/tasks/install.yml` | create | Install script + service | 4 |
| `roles/technitium/tasks/configure.yml` | create | Auth + zone + records + settings via API | 5, 6 |
| `playbooks/02-dns.yml` | create | Run the role on `[dns]` | 7 |
| `site.yml` | modify | Import `02-dns.yml` after bootstrap | 7 |
| `README.md` | modify | DNS appliance section | 8 |

---

## Task 1: Inventory group + group_vars for the `dns` node

**Files:**
- Branch: `feat/ansible-technitium` (off `main`)
- Modify: `inventory/hosts.ini`
- Create: `inventory/group_vars/dns/main.yml`

**Interfaces:**
- Produces: inventory group `dns` (host `dns01` @ `10.1.0.53`), membership in `[linux]`, and the `internal_domain` / `technitium_*` group_vars consumed by the role (Tasks 4–6).

- [ ] **Step 1: Confirm the branch**

Run: `git -C ~/v2e-environment/v2e-ansible branch --show-current`
Expected: `feat/ansible-technitium` (else `git checkout -b feat/ansible-technitium`).

- [ ] **Step 2: Add the `dns` group to the inventory**

In `inventory/hosts.ini`, add a `[dns]` group after `[agent]` (before `[linux:children]`):

```ini
[dns]
dns01 ansible_host=10.1.0.53 ansible_user=ansible
```

and add `dns` to the `[linux:children]` list so it gets bootstrap + hardening:

```ini
[linux:children]
control
services
agent
dns
```

- [ ] **Step 3: Create the `dns` group_vars**

Create `inventory/group_vars/dns/main.yml`:

```yaml
---
# Internal DNS appliance (Technitium) — DNS-1. Consumed by the technitium role.

# devsec ssh_hardening AllowUsers: cluster admin + the automation account.
ssh_allow_users: "v2e ansible"

# Single source of truth for the internal domain (shared contract with v2e-compose
# INTERNAL_DOMAIN). The authoritative zone + wildcard record derive from this.
internal_domain: int.v2e.sh

# Everything *.int.v2e.sh (and the apex) resolves to the services node (Traefik).
technitium_wildcard_target: "10.1.2.10"

# Upstream forwarders over DoT (everything not in the internal zone).
technitium_forwarders:
  - "1.1.1.1"
  - "9.9.9.9"
technitium_forwarder_protocol: Tls

# Recursion only for the lab subnets — NOT an open resolver.
technitium_recursion_networks:
  - "10.1.0.0/24"
  - "10.1.1.0/24"
  - "10.1.2.0/24"
  - "10.1.3.0/24"

# Query logging on (agent DNS visibility — feeds Q1/ANS-6).
technitium_enable_query_logging: true

# Admin API credentials. Password comes from SOPS (see secrets.sops.yaml).
technitium_admin_user: admin
technitium_admin_password: "{{ vault_technitium_admin_password | default('') }}"
```

- [ ] **Step 4: Lint + syntax check**

Run:
```bash
cd ~/v2e-environment/v2e-ansible
ansible-inventory -i inventory/hosts.ini --graph
```
Expected: `dns01` appears under `@dns` and under `@linux`.

Run: `ansible-lint inventory/group_vars/dns/main.yml`
Expected: no errors (warnings about `no-changed-when` etc. apply to tasks, not vars).

- [ ] **Step 5: Commit**

```bash
git add inventory/hosts.ini inventory/group_vars/dns/main.yml
git commit -m "ansible: add dns inventory group + technitium group_vars"
```

---

## Task 2: SOPS secret scaffolding

**Files:**
- Create: `.sops.yaml`
- Create: `inventory/group_vars/dns/secrets.sops.yaml.example`

**Interfaces:**
- Produces: the `vault_technitium_admin_password` secret contract; a `.sops.yaml` creation rule so the operator can `sops --encrypt` the example into `secrets.sops.yaml` (auto-decrypted at runtime by the `community.sops` vars plugin already enabled in `ansible.cfg`).

- [ ] **Step 1: Create the repo-root `.sops.yaml`**

Create `.sops.yaml` (mirrors the v2e-compose pattern):

```yaml
# SOPS creation rule for v2e-ansible secrets (group_vars/**/secrets.sops.yaml).
# Replace the age recipient with YOUR age PUBLIC key (starts with `age1`).
# Public keys are safe to commit. Get yours with:
#   age-keygen -y ~/.config/sops/age/keys.txt
creation_rules:
  - path_regex: secrets\.sops\.ya?ml$
    age: "age1replacewithyourpublickeyxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

- [ ] **Step 2: Create the secrets example**

Create `inventory/group_vars/dns/secrets.sops.yaml.example`:

```yaml
# Technitium admin API password for the dns node. Encrypt to produce the real file:
#   sops --encrypt inventory/group_vars/dns/secrets.sops.yaml.example \
#     > inventory/group_vars/dns/secrets.sops.yaml
# then `sops inventory/group_vars/dns/secrets.sops.yaml` to set the real value.
# The technitium role bootstraps this off Technitium's default admin/admin on the
# first run and logs in with it thereafter.
vault_technitium_admin_password: "<a-strong-password>"
```

- [ ] **Step 3: Confirm `.gitignore` protects the real file**

Run: `grep -n 'secrets.sops.yaml' .gitignore`
Expected: a match. If absent, add these lines to `.gitignore`:

```
# SOPS-encrypted secrets are decryptable by design — but keep the real files out.
inventory/group_vars/**/secrets.sops.yaml
!inventory/group_vars/**/secrets.sops.yaml.example
```

(SOPS files are safe to commit in principle, but the repo's TruffleHog/secret posture prefers keeping them local; the `.example` is tracked.)

- [ ] **Step 4: Commit**

```bash
git add .sops.yaml inventory/group_vars/dns/secrets.sops.yaml.example .gitignore
git commit -m "ansible: sops scaffolding for the technitium admin password"
```

---

## Task 3: Role skeleton — defaults, handler, README

**Files:**
- Create: `roles/technitium/defaults/main.yml`
- Create: `roles/technitium/handlers/main.yml`
- Create: `roles/technitium/README.md`

**Interfaces:**
- Produces: role defaults (install source, paths, API base, timeouts) consumed by Tasks 4–6; a `Restart Technitium` handler.

- [ ] **Step 1: Create `defaults/main.yml`**

```yaml
---
# Technitium DNS Server — native (systemd) install + HTTP-API configuration.
# All values overridable; the group_vars/dns vars supply the site-specific ones.

# --- install ---
technitium_install_url: "https://download.technitium.com/dns/install.sh"
technitium_app_dir: /opt/technitium/dns
technitium_service_name: dns
technitium_marker: "{{ technitium_app_dir }}/DnsServerApp.dll"

# --- API ---
technitium_api_url: "http://127.0.0.1:5380"
technitium_api_timeout: 30

# --- zone / records (overridden in group_vars/dns) ---
internal_domain: int.v2e.sh
technitium_wildcard_target: "10.1.2.10"

# --- upstream / recursion / logging (overridden in group_vars/dns) ---
technitium_forwarders:
  - "1.1.1.1"
  - "9.9.9.9"
technitium_forwarder_protocol: Tls
technitium_recursion_networks:
  - "10.1.0.0/24"
  - "10.1.1.0/24"
  - "10.1.2.0/24"
  - "10.1.3.0/24"
technitium_enable_query_logging: true

# --- credentials ---
technitium_admin_user: admin
technitium_admin_password: ""
technitium_default_password: admin   # Technitium's factory default (first run only)
```

- [ ] **Step 2: Create `handlers/main.yml`**

```yaml
---
- name: Restart Technitium
  become: true
  ansible.builtin.systemd_service:
    name: "{{ technitium_service_name }}"
    state: restarted
```

- [ ] **Step 3: Create `README.md`**

```markdown
# technitium

Installs Technitium DNS Server natively (systemd) on the `dns` node and configures
it through its HTTP API to be authoritative for `{{ internal_domain }}`:

- primary zone `{{ internal_domain }}`, `*.{{ internal_domain }}` + apex A → the
  services node (Traefik), `10.1.2.10`;
- upstream forwarders over DoT (`1.1.1.1`/`9.9.9.9`);
- recursion restricted to the lab subnets (not an open resolver);
- query logging on.

## How it works

`tasks/install.yml` runs Technitium's official `install.sh` once (guarded by a
`stat` on the app DLL) and ensures the `dns` systemd service is enabled + started.
`tasks/configure.yml` talks to `http://127.0.0.1:5380` with `ansible.builtin.uri`,
GET-before-POST so re-runs change nothing. On the first run it logs in with the
factory `admin/admin` and changes the password to `vault_technitium_admin_password`
(SOPS); thereafter it logs in with that password.

## Key variables

See `defaults/main.yml` and `inventory/group_vars/dns/`. The admin password is a
SOPS secret (`vault_technitium_admin_password`).

## Security notes

- The console (`:5380`) is only reached over localhost by this role and, remotely,
  through Traefik + `auth@docker` (`dns.{{ internal_domain }}`, v2e-compose).
- Recursion is `UseSpecifiedNetworks` — do not widen `technitium_recursion_networks`
  to `0.0.0.0/0` (open resolver).
```

- [ ] **Step 4: Lint**

Run: `cd ~/v2e-environment/v2e-ansible && ansible-lint roles/technitium/`
Expected: no errors (tasks come next; defaults/handlers should be clean).

- [ ] **Step 5: Commit**

```bash
git add roles/technitium/defaults/main.yml roles/technitium/handlers/main.yml roles/technitium/README.md
git commit -m "ansible: scaffold the technitium role (defaults, handler, readme)"
```

---

## Task 4: Install Technitium natively (guarded)

**Files:**
- Create: `roles/technitium/tasks/main.yml`
- Create: `roles/technitium/tasks/install.yml`

**Interfaces:**
- Consumes: `technitium_install_url`, `technitium_marker`, `technitium_service_name`, `technitium_api_url` (defaults).
- Produces: an installed + running Technitium service; `main.yml` includes `install.yml` then `configure.yml`.

- [ ] **Step 1: Create `tasks/main.yml`**

```yaml
---
# Technitium: install natively, then configure via the HTTP API. Both halves are
# idempotent (install guarded by a stat; API config is GET-before-POST).

- name: Install Technitium DNS Server
  ansible.builtin.import_tasks: install.yml

- name: Configure Technitium via the HTTP API
  ansible.builtin.import_tasks: configure.yml
```

- [ ] **Step 2: Create `tasks/install.yml`**

```yaml
---
# --- Install (only when the app DLL is absent) ---
- name: Check whether Technitium is already installed
  become: true
  ansible.builtin.stat:
    path: "{{ technitium_marker }}"
  register: technitium_app

- name: Download the Technitium install script
  become: true
  ansible.builtin.get_url:
    url: "{{ technitium_install_url }}"
    dest: /tmp/technitium-install.sh
    mode: "0755"
  when: not technitium_app.stat.exists

- name: Run the Technitium install script
  become: true
  ansible.builtin.command: /bin/bash /tmp/technitium-install.sh
  args:
    creates: "{{ technitium_marker }}"
  when: not technitium_app.stat.exists

- name: Ensure the Technitium service is enabled and running
  become: true
  ansible.builtin.systemd_service:
    name: "{{ technitium_service_name }}"
    state: started
    enabled: true

# --- Wait for the API before configuring ---
- name: Wait for the Technitium web console to answer
  ansible.builtin.uri:
    url: "{{ technitium_api_url }}/api/user/login?user=admin&pass=admin&includeInfo=false"
    method: GET
    status_code: [200]
    timeout: "{{ technitium_api_timeout }}"
  register: technitium_ping
  until: technitium_ping.status == 200
  retries: 30
  delay: 2
  changed_when: false
  failed_when: false
  no_log: true
```

(The install script pulls .NET + Technitium and creates the `dns` systemd unit. The final GET just waits for `:5380` to be live — it doesn't assert the credentials, hence `failed_when: false`.)

- [ ] **Step 3: Syntax check**

Because `configure.yml` doesn't exist yet, create a temporary empty placeholder so the import resolves, then syntax-check:

```bash
cd ~/v2e-environment/v2e-ansible
printf -- '---\n' > roles/technitium/tasks/configure.yml
ansible-playbook -i inventory/hosts.ini --syntax-check playbooks/02-dns.yml 2>/dev/null || \
  echo "02-dns.yml not created yet — will syntax-check in Task 7"
ansible-lint roles/technitium/
```
Expected: `ansible-lint` clean (the empty configure.yml is valid). The placeholder is filled in Task 5.

- [ ] **Step 4: Commit**

```bash
git add roles/technitium/tasks/main.yml roles/technitium/tasks/install.yml roles/technitium/tasks/configure.yml
git commit -m "ansible: technitium native install (guarded) + service"
```

---

## Task 5: API — auth, password bootstrap, zone + records

**Files:**
- Modify: `roles/technitium/tasks/configure.yml`

**Interfaces:**
- Consumes: `technitium_api_url`, `technitium_admin_password`, `technitium_default_password`, `internal_domain`, `technitium_wildcard_target`.
- Produces: `technitium_token` (session token fact); the `${internal_domain}` primary zone; `*.${internal_domain}` + apex A records. Consumed by Task 6 (settings).

- [ ] **Step 1: Fail fast if the password isn't supplied**

Overwrite `roles/technitium/tasks/configure.yml` with (auth + zone + records; settings appended in Task 6):

```yaml
---
- name: Require an admin password (from SOPS)
  ansible.builtin.assert:
    that:
      - technitium_admin_password | length > 0
    fail_msg: >-
      technitium_admin_password is empty. Set vault_technitium_admin_password in
      inventory/group_vars/dns/secrets.sops.yaml (see the .example).

# --- Authenticate (bootstrap the password off the default on the first run) ---
- name: Log in with the configured admin password
  ansible.builtin.uri:
    url: "{{ technitium_api_url }}/api/user/login?user={{ technitium_admin_user }}&pass={{ technitium_admin_password | urlencode }}"
    method: GET
    timeout: "{{ technitium_api_timeout }}"
  register: technitium_login_cfg
  failed_when: false
  changed_when: false
  no_log: true

- name: Log in with the factory default (first run only)
  ansible.builtin.uri:
    url: "{{ technitium_api_url }}/api/user/login?user={{ technitium_admin_user }}&pass={{ technitium_default_password | urlencode }}"
    method: GET
    timeout: "{{ technitium_api_timeout }}"
  register: technitium_login_default
  failed_when: false
  changed_when: false
  no_log: true
  when: technitium_login_cfg.json.status | default('') != 'ok'

- name: Record the admin session token
  ansible.builtin.set_fact:
    technitium_token: >-
      {{ technitium_login_cfg.json.token
         if (technitium_login_cfg.json.status | default('')) == 'ok'
         else (technitium_login_default.json.token | default('')) }}
  no_log: true

- name: Fail if neither login worked
  ansible.builtin.assert:
    that:
      - technitium_token | length > 0
    fail_msg: >-
      Could not authenticate to Technitium with the configured OR the default
      password. If the password was changed out of band, update SOPS.

- name: Change the default admin password (first run only)
  ansible.builtin.uri:
    url: "{{ technitium_api_url }}/api/user/changePassword?token={{ technitium_token }}&pass={{ technitium_admin_password | urlencode }}"
    method: GET
    timeout: "{{ technitium_api_timeout }}"
  register: technitium_pwchange
  failed_when: technitium_pwchange.json.status | default('') != 'ok'
  changed_when: true
  no_log: true
  when: (technitium_login_cfg.json.status | default('')) != 'ok'

# --- Authoritative zone ---
- name: List existing zones
  ansible.builtin.uri:
    url: "{{ technitium_api_url }}/api/zones/list?token={{ technitium_token }}"
    method: GET
    timeout: "{{ technitium_api_timeout }}"
  register: technitium_zones
  changed_when: false
  no_log: true

- name: Create the internal primary zone
  ansible.builtin.uri:
    url: "{{ technitium_api_url }}/api/zones/create?token={{ technitium_token }}&zone={{ internal_domain }}&type=Primary"
    method: GET
    timeout: "{{ technitium_api_timeout }}"
  register: technitium_zone_create
  failed_when: technitium_zone_create.json.status | default('') != 'ok'
  changed_when: true
  no_log: true
  when: internal_domain not in (technitium_zones.json.response.zones | default([]) | map(attribute='name') | list)

# --- Records: wildcard + apex A -> the services node (Traefik) ---
- name: Get current records for the zone
  ansible.builtin.uri:
    url: "{{ technitium_api_url }}/api/zones/records/get?token={{ technitium_token }}&domain={{ internal_domain }}&zone={{ internal_domain }}&listZone=true"
    method: GET
    timeout: "{{ technitium_api_timeout }}"
  register: technitium_records
  changed_when: false
  no_log: true

- name: Ensure the A records exist (wildcard + apex)
  ansible.builtin.uri:
    url: "{{ technitium_api_url }}/api/zones/records/add?token={{ technitium_token }}&domain={{ item }}&zone={{ internal_domain }}&type=A&ipAddress={{ technitium_wildcard_target }}&ttl=3600&overwrite=true"
    method: GET
    timeout: "{{ technitium_api_timeout }}"
  register: technitium_record_add
  failed_when: technitium_record_add.json.status | default('') != 'ok'
  changed_when: not (
    technitium_records.json.response.records | default([])
    | selectattr('name', 'equalto', item if item != internal_domain else internal_domain)
    | selectattr('type', 'equalto', 'A')
    | map(attribute='rData.ipAddress') | select('equalto', technitium_wildcard_target) | list | length > 0)
  no_log: true
  loop:
    - "*.{{ internal_domain }}"
    - "{{ internal_domain }}"
```

(`overwrite=true` makes the add idempotent server-side; the `changed_when` reports "changed" only when the desired A value wasn't already present, so a second run is `changed=0`.)

- [ ] **Step 2: Lint + syntax check via the playbook (created in Task 7 — do a role-only lint now)**

Run: `cd ~/v2e-environment/v2e-ansible && ansible-lint roles/technitium/`
Expected: clean. If ansible-lint flags `risky-shell-pipe`/`no-changed-when`, confirm each write task has an explicit `changed_when` (they do) and each read has `changed_when: false`.

- [ ] **Step 3: Commit**

```bash
git add roles/technitium/tasks/configure.yml
git commit -m "ansible: technitium api — auth bootstrap, zone, wildcard+apex records"
```

---

## Task 6: API — forwarders (DoT), recursion, query logging

**Files:**
- Modify: `roles/technitium/tasks/configure.yml` (append)

**Interfaces:**
- Consumes: `technitium_token`, `technitium_forwarders`, `technitium_forwarder_protocol`, `technitium_recursion_networks`, `technitium_enable_query_logging`.
- Produces: DNS server settings (forwarders over DoT, recursion ACL, logging) applied idempotently.

- [ ] **Step 1: Append the settings tasks to `configure.yml`**

```yaml

# --- Server settings: forwarders (DoT), recursion ACL, query logging ---
- name: Get current DNS settings
  ansible.builtin.uri:
    url: "{{ technitium_api_url }}/api/settings/get?token={{ technitium_token }}"
    method: GET
    timeout: "{{ technitium_api_timeout }}"
  register: technitium_settings
  changed_when: false
  no_log: true

- name: Compute the desired settings state
  ansible.builtin.set_fact:
    technitium_want_forwarders: "{{ technitium_forwarders | join(',') }}"
    technitium_want_recursion_nets: "{{ technitium_recursion_networks | join(',') }}"

- name: Apply DNS server settings (forwarders/recursion/logging)
  ansible.builtin.uri:
    url: >-
      {{ technitium_api_url }}/api/settings/set?token={{ technitium_token }}
      &forwarders={{ technitium_want_forwarders | urlencode }}
      &forwarderProtocol={{ technitium_forwarder_protocol }}
      &recursion=UseSpecifiedNetworks
      &recursionNetworks={{ technitium_want_recursion_nets | urlencode }}
      &enableLogging={{ technitium_enable_query_logging | lower }}
    method: GET
    timeout: "{{ technitium_api_timeout }}"
  register: technitium_settings_set
  failed_when: technitium_settings_set.json.status | default('') != 'ok'
  changed_when: not (
    (technitium_settings.json.response.forwarders | default([]) | map(attribute='address') | sort
      == technitium_forwarders | sort)
    and (technitium_settings.json.response.forwarderProtocol | default('') == technitium_forwarder_protocol)
    and (technitium_settings.json.response.recursion | default('') == 'UseSpecifiedNetworks')
    and (technitium_settings.json.response.enableLogging | default(false) == technitium_enable_query_logging))
  no_log: true
```

> **Implementer note:** the `&` line-continuation in a folded scalar (`>-`) collapses to a single-line URL with the `&`s preserved; verify with `-vvv` that the rendered URL has no stray spaces around `&`. If the installed Technitium version returns forwarders in a different JSON shape (e.g. a plain string vs a list of objects), adjust only the `changed_when` comparison — the `set` call itself is unchanged. The `recursionNetworks` param name is used by v11+; on older builds it is `recursionDeniedNetworks`/`recursionAllowedNetworks` — confirm against `/api/settings/get` output.

- [ ] **Step 2: Lint**

Run: `cd ~/v2e-environment/v2e-ansible && ansible-lint roles/technitium/`
Expected: clean.

- [ ] **Step 3: Commit**

```bash
git add roles/technitium/tasks/configure.yml
git commit -m "ansible: technitium api — DoT forwarders, recursion ACL, query logging"
```

---

## Task 7: Phase playbook + wire into `site.yml`

**Files:**
- Create: `playbooks/02-dns.yml`
- Modify: `site.yml`

**Interfaces:**
- Consumes: the `technitium` role, the `[dns]` group.
- Produces: `site.yml` runs bootstrap → **dns** → services → applications.

- [ ] **Step 1: Create `playbooks/02-dns.yml`**

```yaml
---
# Phase 02-dns — internal DNS appliance (Technitium) on the dns node. Runs after
# 01-bootstrap (which hardens the node as part of the linux group) and before the
# services phase, so the resolver is up early. Idempotent (API GET-before-POST).

- name: Internal DNS appliance (Technitium)
  hosts: dns
  become: true
  roles:
    - technitium
```

- [ ] **Step 2: Import it into `site.yml`**

In `site.yml`, insert the DNS phase between bootstrap and services:

```yaml
- name: Bootstrap phase (reachability, baseline OS, hardening)
  import_playbook: playbooks/01-bootstrap.yml
- name: Internal DNS phase (Technitium appliance)
  import_playbook: playbooks/02-dns.yml
- name: Services phase (container runtime)
  import_playbook: playbooks/02-services.yml
- name: Applications phase (AI identities + workbench)
  import_playbook: playbooks/03-applications.yml
```

- [ ] **Step 3: Syntax-check the whole site**

Run:
```bash
cd ~/v2e-environment/v2e-ansible
ansible-playbook -i inventory/hosts.ini --syntax-check site.yml
```
Expected: lists the plays including `Internal DNS appliance (Technitium)` with no errors.
(If `devsec.hardening`/`geerlingguy.docker` aren't installed locally, syntax-check may error on those imports — if so, syntax-check the DNS play alone: `ansible-playbook -i inventory/hosts.ini --syntax-check playbooks/02-dns.yml`.)

- [ ] **Step 4: Full role lint**

Run: `ansible-lint roles/technitium/ playbooks/02-dns.yml`
Expected: clean.

- [ ] **Step 5: Commit**

```bash
git add playbooks/02-dns.yml site.yml
git commit -m "ansible: run the technitium role as a site.yml phase"
```

---

## Task 8: README

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add a DNS appliance section**

Add a section to `README.md` (near the phase/layout description) covering:

- the `dns` node (`10.1.0.53`, mgmt VLAN, added in v2e-tf) is in `[linux]` (bootstrap + hardening) and `[dns]` (the `technitium` role);
- what the role does: native install + API config → authoritative `int.v2e.sh`, `*.int.v2e.sh`+apex A → `10.1.2.10`, DoT forwarders, recursion limited to lab subnets, query logging;
- the admin password is a SOPS secret (`vault_technitium_admin_password` in `inventory/group_vars/dns/secrets.sops.yaml`; encrypt from the `.example`);
- console access: local `:5380`, or `https://dns.int.v2e.sh` behind auth (v2e-compose);
- the ANS-4 hook: remote resolution is Tailscale split-DNS + subnet route (out of scope here).

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "ansible: document the technitium dns appliance"
```

---

## Final acceptance (runs against the real `dns` node — out of the implementation env)

- [ ] `ansible-playbook -i inventory/hosts.ini playbooks/02-dns.yml` converges; a second run reports `changed=0` for the `technitium` role (idempotent).
- [ ] `dig traefik.int.v2e.sh @10.1.0.53` → `10.1.2.10`; `dig int.v2e.sh @10.1.0.53` → `10.1.2.10`.
- [ ] `dig google.com @10.1.0.53` resolves (forwarded over DoT).
- [ ] recursion from a non-lab source is refused; the console at `:5380` requires the SOPS password.
- [ ] `ansible-lint roles/technitium/` clean.

## PR

Open a PR from `feat/ansible-technitium` → `main`. Title: `ansible: technitium internal DNS role (DNS-1)`. Note the dependency on the v2e-tf PR (needs `10.1.0.53` reachable). This PR merges before the v2e-compose slice (whose `dns.int.v2e.sh` router points at this node).

## Self-review checklist

- Spec coverage: native install (§2) ✓ T4; zone+wildcard+apex (§5) ✓ T5; DoT forwarders (§5) ✓ T6; recursion ACL (§5) ✓ T6; query logging (§5) ✓ T6; SOPS creds (§5) ✓ T2/T5; `internal_domain` group_var (§4) ✓ T1.
- No hardcoded `int.v2e.sh` in role tasks (only in defaults/group_vars) ✓.
- Every API write has explicit `changed_when` + GET-before-POST; every read `changed_when: false` ✓.
- `no_log: true` on all password/token tasks ✓.
