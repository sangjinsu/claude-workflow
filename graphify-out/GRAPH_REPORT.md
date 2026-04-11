# Graph Report - .  (2026-04-11)

## Corpus Check
- Corpus is ~4,008 words - fits in a single context window. You may not need a graph.

## Summary
- 31 nodes · 33 edges · 10 communities detected
- Extraction: 85% EXTRACTED · 15% INFERRED · 0% AMBIGUOUS · INFERRED: 5 edges (avg confidence: 0.81)
- Token cost: 0 input · 0 output

## God Nodes (most connected - your core abstractions)
1. `depends_on DAG` - 5 edges
2. `Parallel Execution (Level-based)` - 5 edges
3. `Workflow Builder` - 4 edges
4. `YAML Workflow Definition` - 4 edges
5. `Variable Substitution (${VAR})` - 4 edges
6. `Step Type: approval` - 3 edges
7. `Step Output Pipeline (${steps.<id>.output})` - 3 edges
8. `on_failure Policy (abort/continue)` - 3 edges
9. `Subcommand: run` - 3 edges
10. `Prompt-Only Architecture` - 3 edges

## Surprising Connections (you probably didn't know these)
- `Determinism Verification (--repeat N)` --conceptually_related_to--> `Parallel Execution (Level-based)`  [INFERRED]
  CHANGELOG.md → skills/workflow-builder/SKILL.md
- `type: mcp (Separate Plugin)` --conceptually_related_to--> `Prompt-Only Architecture`  [INFERRED]
  TODOS.md → CLAUDE.md
- `Workflow Builder` --references--> `YAML Workflow Definition`  [EXTRACTED]
  README.md → skills/workflow-builder/SKILL.md
- `Pipeline Orchestration Harness` --conceptually_related_to--> `Workflow Builder`  [EXTRACTED]
  CLAUDE.md → README.md
- `Pipeline Orchestration Harness` --references--> `depends_on DAG`  [INFERRED]
  CLAUDE.md → skills/workflow-builder/SKILL.md

## Hyperedges (group relationships)
- **Run Pipeline (parse → plan → execute)** — subcommand_run, variable_substitution, depends_on_dag, parallel_execution, step_output_pipeline, on_failure_policy [EXTRACTED 0.95]
- **Workflow Step Types** — step_type_command, step_type_ai, step_type_approval [EXTRACTED 1.00]
- **Deferred Future Step Types** — type_api_deferred, type_skill_deferred, type_mcp_deferred [EXTRACTED 1.00]

## Communities

### Community 0 - "Execution Engine & DAG"
Cohesion: 0.43
Nodes (7): Cycle Detection Algorithm, depends_on DAG, Determinism Verification (--repeat N), Parallel Execution (Level-based), Pipeline Orchestration Harness, Subcommand: run, Subcommand: validate

### Community 1 - "Step Types & Failure Policy"
Cohesion: 0.6
Nodes (5): on_failure Policy (abort/continue), Step Type: ai, Step Type: approval, Step Type: command, YAML Workflow Definition

### Community 2 - "Deferred Features & Roadmap"
Cohesion: 0.4
Nodes (5): Phase Roadmap (P1-P4+), Step Output Pipeline (${steps.<id>.output}), type: api (Deferred Phase 4+), type: mcp (Separate Plugin), type: skill (Deferred - RFC needed)

### Community 3 - "Plugin Architecture"
Cohesion: 0.5
Nodes (4): Claude Code Plugin, Prompt-Only Architecture, Rationale: Prompt-Only Design, Workflow Builder

### Community 4 - "Variable System"
Cohesion: 0.67
Nodes (3): CLI Override (--set KEY=VALUE), Config Profiles (dev/staging/prod), Variable Substitution (${VAR})

### Community 5 - "Security & Trust"
Cohesion: 0.67
Nodes (3): Rationale: Self-Trust Model, Shell Metacharacter Security Warning, Trust Model (self-trust)

### Community 6 - "Workflow Creation"
Cohesion: 1.0
Nodes (1): Subcommand: create

### Community 7 - "Workflow Recording"
Cohesion: 1.0
Nodes (1): Subcommand: record

### Community 8 - "Workflow Listing"
Cohesion: 1.0
Nodes (1): Subcommand: list

### Community 9 - "Workflow Inspection"
Cohesion: 1.0
Nodes (1): Subcommand: show

## Knowledge Gaps
- **11 isolated node(s):** `Config Profiles (dev/staging/prod)`, `CLI Override (--set KEY=VALUE)`, `Subcommand: create`, `Subcommand: record`, `Subcommand: list` (+6 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **Thin community `Workflow Creation`** (1 nodes): `Subcommand: create`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Workflow Recording`** (1 nodes): `Subcommand: record`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Workflow Listing`** (1 nodes): `Subcommand: list`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Workflow Inspection`** (1 nodes): `Subcommand: show`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `Workflow Builder` connect `Plugin Architecture` to `Execution Engine & DAG`, `Step Types & Failure Policy`?**
  _High betweenness centrality (0.182) - this node is a cross-community bridge._
- **Why does `Parallel Execution (Level-based)` connect `Execution Engine & DAG` to `Step Types & Failure Policy`?**
  _High betweenness centrality (0.172) - this node is a cross-community bridge._
- **Why does `depends_on DAG` connect `Execution Engine & DAG` to `Deferred Features & Roadmap`?**
  _High betweenness centrality (0.159) - this node is a cross-community bridge._
- **Are the 2 inferred relationships involving `Parallel Execution (Level-based)` (e.g. with `Determinism Verification (--repeat N)` and `Pipeline Orchestration Harness`) actually correct?**
  _`Parallel Execution (Level-based)` has 2 INFERRED edges - model-reasoned connections that need verification._
- **What connects `Config Profiles (dev/staging/prod)`, `CLI Override (--set KEY=VALUE)`, `Subcommand: create` to the rest of the system?**
  _11 weakly-connected nodes found - possible documentation gaps or missing edges._