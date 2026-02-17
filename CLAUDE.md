# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`kg_builder` is an agent skill that constructs knowledge graphs from textual documents. It is designed to be used as a Claude Code plugin/skill.

## Standard Workflow

The pipeline has four sequential stages:

1. **Entity & Relationship Extraction** — Use an LLM to extract named entities and relationships from raw text documents.
2. **Schema Design** — Infer or apply a graph schema (node types, edge types, properties) from the extracted content.
3. **Entity Resolution & Ontology Mapping** — Deduplicate and normalize entities and relationships against a standard ontology (e.g., merge "NY" and "New York City" into a canonical node).
4. **CSV Generation** — Emit `nodes.csv` and `edges.csv` tables suitable for import into graph databases (e.g., Neo4j, NetworkX, Gephi).

The skill must also support **customized schemas and extraction rubrics** — users can provide their own entity types, relationship types, and extraction guidelines to override or extend the defaults.

## Intended Architecture

- **`skill.md`** — The skill definition file loaded by Claude Code.
- **`pipeline/`** — Core pipeline modules, one file per stage:
  - `extract.py` — Prompts and parsing logic for entity/relationship extraction.
  - `schema.py` — Schema inference and validation.
  - `resolve.py` — Entity resolution and ontology normalization.
  - `export.py` — CSV generation for nodes and edges.
- **`schemas/`** — Default and user-provided schema definitions (JSON or YAML).
- **`rubrics/`** — Default and user-provided extraction rubrics/guidelines.
- **`tests/`** — Unit and integration tests for each pipeline stage.
- **`examples/`** — Sample input documents and expected output CSVs.

## Key Design Decisions

- Each pipeline stage should be independently testable and callable.
- Schema and rubric customization is the primary extension point — keep it easy to swap in user-defined files.
- LLM calls are centralized so the underlying model can be swapped (default: `claude-sonnet-4-6`).
- Output CSV format should follow Neo4j bulk import conventions: `nodes.csv` with `:ID`, `:LABEL` columns and `edges.csv` with `:START_ID`, `:END_ID`, `:TYPE` columns.
