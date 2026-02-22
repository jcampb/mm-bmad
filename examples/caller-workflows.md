# BMAD Caller Workflow Examples

To automate the BMAD methodology in your project, you don't need to define the agents locally. You just need to map GitHub events (like labels being applied to issues) to the central Universal Runner.

Here are examples of how to set up the major lifecycle phases in your project's `.github/workflows/` directory.

### 1. Planning Phase (Triggered by Issue Labels)
This workflow listens for a specific label (`bmad-plan-prd`) and triggers the `create-prd` workflow.

**`.github/workflows/bmad-trigger-prd.yml`**
```yaml
name: BMAD Create PRD
on:
  issues:
    types: [labeled]

jobs:
  create-prd:
    # Only run if the label 'bmad-plan-prd' was added, and ignore if a human is needed
    if: github.event.label.name == 'bmad-plan-prd' && !contains(github.event.issue.labels.*.name, 'needs-human-intervention')
    uses: jcampb/mm-bmad/.github/workflows/bmad-universal-runner.yml@main
    with:
      bmad_workflow_path: '2-plan-workflows/create-prd'
    secrets:
      BMAD_CENTRAL_TOKEN: ${{ secrets.BMAD_CENTRAL_TOKEN }}
```

### 2. Implementation Phase (Create Story)
This triggers when a user opens an issue and wants it converted into a formal BMAD Story document.

**`.github/workflows/bmad-trigger-create-story.yml`**
```yaml
name: BMAD Create Story
on:
  issues:
    types: [opened]

jobs:
  create-story:
    if: contains(github.event.issue.labels.*.name, 'bmad-story') && !contains(github.event.issue.labels.*.name, 'needs-human-intervention')
    uses: jcampb/mm-bmad/.github/workflows/bmad-universal-runner.yml@main
    with:
      bmad_workflow_path: '4-implementation/create-story'
    secrets:
      BMAD_CENTRAL_TOKEN: ${{ secrets.BMAD_CENTRAL_TOKEN }}
```

### 3. The Dev & Review Loop
This handles the autonomous coding and adversarial review loop.

**`.github/workflows/bmad-trigger-dev-loop.yml`**
```yaml
name: BMAD Dev & Review Loop
on:
  pull_request:
    types: [opened, reopened, synchronize]
  pull_request_review:
    types: [submitted]

jobs:
  dev-story:
    # Trigger dev-story when a PR is opened, or when changes are requested
    if: (!contains(github.event.pull_request.labels.*.name, 'needs-human-intervention')) && (github.event.action == 'opened' || (github.event.action == 'submitted' && github.event.review.state == 'changes_requested'))
    uses: jcampb/mm-bmad/.github/workflows/bmad-universal-runner.yml@main
    with:
      bmad_workflow_path: '4-implementation/dev-story'
    secrets:
      BMAD_CENTRAL_TOKEN: ${{ secrets.BMAD_CENTRAL_TOKEN }}

  code-review:
    # Trigger code-review when new commits are pushed (synchronize)
    if: (!contains(github.event.pull_request.labels.*.name, 'needs-human-intervention')) && (github.event.action == 'synchronize')
    uses: jcampb/mm-bmad/.github/workflows/bmad-universal-runner.yml@main
    with:
      bmad_workflow_path: '4-implementation/code-review'
    secrets:
      BMAD_CENTRAL_TOKEN: ${{ secrets.BMAD_CENTRAL_TOKEN }}
```

### 4. Correct Course (Bug Fixes & Scope Changes)
This triggers the BMAD course correction workflow when a user adds the `bmad-correct-course` label to a bug report.

**`.github/workflows/bmad-trigger-correct-course.yml`**
```yaml
name: BMAD Correct Course
on:
  issues:
    types: [labeled]

jobs:
  correct-course:
    if: github.event.label.name == 'bmad-correct-course' && !contains(github.event.issue.labels.*.name, 'needs-human-intervention')
    uses: jcampb/mm-bmad/.github/workflows/bmad-universal-runner.yml@main
    with:
      bmad_workflow_path: '4-implementation/correct-course'
    secrets:
      BMAD_CENTRAL_TOKEN: ${{ secrets.BMAD_CENTRAL_TOKEN }}
```
