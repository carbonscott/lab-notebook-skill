---
name: lab-notebook
description: "Log and query research notebook entries. Use when the user asks to record observations, decisions, dead-ends, questions, or milestones. Also use when they ask what we've tried, what decisions were made, or want to search the notebook. Trigger on: 'log this', 'note this', 'record', 'what have we tried', 'what did we decide', 'search the notebook', 'document progress', 'what dead-ends', 'what's open'."
user-invocable: true
argument-hint: <action or query>
---

# Lab Notebook

A structured, append-only research notebook shared across repos, agents, and humans.

## Prerequisites

These environment variables must be set (source the notebook's `.env` file):

- `LAB_NOTEBOOK_DIR` — path to the notebook data directory
- `LAB_NOTEBOOK_WRITER` — your writer ID (defaults to `$USER`)

If `LAB_NOTEBOOK_DIR` is not set, tell the user to run `lab-notebook init` first.

## Logging Entries

To record something, run `lab-notebook emit`. Two flags are required: `--context` and `--type`. Content is a positional argument.

```bash
lab-notebook emit --context <context> --type <type> [--field value] [--extra K=V] "content"
```

### Choosing the type

Types are defined in the notebook's `schema.yaml` and may vary per notebook. Run `lab-notebook schema` to see available types.

The default types and when to use them:

| Type | When to use | Example trigger |
|------|-------------|-----------------|
| `observation` | User reports a measurement, finding, or noticed behavior | "I saw that...", "the loss is...", "it turns out..." |
| `decision` | User made or confirmed a choice | "let's go with...", "we decided...", "use X because Y" |
| `dead-end` | Something was tried and failed | "that didn't work", "X failed because Y", "don't try X" |
| `question` | An open question that needs investigation | "we don't know...", "should we...", "what if..." |
| `milestone` | Something is done, working, merged, or shipped | "X is complete", "merged PR", "pipeline working" |

### Choosing the context

Use hierarchical `project/topic` naming. Context groups related entries into a research thread.

Examples: `maxie/ssl-comparison`, `maxie/scaling-laws`, `broker/migration`, `data/loading-pipeline`

If unsure, ask the user what context this falls under, or check existing contexts with `lab-notebook contexts`.

### Writing content

Summarize the key insight in 1-3 sentences. Do not dump the entire conversation — distill it. Include specific numbers, file names, or commit hashes when relevant.

### Schema-defined flags

CLI flags are generated dynamically from the notebook's `schema.yaml`. Run `lab-notebook schema` to see available fields and their types. Fields vary depending on which template was used.

For example, the `research-notebook` template (the default) includes:
- `--repo` — text field (e.g. `research-lrn091`)
- `--branch` — text field (e.g. `main`)
- `--tags` — list field, comma-separated (e.g. `mae,masking,phase0`)
- `--artifacts` — list field, comma-separated (e.g. `research-lrn091:results/S01.csv`)

The `ml-experiment-log` template includes fields like `--method`, `--dataset`, `--backbone`, `--epochs`, `--lr`, `--gpu_hours`, `--loss`, etc.

### Templates

Two bundled schema templates ship with the tool:
- `research-notebook` (default) — observations, decisions, dead-ends, questions, milestones
- `ml-experiment-log` — run-start, run-end, config-change, crash, checkpoint, comparison

```bash
# List available templates
lab-notebook template

# Initialize a new notebook with a specific template
lab-notebook init /path/to/notebook --template ml-experiment-log

# Apply a template to an existing notebook (overwrites schema.yaml)
lab-notebook template ml-experiment-log --force
```

### Extra fields (`--extra`)

For one-off fields not declared in `schema.yaml`, use `--extra key=value` (repeatable):

```bash
lab-notebook emit --context proj/topic --type observation \
    --extra reviewer=alice --extra priority=high \
    "Found a regression in the validation set."
```

- Stored as top-level keys in JSONL, as a JSON blob column in SQLite
- Values are always strings (use schema fields for typed data)
- Cannot collide with core field names (id, ts, writer_id, context, type, content) or schema field names

## Querying

Three approaches, from simplest to most powerful:

### Quick search

```bash
lab-notebook search "broker manifest"
lab-notebook search "scaling" --context maxie/scaling-laws --type decision
```

### Raw SQL

For structured queries, use `lab-notebook sql`. Run `lab-notebook schema` first if you need to see the table structure and example queries.

```bash
# Recent entries
lab-notebook sql "SELECT ts, type, substr(content,1,80) FROM entries ORDER BY ts DESC LIMIT 10"

# All dead-ends
lab-notebook sql "SELECT context, ts, substr(content,1,80) FROM entries WHERE type='dead-end' ORDER BY ts DESC"

# Decisions in a context
lab-notebook sql "SELECT ts, substr(content,1,80) FROM entries WHERE context='maxie/ssl-comparison' AND type='decision'"

# Entries by tag
lab-notebook sql "SELECT ts, context, type FROM entries WHERE EXISTS (SELECT 1 FROM json_each(tags) WHERE value='scaling')"
```

### List contexts

```bash
lab-notebook contexts
```

Shows all research threads with entry counts and date ranges. Use this for orientation when you don't know what contexts exist.

## Examples

**User says:** "log the decision that we're using cxi101235425 for early phase"

```bash
lab-notebook emit --context maxie/data-strategy --type decision \
    --tags phase0,data-selection \
    "Use only cxi101235425 for the early phase. Single-experiment training removes detector geometry and intensity distribution as confounders during initial SSL method comparison."
```

**User says:** "what dead-ends have we hit?"

```bash
lab-notebook sql "SELECT ts, context, substr(content,1,100) FROM entries WHERE type='dead-end' ORDER BY ts DESC"
```

**User says:** "search for anything about the broker"

```bash
lab-notebook search "broker"
```

$ARGUMENTS
