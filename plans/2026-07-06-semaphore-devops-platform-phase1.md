# Semaphore DevOps Platform — Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Seed the empty Semaphore (v2.18.14) with an admin "v2e system" project + repository + inventory + key store + three survey-driven maintenance templates (patch/upgrade, add-user, read-only health-check), all created idempotently from Ansible so a rebuild restores them.

**Architecture:** Reuse the existing `patch` role + `playbooks/ops/patch.yml`; add two small ops playbooks (`add-user.yml`, `health-check.yml`); build a new `semaphore_bootstrap` role that calls Semaphore's REST API (via `ansible.builtin.uri`) from control to create the project and templates. Run it from a new `playbooks/ops/semaphore-bootstrap.yml`.

**Tech Stack:** Ansible (core 2.19.x), `ansible.builtin.uri` for the Semaphore API, community.sops for secrets, Semaphore v2.18.14 REST API.

**Spec:** `v2e-docs/specs/2026-07-06-semaphore-devops-platform-design.md` (Phase 1 only; Phase 2 multi-user tier is a later plan).

## Global Constraints

- Ansible playbooks live in `v2e-ansible/playbooks/ops/`, roles in `v2e-ansible/roles/`. Follow existing style (see `playbooks/ops/patch.yml`, `roles/patch`).
- Secrets come from SOPS `inventory/group_vars/all.sops.yaml` (lowercase keys). Nothing sensitive in git; `no_log: true` on secret-handling tasks.
- CI must stay green: `ansible-lint` + `yamllint` + `ansible-playbook --syntax-check` (the v2e-ansible CI). Pinned collections only (community.general 13.1.0, ansible.posix, community.sops).
- The bootstrap role runs **from control** (the privileged runner host) targeting the Semaphore API; it does NOT run on the semaphore container.
- Semaphore API base (internal, from control over the VLAN): `http://10.1.2.10:3000` (the `SEMAPHORE_BIND_ADDR` published port). Auth via a pre-created admin API token stored in SOPS as `semaphore_api_token`.
- Git: short commit messages, no AI attribution/co-author trailers (user preference).
- All Semaphore objects must be **idempotent** — create-if-absent by name, never duplicate on re-run.

---

### Task 1: `add-user` ops playbook

**Files:**
- Create: `v2e-ansible/playbooks/ops/add-user.yml`

**Interfaces:**
- Produces: a playbook runnable as `ansible-playbook playbooks/ops/add-user.yml -e target=<host|group> -e username=<u> -e ssh_public_key=<key> -e grant_sudo=<bool> -e login_shell=<shell>`. Semaphore template (Task 6) maps these to survey vars.

- [ ] **Step 1: Write the playbook**

```yaml
---
# On-demand: add a login user to chosen node(s). Survey-driven from Semaphore, or:
#   ansible-playbook playbooks/ops/add-user.yml -e target=services \
#     -e username=alice -e "ssh_public_key='ssh-ed25519 AAAA... alice'" -e grant_sudo=false
# Key-only (no passwords). Idempotent.

- name: Add a login user
  hosts: "{{ target | default('none') }}"
  become: true
  gather_facts: false
  vars:
    grant_sudo: false
    login_shell: /bin/bash
  tasks:
    - name: Require username + ssh_public_key + a real target
      ansible.builtin.assert:
        that:
          - target != 'none'
          - username is defined and username | length > 0
          - ssh_public_key is defined and ssh_public_key | length > 0
        fail_msg: "Pass -e target=<host|group>, -e username=..., -e ssh_public_key='...'"

    - name: Create the user
      ansible.builtin.user:
        name: "{{ username }}"
        shell: "{{ login_shell }}"
        create_home: true
        state: present

    - name: Install the authorized key
      ansible.posix.authorized_key:
        user: "{{ username }}"
        key: "{{ ssh_public_key }}"
        state: present

    - name: Grant passwordless sudo (only when requested)
      community.general.sudoers:
        name: "10-{{ username }}"
        user: "{{ username }}"
        commands: ALL
        nopassword: true
        state: present
      when: grant_sudo | bool
```

- [ ] **Step 2: Syntax check**

