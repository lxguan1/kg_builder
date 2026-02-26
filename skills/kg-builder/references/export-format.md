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
5. `evidence_location` — 1-based line number of the evidence sentence in the source document; populated by the post-processing script below, leave empty when writing
6. `references` — the citation keys from Stage 1 `citation_keys` field, semicolon-separated (e.g. `"[14]; [15]"`); write raw keys when creating the file, or empty string if none; expanded to full reference strings by the post-processing script below

**Header row example:**
```
:START_ID,:END_ID,:TYPE,evidence,evidence_location,references
```

**Data row examples:**
```
e1,e3,WORKS_FOR,"Marie Curie joined the University of Paris faculty in 1906.",42,""
e1,e2,BORN_IN,"Marie Curie was born in Warsaw in 1867. [14]",17,"Smith J, Jones A. Title. Journal. 2020;12(3):45-67."
```

### Rules
- Relation label in `:TYPE` must match a label defined in the schema.
- `evidence` must be the verbatim sentence from the document, double-quoted.
- `evidence_location` is populated by the location detection script; write an empty value when creating the file.
- `references` is written as raw semicolon-separated citation keys and expanded by the citation expansion script; write empty string if the triple has no citations.
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

After writing `edges.csv`, run the evidence location detection script to populate `evidence_location`.
Substitute `{DOCUMENT_PATH}` with the path to the input document from Step 0a:

```bash
python3 << 'PYEOF'
import csv

DOC_PATH = "{DOCUMENT_PATH}"
EDGES_PATH = "kg_output/edges.csv"

with open(DOC_PATH, "r", encoding="utf-8") as f:
    lines = f.readlines()
doc_text = "".join(lines)

def find_line(evidence):
    if not evidence:
        return ""
    # Verbatim search: count newlines before the match position
    idx = doc_text.find(evidence)
    if idx >= 0:
        return doc_text[:idx].count("\n") + 1
    # Fuzzy fallback using rapidfuzz
    try:
        from rapidfuzz import process, fuzz
        stripped = [l.rstrip("\n") for l in lines]
        result = process.extractOne(evidence, stripped, scorer=fuzz.partial_ratio)
        if result and result[1] >= 80:
            return result[2] + 1  # result[2] is the 0-based index
    except ImportError:
        pass
    return ""

rows = []
with open(EDGES_PATH, "r", encoding="utf-8-sig", newline="") as f:
    reader = csv.DictReader(f)
    fieldnames = list(reader.fieldnames)
    if "evidence_location" not in fieldnames:
        fieldnames.append("evidence_location")
    for row in reader:
        if not row.get("evidence_location"):
            row["evidence_location"] = find_line(row.get("evidence", ""))
        rows.append(row)

with open(EDGES_PATH, "w", encoding="utf-8-sig", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(rows)

print("Evidence locations written to", EDGES_PATH)
PYEOF
```

After running the evidence location script, run the citation expansion script to replace raw citation keys in `references` with full reference strings from `kg_output/citation_map.json`:

```bash
python3 << 'PYEOF'
import csv, json, os

EDGES_PATH = "kg_output/edges.csv"
CITATION_MAP_PATH = "kg_output/citation_map.json"

if not os.path.exists(CITATION_MAP_PATH):
    print("No citation_map.json found; skipping reference expansion.")
    exit()

with open(CITATION_MAP_PATH, "r", encoding="utf-8") as f:
    citation_map = json.load(f)

if not citation_map:
    print("citation_map is empty; skipping reference expansion.")
    exit()

rows = []
with open(EDGES_PATH, "r", encoding="utf-8-sig", newline="") as f:
    reader = csv.DictReader(f)
    fieldnames = list(reader.fieldnames)
    if "references" not in fieldnames:
        fieldnames.append("references")
    for row in reader:
        raw = row.get("references", "")
        if raw:
            keys = [k.strip() for k in raw.split(";") if k.strip()]
            expanded = [citation_map.get(k, k) for k in keys]
            row["references"] = " | ".join(expanded)
        rows.append(row)

with open(EDGES_PATH, "w", encoding="utf-8-sig", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(rows)

print("Citation references expanded in", EDGES_PATH)
PYEOF
```

After writing both files, re-encode them with a UTF-8 BOM so that Excel, Google Sheets, and other tools auto-detect the encoding correctly:

```bash
python3 -c "
for fname in ['kg_output/nodes.csv', 'kg_output/edges.csv']:
    with open(fname, 'r', encoding='utf-8-sig') as f:
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

