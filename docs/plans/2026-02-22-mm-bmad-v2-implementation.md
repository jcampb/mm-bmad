# mm-bmad v2 Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Convert mm-bmad from a runtime Universal Wrapper to a build pipeline that compiles BMAD methodology into native GitHub Agentic Workflows (gh-aw).

**Architecture:** An orchestrator Claude Code skill reads BMAD agent definitions and workflow files from `_bmad/`, launches converter subagents per workflow, and produces self-contained gh-aw `.md` files with full frontmatter, personas, instructions, and guardrails. Target repos install via `gh aw add-wizard`.

**Tech Stack:** GitHub Agentic Workflows (gh-aw), Claude (engine), BMAD-METHOD (source), Claude Code skills (builder)

**Design doc:** `docs/plans/2026-02-22-mm-bmad-v2-design.md`

---

## Phase 1: Setup & Cleanup

### Task 1: Install BMAD source material

**Files:**
- Create: `_bmad/` (via npx installer)

**Step 1: Install BMAD-METHOD**

```bash
cd /Users/joshcampbell/code/mm-bmad
npx bmad-method install --yes
```

If interactive, select all modules (BMM core at minimum).

**Step 2: Verify installation**

```bash
ls _bmad/bmm/agents/
# Expected: analyst.agent.yaml, architect.agent.yaml, dev.agent.yaml,
#           pm.agent.yaml, qa.agent.yaml, sm.agent.yaml,
#           ux-designer.agent.yaml, quick-flow-solo-dev.agent.yaml,
#           tech-writer/

ls _bmad/bmm/workflows/
# Expected: 1-analysis/, 2-plan-workflows/, 3-solutioning/,
#           4-implementation/, bmad-quick-flow/, document-project/,
#           generate-project-context/, qa-generate-e2e-tests/
```

**Step 3: Commit**

```bash
git add _bmad/
git commit -m "feat: install BMAD-METHOD source material for v2 build pipeline"
```

---

### Task 2: Delete v1 files and create v2 structure

**Files:**
- Delete: `agents/` (entire directory)
- Delete: `.github/workflows/` (entire directory)
- Delete: `architecture/` (entire directory)
- Delete: `examples/` (entire directory)
- Create: `workflows/` (empty, will hold builder output)
- Create: `.claude/skills/` (will hold builder skill files)

**Step 1: Delete v1 files**

```bash
cd /Users/joshcampbell/code/mm-bmad
rm -rf agents/ .github/workflows/ architecture/ examples/
```

**Step 2: Create v2 directories**

```bash
mkdir -p workflows
mkdir -p .claude/skills
```

**Step 3: Verify structure**

```bash
ls -la
# Expected: _bmad/, workflows/, .claude/, docs/, README.md, .git/
# NOT expected: agents/, .github/, architecture/, examples/
```

**Step 4: Commit**

```bash
git add -A
git commit -m "refactor: remove v1 wrapper architecture, create v2 directory structure"
```

---

## Phase 2: Write the Converter Prompt

The converter is the core translation engine. It takes one BMAD workflow + its owning agent definition(s) and produces one gh-aw workflow `.md` file. We write it first and test it in isolation before building the orchestrator.

### Task 3: Write the converter prompt

**Files:**
- Create: `.claude/skills/bmad-converter-prompt.md`

**Step 1: Write the converter prompt**

The converter prompt must contain:
1. Role definition — "You translate BMAD workflows into gh-aw workflow files"
2. Input specification — what the orchestrator will provide (phase, workflow name, trigger pattern, safe-outputs, pre-steps, workflow source paths, agent definitions)
3. Output specification — exact structure of the gh-aw `.md` file
4. The structural template from the design doc (frontmatter + persona + principles + critical rules + instructions + checklist + guardrails)
5. Translation rules table (agent.persona.role → `## Your Persona`, etc.)
6. Multi-perspective rules — when multiple agents are provided, use the `## Your Perspectives` pattern
7. Guardrail boilerplate — the exact blocker protocol text to inject into every workflow
8. Pre-step templates — config loader shell script, cycle counter shell script
9. Constraints — no cross-workflow references, no raw write operations, scope constraints required

Write the full file to `.claude/skills/bmad-converter-prompt.md`. Target ~200 lines. Reference the design doc for exact specifications.

