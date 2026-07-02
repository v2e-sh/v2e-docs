# v2e-docs

Documentation and operational runbooks for the v2e homelab. Browsable site:
<https://v2e-sh.github.io/v2e-docs/>

- [RUNBOOK.md](RUNBOOK.md) — first-run deployment guide and day-2 operations
  (Proxmox + VyOS + cloud-init + OpenTofu + Ansible + Docker).
- [CONFIGURATION.md](CONFIGURATION.md) — every variable in one place: tfvars,
  SOPS secrets (with generation commands), group_vars, template build config.
- Design specs: [VM templates](specs/2026-06-30-packer-templates-design.md) ·
  [Traefik + TLS](specs/2026-06-30-compose-traefik-tls-design.md) ·
  [Auth](specs/2026-06-30-compose-auth-design.md) ·
  [DNS appliance](specs/2026-06-30-dns-appliance-design.md)

## Related repositories

- [v2e-tf](https://github.com/v2e-sh/v2e-tf) — Terraform/OpenTofu infrastructure
- [v2e-ansible](https://github.com/v2e-sh/v2e-ansible) — configuration management
- [v2e-templates](https://github.com/v2e-sh/v2e-templates) — VM template builds
- [v2e-compose](https://github.com/v2e-sh/v2e-compose) — Docker Compose stacks