Run: `cd v2e-ansible && ansible-playbook playbooks/ops/add-user.yml --syntax-check`
Expected: `playbook: playbooks/ops/add-user.yml` with no error.

- [ ] **Step 3: Lint**

Run: `cd v2e-ansible && ansible-lint playbooks/ops/add-user.yml`
Expected: no errors (warnings about `hosts: "{{ target }}"` are acceptable — it is deliberate).

- [ ] **Step 4: Assertion guard fires without args**

Run: `cd v2e-ansible && ansible-playbook playbooks/ops/add-user.yml --check`
Expected: FAILS at "Require username..." assert (proves the guard) — this is the passing result.

- [ ] **Step 5: Commit**

```bash
cd v2e-ansible && git add playbooks/ops/add-user.yml
git commit -m "ops: add-user playbook (survey-driven)"
```

---

### Task 2: `health-check` ops playbook (read-only)

**Files:**
- Create: `v2e-ansible/playbooks/ops/health-check.yml`

**Interfaces:**
- Produces: a read-only playbook `ansible-playbook playbooks/ops/health-check.yml -e target=<host|group>` that changes nothing and prints a per-node health summary. This is the FIRST template we run end-to-end (Task 7) because it is safe.

- [ ] **Step 1: Write the playbook**

```yaml
---
# On-demand read-only health report across chosen node(s). Changes nothing.
#   ansible-playbook playbooks/ops/health-check.yml -e target=all
# Reports: reachability, disk %, memory %, and (on Docker hosts) unhealthy containers.

- name: Estate health report (read-only)
  hosts: "{{ target | default('all') }}"
  become: true
  gather_facts: true
  tasks:
    - name: Disk + memory snapshot
      ansible.builtin.set_fact:
        _disk_pct: "{{ (ansible_mounts | selectattr('mount', 'eq', '/') | map(attribute='size_available') | first | float
                        / (ansible_mounts | selectattr('mount', 'eq', '/') | map(attribute='size_total') | first | float) * 100) | round(1) }}"
        _mem_pct: "{{ (ansible_memory_mb.real.available | float / ansible_memory_mb.real.total | float * 100) | round(1) }}"

    - name: Check for unhealthy Docker containers (Docker hosts only)
      ansible.builtin.command:
        cmd: docker ps --filter health=unhealthy --format '{{ "{{" }}.Names{{ "}}" }}'
      register: _unhealthy
      changed_when: false
      failed_when: false
      check_mode: false

    - name: Report
      ansible.builtin.debug:
        msg:
          - "host={{ inventory_hostname }} reachable=yes"
          - "disk_free_pct={{ _disk_pct }} mem_avail_pct={{ _mem_pct }}"
          - "unhealthy_containers={{ _unhealthy.stdout_lines | default([]) | join(',') or 'none' }}"
```

- [ ] **Step 2: Syntax check + lint**

Run: `cd v2e-ansible && ansible-playbook playbooks/ops/health-check.yml --syntax-check && ansible-lint playbooks/ops/health-check.yml`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
cd v2e-ansible && git add playbooks/ops/health-check.yml
git commit -m "ops: read-only estate health-check playbook"
```

---

### Task 2b: Make patch serial + health-gated (spec safety requirement)

**Files:**
- Modify: `v2e-ansible/playbooks/ops/patch.yml`

**Interfaces:**
- Produces: `playbooks/ops/patch.yml` that patches **one node at a time** with a reachability gate between nodes — the safety the spec requires, reusing the existing `patch` role's reboot logic unchanged.

- [ ] **Step 1: Add `serial: 1` + a post-patch reachability gate to the play**

Replace the play body of `playbooks/ops/patch.yml` (keep the header comment) with:

```yaml
- name: Orchestrated full patch (one node at a time)
  hosts: "{{ target | default('linux') }}"
  become: true
  serial: 1                      # patch + reboot a single node, then move on
  max_fail_percentage: 0         # stop the whole run if any node fails to come back
  tags:
    - patch
  roles:
    - patch
  post_tasks:
    - name: Confirm the node is back and reachable after patch/reboot
      ansible.builtin.wait_for_connection:
        timeout: 120