**Step 2: Commit**

```bash
git add .claude/skills/bmad-converter-prompt.md
git commit -m "feat: add BMAD-to-gh-aw converter prompt"
```

---

### Task 4: Test converter on dev-story (canonical single-agent workflow)

This is the most important test — dev-story has all the patterns: event triggers, pre-steps (cycle counter + config loader), safe-outputs (push-to-pr), and a single owning agent (Amelia/dev.agent.yaml).

**Files:**
- Read: `_bmad/bmm/workflows/4-implementation/dev-story/workflow.yaml`
- Read: `_bmad/bmm/workflows/4-implementation/dev-story/instructions.xml`
- Read: `_bmad/bmm/workflows/4-implementation/dev-story/checklist.md`
- Read: `_bmad/bmm/agents/dev.agent.yaml`
- Create: `workflows/bmad-dev-story.md` (test output)

**Step 1: Read all source files**

Read each file listed above to understand what the converter will receive.

**Step 2: Invoke the converter manually**

In Claude Code, simulate what the orchestrator would do: provide the converter prompt with the specific inputs for dev-story:

```
phase: "4-implementation"
workflow: "dev-story"
trigger_pattern: "pr_review_cycle"
safe_outputs: ["push-to-pr", "add-comment", "add-label"]
pre_steps: ["config-loader", "cycle-counter"]
workflow_source:
  instructions: "_bmad/bmm/workflows/4-implementation/dev-story/instructions.xml"
  checklist: "_bmad/bmm/workflows/4-implementation/dev-story/checklist.md"
agent_definitions:
  - (full contents of dev.agent.yaml)
```

**Step 3: Validate the output against structural checks**

Verify:
- [ ] Frontmatter starts with `---` and has valid YAML
- [ ] `on:` includes `pull_request: types: [ready_for_review]` and `pull_request_review: types: [submitted]`
- [ ] `engine: claude` present
- [ ] `permissions:` present with minimal scope
- [ ] `tools:` present with `github: toolsets: [repos, pull_requests, issues]`
- [ ] `safe-outputs:` includes `push-to-pr`, `add-comment`, `add-label`
- [ ] `source: jcampb/mm-bmad/workflows/bmad-dev-story@main`
- [ ] Pre-steps include config loader and cycle counter
- [ ] `needs-human-intervention` guard is present
- [ ] `## Your Persona` has Amelia's actual role, identity, communication style
- [ ] `## Principles` has dev agent's principles
- [ ] `## Critical Rules` has all critical_actions from dev.agent.yaml
- [ ] `## Instructions` contains translated content from instructions.xml (no steps dropped)
- [ ] `## Checklist` contains all items from checklist.md
- [ ] `## Guardrails` has blocker protocol with exact add-comment + add-label pattern
- [ ] No references to other workflows by name
- [ ] No raw "commit" or "push" in instructions — only safe-output references

**Step 4: Fix converter prompt if any checks fail**

Iterate on `.claude/skills/bmad-converter-prompt.md` until dev-story output passes all checks.

**Step 5: Write validated output**

```bash
# Write the validated output to workflows/bmad-dev-story.md
git add workflows/bmad-dev-story.md
git commit -m "feat: generate bmad-dev-story.md gh-aw workflow (converter test)"
```

---

### Task 5: Test converter on code-review (single-agent, different trigger pattern)

