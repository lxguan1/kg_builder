# kg-builder Skill Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a Claude Code skill (`skills/kg-builder/`) that turns plain text or Markdown documents into a knowledge graph (nodes.csv + edges.csv) using four LLM-driven pipeline stages.

**Architecture:** Lean `SKILL.md` (~1500–2000 words) orchestrates the pipeline and delegates prompt detail to six `references/` files — one per stage plus defaults for schema and rubric. All KG reasoning is done via LLM prompts; Python is used only to write the final CSV files.

**Tech Stack:** Markdown (SKILL.md), YAML (default schema), plain text prompt templates. No Python libraries required; CSV written via Claude's Write tool.

---

## Task 1: Create skill directory structure

**Files:**
- Create: `skills/kg-builder/SKILL.md` (empty placeholder)
- Create: `skills/kg-builder/references/extract-prompts.md` (empty placeholder)
- Create: `skills/kg-builder/references/schema-prompts.md` (empty placeholder)
- Create: `skills/kg-builder/references/resolve-prompts.md` (empty placeholder)
- Create: `skills/kg-builder/references/export-format.md` (empty placeholder)
- Create: `skills/kg-builder/references/default-rubric.md` (empty placeholder)
- Create: `skills/kg-builder/references/default-schema.yaml` (empty placeholder)

**Step 1: Create directories**

```bash
mkdir -p skills/kg-builder/references
```

**Step 2: Create placeholder files**

```bash
touch skills/kg-builder/SKILL.md
touch skills/kg-builder/references/extract-prompts.md
touch skills/kg-builder/references/schema-prompts.md
touch skills/kg-builder/references/resolve-prompts.md
touch skills/kg-builder/references/export-format.md
touch skills/kg-builder/references/default-rubric.md
touch skills/kg-builder/references/default-schema.yaml
```

**Step 3: Verify structure**

```bash
find skills/ -type f | sort
```

Expected output:
```
skills/kg-builder/SKILL.md
skills/kg-builder/references/default-rubric.md
skills/kg-builder/references/default-schema.yaml
skills/kg-builder/references/export-format.md
skills/kg-builder/references/extract-prompts.md
skills/kg-builder/references/resolve-prompts.md
skills/kg-builder/references/schema-prompts.md
```

**Step 4: Commit**

```bash
git add skills/
git commit -m "chore: scaffold kg-builder skill directory structure"
```

---

## Task 2: Write `references/default-rubric.md`

This file defines the default extraction guidelines Claude follows when no user `kg_rubric.md` is provided.

**Files:**
- Write: `skills/kg-builder/references/default-rubric.md`

**Step 1: Write the file**

Content:

```markdown
# Default Extraction Rubric

This rubric defines what to extract when no user-supplied `kg_rubric.md` is present.

## Entity Types to Extract

| Type | Description | Examples |
|------|-------------|---------|
| Person | Any named individual | "Marie Curie", "the CEO", "Dr. Smith" |
| Organization | Companies, institutions, agencies, groups | "NASA", "Apple Inc.", "the UN" |
| Location | Geographic places, regions, addresses | "Paris", "the Amazon", "Building 4" |
| Event | Named occurrences with temporal bounds | "World War II", "the 2023 merger", "the summit" |
| Concept | Abstract ideas, theories, technologies, products | "machine learning", "the treaty", "COVID-19" |

## Extraction Rules

1. **Use noun phrases or proper names only.** Do not extract pronouns ("he", "they") or bare articles ("the company").
2. **Normalize surface forms minimally.** Keep the text close to what appears in the document; do not invent canonical forms yet (resolution happens in Stage 3).
3. **Capture context.** For each entity, record the sentence or clause where it appears as `context`.
4. **Extract all explicit relationships.** A relationship is explicit when the text directly states a connection between two entities (e.g., "X founded Y", "X is located in Y").
5. **Extract strongly implied relationships.** A relationship is implied when the surrounding discourse makes it highly likely (e.g., "X, the CEO of Y" implies WORKS_FOR).
6. **Do not invent relationships.** If a connection requires external knowledge not present in the document, omit it.
7. **Record evidence.** For each relationship triple, record the sentence that supports it as `evidence`.
8. **Co-reference within the document is fine.** If "the company" clearly refers to "Acme Corp", treat them as the same entity in context — but keep both surface forms; resolution happens later.

## Relationship Labels

Use concise, uppercase, underscore-separated labels. Prefer specific labels over generic ones:

| Preferred | Avoid |
|-----------|-------|
| FOUNDED_BY | RELATED_TO |
| WORKS_FOR | HAS |
| LOCATED_IN | IS |
| PARTICIPATED_IN | CONNECTED_TO |
| CAUSED_BY | — |
| MENTIONS | — |

When no specific label fits, use RELATED_TO with a note in `evidence`.

## Output Format

Stage 1 must produce valid JSON in this exact structure:

```json
{
  "entities": [
    { "id": "e1", "text": "Marie Curie", "type": "Person", "context": "Marie Curie was born in Warsaw." },
    { "id": "e2", "text": "Warsaw", "type": "Location", "context": "Marie Curie was born in Warsaw." }
  ],
  "triples": [
    { "source_id": "e1", "relation": "BORN_IN", "target_id": "e2", "evidence": "Marie Curie was born in Warsaw." }
  ]
}
```

Every entity must have a unique `id` (e1, e2, …). Triples reference entity ids, not text.
```

