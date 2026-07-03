# v2e-docs

Documentation and operational runbooks for the **v2e** homelab — a single-host lab
defined entirely in code: five VLAN-isolated VMs on one Proxmox host, provisioned by
OpenTofu and Ansible, serving a Docker application estate behind Traefik and Tailscale.

**📖 Browsable documentation: <https://v2e-sh.github.io/v2e-docs/>**

The rendered site is the best way to read these docs. This repository holds the source.

## Contents

- **[System reference](https://v2e-sh.github.io/v2e-docs/system/architecture/)** —
  how the lab is built: nodes, network and firewall, application estate, secrets flow,
  deploy lifecycle, and observability.
- **[Runbook](RUNBOOK.md)** — first-run deployment and day-2 operations
  (Proxmox · VyOS · cloud-init · OpenTofu · Ansible · Docker).
- **[Configuration reference](CONFIGURATION.md)** — every variable in one place:
  `terraform.tfvars`, SOPS secrets (with generation commands), Ansible `group_vars`,
  and template build config.
- **Design specs** —
  [VM templates](specs/2026-06-30-packer-templates-design.md) ·
  [Traefik + TLS](specs/2026-06-30-compose-traefik-tls-design.md) ·
  [Auth](specs/2026-06-30-compose-auth-design.md) ·
  [DNS appliance](specs/2026-06-30-dns-appliance-design.md)
- **[Audits](https://v2e-sh.github.io/v2e-docs/audits/AUDIT-2026-07-03/)** —
  automated best-practice / official-documentation audit findings, each confirmed
  against the code.

## The platform

The lab is split across five single-purpose repositories. **The code is the source of
truth** — the documentation describes what the code does.

| Repository | Role |
|---|---|
| [v2e-tf](https://github.com/v2e-sh/v2e-tf) | OpenTofu — Proxmox VMs, VLAN network, VyOS router, cloud-init. The deploy entry point. |
| [v2e-ansible](https://github.com/v2e-sh/v2e-ansible) | Configuration management — `site.yml` and the roles that converge every node. |
| [v2e-compose](https://github.com/v2e-sh/v2e-compose) | The Docker Compose application stacks. |
| [v2e-templates](https://github.com/v2e-sh/v2e-templates) | VM template builds (`virt-customize`). |
| **v2e-docs** | This documentation site. |

## Building the docs locally

The site is [MkDocs](https://www.mkdocs.org/) with the
[Material](https://squidfunk.github.io/mkdocs-material/) theme.

```bash
pip install mkdocs-material==9.7.6
mkdocs serve        # live preview at http://127.0.0.1:8000/v2e-docs/
```

Pushes to `main` build and publish to GitHub Pages automatically via
`.github/workflows/docs.yml`.
