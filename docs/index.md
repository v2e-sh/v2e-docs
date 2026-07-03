---
hide:
  - toc
---

# v2e homelab

**v2e** is a single-host homelab defined entirely in code. Five VLAN-isolated
virtual machines run on one Proxmox host, provisioned by OpenTofu and Ansible and
serving a Docker application estate behind Traefik, TinyAuth, and Tailscale. This
site documents how the lab is built and how to operate it — and the **code in the
[repositories](#the-repositories) is the source of truth** for every claim here.

[:material-map-outline: Architecture overview](system/architecture.md){ .md-button .md-button--primary }
[:material-rocket-launch-outline: Deploy it](RUNBOOK.md){ .md-button }

## Start here

<div class="grid cards" markdown>

-   :material-sitemap:{ .lg .middle } &nbsp; __System reference__

    ---

    How the lab fits together — the nodes, the network and firewall, the
    application estate, the secrets flow, the deploy lifecycle, and observability.

    [:octicons-arrow-right-24: Explore the system](system/architecture.md)

-   :material-rocket-launch:{ .lg .middle } &nbsp; __Runbook__

    ---

    First-run deployment and day-2 operations, end to end and copy-paste — from
    building templates to a running, browsable estate.

    [:octicons-arrow-right-24: Deploy and operate](RUNBOOK.md)

-   :material-tune-variant:{ .lg .middle } &nbsp; __Configuration reference__

    ---

    Every variable in one place: `terraform.tfvars`, SOPS secrets with their
    generation commands, Ansible `group_vars`, and the template build config.

    [:octicons-arrow-right-24: Look up a variable](CONFIGURATION.md)

-   :material-shield-search:{ .lg .middle } &nbsp; __Audits__

    ---

    Findings from the automated best-practice and official-documentation audit —
    each confirmed against the code at `file:line`.

    [:octicons-arrow-right-24: Read the latest audit](audits/AUDIT-2026-07-03.md)

</div>

## At a glance

The whole lab is one Proxmox host and five virtual machines, each on its own VLAN
behind a default-deny firewall:

| | |
|---|---|
| :material-server: **Host** | One Proxmox VE host; five VMs — `router`, `control`, `services`, `agent`, `infra` |
| :material-lan: **Network** | VLAN-segmented, default-deny VyOS firewall; one WAN DNAT to control's SSH |
| :material-cog-transfer: **Provisioning** | OpenTofu builds the VMs and cloud-init; Ansible `site.yml` converges them — one unattended pass |
| :material-docker: **Applications** | Docker estate behind Traefik + TinyAuth on `*.int.v2e.sh`, with a Let's Encrypt wildcard cert |
| :material-vpn: **Mesh & DNS** | Tailscale mesh, residential and Mullvad exit nodes, split-DNS to a Technitium resolver |
| :material-key-chain: **Secrets** | Everything sensitive lives in SOPS/age; a clean rebuild reproduces the estate from nothing |

!!! tip "New to the lab?"
    Read the [Architecture overview](system/architecture.md) to see how the pieces
    fit, then follow the [Runbook](RUNBOOK.md) to stand it up.

## The repositories

The platform is split across five repositories. Each is small, single-purpose, and
independently versioned.

<div class="grid cards" markdown>

-   :material-terraform:{ .middle } &nbsp; __[v2e-tf](https://github.com/v2e-sh/v2e-tf)__

    ---

    OpenTofu — the Proxmox VMs, the VLAN network, the VyOS router, and the
    cloud-init that bootstraps each node. The deploy entry point.

-   :material-ansible:{ .middle } &nbsp; __[v2e-ansible](https://github.com/v2e-sh/v2e-ansible)__

    ---

    Configuration management — `site.yml` and the roles that harden the nodes,
    bring up Docker, and render every stack's environment from SOPS.

-   :material-docker:{ .middle } &nbsp; __[v2e-compose](https://github.com/v2e-sh/v2e-compose)__

    ---

    The Docker Compose stacks — Traefik, TinyAuth, the observability stack,
    Technitium DNS, the Mullvad exit, and the rest of the application estate.

-   :material-package-variant-closed:{ .middle } &nbsp; __[v2e-templates](https://github.com/v2e-sh/v2e-templates)__

    ---

    VM template builds — the `virt-customize` scripts that turn cloud images into
    the Proxmox templates every node is cloned from.

-   :material-book-open-variant:{ .middle } &nbsp; __[v2e-docs](https://github.com/v2e-sh/v2e-docs)__

    ---

    This documentation site — the system reference, runbook, configuration
    reference, design specs, and audit reports.

</div>
