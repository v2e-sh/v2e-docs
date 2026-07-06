# Semaphore DevOps Platform — Design

**Date:** 2026-07-06
**Status:** draft for review
**Scope:** Turn the (currently empty) Semaphore instance into (1) a system-maintenance
toolbox for admins and (2) a foundation for a shared, multi-tenant DevOps platform where
other people can add their own repositories and run their own playbooks — safely isolated
from the lab's secrets and nodes.

> Out of scope (separate future specs): Arcane's redeploy-on-change CI/CD role; the
> CrowdSec IPS; the Arcane multi-node (infra) environment. All discussed, all deferred.

## Goal

Semaphore runs but is unconfigured (no projects) — pure click-ops emptiness, and its
runner already sits on `control`. Two outcomes:

1. **Maintenance tier (build now):** admins run small, parameterised ops tasks across the
   fleet from a button — patch/upgrade, add-user, health check — instead of hand-running
   Ansible. Playbooks live in git; Semaphore is the trigger + audit trail.
2. **Multi-user tier (design now, build when onboarding):** the v2e env is a shared DevOps
   environment. Other people add their own repository to their own Semaphore project and
   run their own playbooks — **without** touching the lab's mesh SSH keys, SOPS age key,
   or nodes.

## The load-bearing constraint

The existing Semaphore **runner runs on `control` as the `ansible` user**, which holds the
**mesh SSH private keys + the SOPS age key** — effectively root over the entire estate.
Any playbook that runs on that runner can reach every node and decrypt every secret.
Therefore a multi-user Semaphore **cannot** let guests share that runner. Isolation is not
optional; it is the whole security model.

## Architecture — two tiers, isolated by project + runner

```
                         Semaphore (services node, Traefik + Authelia gated)
                         │
   ┌─────────────────────┴──────────────────────┐
   │ PRIVILEGED TIER (admins only)               │  UNPRIVILEGED TIER (other people)
   │ Project: "v2e system"                        │  Projects: one per user/team
   │  repos:  v2e-ansible (ops/ playbooks)        │   repos:  the user's own repo(s)
   │  keys:   mesh SSH + SOPS-decrypt (control)   │   keys:   the user's own keys
   │  runner: control runner (mesh + SOPS)  ──────┼──► runner: sandbox runner
   │  role:   estate maintenance                  │           (NO mesh keys, NO SOPS,
   │                                              │            no estate reachability)
   └──────────────────────────────────────────────┘
```

- **Project** is Semaphore's isolation boundary (own repos, key store, inventories, RBAC).
- **Runner assignment** keeps privileged and guest work on physically different runners.
  The control runner is reserved for the `v2e system` project only; guest projects are
  pinned to the sandbox runner.

## Phase 1 — Maintenance tier (build now)

### Where the automation lives
A new **`ops/` directory in `v2e-ansible`** (`playbooks/ops/*.yml`). Reuses the existing
inventory, group_vars, roles, SOPS integration, and CI. The playbooks are the durable
artifact (in git, reviewed, rebuild-safe); the Semaphore templates are thin pointers.

### Project: "v2e system"
- **Repository:** `v2e-ansible` (https, `main`).
- **Inventory:** the existing estate inventory (hosts.ini + group_vars). The control
  runner decrypts SOPS with its age key, so `become`/secret-backed tasks work.
- **Key store:** the mesh SSH key (already on control) + the become/sudo password from SOPS.
- **Runner:** the existing privileged control runner.
- **Access:** admins only.

### Standard task templates (survey-driven)

1. **Patch & upgrade** — `playbooks/ops/patch-upgrade.yml`
   - `apt update` + `apt full-upgrade` on the **Debian-family nodes only** (services,
     infra, control, agent). **Router excluded** — VyOS uses image-based upgrades, not apt.
   - Safety: `serial: 1` (one node at a time) + a health check between nodes; a **dry-run**
     survey toggle (`--check`); a **reboot-if-required** survey toggle (default off) with
     an ordered, health-gated reboot. Never patches all nodes simultaneously.
   - Survey vars: `target` (all / group / single host), `do_reboot` (bool), `check_mode` (bool).