**Step 2: Verify word count is reasonable**

```bash
wc -w skills/kg-builder/references/default-rubric.md
```

Expected: 300–500 words.

**Step 3: Commit**

```bash
git add skills/kg-builder/references/default-rubric.md
git commit -m "feat: add default extraction rubric"
```

---

## Task 3: Write `references/default-schema.yaml`

This file defines the default graph schema used when no user `kg_schema.yaml` is present.

**Files:**
- Write: `skills/kg-builder/references/default-schema.yaml`

**Step 1: Write the file**

Content:

```yaml
# Default Knowledge Graph Schema
# Override by placing kg_schema.yaml in the working directory.

node_types:
  Person:
    description: A named individual
    properties:
      - name: string          # canonical name (filled in after Stage 3)
      - role: string          # optional role/title mentioned in text
  Organization:
    description: A company, institution, agency, or group
    properties:
      - name: string
      - industry: string      # optional
  Location:
    description: A geographic place, region, or address
    properties:
      - name: string
      - country: string       # optional
  Event:
    description: A named occurrence with temporal bounds
    properties:
      - name: string
      - date: string          # optional; ISO 8601 if known
  Concept:
    description: An abstract idea, theory, technology, or product
    properties:
      - name: string

edge_types:
  WORKS_FOR:
    description: Person works for or is employed by Organization
    source: Person
    target: Organization
  FOUNDED_BY:
    description: Organization was founded by Person
    source: Organization
    target: Person
  LOCATED_IN:
    description: Entity is geographically located in Location
    source: [Person, Organization, Event]
    target: Location
  PARTICIPATED_IN:
    description: Person or Organization participated in Event
    source: [Person, Organization]
    target: Event
  CAUSED_BY:
    description: Event or Concept was caused by another entity
    source: [Event, Concept]
    target: [Person, Organization, Event, Concept]
  MENTIONS:
    description: Generic relationship when a more specific type does not apply
    source: any
    target: any
  RELATED_TO:
    description: Fallback for implied or unclear relationships
    source: any
    target: any
```

**Step 2: Validate YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('skills/kg-builder/references/default-schema.yaml'))" && echo "VALID"
```

Expected: `VALID`

**Step 3: Commit**

```bash
git add skills/kg-builder/references/default-schema.yaml
git commit -m "feat: add default graph schema"
```

---

## Task 4: Write `references/extract-prompts.md`

LLM prompt templates for Stage 1 (entity + relationship extraction).

**Files:**
- Write: `skills/kg-builder/references/extract-prompts.md`

**Step 1: Write the file**

Content:

````markdown
# Stage 1 — Entity & Relationship Extraction Prompts

Stage 1 runs two sequential LLM passes on the input document.

## Pass 1 — Entity Extraction

Send this prompt (fill in `{DOCUMENT}` and `{RUBRIC}`):

```
You are an expert information-extraction assistant.

RUBRIC:
{RUBRIC}

DOCUMENT:
{DOCUMENT}

Task: Extract all entities from the DOCUMENT that match the types defined in the RUBRIC.
For each entity, record:
  - id: unique identifier (e1, e2, ...)
  - text: the exact surface form as it appears in the document
  - type: the entity type from the rubric
  - context: the sentence or clause where the entity appears

