# Version

- 2026-08-28 nms-knowledge-control-loop-v1
  - Added `core/playbooks/harness-routing.md` as the canonical request-routing entry point.
  - Replaced the shared learning skill with thin verify, retro, and refresh control skills.
  - Moved NMS DOCX structure knowledge into the NMS/EMS Pack and reduced every shared skill folder to `SKILL.md`.
  - Linked the routing-first workflow from the three current consumer `AGENTS.md` files without global skill installation.
- 2026-08-28 legacy-retirement-v1
  - Moved NMS/EMS canonical documents to `packs/nms-ems/` and rewired current consumers.
  - Removed legacy `harness/` and `nms-knowledge/` after the deletion gate passed.
  - Post-removal structure, live-reference, placeholder, and six-skill validation checks passed.
- 2026-08-28 universal-core-v0
  - Added reusable `core/` policies, playbooks, and result template.
  - Added the `packs/nms-ems/` entry point for optional domain rules.
  - This initial compatibility state was superseded by `legacy-retirement-v1`.
- Seed version: local-seed-2026-06-12
- Source reference: `<FRONTEND_ROOT>`
- Scope: NMS/EMS AI harness, NMS knowledge, repo-local skills

## 2026-06-12 update
- Added FE architecture analysis guide.
- Added FE new feature flow guide.
- Added frontend feature playbook.
- Linked FE guides from three local project AGENTS.md files.
- Added six-gen-flexible runtime flow guide.
- Added WC component usage map.
- Added NE Configuration Command flow guide.
