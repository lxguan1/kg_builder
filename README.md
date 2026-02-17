# kg-builder

A Claude Code plugin that builds knowledge graphs from plain text or Markdown documents.

## What it does

Give it a document, get back a knowledge graph. The plugin runs a four-stage LLM pipeline:

1. **Extract** — identifies entities (Person, Organization, Location, Event, Concept) and relationships
2. **Schema** — infers or validates a graph schema from the extracted content
3. **Resolve** — deduplicates entity surface forms and maps to canonical names
4. **Export** — writes `kg_output/nodes.csv` and `kg_output/edges.csv` in [Neo4j bulk-import format](https://neo4j.com/docs/operations-manual/current/tools/neo4j-admin/neo4j-admin-import/)

No Python libraries required — all reasoning is done via LLM prompts.

## Installation

```
/plugin marketplace add https://github.com/yuanhao96/kg_builder
/plugin install kg-builder 
```

## Usage

Once installed, trigger the skill by saying something like:

- *"Build a knowledge graph from this document"*
- *"Extract entities from report.md"*
- *"Construct a KG from this text"*
- *"Map entities to an ontology"*

The skill will ask whether to run **interactively** (confirm at each stage) or **single-shot** (produce CSVs in one go).

## Output

```
kg_output/
├── nodes.csv    # :ID, :LABEL, name, ontology_id
└── edges.csv    # :START_ID, :END_ID, :TYPE, evidence
```

Compatible with Neo4j, NetworkX, Gephi, and any tool that accepts CSV edge lists.

## Customization

Place these files in your working directory to override the defaults:

| File | Purpose |
|------|---------|
| `kg_schema.yaml` | Custom node/edge types and properties |
| `kg_rubric.md` | Custom extraction guidelines |
| `kg_ontology.yaml` | Target ontology for entity mapping (optional) |

**Example `kg_schema.yaml`** for a biomedical domain:
```yaml
node_types:
  Drug:
    description: A pharmaceutical compound
    properties: [name]
  Disease:
    description: A medical condition
    properties: [name]

edge_types:
  TREATS:
    description: Drug treats Disease
    source: [Drug]
    target: [Disease]
```

If no files are provided, the skill uses a general-purpose schema covering Person, Organization, Location, Event, and Concept.

## Local development

```bash
# Test locally
cc --plugin-dir /path/to/kg_builder

# Validate schema YAML
python3 -c "import yaml; yaml.safe_load(open('skills/kg-builder/references/default-schema.yaml'))" && echo VALID

# Check skill word count
wc -w skills/kg-builder/SKILL.md
```

## License

MIT
