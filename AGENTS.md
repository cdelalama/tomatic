# LLM Agent Context

This project participates in a homelab whose source of truth lives in
`~/src/home-infra/`. Any LLM agent (Claude Code, Codex CLI, Cursor, or
any other) working in this repository must read the relevant homelab
context before making infrastructure-affecting changes.

This file is the canonical agent-instructions document for this project.
`CLAUDE.md` (if present) is a symlink to this file so Claude Code's
loader picks it up; the content stays LLM-neutral on purpose.

## Required reading (in this order)

1. `~/src/home-infra/docs/CONVENTIONS.md` — homelab build, deploy,
   secrets (Doppler), and NAS/QNAP quirks.
2. `~/src/home-infra/docs/INVENTORY.md` — current observed state of
   hosts, IPs, ports.
3. `~/src/home-infra/docs/SERVICES.md` — services already running per
   host.
4. `.claude/checklists/homelab-project.md` — local deploy checklist
   for this project (installed by the homelab profile).
5. (Optional, if declaring an `infra.contract.yml`)
   `~/src/home-infra-protocol/docs/PROJECT_CONTRACTS.md` and
   `~/src/home-infra-protocol/SPEC.md`.

## Mandatory updates

When this project's changes affect the homelab — deployment target,
exposed URL, secrets, host placement, runtime version, image tag —
the agent must update in the same session, in `~/src/home-infra/`:

- `docs/INVENTORY.md` — when hosts, IPs, or ports change.
- `docs/SERVICES.md` — when a service is added, removed, or
  relocated to another host.
- `docs/PROJECTS.md` — when **deployed reality** changes: the
  project is created or retired, the deployed version on a host
  changes, status changes, host placement changes, exposure (UI /
  API / URL) changes. Internal patch releases that never reach a
  host do not warrant a PROJECTS.md update — the table tracks
  observed state, not repo HEAD.
- `catalog/services.yml` — only if the service is portal-visible
  (`infra-portal` will render it).

These updates are not optional. The operator's global rule
(`~/.claude/CLAUDE.md`) classifies them as mandatory.

## Project Contract (optional, experimental)

This project may ship an `infra.contract.yml` describing how it
participates in the infrastructure. The format is documented in
`~/src/home-infra-protocol/docs/PROJECT_CONTRACTS.md`.

As of 2026-05, the contract is **experimental**: no project has
implemented one yet, and `infra-portal` does not consume it.
`home-infra/catalog/services.yml` remains the authoritative input.
Filling this contract validates the protocol shape on a real project;
it does not yet replace any operational step.

Compliance with `home-infra-protocol` is **not claimed** until the
protocol stabilizes and the implementation is audited
(see `home-infra-protocol/docs/GOVERNANCE.md` *Compliance Claims*).

## Anti-rules

- Do not invent infrastructure facts. Real state lives in
  `~/src/home-infra/`. If you do not see something there, it is not
  true.
- Do not duplicate `home-infra/docs/CONVENTIONS.md` content into this
  project. Link to it from `docs/operations/DEPLOY_PLAYBOOK.md`.
- Do not deploy without walking through
  `.claude/checklists/homelab-project.md`. Every checked item is a
  guarantee.
- Do not put secret values in this file or in `infra.contract.yml`.
  Secrets are referenced by Doppler variable name only.
- Do not edit `home-infra` from inside this project's automation. The
  operator does it during deploy, with the checklist as the gate.
