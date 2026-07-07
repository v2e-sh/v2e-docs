# Nightly autonomous backlog

Low-risk improvement tasks that Claude may work through **unattended overnight** and
leave as PRs for morning review. This file is the durable queue; the nightly cron reads
the top not-`[done]` task, does it, and opens a PR.

## Autonomous rules (do NOT break these when running unattended)

1. **PRs only.** Every task ends in a pushed branch + a PR. **Never merge**, never
   `converge`/`ansible-playbook`/`docker` against the estate, never mutate production.
2. **No secrets.** Never read, write, decrypt (`sops -d`), or move SOPS/age material or
   any credential. Never `sops set`.
3. **Live estate is read-only at most.** Prefer working purely from the local repos. If a
   task turns out to need a converge, a decision, or a secret → **do not do it**; mark it
   `[blocked: needs review]` with a one-line reason and move on.
4. **One task, one focused PR.** Small, reviewable. Keep CI green (lint/syntax).
5. **Report.** Append what you did (PR link) to that night's `nightly-summary.md`. Update
   this file: flip the task to `[done]` (with the PR number) or `[blocked]`.
6. **Pick order:** top non-`[done]`, non-`[blocked]` task first. If all are done/blocked,
   do nothing (don't invent scope).

## Queue

- [done #26] **1. Domain-generalize the user-facing docs.** `docs/guides/using-the-apps.md` and
  the "Using the installed apps" section of `RUNBOOK.md` hardcode this deployment's real
  domain (`int.v2e.sh`, `vault.int.v2e.sh`, `grafana.int.v2e.sh`, …). Replace with the
  `int.<your-domain>` placeholder convention the rest of the runbook already uses (see the
  runbook's IP/placeholder note), and add one line noting this deployment uses `v2e.sh`.
  The *code* is already domain-agnostic (`${INTERNAL_DOMAIN}`) — this is docs only.
  Acceptance: `mkdocs build --strict` clean; PR. *(If PR #22 is still open, target its
  content; if merged, target main.)*

- [ ] **2. Per-container OOM/restart alert.** Add a Grafana alert rule
  (`v2e-compose/observability/config/grafana-alerting.yaml`) that fires when any container
  restarts/OOMs (cadvisor `container_start_time_seconds` change, or the restart metric),
  so a single-container crash-loop pages instead of going unnoticed (the Grafana-OOM class
  that was invisible until spotted by eye). **PR only — do NOT converge**; note in the PR
  that it takes effect on the next services converge.

- [ ] **3. Pin the ansible CI toolchain (nightly audit N7).** In `v2e-ansible/.github/
  workflows/ci.yml`, the syntax-check/lint jobs `pip install ansible[-lint]` unpinned.
  Pin `ansible-core` to the estate's target (2.19.5) — verify the pin doesn't break the
  jobs (they run in CI on the PR). PR.

- [ ] **4. Docs domain-consistency sweep.** After task 1, grep all `docs/` + root markdown
  for stray real-domain leaks vs the placeholder convention; fix any in a follow-up PR.

- [ ] **5. Remove AI/session-scaffolding docs (files only — keep history).** Strip the
  process fingerprints so the repo reads as ordinary human work. **Plain `git rm`** — the
  files REMAIN in git history; do **NOT** rewrite history or force-push (full scrub is a
  separate supervised job). Remove: `HANDOVER.md`, `MASTER-PLAN.md`,
  `RUNBOOK-DEVIATIONS-2026-07-01.md`. **KEEP** the product docs (`RUNBOOK.md`,
  `CONFIGURATION.md`, `docs/system/*`, `docs/guides/*`, `docs/index.md`, `README.md`) and
  `NIGHTLY-BACKLOG.md`; **LEAVE** `specs/`, `plans/`, `audits/`, `PRIVACY-ROADMAP.md`
  untouched (later full-cleanup phase). Fix inbound links so the site stays valid — known
  referrers: `CONFIGURATION.md` links to HANDOVER, the `mkdocs.yml` comment lists these,
  and HANDOVER cross-links MASTER-PLAN/DEVIATIONS (going away with it). Acceptance:
  `mkdocs build --strict` clean; PR only. Note in the PR that history scrubbing is deferred.

<!-- Add new overnight-safe tasks below. Anything needing a converge, a decision, or a
     secret does NOT belong here — those stay operator-driven. -->