Output ONLY valid JSON. Do not include explanation or commentary.

Format:
{
  "entities": [
    { "id": "e1", "text": "...", "type": "...", "context": "..." }
  ]
}
```

### Filling in {RUBRIC}

- If `kg_rubric.md` exists in the working directory: insert its full contents.
- Otherwise: insert the full contents of `references/default-rubric.md`.
- If the user described a rubric inline: insert their description verbatim.

### Filling in {DOCUMENT}

Insert the full contents of the input document.
For documents longer than ~8,000 words, process in overlapping chunks of ~4,000 words (500-word overlap) and merge entity lists afterward, deduplicating by `text` + `type`.

---

## Pass 2 — Relationship Extraction

After Pass 1 completes, send this prompt (fill in `{ENTITIES}`, `{DOCUMENT}`, `{RUBRIC}`):

```
You are an expert information-extraction assistant.

RUBRIC:
{RUBRIC}

DOCUMENT:
{DOCUMENT}

ENTITIES ALREADY IDENTIFIED:
{ENTITIES}

Task: Extract all relationships between the entities listed above.
Only use entity ids from the ENTITIES list above. Do not create new entity ids.
For each relationship, record:
  - source_id: id of the source entity
  - relation: relationship label (uppercase, underscore-separated)
  - target_id: id of the target entity
  - evidence: the exact sentence that supports this relationship

Output ONLY valid JSON. Do not include explanation or commentary.

Format:
{
  "triples": [
    { "source_id": "e1", "relation": "BORN_IN", "target_id": "e2", "evidence": "..." }
  ]
}
```

### Filling in {ENTITIES}

Insert the JSON output from Pass 1.

---

## Merging Pass 1 + Pass 2 Output

Combine into a single JSON object:

```json
{
  "entities": [ ...from Pass 1... ],
  "triples": [ ...from Pass 2... ]
}
```

Store this as the Stage 1 output to pass to Stage 2.
````

**Step 2: Commit**

```bash
git add skills/kg-builder/references/extract-prompts.md
git commit -m "feat: add entity/relationship extraction prompt templates"
```

---

## Task 5: Write `references/schema-prompts.md`

LLM prompt templates for Stage 2 (schema design/validation).

**Files:**
- Write: `skills/kg-builder/references/schema-prompts.md`

**Step 1: Write the file**

Content:

````markdown
# Stage 2 — Schema Design Prompts

Stage 2 has two branches depending on whether a user schema is provided.

## Branch A — Infer Schema (no user schema)

Use this when no `kg_schema.yaml` exists in the working directory.

Send this prompt (fill in `{EXTRACTION}`, `{DEFAULT_SCHEMA}`):

```
You are a knowledge graph schema designer.

DEFAULT SCHEMA:
{DEFAULT_SCHEMA}

EXTRACTION OUTPUT (Stage 1):
{EXTRACTION}

Task: Produce a validated, final schema for this knowledge graph.

Steps:
1. Review all entity types found in the extraction.
2. Check each entity type against the DEFAULT SCHEMA.
3. If new entity types appear that are not in the DEFAULT SCHEMA, add them with appropriate properties.
4. Review all relation labels found in the extraction.
5. If new relation labels appear, add them to the schema with source/target constraints.
6. Remove any schema types that have zero instances in the extraction.

Output ONLY valid JSON in this format:
{
  "node_types": {
    "TypeName": {
      "description": "...",
      "properties": ["name", "prop2", ...]
    }
  },
  "edge_types": {
    "RELATION_LABEL": {
      "description": "...",
      "source": ["TypeName", ...],
      "target": ["TypeName", ...]
    }
  }
}
```

### Filling in {DEFAULT_SCHEMA}

Insert the full contents of `references/default-schema.yaml`.

---

## Branch B — Validate Against User Schema

Use this when `kg_schema.yaml` exists in the working directory.

Send this prompt (fill in `{EXTRACTION}`, `{USER_SCHEMA}`):

```
You are a knowledge graph schema validator.

USER SCHEMA:
{USER_SCHEMA}

EXTRACTION OUTPUT (Stage 1):
{EXTRACTION}

Task: Validate and reconcile the extraction against the user schema.

Steps:
1. For each entity in the extraction, verify its type is defined in USER SCHEMA.
   - If an entity type is missing from the schema, flag it and suggest the closest match.
   - Do NOT add new types to the schema; report mismatches.