**Files:**
- Read: `_bmad/bmm/workflows/4-implementation/code-review/workflow.yaml`
- Read: `_bmad/bmm/workflows/4-implementation/code-review/instructions.xml`
- Read: `_bmad/bmm/workflows/4-implementation/code-review/checklist.md`
- Read: `_bmad/bmm/agents/dev.agent.yaml` (code-review is in dev agent's menu)
- Create: `workflows/bmad-code-review.md`

**Step 1: Invoke converter with code-review inputs**

```
phase: "4-implementation"
workflow: "code-review"
trigger_pattern: "pr_synchronize"
safe_outputs: ["submit-pr-review", "add-comment"]
pre_steps: ["config-loader", "skip-if-human-push"]
workflow_source:
  instructions: "_bmad/bmm/workflows/4-implementation/code-review/instructions.xml"
  checklist: "_bmad/bmm/workflows/4-implementation/code-review/checklist.md"
agent_definitions:
  - (dev.agent.yaml — code review is in Amelia's menu)
```

**Step 2: Validate output**

Same structural checks as Task 4, plus:
- [ ] `on:` includes `pull_request: types: [synchronize]`
- [ ] `safe-outputs:` includes `submit-pr-review` (not `push-to-pr`)
- [ ] Pre-step to skip if last push was by a human (only review agent-pushed code)
- [ ] Instructions reflect adversarial review mindset from instructions.xml

**Step 3: Iterate on converter if needed, write output**

```bash
git add workflows/bmad-code-review.md
git commit -m "feat: generate bmad-code-review.md gh-aw workflow"
```

---

### Task 6: Test converter on create-prd (planning phase, different frontmatter pattern)

**Files:**
- Read: `_bmad/bmm/workflows/2-plan-workflows/create-prd/workflow-create-prd.md`
- Read: `_bmad/bmm/workflows/2-plan-workflows/create-prd/steps-c/` (all step files)
- Read: `_bmad/bmm/workflows/2-plan-workflows/create-prd/templates/prd-template.md`
- Read: `_bmad/bmm/agents/pm.agent.yaml`
- Create: `workflows/bmad-create-prd.md`

**Step 1: Invoke converter with create-prd inputs**

```
phase: "2-plan-workflows"
workflow: "create-prd"
trigger_pattern: "issue_labeled"
safe_outputs: ["create-pull-request", "add-comment", "add-label"]
pre_steps: ["config-loader"]
workflow_source:
  instructions: "_bmad/bmm/workflows/2-plan-workflows/create-prd/workflow-create-prd.md"
  steps: "_bmad/bmm/workflows/2-plan-workflows/create-prd/steps-c/"
  template: "_bmad/bmm/workflows/2-plan-workflows/create-prd/templates/prd-template.md"
agent_definitions:
  - (pm.agent.yaml — John, the Product Manager)
```

Note: Planning workflows have multi-step processes with step files. The converter must synthesize all step files into the `## Instructions` section.

**Step 2: Validate output**

- [ ] `on:` includes `issues: types: [labeled]` with label filter
- [ ] `safe-outputs:` includes `create-pull-request` (not `push-to-pr`)
- [ ] No cycle counter pre-step (planning workflows don't loop)
- [ ] `## Your Persona` has John's PM persona
- [ ] `## Instructions` synthesizes all step-*.md files into a coherent sequence
- [ ] Template content is referenced or embedded for the agent to use

**Step 3: Iterate and commit**

```bash
git add workflows/bmad-create-prd.md
git commit -m "feat: generate bmad-create-prd.md gh-aw workflow"
```

---

### Task 7: Test converter on check-implementation-readiness (multi-agent workflow)

This tests the multi-perspective pattern. Both pm.agent.yaml and architect.agent.yaml list this workflow in their menus.

**Files:**
- Read: `_bmad/bmm/workflows/3-solutioning/check-implementation-readiness/workflow.md`
- Read: `_bmad/bmm/workflows/3-solutioning/check-implementation-readiness/steps/` (all)
- Read: `_bmad/bmm/agents/pm.agent.yaml`
- Read: `_bmad/bmm/agents/architect.agent.yaml`
- Create: `workflows/bmad-check-readiness.md`

**Step 1: Invoke converter with multi-agent inputs**

```
phase: "3-solutioning"
workflow: "check-implementation-readiness"
trigger_pattern: "issue_labeled"
safe_outputs: ["create-pull-request", "add-comment", "add-label"]
pre_steps: ["config-loader"]
workflow_source:
  instructions: "_bmad/bmm/workflows/3-solutioning/check-implementation-readiness/workflow.md"
  steps: "_bmad/bmm/workflows/3-solutioning/check-implementation-readiness/steps/"
agent_definitions:
  - (pm.agent.yaml — John)
  - (architect.agent.yaml — Winston)
```

**Step 2: Validate multi-perspective output**

- [ ] Uses `## Your Perspectives` (not `## Your Persona`)
- [ ] Has Product Manager Lens section with John's persona details
- [ ] Has Architect Lens section with Winston's persona details
- [ ] Instructions include "evaluate through each perspective" pattern
- [ ] Synthesis step at the end
- [ ] Both agents' principles are represented

**Step 3: Iterate and commit**

```bash
git add workflows/bmad-check-readiness.md
git commit -m "feat: generate bmad-check-readiness.md multi-perspective workflow"
```

---

## Phase 3: Write the Orchestrator Skill

Now that the converter is validated against three different patterns (implementation loop, planning, multi-agent), write the orchestrator that automates the full build.

### Task 8: Write the orchestrator skill

**Files:**
- Create: `.claude/skills/bmad-build-workflows.md`

**Step 1: Write the orchestrator**

The orchestrator must contain:
1. Skill metadata (name, description, trigger)
2. Discovery logic — how to scan `_bmad/bmm/workflows/` and `_bmad/bmm/agents/`
3. Agent-to-workflow reverse mapping logic — parse `menu` fields from all agent YAMLs
4. The frontmatter decision tree (from design doc) — which phase gets which trigger/safe-output pattern
5. Subagent dispatch instructions — how to invoke the converter with the right inputs
6. The full evaluation checklist — structural, content, and cross-workflow checks
7. Failure handling — flag, retry once, present both attempts on second failure
8. User interaction flow — present discovered workflows, let user select, show results for approval

Key sections:
- `## Discovery` — scan directories, parse agent menus, build mapping
- `## Workflow Classification` — the frontmatter decision tree
- `## Subagent Dispatch` — how to launch converter with inputs
- `## Evaluation` — all three check categories (structural, content, cross-workflow)
- `## User Interaction` — approval flow

**Step 2: Commit**

```bash
git add .claude/skills/bmad-build-workflows.md
git commit -m "feat: add orchestrator skill for BMAD-to-gh-aw build pipeline"
```

---

### Task 9: Test orchestrator on implementation phase subset

**Step 1: Invoke the orchestrator skill**

```
> /bmad-build-workflows
```

When prompted, select only the 4-implementation phase workflows:
- dev-story
- code-review
- create-story
- correct-course

**Step 2: Verify orchestrator behavior**

- [ ] Orchestrator correctly discovers all workflows in `_bmad/bmm/workflows/4-implementation/`
- [ ] Orchestrator correctly reads all agent YAML files
- [ ] Orchestrator correctly maps: dev-story → dev.agent, code-review → dev.agent, create-story → sm.agent, correct-course → sm.agent + pm.agent
- [ ] Orchestrator launches converter subagent for each workflow
- [ ] Orchestrator runs structural checks on each output
- [ ] Orchestrator runs content checks on each output
- [ ] Orchestrator presents results for approval

**Step 3: Run cross-workflow checks**

- [ ] No duplicate trigger labels across the 4 workflows
- [ ] Guardrail wording is identical
- [ ] Label vocabulary is consistent

**Step 4: Fix orchestrator if any issues, commit outputs**

```bash
git add workflows/bmad-create-story.md workflows/bmad-correct-course.md
git add .claude/skills/bmad-build-workflows.md  # if updated
git commit -m "feat: generate implementation phase gh-aw workflows via orchestrator"
```

---

## Phase 4: Full Build

### Task 10: Run builder against all remaining phases

**Step 1: Invoke orchestrator for all workflows**

```
> /bmad-build-workflows
```

Select all workflows. The orchestrator should process:

- **1-analysis:** create-product-brief, research (domain, market, technical — may produce multiple workflows or one composite)
- **2-plan-workflows:** create-prd (create, validate, edit variants), create-ux-design
- **3-solutioning:** check-implementation-readiness, create-architecture, create-epics-and-stories
- **4-implementation:** (already done in Task 9, skip or regenerate)
- **bmad-quick-flow:** quick-dev, quick-spec
- **document-project:** document-project (with sub-workflows)
- **generate-project-context:** generate-project-context
- **qa-generate-e2e-tests:** qa-automate

**Step 2: Review each generated workflow**

For each workflow, check:
- [ ] Correct trigger pattern for its phase
- [ ] Correct agent persona(s)
- [ ] Instructions fully translated
- [ ] Checklist complete
- [ ] Guardrails present

**Step 3: Run full cross-workflow validation**

- [ ] No duplicate trigger labels across ALL workflows
- [ ] All labels follow `bmad-{phase}-{workflow}` convention
- [ ] Guardrail text identical everywhere
- [ ] Label vocabulary is closed (no references to undefined labels)

**Step 4: Commit all workflows**

```bash
git add workflows/
git commit -m "feat: generate full BMAD lifecycle gh-aw workflows"
```

---

### Task 11: Add special workflows not from BMAD source

These workflows don't come from BMAD source files — they're part of our gh-aw coordination architecture.

**Files:**
- Create: `workflows/bmad-resume.md`
- Create: `workflows/bmad-party-mode.md` (if not already generated from a BMAD workflow)

**Step 1: Write bmad-resume.md**

This workflow handles resumption after human intervention:

```yaml
---
on:
  issues:
    types: [unlabeled]
  issue_comment:
    types: [created]
engine: claude
# ... (see design doc for details)
---
```

Trigger: `needs-human-intervention` label removed, or human comment on blocked issue/PR. Reads the thread, understands what was blocking, and dispatches the appropriate next workflow.

**Step 2: Write bmad-party-mode.md (if needed)**

Single-agent multi-perspective analysis workflow. May already be generated from a BMAD source workflow. If not, write manually following the design doc's multi-perspective pattern.

**Step 3: Commit**

```bash
git add workflows/bmad-resume.md workflows/bmad-party-mode.md
git commit -m "feat: add resume and party-mode coordination workflows"
```

---

### Task 12: Compile all workflows with gh-aw

**Step 1: Install gh-aw CLI if not present**

```bash
gh extension install github/gh-aw
```

**Step 2: Compile**

```bash
cd /Users/joshcampbell/code/mm-bmad
gh aw compile
```

This should produce a `.lock.yml` file alongside each `.md` file in `workflows/`.

**Step 3: Verify compilation**

```bash
ls workflows/*.lock.yml
# Expected: one .lock.yml per .md file
```

**Step 4: Commit compiled files**

```bash
git add workflows/*.lock.yml
git commit -m "feat: compile gh-aw workflows to lock files"
```

Note: If `gh aw compile` is not yet available or the workflows/ directory is not the standard `.github/workflows/` location, we may need to adjust the directory structure or compilation command. Verify gh-aw documentation for compiling from a non-standard path.

---

## Phase 5: Documentation & Cleanup

### Task 13: Rewrite README.md

**Files:**
- Modify: `README.md`

**Step 1: Rewrite README**

The README should cover:
1. What mm-bmad is (one paragraph — build pipeline, not runtime wrapper)
2. How it works (the build pipeline: BMAD source → builder skill → gh-aw workflows)
3. For target repos: installation via `gh aw add-wizard` (3-step quickstart)
4. For maintainers: how to update (re-run BMAD install + builder)
5. Link to design doc
6. List of available workflows with one-line descriptions

Do NOT include: generic development practices, detailed architecture (that's in the design doc), or v1 references.

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: rewrite README for v2 build pipeline architecture"
```

---

### Task 14: Final review and tag

**Step 1: Review full repo structure**

```bash
find /Users/joshcampbell/code/mm-bmad -not -path '*/.git/*' -type f | sort
```

Verify it matches the design doc's repo structure.

**Step 2: Run one final cross-workflow check**

Read every `workflows/*.md` file and verify:
- [ ] All have `source:` pointing to `jcampb/mm-bmad/workflows/...@main`
- [ ] All have `engine: claude`
- [ ] All have the `needs-human-intervention` skip guard
- [ ] No workflow references another by name

**Step 3: Tag the release**

```bash
git tag v2.0.0
```

Do NOT push unless user explicitly requests it.

---

## Summary: Task Dependency Graph

```
Task 1: Install BMAD
    ↓
Task 2: Delete v1, create v2 structure
    ↓
Task 3: Write converter prompt
    ↓
Task 4: Test converter → dev-story ──────┐
Task 5: Test converter → code-review ────┤
Task 6: Test converter → create-prd ─────┤ (iterate on converter)
Task 7: Test converter → readiness check ┘
    ↓
Task 8: Write orchestrator skill
    ↓
Task 9: Test orchestrator → implementation phase
    ↓
Task 10: Full build → all phases
    ↓
Task 11: Add coordination workflows (resume, party-mode)
    ↓
Task 12: gh aw compile
    ↓
Task 13: Rewrite README
    ↓
Task 14: Final review and tag
```