```

(`target` defaults to the `linux` group so "patch everything" still works; the Semaphore template's `target` survey var can narrow it. The `patch` role already excludes the controller from reboots and skips the router — the router is not in `linux` for apt patching.)

- [ ] **Step 2: Syntax check + lint**

Run: `cd v2e-ansible && ansible-playbook playbooks/ops/patch.yml --syntax-check && ansible-lint playbooks/ops/patch.yml`
Expected: no errors.

- [ ] **Step 3: Dry-run against one node**

Run: `cd v2e-ansible && ansible-playbook playbooks/ops/patch.yml --check --limit services`
Expected: runs in check mode, no changes applied, no error.

- [ ] **Step 4: Commit**

```bash
cd v2e-ansible && git add playbooks/ops/patch.yml
git commit -m "ops: patch one node at a time with a post-reboot reachability gate"
```

---

### Task 3: Semaphore API token secret + live API-shape verification

**Files:**
- Modify: `v2e-ansible/inventory/group_vars/all.sops.yaml` (add `semaphore_api_token`)
- Modify: `v2e-ansible/inventory/group_vars/services.yml` (document the new secret; do NOT add to compose_stack_required_secrets — the bootstrap is a separate play)

**Interfaces:**
- Produces: SOPS var `semaphore_api_token` (a Semaphore admin API token) usable as `{{ semaphore_api_token }}` on control. Confirmed request/response shapes for the v2.18 endpoints used in Tasks 4-6.

- [ ] **Step 1: Create the API token in Semaphore (one-time, via the UI or API)**

In the Semaphore UI (logged in as admin) → user settings → create an API token, OR:
Run from control (session-cookie login then token create):
```bash
BASE=http://10.1.2.10:3000
SPW=$(ssh services 'docker exec semaphore-semaphore-1 printenv SEMAPHORE_ADMIN_PASSWORD')
curl -s -c /tmp/sc.txt -X POST "$BASE/api/auth/login" -H 'Content-Type: application/json' \
  -d "{\"auth\":\"admin\",\"password\":\"$SPW\"}" -o /dev/null -w "login=%{http_code}\n"
curl -s -b /tmp/sc.txt -X POST "$BASE/api/user/tokens" | tee /tmp/tok.json   # -> {"id":"<token>","created":...}
```
Expected: `login=204` (or 200) and a token id in `/tmp/tok.json`.

- [ ] **Step 2: Store the token in SOPS**

```bash
cd v2e-ansible
sops set inventory/group_vars/all.sops.yaml '["semaphore_api_token"]' "\"$(python3 -c 'import json;print(json.load(open("/tmp/tok.json"))["id"])')\""
sops -d inventory/group_vars/all.sops.yaml | grep -c '^semaphore_api_token:'   # -> 1
rm -f /tmp/tok.json /tmp/sc.txt
```
Expected: `1`. (Note: SOPS file is gitignored/local per the estate model; the encrypted copy syncs to control.)

- [ ] **Step 3: Verify the v2.18 API object shapes (interface exploration — this de-risks Tasks 4-6)**

```bash
BASE=http://10.1.2.10:3000; TOK=$(sops -d inventory/group_vars/all.sops.yaml | awk -F': ' '/^semaphore_api_token:/{print $2}' | tr -d '"')
H="-H Authorization:\ Bearer\ $TOK"
curl -s $H "$BASE/api/projects" | python3 -m json.tool | head          # project list shape
curl -s $H "$BASE/api/ping"                                            # -> "pong" (token works)
```
Record the exact field names for: project (`id`,`name`), key (`/api/project/{id}/keys`: `name`,`type`,`ssh`/`login_password`), repository (`name`,`git_url`,`git_branch`,`ssh_key_id`), inventory (`name`,`type`,`inventory`,`ssh_key_id`,`become_key_id`), template (`name`,`playbook`,`inventory_id`,`repository_id`,`environment_id`,`app`,`survey_vars`). If any field differs from what Tasks 4-6 assume, adjust those tasks' JSON before implementing.

- [ ] **Step 4: Commit the (encrypted) secret reference + doc note**

```bash
cd v2e-ansible && git add inventory/group_vars/services.yml   # comment noting semaphore_api_token in SOPS
git commit -m "semaphore: document the bootstrap API token secret"
```
(The SOPS file itself is not committed per the estate's local-SOPS model.)

---

### Task 4: `semaphore_bootstrap` role — auth check, project, key store

**Files:**
- Create: `v2e-ansible/roles/semaphore_bootstrap/defaults/main.yml`
- Create: `v2e-ansible/roles/semaphore_bootstrap/tasks/main.yml`

**Interfaces:**
- Consumes: `semaphore_api_token` (SOPS), `semaphore_api_base` (default below).
- Produces: role facts `_sem_project_id`, `_sem_ssh_key_id`, `_sem_become_key_id` used by Tasks 5-6.

- [ ] **Step 1: Write defaults**

```yaml
---
# roles/semaphore_bootstrap/defaults/main.yml
semaphore_api_base: "http://10.1.2.10:3000"
semaphore_project_name: "v2e system"
semaphore_repo_url: "https://github.com/v2e-sh/v2e-ansible.git"
semaphore_repo_branch: "main"
# The mesh SSH private key the runner uses to reach nodes, + the sudo/become password.
# Both already exist in SOPS for the estate; referenced by name here.
semaphore_ssh_private_key: "{{ mesh_ssh_private_key | default('') }}"
semaphore_become_password: "{{ sudo_password | default('') }}"
```

- [ ] **Step 2: Write the auth + project + keys tasks**

```yaml
---
# roles/semaphore_bootstrap/tasks/main.yml
# Idempotent seed of the "v2e system" project. Runs from control (delegate_to: localhost
# is NOT used — this play targets control which reaches the Semaphore VLAN port).
- name: Semaphore — auth headers
  ansible.builtin.set_fact:
    _sem_headers:
      Authorization: "Bearer {{ semaphore_api_token }}"
      Content-Type: "application/json"
  no_log: true

