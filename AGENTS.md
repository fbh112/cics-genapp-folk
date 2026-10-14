# AGENTS.md

## Project overview

CICS GenApp ("general insurance application") – a COBOL / CICS / Db2 / VSAM sample application.
All application sources are under `base/`:

| Path | Content |
| --- | --- |
| `base/src/` | COBOL programs (`*.cbl`), copybooks (`*.cpy`), BMS map (`ssmap.bms`) |
| `base/cntl/` | JCL (Db2 create/bind, VSAM define, CICS resource definitions, web service assistant, servers) |
| `base/data/` | Sample VSAM load data |
| `base/exec/` | REXX utilities |
| `base/event-bindings/` | CICS event binding |
| `base/wsim/` | Workload Simulator scripts |
| `base/*.md` | Existing documentation (Architecture, Reference, Installation, Building, Testing) |

## Local metadata database

- Created by `scan_program` on `base/src` (31 COBOL programs; copybooks and BMS are resolved as includes).
- Location is stored in `.bobz/local-settings.json` (`databaseLocation`). `.bobz/` is git-ignored.
- Use `execute_sql_query`, `get_program_dependencies`, `get_paragraphs`, `get_control_flow` against it.
- Data dictionary (if created) is stored at `.bobz/DD.json`.

## Output location rule (MANDATORY)

**All files created by the agent (inventories, CRUD matrices, domain maps, boundary analyses,
business rules, hotspot reports, diagrams, documentation, implementation plans, impact analyses, etc.)
MUST be saved under the `docs/` folder at the repository root.** Never write generated files elsewhere
(do not modify files under `base/`).

Suggested layout:

```
docs/
  01-inventory.md
  02-db2-crud-matrix.md
  03-functional-domains.md
  04-domain-<name>-boundary.md
  05-domain-<name>-business-rules.md
  06-domain-<name>-migration-hotspots.md
  program-docs/        (per-program documentation)
  app-docs/            (application documentation)
  implementation-plans/
  impact-analysis/
```

## Conventions

- Diagrams: Mermaid fenced code blocks.
- Markdown tables for matrices; wiki-ready (Confluence-friendly) headings and tables for business rules.
- Do not change program source in Architect mode; switch to Z Code mode for code changes.