2. For each triple in the extraction, verify its relation label is defined in USER SCHEMA.
   - If a relation label is missing, flag it and suggest the closest match.
3. Produce a reconciled extraction where all entity types and relation labels are mapped to valid schema types.

Output ONLY valid JSON:
{
  "validated_schema": { ...user schema as-is... },
  "reconciled_extraction": {
    "entities": [ { "id": "...", "text": "...", "type": "<validated type>", "context": "..." } ],
    "triples": [ { "source_id": "...", "relation": "<validated label>", "target_id": "...", "evidence": "..." } ]
  },
  "warnings": [ "Entity 'XYZ' typed as 'Product' not found in schema; mapped to 'Concept'" ]
}
```

---

## Stage 2 Output

Pass the final schema JSON (from Branch A or B) + the (possibly reconciled) extraction JSON to Stage 3.
````

**Step 2: Commit**

```bash
git add skills/kg-builder/references/schema-prompts.md
git commit -m "feat: add schema design/validation prompt templates"
```

---

## Task 6: Write `references/resolve-prompts.md`

LLM prompt templates for Stage 3 (entity resolution + ontology mapping).

**Files:**
- Write: `skills/kg-builder/references/resolve-prompts.md`

**Step 1: Write the file**

Content:

````markdown
# Stage 3 — Entity Resolution Prompts

Stage 3 deduplicates and normalizes entity surface forms, then optionally maps them to a user-supplied ontology.

## Pass 1 — Intra-document Deduplication

Send this prompt (fill in `{ENTITIES}`):

```
You are an entity resolution expert.

ENTITIES:
{ENTITIES}

Task: Identify groups of entities that refer to the same real-world thing within this document.
Criteria for grouping:
- Same name with different capitalization or punctuation ("apple inc" = "Apple Inc.")
- Abbreviation or acronym expansion ("the US" = "United States", "ML" = "machine learning")
- Pronoun/referent resolution ("the company" when only one Organization exists)
- Partial name matches when unambiguous ("Curie" = "Marie Curie" if only one Curie appears)

Do NOT merge entities that are genuinely different.

Output ONLY valid JSON:
{
  "clusters": [
    {
      "canonical_id": "e1",
      "canonical_text": "Marie Curie",
      "member_ids": ["e1", "e7", "e12"],
      "reason": "e7='Curie', e12='Dr. Curie' are the same person"
    }
  ]
}

Rules:
- canonical_id must be one of the existing entity ids.
- canonical_text should be the most complete, formal form of the name.
- Entities that have no duplicates still appear as single-member clusters.
```

### Filling in {ENTITIES}

Insert the `entities` array from Stage 2's output.

---

## Pass 2 — Update Triples

After deduplication, rewrite all triples to use canonical ids:

For each triple in the Stage 2 output:
- Replace `source_id` with the `canonical_id` of its cluster.
- Replace `target_id` with the `canonical_id` of its cluster.
- Remove self-loops (where source_id == target_id after resolution).
- Remove duplicate triples (same source, relation, target after resolution).

This pass is mechanical — no LLM call needed. Apply the cluster mapping in code.

---

## Pass 3 — User Ontology Mapping (optional)

Only run this pass if the user provides an ontology file or describes one inline.

Send this prompt (fill in `{CANONICAL_ENTITIES}`, `{ONTOLOGY}`):

```
You are an ontology mapping expert.

CANONICAL ENTITIES (after deduplication):
{CANONICAL_ENTITIES}

TARGET ONTOLOGY:
{ONTOLOGY}

Task: Map each canonical entity to the best matching concept in the TARGET ONTOLOGY.
- If a clear match exists, record the ontology URI or ID.
- If no match exists, leave ontology_id as null.
- Do not force mappings; prefer null over a bad match.

