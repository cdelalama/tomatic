# Changelog

All notable changes to this project are documented in this file.

This project follows Semantic Versioning (SemVer): MAJOR.MINOR.PATCH.

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
