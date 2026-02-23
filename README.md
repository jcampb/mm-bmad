# mm-bmad

A build pipeline that compiles the [BMAD Method](https://github.com/bmadcode/BMAD-METHOD) into native [GitHub Agentic Workflows](https://github.com/github/gh-aw).

BMAD agent definitions, workflow instructions, and checklists in `_bmad/` are compiled by a Claude Code builder skill into self-contained gh-aw `.md` files. Target repos install workflows via `gh aw add` -- no BMAD knowledge required.

## Quick Start (Target Repos)

```bash
# 1. Add the workflows you need
gh aw add jcampb/mm-bmad/workflows/bmad-create-story
gh aw add jcampb/mm-bmad/workflows/bmad-dev-story
gh aw add jcampb/mm-bmad/workflows/bmad-code-review

# 2. Compile
gh aw compile

# 3. (Optional) Add project config
mkdir -p .bmad
cat > .bmad/config.yaml << 'EOF'
language: TypeScript
framework: Next.js
test_runner: vitest
EOF

# 4. Commit and push
git add .github/workflows/ .bmad/
git commit -m "Add BMAD workflows"
git push
```

Then trigger workflows by adding labels to issues or creating PRs.

## Available Workflows

### Analysis Phase
| Workflow | Trigger Label | Description |
|---|---|---|
| `bmad-create-product-brief` | `bmad-analysis-product-brief` | Create product briefs through step-by-step discovery |
| `bmad-market-research` | `bmad-analysis-market-research` | Market research with citations |
| `bmad-domain-research` | `bmad-analysis-domain-research` | Domain and industry research |
| `bmad-technical-research` | `bmad-analysis-technical-research` | Technical research and analysis |

### Planning Phase
| Workflow | Trigger Label | Description |
|---|---|---|
| `bmad-create-prd` | `bmad-plan-prd` | Product Requirements Document creation |
| `bmad-create-ux-design` | `bmad-plan-ux-design` | UX design specifications |

### Solutioning Phase
| Workflow | Trigger Label | Description |
|---|---|---|
| `bmad-create-architecture` | `bmad-solution-architecture` | System architecture documentation |
| `bmad-create-epics-and-stories` | `bmad-solution-epics` | Epics and user stories from PRD |
| `bmad-check-readiness` | `bmad-solution-readiness` | Validate artifacts before implementation |

### Implementation Phase
| Workflow | Trigger | Description |
|---|---|---|
| `bmad-create-story` | Label: `bmad-story` | Create implementation-ready story files |
| `bmad-dev-story` | PR ready for review / review submitted | Execute story implementation with TDD |
| `bmad-code-review` | PR synchronize | Adversarial code review against story claims |
| `bmad-correct-course` | Label: `bmad-correct-course` | Sprint change proposal and impact analysis |
| `bmad-sprint-planning` | Label: `bmad-sprint-planning` | Sprint planning and story preparation |
| `bmad-sprint-status` | Label: `bmad-sprint-status` | Sprint status report |
| `bmad-retrospective` | Label: `bmad-retrospective` | Sprint retrospective with multi-perspective analysis |

### Coordination
| Workflow | Trigger | Description |
|---|---|---|
| `bmad-party-mode` | Label: `bmad-party-mode` | Multi-perspective analysis from all agent viewpoints |
| `bmad-resume` | `needs-human-intervention` label removed | Resume after human resolves a blocker |

## How It Works

Each workflow is a self-contained "fat" `.md` file containing everything the AI agent needs: persona, principles, instructions, checklists, and guardrails. The `gh aw compile` command converts these into GitHub Actions `.lock.yml` files.

**Safety mechanisms:**
- **Labels as state machine** -- workflow state is visible in the GitHub UI
- **Safe-outputs** -- all write operations go through pre-approved channels
- **Circuit breaker** -- `needs-human-intervention` label halts all workflows
- **3-strike rule** -- dev/review loop auto-halts after 3 cycles (deterministic pre-step)

## Updating

```bash
gh aw upgrade    # pulls latest from source: field in each workflow
gh aw compile    # recompile
```

## For Maintainers

To regenerate workflows after BMAD updates:

```bash
npx bmad-method install --yes    # update _bmad/ source
# Run the builder skill in Claude Code:
# /bmad-build-workflows
```

See `docs/plans/2026-02-22-mm-bmad-v2-design.md` for architecture details.