- name: Semaphore — ping (token valid?)
  ansible.builtin.uri:
    url: "{{ semaphore_api_base }}/api/ping"
    headers: "{{ _sem_headers }}"
    return_content: true
  register: _sem_ping
  failed_when: "'pong' not in _sem_ping.content"

- name: Semaphore — list projects
  ansible.builtin.uri:
    url: "{{ semaphore_api_base }}/api/projects"
    headers: "{{ _sem_headers }}"
    return_content: true
  register: _sem_projects

- name: Semaphore — create project if absent
  ansible.builtin.uri:
    url: "{{ semaphore_api_base }}/api/projects"
    method: POST
    headers: "{{ _sem_headers }}"
    body_format: json
    body:
      name: "{{ semaphore_project_name }}"
    status_code: [200, 201]
  register: _sem_new_project
  when: semaphore_project_name not in (_sem_projects.json | map(attribute='name') | list)

- name: Semaphore — resolve project id
  ansible.builtin.set_fact:
    _sem_project_id: >-
      {{ (_sem_new_project.json.id if _sem_new_project is not skipped
          else (_sem_projects.json | selectattr('name','eq',semaphore_project_name) | map(attribute='id') | first)) }}

- name: Semaphore — list keys
  ansible.builtin.uri:
    url: "{{ semaphore_api_base }}/api/project/{{ _sem_project_id }}/keys"
    headers: "{{ _sem_headers }}"
    return_content: true
  register: _sem_keys

- name: Semaphore — create SSH mesh key if absent
  ansible.builtin.uri:
    url: "{{ semaphore_api_base }}/api/project/{{ _sem_project_id }}/keys"
    method: POST
    headers: "{{ _sem_headers }}"
    body_format: json
    body:
      name: "mesh-ssh"
      type: "ssh"
      project_id: "{{ _sem_project_id | int }}"
      ssh:
        login: "ansible"
        private_key: "{{ semaphore_ssh_private_key }}"
    status_code: [200, 201]
  no_log: true
  when: "'mesh-ssh' not in (_sem_keys.json | map(attribute='name') | list)"

- name: Semaphore — create become (sudo) key if absent
  ansible.builtin.uri:
    url: "{{ semaphore_api_base }}/api/project/{{ _sem_project_id }}/keys"
    method: POST
    headers: "{{ _sem_headers }}"
    body_format: json
    body:
      name: "become-sudo"
      type: "login_password"
      project_id: "{{ _sem_project_id | int }}"
      login_password:
        login: ""
        password: "{{ semaphore_become_password }}"
    status_code: [200, 201]
  no_log: true
  when: "'become-sudo' not in (_sem_keys.json | map(attribute='name') | list)"