Output ONLY valid JSON:
{
  "mappings": [
    { "canonical_id": "e1", "canonical_text": "Marie Curie", "ontology_id": "Q7186", "ontology_label": "Marie Curie" },
    { "canonical_id": "e2", "canonical_text": "Warsaw", "ontology_id": null, "ontology_label": null }
  ]
}
```

---

## Stage 3 Output

Pass to Stage 4:
1. Resolved entity list: one entry per canonical entity (with optional ontology_id).
2. Resolved triple list: all triples using canonical_ids, duplicates removed.
3. The validated schema from Stage 2.
````

**Step 2: Commit**

```bash
git add skills/kg-builder/references/resolve-prompts.md
git commit -m "feat: add entity resolution prompt templates"
```

---

## Task 7: Write `references/export-format.md`

Specification for the final CSV output in Neo4j bulk-import format.

**Files:**
- Write: `skills/kg-builder/references/export-format.md`

**Step 1: Write the file**

Content:

````markdown
# Stage 4 — CSV Export Format

Stage 4 writes two CSV files using Claude's Write tool.

## nodes.csv

**Required columns** (in this order):
1. `:ID` — the entity's canonical_id (e.g., `e1`)
2. `:LABEL` — the entity type from the schema (e.g., `Person`)
3. `name` — the canonical_text from Stage 3
4. Additional property columns from the schema (one column per property; empty string if unknown)
5. `ontology_id` — the mapped ontology ID from Stage 3, or empty string

**Header row example:**
```
:ID,:LABEL,name,role,industry,country,date,ontology_id
```

**Data row examples:**
```
e1,Person,Marie Curie,physicist,,,,"Q7186"
e2,Location,Warsaw,,,Poland,,"Q270"
e3,Organization,University of Paris,,education,,,"Q209842"
```

### Rules
- All values must be quoted with double-quotes if they contain commas or newlines.
- Empty values are represented as empty strings (not NULL or N/A).
- `:ID` values must be unique across all rows.
- Each row corresponds to exactly one canonical entity.

---

## edges.csv

**Required columns** (in this order):
1. `:START_ID` — canonical_id of the source entity
2. `:END_ID` — canonical_id of the target entity
3. `:TYPE` — the relation label (e.g., `WORKS_FOR`)
4. `evidence` — the supporting sentence from Stage 1

**Header row example:**
```
:START_ID,:END_ID,:TYPE,evidence
```

**Data row examples:**
```
e1,e3,WORKS_FOR,"Marie Curie joined the University of Paris faculty in 1906."
e1,e2,BORN_IN,"Marie Curie was born in Warsaw in 1867."
```

### Rules
- Relation label in `:TYPE` must match a label defined in the schema.
- `evidence` should be the verbatim sentence from the document, double-quoted.
- No self-loops (`:START_ID` must differ from `:END_ID`).
- Duplicate rows (same START, END, TYPE) are allowed only if evidence differs.

---

## Writing the Files

Use the Write tool to create both files in the working directory:

```
Write nodes.csv with the header and one row per canonical entity.
Write edges.csv with the header and one row per resolved triple.
```

Default output location: `kg_output/nodes.csv` and `kg_output/edges.csv`.
Create the `kg_output/` directory if it does not exist (use Bash: `mkdir -p kg_output`).

---

## Verification

After writing, verify:
1. `nodes.csv` has N+1 lines (header + one per entity).
2. `edges.csv` has M+1 lines (header + one per triple).
3. Every `:START_ID` and `:END_ID` in edges.csv appears as a `:ID` in nodes.csv.

Report any orphaned edge references to the user.
````

**Step 2: Commit**

```bash
git add skills/kg-builder/references/export-format.md
git commit -m "feat: add CSV export format specification"
```

---

## Task 8: Write `SKILL.md` (core orchestration)

This is the main skill file — lean (~1500–2000 words), imperative style, third-person frontmatter.

**Files:**
- Write: `skills/kg-builder/SKILL.md`

**Step 1: Write the file**

Content:

````markdown
---
name: kg-builder
description: This skill should be used when the user asks to "build a knowledge graph", "extract entities from a document", "create a KG from", "construct a knowledge graph from", "turn this document into a graph", "knowledge graph from text", or "extract relationships from". Guides a four-stage LLM pipeline that produces nodes.csv and edges.csv from plain text or Markdown documents.
version: 0.1.0
---

# kg-builder

Construct a knowledge graph from plain text or Markdown documents using a four-stage LLM pipeline:

1. **Extract** — identify entities and relationships in the source document
2. **Schema** — design or validate the graph schema
3. **Resolve** — deduplicate entities and normalize to canonical forms
4. **Export** — emit `nodes.csv` and `edges.csv` (Neo4j bulk-import format)

All reasoning is done via LLM prompts. Python is used only to write the final CSV files.

---

## Step 0 — Session Setup

At the start of every kg-builder session, complete these checks before running any stage.

### 0a. Confirm input document

Ask the user (or accept from their message):
- The path to the input document (must be plain text or Markdown)
- Read the document using the Read tool

### 0b. Choose interaction mode

Ask: **"Run interactively (confirm after each stage) or single-shot (run all stages, produce CSVs)?"**

- **Interactive** — show output at each stage checkpoint and ask: `Proceed / Edit / Re-run`
- **Single-shot** — run all four stages without stopping, then show final CSV paths

### 0c. Resolve schema and rubric

Check the working directory for customization files:

| File | Purpose | Fallback |
|------|---------|---------|
| `kg_schema.yaml` | Graph schema (node types, edge types) | `references/default-schema.yaml` |
| `kg_rubric.md` | Extraction guidelines | `references/default-rubric.md` |

To check: use the Glob tool with pattern `kg_schema.yaml` and `kg_rubric.md`.

If neither file exists and the user described a schema/rubric inline, use their description verbatim in place of the file content.

Report to the user which files will be used before proceeding.

---

## Stage 1 — Entity & Relationship Extraction

**Load:** `references/extract-prompts.md`

Follow the two-pass procedure in that file exactly:
1. **Pass 1** — extract entities from the document
2. **Pass 2** — extract relationships between entities

Fill in `{RUBRIC}` with the resolved rubric (Step 0c).
Fill in `{DOCUMENT}` with the document content.

**Output:** JSON object with `entities` array and `triples` array.

**Interactive checkpoint:** Show entity count and triple count. Ask: `Proceed / Edit / Re-run`
- *Edit* — user can add, remove, or correct specific entities/triples before continuing
- *Re-run* — discard output and re-run Stage 1

---

## Stage 2 — Schema Design

**Load:** `references/schema-prompts.md`

Choose the branch:
- **Branch A** (infer schema) — if no `kg_schema.yaml` in working directory
- **Branch B** (validate against schema) — if `kg_schema.yaml` exists

Fill in placeholders as described in the reference file.

**Output:** Validated schema JSON + (possibly reconciled) extraction JSON.

If Branch B produces warnings, surface them to the user before continuing (even in single-shot mode).

**Interactive checkpoint:** Show schema node types and edge types. Ask: `Proceed / Edit / Re-run`

---

## Stage 3 — Entity Resolution

**Load:** `references/resolve-prompts.md`

Run in order:
1. **Pass 1** — deduplication clustering
2. **Pass 2** — update triples to use canonical ids (mechanical, no LLM call)
3. **Pass 3** — ontology mapping (only if user provided an ontology)

**Output:** Resolved entity list, resolved triple list, validated schema.

**Interactive checkpoint:** Show resolution table (surface forms → canonical names). Ask: `Proceed / Edit / Re-run`

---

## Stage 4 — CSV Export

**Load:** `references/export-format.md`

Follow the export format specification exactly.

1. Run `mkdir -p kg_output` via Bash tool
2. Write `kg_output/nodes.csv` via Write tool
3. Write `kg_output/edges.csv` via Write tool
4. Verify integrity (all edge `:START_ID`/`:END_ID` present in nodes `:ID` column)

Report the final file paths and row counts to the user:

```
Knowledge graph complete.
  nodes.csv  — N nodes  (kg_output/nodes.csv)
  edges.csv  — M edges  (kg_output/edges.csv)
