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
