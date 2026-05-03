# Changelog

All notable changes to this project are documented in this file.

This project follows Semantic Versioning (SemVer): MAJOR.MINOR.PATCH.

## [0.1.6] - 2026-05-03

### Added

- `scripts/dockit-bootstrap-context.sh`: copied directly from LLM-DocKit
  4.7.0 (registered upstream in `dockit-sync-manifest.yml` with
  `strategy: copy`). POSIX shell, ~7 KB, zero external dependencies.
  Reads this repo's `LLM_START_HERE.md` "Recommended reading order:"
  section dynamically and emits a Claude Code SessionStart
  `additionalContext` JSON payload (`--json`, default) or plain text
  (`--human`) for non-Claude LLMs. Tomatic's reading order has 9
  entries; output is ~2.4 KB, well under the 10 KB SessionStart hook
  limit. Closes upstream LLM-DocKit DF-033 on the Claude Code axis for
  this repo.
- `.claude/settings.json`: new `SessionStart` hook block calling the
  script with `--json`, wrapped in
  `sh -c 'if [ -x scripts/dockit-bootstrap-context.sh ]; then ...; fi'`
  for graceful degradation. The protocol enforced: first substantive
  reply must begin with literally `Onboarding loaded.` (after reading
  the listed files) or `Onboarding skipped: <reason>` (for trivial
  requests that do not depend on architectural context). Existing
  Stop / PostToolUse / PreCompact hooks unchanged.

### Changed

### Fixed

## [0.1.5] - 2026-05-03

### Added

- D-011 addendum (2026-05-03) in `docs/llm/DECISIONS.md` recording
  the rename of the Pre-H0 service from `tomatic-esphome` to
  `esphome-builder` after audit feedback. Shared homelab services
  do not carry first-adopter project names.
- `infra.contract.yml`: new top-level `consumes:` block listing the
  shared homelab services this project depends on (`mosquitto` on
  zigbee, `esphome-builder` on NAS) with role + detail. Tomatic-local
  contract extension (additionalProperties allowed by the schema);
  migrates if `home-infra-protocol` formalises a canonical name.
  Also added planned services `tomatic-bridge` (H1) and
  `tomatic-control-core` (H4) to the existing `services:` array
  with their dependency edges.

### Changed

- Container renamed: `tomatic-esphome` → `esphome-builder`. Compose
  dir on NAS moved from `/share/Container/compose/tomatic/` to
  `/share/Container/compose/esphome-builder/`. The `tomatic/` path
  is now reserved for tomatic-owned containers (bridge, control-core,
  web operativa) starting at H1 — currently does not exist on NAS.
- `home-infra/catalog/services.yml`: id `tomatic-esphome` →
  `esphome-builder`; tags drop `tomatic`; description rewritten as
  shared homelab service with Tomatic as first adopter.
- `home-infra/docs/{INVENTORY,SERVICES,PROJECTS}.md` updated.
  PROJECTS notes that tomatic does not yet own any container; it
  consumes the shared `esphome-builder` and (planned) `mosquitto`.