- name: Semaphore — re-list keys (resolve ids)
  ansible.builtin.uri:
    url: "{{ semaphore_api_base }}/api/project/{{ _sem_project_id }}/keys"
    headers: "{{ _sem_headers }}"
    return_content: true
  register: _sem_keys2

- name: Semaphore — set key id facts
  ansible.builtin.set_fact:
    _sem_ssh_key_id: "{{ _sem_keys2.json | selectattr('name','eq','mesh-ssh') | map(attribute='id') | first }}"
    _sem_become_key_id: "{{ _sem_keys2.json | selectattr('name','eq','become-sudo') | map(attribute='id') | first }}"
```

- [ ] **Step 3: Verify field names against Task 3's findings**

Cross-check the `keys` POST body (`type`, `ssh.login`, `ssh.private_key`, `login_password.password`) against what Task 3 Step 3 recorded for v2.18. Fix inline if different (older/newer Semaphore nested these differently).

- [ ] **Step 4: Commit**

```bash
cd v2e-ansible && git add roles/semaphore_bootstrap/
git commit -m "semaphore_bootstrap: project + key store (idempotent)"
```

---

### Task 5: `semaphore_bootstrap` role — repository, inventory, environment

**Files:**
- Modify: `v2e-ansible/roles/semaphore_bootstrap/tasks/main.yml` (append)

**Interfaces:**
- Consumes: `_sem_project_id`, `_sem_ssh_key_id`, `_sem_become_key_id` (Task 4).
- Produces: `_sem_repo_id`, `_sem_inventory_id`, `_sem_env_id` (Task 6).

- [ ] **Step 1: Append list-guard-create for repository, inventory, environment (all idempotent)**

```yaml
# --- repository ---
- name: Semaphore — list repositories
  ansible.builtin.uri:
    url: "{{ semaphore_api_base }}/api/project/{{ _sem_project_id }}/repositories"
    headers: "{{ _sem_headers }}"
    return_content: true
  register: _sem_repos

- name: Semaphore — create v2e-ansible repository if absent
  ansible.builtin.uri:
    url: "{{ semaphore_api_base }}/api/project/{{ _sem_project_id }}/repositories"
    method: POST
    headers: "{{ _sem_headers }}"
    body_format: json
    body:
      name: "v2e-ansible"
      project_id: "{{ _sem_project_id | int }}"
      git_url: "{{ semaphore_repo_url }}"
      git_branch: "{{ semaphore_repo_branch }}"
      ssh_key_id: "{{ _sem_ssh_key_id | int }}"   # HTTPS public repo: may need a 'none'-type key — see Task 3 / Notes
    status_code: [200, 201]
  when: "'v2e-ansible' not in (_sem_repos.json | map(attribute='name') | list)"

# --- inventory ---
- name: Semaphore — list inventories
  ansible.builtin.uri:
    url: "{{ semaphore_api_base }}/api/project/{{ _sem_project_id }}/inventory"
    headers: "{{ _sem_headers }}"
    return_content: true
  register: _sem_invs

- name: Semaphore — create inventory if absent
  ansible.builtin.uri:
    url: "{{ semaphore_api_base }}/api/project/{{ _sem_project_id }}/inventory"
    method: POST
    headers: "{{ _sem_headers }}"
    body_format: json
    body:
      name: "v2e estate"
      project_id: "{{ _sem_project_id | int }}"
      type: "file"                       # inventory file inside the repo
      inventory: "inventory/hosts.ini"
      ssh_key_id: "{{ _sem_ssh_key_id | int }}"
      become_key_id: "{{ _sem_become_key_id | int }}"
    status_code: [200, 201]
  when: "'v2e estate' not in (_sem_invs.json | map(attribute='name') | list)"

# --- environment ---
- name: Semaphore — list environments
  ansible.builtin.uri:
    url: "{{ semaphore_api_base }}/api/project/{{ _sem_project_id }}/environment"
    headers: "{{ _sem_headers }}"
    return_content: true
  register: _sem_envs