```

---

## Customization

### User-supplied schema (`kg_schema.yaml`)

Place `kg_schema.yaml` in the working directory before running the skill. Format:

```yaml
node_types:
  Drug:
    description: A pharmaceutical compound
    properties: [name, cas_number, molecular_weight]
  Disease:
    description: A medical condition
    properties: [name, icd_code]

edge_types:
  TREATS:
    description: Drug treats Disease
    source: [Drug]
    target: [Disease]
```

### User-supplied rubric (`kg_rubric.md`)

Place `kg_rubric.md` in the working directory. Write it as extraction guidelines in plain English:

```markdown
Extract only Drug and Disease entities. Ignore persons and locations.
For each drug, record the drug name as text. For each disease, record the disease name.
Relationships: TREATS (drug → disease), CONTRAINDICATED_FOR (drug → disease).
Include the dosage or clinical study as evidence where available.
```

### Inline customization

If no files are provided, describe the schema and rubric conversationally at Step 0 and the skill will use your description in place of the default files.

---

## Additional Resources

### Reference Files

- **`references/extract-prompts.md`** — Prompt templates for Pass 1 (entity extraction) and Pass 2 (relationship extraction)
- **`references/schema-prompts.md`** — Prompt templates for schema inference (Branch A) and schema validation (Branch B)
- **`references/resolve-prompts.md`** — Prompt templates for deduplication, triple rewriting, and ontology mapping
- **`references/export-format.md`** — Neo4j bulk-import CSV specification and write instructions
- **`references/default-rubric.md`** — Default extraction guidelines (Person, Organization, Location, Event, Concept)
- **`references/default-schema.yaml`** — Default schema with 5 node types and 7 edge types
````

**Step 2: Count words to verify within target range**

```bash
wc -w skills/kg-builder/SKILL.md
```

Expected: 700–900 words for the body (not counting frontmatter). If significantly over 1000, move a section to references/.

**Step 3: Validate frontmatter**

```bash
python3 -c "
import re
text = open('skills/kg-builder/SKILL.md').read()
fm = re.match(r'^---\n(.*?)\n---', text, re.DOTALL)
assert fm, 'No frontmatter found'
assert 'name:' in fm.group(1), 'Missing name'
assert 'description:' in fm.group(1), 'Missing description'
assert 'This skill should be used when' in fm.group(1), 'Description not third-person'
print('Frontmatter valid')
"
```

Expected: `Frontmatter valid`

**Step 4: Commit**

```bash
git add skills/kg-builder/SKILL.md
git commit -m "feat: add kg-builder SKILL.md with four-stage pipeline orchestration"
```

---

## Task 9: Update CLAUDE.md with actual skill location

**Files:**
- Modify: `CLAUDE.md`

**Step 1: Add skill location and commands section to CLAUDE.md**

Add the following section after the existing content:

```markdown
## Skill Location

