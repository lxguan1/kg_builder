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

To check: run the Glob tool twice — once with pattern `kg_schema.yaml`, once with pattern `kg_rubric.md`.

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
3. **Pass 3** — ontology mapping (only if user provided an ontology file named `kg_ontology.txt` or `kg_ontology.yaml` in the working directory, or described an ontology inline during Step 0)

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

## Error Handling

- **Invalid JSON from LLM:** If a stage produces malformed JSON, re-run that stage once. If it fails again, show the raw output to the user and ask whether to retry or abort.
- **Interactive mode — Edit:** When the user selects "Edit", ask them to describe the changes in plain English (e.g., "remove entity e3", "change the relation on triple 2 to FOUNDED_BY"). Apply the described changes to the JSON manually, then continue.
- **Interactive mode — Re-run:** Discard the current stage output entirely and re-run the stage prompt with the original inputs.
- **Referential integrity failure (Stage 4):** If any `:START_ID` or `:END_ID` in edges.csv has no corresponding `:ID` in nodes.csv, report the orphaned edge IDs to the user and ask whether to remove them or abort.

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