2. **Add user** — `playbooks/ops/add-user.yml`
   - Create a user on chosen node(s): `username`, `ssh_public_key`, `sudo` (bool), `shell`.
     Idempotent; key-only (no passwords).
   - Survey vars: `target`, `username`, `ssh_public_key`, `grant_sudo`, `shell`.

3. **Estate health check** — `playbooks/ops/health-check.yml` *(read-only — the first one
   we run to prove the pipe without changing state)*
   - Container health across services + infra, Prometheus targets up, the Authelia
     auth-path probe, cert days, DNS resolution. Reports; changes nothing. Optionally
     posts a summary to ntfy.

Establish the pattern: a new maintenance task = drop a playbook in `ops/` + a thin template.

### Template provisioning — IaC vs click-ops
Semaphore stores projects/repos/templates in **its own database** (the same click-ops
fragility that left it empty). Two options:

- **(Recommended) Seed via Semaphore's API from Ansible.** A `semaphore_bootstrap` role
  (run from control, using a Semaphore admin API token from SOPS) creates the `v2e system`
  project, its repository, inventory, key-store entries, and the three templates —
  idempotently. Fully rebuild-safe: a wiped Semaphore is re-seeded on the next converge.
- **(Fallback) One-time UI setup**, documented in the runbook. The playbooks stay in git;
  only the thin templates are manual. Lower upfront effort, but a rebuild needs re-clicking.

Recommendation: **API-seed** — it's the difference between "rebuildable" and "click-ops",
which is the exact gap this whole exercise is closing.

### Verification
Run the **read-only health check** template first (proves runner + repo + inventory +
keys end to end, zero state change). Then a **`--check` dry-run** of patch-upgrade against
a single node. Only then a real patch of one low-risk node with reboot off.

## Phase 2 — Multi-user tier (design now, build at onboarding)

- **Sandbox runner:** a second Semaphore runner with **no** mesh keys, **no** SOPS age key,
  and no route to the estate's privileged nodes — its own isolated identity. Candidates:
  a container on the services node on an egress-restricted network, or a dedicated small
  VM. Registered to Semaphore with its own token; **not** `is_default`.
- **User projects:** each user/team gets a project pinned to the sandbox runner. They add
  their **own repository** (Semaphore Repositories are first-class — this is the literal
  "add a repository" the platform is for) and their **own keys** in their project's key
  store. They cannot see or use the `v2e system` project's repos, keys, or runner.
- **RBAC:** Semaphore users with per-project roles (admin/manager/task-runner). Admins own
  `v2e system`; guests are scoped to their own project(s).
- **Onboarding flow (documented):** create the user → create their project (sandbox runner)
  → they add repo + keys + templates. Optionally seed each new user project with a copy of
  a couple of harmless standard templates as a starting point.

Explicitly **not** in Phase 2 until decided: giving guest runners any estate reachability.
Default posture is total isolation; any exception is a deliberate, per-target firewall rule.

## Security model (summary)

| Concern | Control |
|---|---|
| Guests reaching the estate / secrets | Separate sandbox runner (no mesh keys, no SOPS, no route) |
| Guest work leaking into admin scope | Project isolation (own repos/keys) + RBAC |
| Privileged runner misuse | `v2e system` project is admin-only; control runner reserved to it |
| Arbitrary-repo risk on the privileged runner | Never — guest repos only ever run on the sandbox runner |
| Auditability | Semaphore records every task run (who/what/when/output) |

## Open decisions (for the reviewer)

1. **Template provisioning:** API-seed (recommended, rebuild-safe) vs documented manual UI?
2. **Sandbox-runner host (Phase 2):** container on services vs a dedicated VM? (Decide at
   onboarding time; not blocking Phase 1.)
3. **Starter template set:** the three above (patch-upgrade, add-user, health-check) —
   add/remove any before build?
4. **ops/ location:** in `v2e-ansible` (recommended, reuses everything) vs a dedicated repo?

## Rollout

1. Phase 1: `ops/` playbooks + (recommended) `semaphore_bootstrap` role → seed the
   `v2e system` project + templates → verify with the read-only health check.
2. Phase 2 (at onboarding): sandbox runner + a sample user project + onboarding doc.
