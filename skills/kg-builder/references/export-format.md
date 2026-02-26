# Stage 4 — CSV Export Format

Stage 4 writes two CSV files using Claude's Write tool.

## nodes.csv

**Required columns** (in this order):
1. `:ID` — the entity's canonical_id (e.g., `e1`)
2. `:LABEL` — the entity type from the schema (e.g., `Person`)
3. `name` — the canonical_text from Stage 3
4. Additional property columns from the schema — only include properties that were actually extracted; omit columns with no data
5. `ontology_id` — the mapped ontology ID from Stage 3, or empty string

**Header row example:**
```
:ID,:LABEL,name,ontology_id
```

**Data row examples:**
```
e1,Person,Marie Curie,"Q7186"
e2,Location,Warsaw,"Q270"
e3,Organization,University of Paris,"Q209842"
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

After writing both files, re-encode them with a UTF-8 BOM so that Excel, Google Sheets, and other tools auto-detect the encoding correctly:

```bash
python3 -c "
for fname in ['kg_output/nodes.csv', 'kg_output/edges.csv']:
    with open(fname, 'r', encoding='utf-8') as f:
        content = f.read()
    with open(fname, 'w', encoding='utf-8-sig', newline='') as f:
        f.write(content)
print('Re-encoded as UTF-8 with BOM.')
"
```

---

## Verification

After writing, verify:
1. `nodes.csv` has N+1 lines (header + one per entity).
2. `edges.csv` has M+1 lines (header + one per triple).
3. Every `:START_ID` and `:END_ID` in edges.csv appears as a `:ID` in nodes.csv.

Report any orphaned edge references to the user.

---