- name: Semaphore — create empty environment if absent
  ansible.builtin.uri:
    url: "{{ semaphore_api_base }}/api/project/{{ _sem_project_id }}/environment"
    method: POST
    headers: "{{ _sem_headers }}"
    body_format: json
    body:
      name: "default"
      project_id: "{{ _sem_project_id | int }}"
      json: "{}"
    status_code: [200, 201]
  when: "'default' not in (_sem_envs.json | map(attribute='name') | list)"
```

- [ ] **Step 2: Resolve the ids (re-GET each collection, set facts)**

```yaml
- name: Semaphore — re-list repositories/inventories/environments to resolve ids
  ansible.builtin.uri:
    url: "{{ semaphore_api_base }}/api/project/{{ _sem_project_id }}/{{ item }}"
    headers: "{{ _sem_headers }}"
    return_content: true
  loop: [repositories, inventory, environment]
  register: _sem_relist

- name: Semaphore — set repo/inventory/env id facts
  ansible.builtin.set_fact:
    _sem_repo_id: "{{ _sem_relist.results[0].json | selectattr('name','eq','v2e-ansible') | map(attribute='id') | first }}"
    _sem_inventory_id: "{{ _sem_relist.results[1].json | selectattr('name','eq','v2e estate') | map(attribute='id') | first }}"
    _sem_env_id: "{{ _sem_relist.results[2].json | selectattr('name','eq','default') | map(attribute='id') | first }}"
```

- [ ] **Step 3: Syntax check the role via a throwaway play**

Run: `cd v2e-ansible && ansible-playbook playbooks/ops/semaphore-bootstrap.yml --syntax-check` (playbook created in Task 7; if running Task 5 first, use a 2-line temp play importing the role).
Expected: no syntax error.

- [ ] **Step 4: Commit**

```bash
cd v2e-ansible && git add roles/semaphore_bootstrap/tasks/main.yml
git commit -m "semaphore_bootstrap: repository + inventory + environment"
```

---

### Task 6: `semaphore_bootstrap` role — the three templates

**Files:**
- Modify: `v2e-ansible/roles/semaphore_bootstrap/tasks/main.yml` (append)
- Create: `v2e-ansible/roles/semaphore_bootstrap/vars/templates.yml`

**Interfaces:**
- Consumes: `_sem_project_id`, `_sem_repo_id`, `_sem_inventory_id`, `_sem_env_id`.
- Produces: three Semaphore task templates (patch-upgrade, add-user, health-check) with survey vars.

- [ ] **Step 1: Define the template list**

```yaml
---
# roles/semaphore_bootstrap/vars/templates.yml
semaphore_templates:
  - name: "Estate health check (read-only)"
    playbook: "playbooks/ops/health-check.yml"
    survey_vars:
      - {name: target, title: "Target (host/group/all)", type: "", required: true}
  - name: "Patch & upgrade"
    playbook: "playbooks/ops/patch.yml"
    survey_vars:
      - {name: target, title: "Limit (host/group; blank=all linux)", type: "", required: false}
      - {name: patch_upgrade, title: "apt mode (dist/safe)", type: "", required: false}
  - name: "Add user"
    playbook: "playbooks/ops/add-user.yml"
    survey_vars:
      - {name: target, title: "Target (host/group)", type: "", required: true}
      - {name: username, title: "Username", type: "", required: true}
      - {name: ssh_public_key, title: "SSH public key", type: "", required: true}
      - {name: grant_sudo, title: "Grant sudo (true/false)", type: "", required: false}
```

- [ ] **Step 2: Append the idempotent template-create loop**

```yaml
- name: Semaphore — load template definitions
  ansible.builtin.include_vars: templates.yml

- name: Semaphore — list existing templates
  ansible.builtin.uri:
    url: "{{ semaphore_api_base }}/api/project/{{ _sem_project_id }}/templates"
    headers: "{{ _sem_headers }}"
    return_content: true
  register: _sem_tmpls

