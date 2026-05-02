# Changelog

All notable changes to this project are documented in this file.

This project follows Semantic Versioning (SemVer): MAJOR.MINOR.PATCH.

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
