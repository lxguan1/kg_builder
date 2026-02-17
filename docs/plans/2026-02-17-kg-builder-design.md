# kg-builder Skill Design

**Date:** 2026-02-17
**Status:** Approved

## Overview

`kg-builder` is a Claude Code skill that constructs knowledge graphs from plain text or Markdown documents. It extracts entities and relationships using LLM prompts, designs or validates a graph schema, resolves entities to canonical forms, and emits `nodes.csv` + `edges.csv` in Neo4j bulk-import format.

## Decisions Made

| Question | Decision |
|----------|----------|
| Input format | Plain text / Markdown |
| Interaction model | Configurable — user chooses interactive or single-shot at the start |
| Customization source | File first (`kg_schema.yaml`, `kg_rubric.md`); inline fallback |
| Python tooling | LLM-only; Python used only for file I/O (Write tool) |
| Entity resolution | Infer canonical forms from document; optionally apply user ontology |
| Architecture | Option B — lean SKILL.md + references/ for prompts and schemas |

## File Structure

```
kg_builder/
├── CLAUDE.md
├── docs/plans/
│   └── 2026-02-17-kg-builder-design.md
└── skills/
    └── kg-builder/
        ├── SKILL.md
        └── references/
            ├── extract-prompts.md
            ├── schema-prompts.md
            ├── resolve-prompts.md
            ├── export-format.md
            ├── default-rubric.md
            └── default-schema.yaml
```

## SKILL.md Responsibilities

- Frontmatter description with trigger phrases: "build a knowledge graph", "extract entities from", "create KG from", "construct knowledge graph from", "knowledge graph from document"
- Mode selection at session start: interactive vs. single-shot
- Config resolution: check for `kg_schema.yaml` / `kg_rubric.md` in working directory; fall back to `references/default-*`
- Stage orchestration: load the appropriate `references/` file for each stage and execute the pipeline
- Interactive checkpoints (interactive mode): show output after each stage, ask Proceed / Edit / Re-run

## Stage Details

### Stage 1 — Entity & Relationship Extraction (`references/extract-prompts.md`)
- Two-pass extraction: (1) entity mentions with types + context, (2) relationships between entities
- Output: JSON list of entities `{text, type, context}` and triples `{source, relation, target, evidence}`

### Stage 2 — Schema Design (`references/schema-prompts.md`)
- Either infer schema from extraction output, or validate against user-supplied schema
- Output: schema definition with node types + property lists + edge types

### Stage 3 — Entity Resolution (`references/resolve-prompts.md`)
- Cluster near-duplicate entity surface forms, elect canonical names
- Optionally map canonicals to user-supplied ontology
- Output: resolution table `{surface_form → canonical_id}`

### Stage 4 — CSV Export (`references/export-format.md`)
- Produce `nodes.csv` (`:ID`, `:LABEL`, `name`, additional properties)
- Produce `edges.csv` (`:START_ID`, `:END_ID`, `:TYPE`, additional properties)
- Format follows Neo4j bulk import conventions

## Default Rubric (`references/default-rubric.md`)
- Extract named entities: Person, Organization, Location, Event, Concept
- Capture all explicit and strongly implied relationships
- Ignore pronouns; use noun phrases or proper names only
- Include sentence context as evidence for each triple

## Default Schema (`references/default-schema.yaml`)
- Node types: Person, Organization, Location, Event, Concept
- Edge types: RELATED_TO, WORKS_FOR, LOCATED_IN, PARTICIPATED_IN, MENTIONS, CAUSED_BY

## Customization Contract
Users provide:
- `kg_schema.yaml` — overrides default schema (node types, edge types, property definitions)
- `kg_rubric.md` — overrides default extraction rubric (guidelines for what to extract)
- Inline description — if no file, user can describe the schema/rubric conversationally at mode-selection step
