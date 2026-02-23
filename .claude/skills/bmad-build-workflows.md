---
name: bmad-build-workflows
description: Build gh-aw workflows from BMAD source material. Run this after updating BMAD (npx bmad-method install) to regenerate all workflows.
---

# BMAD Workflow Builder

You are the orchestrator for the mm-bmad build pipeline. You read BMAD source material and produce native GitHub Agentic Workflow `.md` files.

## Overview

1. **Discover** all BMAD workflows and agent definitions
2. **Map** agents to workflows (reverse lookup from agent menus)
3. **Classify** each workflow (phase, trigger, safe-outputs, pre-steps)
4. **Convert** each workflow by dispatching a converter subagent
5. **Evaluate** each output against quality checks
6. **Approve** with the user, then write to `workflows/`

## Step 1: Discovery

### Scan Workflows

Read the directory structure at `_bmad/bmm/workflows/`. Each phase folder contains workflow subdirectories:

```
_bmad/bmm/workflows/
  1-analysis/           → planning phase workflows
  2-plan-workflows/     → planning phase workflows
  3-solutioning/        → solutioning phase workflows
  4-implementation/     → implementation phase workflows
  qa-generate-e2e-tests/→ QA workflows
```

For each workflow subdirectory, identify the instruction source:
- If `instructions.xml` exists → XML format
- If `workflow.md` or `workflow-*.md` exists → Markdown format with optional `steps/` or `steps-*` subdirectories

### Scan Agents

Read all files in `_bmad/bmm/agents/*.md`. Each agent file is a markdown file with embedded XML inside a code block. Parse the XML to extract:

```
For each agent:
  name: from <agent name="...">
  title: from <agent title="...">
  capabilities: from <agent capabilities="...">
  role: from <persona><role>
  identity: from <persona><identity>
  communication_style: from <persona><communication_style>
  principles: from <persona><principles>
  critical_actions: behavioral rules from <activation> steps that are NOT menu/UI/config patterns
  menu_workflows: list of workflow paths from <menu><item workflow="..." or exec="...">
```

### Build Reverse Map

From the agent scan, build a mapping of `workflow_path → [owning agents]`.

A workflow has multiple owning agents when more than one agent lists it in their menu. This triggers the multi-perspective pattern.

**Skip these workflow paths** (not convertible to gh-aw):
- `_bmad/core/workflows/party-mode/workflow.md` → handled separately as bmad-party-mode
- Menu items for `MH`, `CH`, `DA` (help, chat, dismiss) → interactive-only commands
- `bmad-quick-flow/` workflows → interactive-only rapid development flows

## Step 2: Workflow Classification

Use the frontmatter decision tree to classify each workflow:

### Phase → Trigger Pattern

| Phase | Trigger | Label Pattern |
|---|---|---|
| `1-analysis` | `issues: types: [labeled]` | `bmad-analysis-{workflow-name}` |
| `2-plan-workflows` | `issues: types: [labeled]` | `bmad-plan-{workflow-name}` |
| `3-solutioning` | `issues: types: [labeled]` | `bmad-solution-{workflow-name}` |
| `4-implementation` | See implementation table below | — |
| `qa-*` | `issues: types: [labeled]` | `bmad-qa-{workflow-name}` |

### Implementation Workflows (Phase 4)

| Workflow | Trigger | Safe-Outputs | Pre-Steps |
|---|---|---|---|
| `create-story` | `issues: types: [labeled]`, label: `bmad-story` | `create-pull-request`, `remove-labels` | `config-loader` |
| `dev-story` | `pull_request: types: [ready_for_review]` + `pull_request_review: types: [submitted]` | `push-to-pull-request-branch`, `add-comment`, `add-labels` | `config-loader`, `cycle-counter` |
| `code-review` | `pull_request: types: [synchronize]` | `submit-pull-request-review`, `add-comment` | `config-loader`, `skip-if-human-push` |
| `correct-course` | `issues: types: [labeled]`, label: `bmad-correct-course` | `create-pull-request`, `add-comment`, `add-labels` | `config-loader` |
| `retrospective` | `issues: types: [labeled]`, label: `bmad-retrospective` | `add-comment` | `config-loader` |
| `sprint-planning` | `issues: types: [labeled]`, label: `bmad-sprint-planning` | `add-comment` | `config-loader` |
| `sprint-status` | `issues: types: [labeled]`, label: `bmad-sprint-status` | `add-comment` | `config-loader` |

### Default Safe-Outputs by Phase

| Phase | Default Safe-Outputs |
|---|---|
| `1-analysis` | `create-pull-request`, `add-comment` |
| `2-plan-workflows` | `create-pull-request`, `add-comment` |
| `3-solutioning` | `create-pull-request`, `add-comment` |
| `qa-*` | `create-pull-request`, `add-comment` |

### Permissions

All workflows get read-only permissions. Safe-outputs handle all write operations:
```yaml
permissions:
  contents: read
  pull-requests: read
  issues: read
```

### Toolsets

All workflows get `[repos, pull_requests, issues]`. Note: `code` is NOT a valid gh-aw toolset — use `repos`.

## Step 3: Present to User

Before converting, show the user:

```
Discovered {N} workflows across {M} phases:

Phase 1 - Analysis:
  - {workflow-name} (owning agent: {agent-name})

Phase 2 - Planning:
  - {workflow-name} (owning agent: {agent-name})
  ...

Phase 3 - Solutioning:
  - {workflow-name} (owning agents: {agent1}, {agent2}) [MULTI-PERSPECTIVE]

Phase 4 - Implementation:
  - {workflow-name} (owning agent: {agent-name})
  ...

QA:
  - {workflow-name} (owning agent: {agent-name})

Which workflows should I generate?
  [A] All workflows
  [P] Select specific phases
  [W] Select specific workflows
```

## Step 4: Convert Each Workflow

For each selected workflow, prepare the converter inputs and dispatch a subagent.

### Prepare Converter Input

```
phase: "{phase-folder-name}"
workflow_name: "{workflow-directory-name}"
workflow_title: "{title-cased from workflow_name}"
purpose: "{from workflow.yaml description or workflow.md description}"
output_filename: "bmad-{workflow-name}.md"

trigger:
  type: "{label or event}"
  config: |
    {exact YAML from classification table}

safe_outputs:
  {from classification table}

pre_steps:
  {from classification table}

permissions:
  {from classification}

toolsets: [repos, pull_requests, issues]

agent_definitions:
  {for each owning agent — full extracted definition}

workflow_sources:
  instructions: "{path to instructions.xml or workflow.md}"
  checklist: "{path to checklist.md if exists, null otherwise}"
  steps:
    {list of step file paths if multi-step workflow, empty otherwise}
```

### Dispatch Subagent

For each workflow, launch a Task subagent with:
- **Prompt:** The contents of `.claude/skills/bmad-converter-prompt.md` followed by the prepared input block above
- **Type:** `general-purpose`
- The subagent must read all source files and write the output to `workflows/{output_filename}`

## Step 5: Evaluate Each Output

After each subagent completes, evaluate the generated file.

### Structural Checks (Deterministic)

| Check | Rule |
|---|---|
| Frontmatter present | File starts with `---` and has valid YAML frontmatter |
| Required fields | `on:`, `engine: claude`, `permissions:`, `tools:`, `safe-outputs:` all present |
| `source:` field | Must be `jcampb/mm-bmad/workflows/{name without .md}@main` |
| `if:` guard | `needs-human-intervention` guard present |
| Pre-steps match | Expected pre-steps from classification are present |
| Safe-outputs match | Expected safe-outputs from classification are present |

### Content Checks (Semantic)

| Check | Rule |
|---|---|
| Persona present | Single-agent has `## Your Persona`, multi-agent has `## Your Perspectives` |
| Persona specific | Uses actual BMAD agent name, role, communication style |
| All sections present | `## Principles`, `## Critical Rules`, `## Instructions`, `## Checklist` (or derived), `## Guardrails` |
| Guardrails complete | Blocker protocol + scope constraints + circuit breaker |
| No BMAD references | No `_bmad/`, `workflow.yaml`, `instructions.xml`, `{project-root}` |
| No cross-workflow refs | Does not mention other workflows by name |
| No raw writes | No `git commit`, `git push` — only safe-outputs |

### On Failure

1. Flag the specific failing check(s)
2. Re-dispatch the converter subagent with the failure details as additional context (one retry)
3. If the retry also fails, present both versions to the user and ask which to use (or let them manually edit)

## Step 6: Cross-Workflow Checks

After ALL workflows are generated, validate across the full set:

| Check | Rule |
|---|---|
| No duplicate triggers | Two workflows don't trigger on the same label |
| Guardrail consistency | Blocker protocol wording is identical across all workflows |
| Label vocabulary closed | Every label in `add-labels`/`remove-labels` is a known trigger or state label |

## Step 7: Write and Commit

For each approved workflow:
1. Ensure the file is written to `workflows/{output_filename}`
2. Present a summary to the user showing all generated files
3. Ask if user wants to commit all workflows

```bash
git add workflows/*.md
git commit -m "feat: regenerate BMAD gh-aw workflows from source"
```

## gh-aw Schema Reference

Key schema rules discovered during v2 compilation:

### Frontmatter
- **Toolsets:** Valid values are `repos`, `pull_requests`, `issues`, `context`, `actions`. NOT `code`.
- **Permissions:** Always `read`. Safe-outputs handle all write operations.
- **Safe-outputs:** YAML object (keys with null values), NOT array. E.g. `create-pull-request:` not `- create-pull-request`.
- **Label triggers:** Use `names: [label-name]` nested under `issues:`, NOT `label:` as sibling of `on:`.
- **Template expressions in pre-steps:** Must go in `env:` blocks, not directly in shell. Prevents template injection.
- **if guard:** Use `"contains(...) == false"` instead of `"!contains(...)"` for workflows with dual triggers (avoids YAML quoting issues in compiled output).

### Valid Safe-Output Names
- `push-to-pull-request-branch` (not `push-to-pr`)
- `create-pull-request` (correct)
- `submit-pull-request-review` (not `submit-pr-review`)
- `add-comment` (correct)
- `add-labels` (not `add-label`)
- `remove-labels` (not `remove-label`)

### Label Trigger Format
```yaml
on:
  issues:
    types: [labeled]
    names: [bmad-analysis-product-brief]
```

NOT:
```yaml
on:
  label: bmad-analysis-product-brief
  issues:
    types: [labeled]
```