- name: Semaphore — create each template if absent
  ansible.builtin.uri:
    url: "{{ semaphore_api_base }}/api/project/{{ _sem_project_id }}/templates"
    method: POST
    headers: "{{ _sem_headers }}"
    body_format: json
    body:
      project_id: "{{ _sem_project_id | int }}"
      name: "{{ item.name }}"
      playbook: "{{ item.playbook }}"
      inventory_id: "{{ _sem_inventory_id | int }}"
      repository_id: "{{ _sem_repo_id | int }}"
      environment_id: "{{ _sem_env_id | int }}"
      app: "ansible"
      type: ""
      survey_vars: "{{ item.survey_vars }}"
    status_code: [200, 201]
  loop: "{{ semaphore_templates }}"
  loop_control:
    label: "{{ item.name }}"
  when: item.name not in (_sem_tmpls.json | map(attribute='name') | list)
```

- [ ] **Step 3: Verify template body fields against Task 3 findings; fix inline if v2.18 differs (`survey_vars` shape, `app`, `type`).**

- [ ] **Step 4: Commit**

```bash
cd v2e-ansible && git add roles/semaphore_bootstrap/
git commit -m "semaphore_bootstrap: seed the three maintenance templates"
```

---

### Task 7: Bootstrap playbook + converge + end-to-end verification

**Files:**
- Create: `v2e-ansible/playbooks/ops/semaphore-bootstrap.yml`

**Interfaces:**
- Consumes: the `semaphore_bootstrap` role (Tasks 4-6).
- Produces: a runnable seed + a verified end-to-end run of the health-check template.

- [ ] **Step 1: Write the bootstrap playbook**

```yaml
---
# Seed / reconcile the "v2e system" Semaphore project + templates. Idempotent.
# Runs on control (which can reach the Semaphore VLAN port + holds SOPS).
#   ansible-playbook playbooks/ops/semaphore-bootstrap.yml
- name: Seed the v2e system Semaphore project
  hosts: control
  gather_facts: false
  roles:
    - semaphore_bootstrap
```

- [ ] **Step 2: Syntax check + lint**

Run: `cd v2e-ansible && ansible-playbook playbooks/ops/semaphore-bootstrap.yml --syntax-check && ansible-lint playbooks/ops/semaphore-bootstrap.yml roles/semaphore_bootstrap/`
Expected: no errors.

- [ ] **Step 3: Run the bootstrap (from control)**

Run (on control, as ansible): `cd ~ansible/ansible && git pull && ansible-playbook playbooks/ops/semaphore-bootstrap.yml`
Expected: PLAY RECAP `ok`, `changed` on first run (creates objects).

- [ ] **Step 4: Idempotence — run it again**

Run: `ansible-playbook playbooks/ops/semaphore-bootstrap.yml`
Expected: `changed=0` (every object already exists; all `when` guards skip the POSTs).

- [ ] **Step 5: Verify in Semaphore + run the read-only template end-to-end**

- Confirm via API: `curl -s -H "Authorization: Bearer $TOK" $BASE/api/project/<id>/templates | python3 -m json.tool` shows the three templates.
- In the Semaphore UI (semaphore.int.v2e.sh, behind Authelia), open **"Estate health check (read-only)"**, run it with `target=services`, and confirm the task task succeeds and prints the health summary. This proves runner + repo + inventory + keys + template end to end, with zero state change.

- [ ] **Step 6: Commit**

```bash
cd v2e-ansible && git add playbooks/ops/semaphore-bootstrap.yml
git commit -m "ops: semaphore-bootstrap playbook (seed the v2e system project)"
```

---

## Notes for the implementer

- **Reused, not rebuilt:** `playbooks/ops/patch.yml` + `roles/patch` already exist and handle reboot-if-required / never-the-controller. Task 6 just points a template at `playbooks/ops/patch.yml` — do not rewrite the patch logic.
- **HTTPS repo auth:** `v2e-ansible` is public, so the repository may need a `none`-type Semaphore key rather than the SSH key. Task 3's exploration confirms whether an anon HTTPS clone works; if the repo is cloned over HTTPS with no auth, create a `type: none` key and use its id for `ssh_key_id` on the repository (Semaphore requires a key id even for anon).
- **Where it runs in the deploy:** `semaphore-bootstrap.yml` is operator-run (like the other `playbooks/ops/*`), not folded into the unattended `site.yml`. It can later be added to the services converge once proven.
- **Secret hygiene:** `no_log: true` on every task carrying the SSH private key, become password, or API token. The SOPS file stays local per the estate model.
