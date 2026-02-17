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