- `LLM_START_HERE.md` Current Focus synced with HANDOFF (it was
  still saying "Initial scaffold + doc rewrite. No application code
  yet.").
- `infra.contract.yml`: removed the planned `tomatic-esphome-builder`
  from the `services:` array. The Builder is no longer claimed as a
  tomatic-owned service; it is consumed via the new `consumes:`
  block.

### Fixed

- Stale `LLM_START_HERE.md` Current Focus contradicting HANDOFF. The
  drift was invisible to the validator because both files were
  marker-synced.

## [0.1.4] - 2026-05-02

### Added

- **Pre-H0 deploy:** `tomatic-esphome` container live on NAS at
  `http://10.0.0.220:6052/` (image `ghcr.io/esphome/esphome:latest`,
  `network_mode: host`, compose at `/share/Container/compose/tomatic/`,
  volume `firmware/esphome/`, no `privileged`). First Tomatic service
  on the homelab. ESPHome Builder UI available for YAML exploration
  ahead of H10.
- D-011 in `docs/llm/DECISIONS.md` — pull ESPHome Builder out of source-doc
  H10 to Pre-H0 so the NAS deploy chain is validated early and the YAML
  editor is available before firmware work begins. Diff vs the source-doc
  compose: `privileged: true` removed (NAS never holds USB, first flash
  is from the operator's PC via `web.esphome.io`); compose file is
  single-service today and grows additively from H1+.
- `home-infra/catalog/services.yml` registers `tomatic-esphome` as
  `category: tools`, `interface: web` (using the just-shipped
  `home-infra-protocol` 0.2.0 field), `status.type: http`. Verified live
  by `infra-portal` at 23:28 UTC: `state=up`, HTTP 200.

### Changed

- `home-infra/docs/{INVENTORY,SERVICES,PROJECTS}.md` reflect the new
  container and the Pre-H0 status of Tomatic.

### Fixed

## [0.1.3] - 2026-05-02

### Added

- D-010 in `docs/llm/DECISIONS.md` — single shared MQTT broker on the `zigbee` RPi
  for Tomatic. Mirrors `home-infra` ADR-0011.
- `infra.contract.yml` now declares the planned MQTT topology
  (`mqtt.broker_service_id: mosquitto`, ACL topic shapes for the
  `tomatic_bridge` Mosquitto user) and `deps: [mosquitto]` on the
  planned `tomatic-web` and `tomatic-public-dashboard` services.

### Changed

- `docs/ARCHITECTURE.md` deployment table now lists Mosquitto as a shared
  service on `zigbee.home.arpa:1883` (reused, not new) instead of a
  Tomatic-only broker on NAS; storage layout drops `data/mosquitto/` on
  NAS; MQTT ACL section spells out the per-client publish/subscribe
  scopes that the broker enforces.
- `docs/STRUCTURE.md` clarifies that the simulator profile keeps its
  own local Mosquitto, but production NAS does not.
- `docs/llm/DECISIONS.md` D-004 amended to point at D-010 — the
  "two MQTT clients in the bridge" sketch from the source design
  document is replaced by a single client against `zigbee.home.arpa:1883`.
- `docs/llm/HANDOFF.md` Open Questions 1 (Mosquitto topology) and 4
  (cross-broker routing) marked as resolved by D-010.

### Fixed

## [0.1.2] - 2026-05-02

### Added

- Added `docs/reference/Tomatic_v3_2.docx` to keep the canonical Word source
  design document inside the repository.
- Added `docs/reference/README.md` documenting the reference document policy.

### Changed

- Updated documentation links and onboarding notes to point at the in-repo
  Word document instead of the external `~/src/Tomatic_v3_2.docx` path.

### Fixed

## [0.1.1] - 2026-05-02

### Added

- Project documentation populated from `~/src/Tomatic_v3_2.docx` (Carlos, May 2026):
  - README rewritten with Tomatic positioning ("agentic grow system, safe by default"), one-paragraph architecture summary, current hardware inventory.
  - PROJECT_CONTEXT with vision, objectives + success criteria, stakeholders, key components, current status, V1.0-kernel milestones H0–H11.
  - ARCHITECTURE with the 8-layer stack, hard rules R1–R12, three key flows (LLM intent → command, local emergency, catch-up), MQTT topic excerpt with retain policy, command lifecycle state machine, Plant MCP tool surface, deployment topology table, roadmap V1.0–V1.2 + future.
  - STRUCTURE with the planned monorepo layout (apps/, packages/, firmware/esphome/, presets/, prompts/, hardware/, compose/, scripts/, tests/), what exists today, naming conventions, onboarding notes.
  - HANDOFF set to **H0 — Schemas + DB** as the next session focus, with concrete next-step list (pnpm workspace, Zod schemas, drizzle, SQLite WAL, vitest, GHA test workflow), open questions (Mosquitto topology, public dashboard host, Doppler layout, MQTT cross-broker routing), and Do-Not-Touch zones.
  - DECISIONS pre-populated with D-001..D-009 extracted from source doc §20: control determinista primero, propose-vs-commands, ESPHome único, NAS-first, SQLite + WAL, cabinet prefix from V1.0, web operativa over CLI, CO₂ disabled until alarm, Vercel AI SDK from V1.1.
- LLM_START_HERE customized for Tomatic: reading order with the source `.docx` first, project-specific rules section (R1–R12 axioms, source doc is read-only, mandatory homelab updates, R11 acceptance gates).
- `infra.contract.yml` updated with concrete homelab values: id, repository, planned services (`tomatic-web`, `tomatic-public-dashboard`, `tomatic-esphome-builder`), `host: nas`, `compose: /share/Container/compose/tomatic/`, Doppler `secret_refs` for MQTT/admin/Anthropic/OTA/WiFi/Telegram.

### Changed

### Fixed

## [0.1.0] - 2026-05-02

### Added

- Initial scaffold from LLM-DocKit 4.6.1.
- Homelab profile applied: AGENTS.md, CLAUDE.md symlink, infra.contract.yml stub, `.claude/checklists/homelab-project.md`.
- GitHub repository created at https://github.com/cdelalama/tomatic (public, MIT).
- Registered in `~/src/home-infra/docs/PROJECTS.md` (Active Projects table + Project Details section).

### Changed

### Fixed