The skill lives at `skills/kg-builder/`. It is a Claude Code plugin skill — no build step required.

## Development Commands

```bash
# Validate YAML syntax of schema files
python3 -c "import yaml; yaml.safe_load(open('skills/kg-builder/references/default-schema.yaml'))" && echo "VALID"

# Check skill word count (body should be 700-1000 words)
wc -w skills/kg-builder/SKILL.md

# Check all referenced files exist
find skills/kg-builder/references/ -name "*.md" -o -name "*.yaml" | sort

# Test skill locally with Claude Code
# cc --plugin-dir /Users/huangyuanhao/projects/kg_builder
```
```

**Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md with skill location and dev commands"
```

---

## Task 10: Final validation pass

**Step 1: Run the skill-reviewer agent**

In Claude Code, say: "Review my skill at skills/kg-builder/ and check if it follows best practices"

The skill-reviewer agent will check:
- Description quality (trigger phrases, third-person)
- Content organization (progressive disclosure)
- Writing style (imperative form)

**Step 2: Verify all referenced files exist**

```bash
python3 -c "
import re, os
skill = open('skills/kg-builder/SKILL.md').read()
refs = re.findall(r'\`references/([\w.-]+)\`', skill)
missing = [r for r in refs if not os.path.exists(f'skills/kg-builder/references/{r}')]
if missing:
    print('MISSING:', missing)
else:
    print('All references exist:', refs)
"
```

Expected: `All references exist: [...]` with no missing files.

**Step 3: Final commit**

```bash
git add -A
git status  # verify nothing unexpected staged
git commit -m "feat: complete kg-builder skill v0.1.0"
```

---

## Summary of Files Created

| File | Purpose |
|------|---------|
| `skills/kg-builder/SKILL.md` | Main skill — pipeline orchestration, mode selection, config resolution |
| `skills/kg-builder/references/extract-prompts.md` | Two-pass entity + relationship extraction prompts |
| `skills/kg-builder/references/schema-prompts.md` | Schema inference (Branch A) and validation (Branch B) prompts |
| `skills/kg-builder/references/resolve-prompts.md` | Deduplication, triple rewriting, ontology mapping prompts |
| `skills/kg-builder/references/export-format.md` | Neo4j bulk-import CSV specification |
| `skills/kg-builder/references/default-rubric.md` | Default extraction guidelines |
| `skills/kg-builder/references/default-schema.yaml` | Default schema (5 node types, 7 edge types) |
