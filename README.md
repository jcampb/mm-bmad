# mm-bmad (Central BMAD Brain)

This private repository acts as the central intelligence hub for your GitHub Agentic Workflows. It contains the reusable GitHub Actions YAML wrappers and the natural language Markdown agent prompts.

## How to Deploy to Other Projects

Because this is a **private repository**, your other projects will need a Personal Access Token (PAT) to read these workflows.

### 1. Create a PAT
1. Go to your GitHub Settings -> Developer Settings -> Personal access tokens.
2. Create a classic token with `repo` scope, or a fine-grained token with read access to `jcampb/mm-bmad`.
3. In your target project repository, add this token as a Repository Secret named `BMAD_CENTRAL_TOKEN`.

### 2. Add the Caller Workflows to your Target Project
In the repository where you want the agents to operate, create these files:

**`.github/workflows/bmad-create-story.yml`**
```yaml
name: BMAD Create Story
on:
  issues:
    types: [opened]

jobs:
  call-central:
    if: contains(github.event.issue.labels.*.name, 'bmad-story')
    uses: jcampb/mm-bmad/.github/workflows/bmad-create-story.yml@main
    secrets:
      BMAD_CENTRAL_TOKEN: ${{ secrets.BMAD_CENTRAL_TOKEN }}
```

**`.github/workflows/bmad-dev-story.yml`**
```yaml
name: BMAD Dev Story
on:
  pull_request:
    types: [opened, reopened]
  pull_request_review:
    types: [submitted]

jobs:
  call-central:
    if: github.event.action == 'opened' || (github.event.action == 'submitted' && github.event.review.state == 'changes_requested')
    uses: jcampb/mm-bmad/.github/workflows/bmad-dev-story.yml@main
    secrets:
      BMAD_CENTRAL_TOKEN: ${{ secrets.BMAD_CENTRAL_TOKEN }}
```

**`.github/workflows/bmad-code-review.yml`**
```yaml
name: BMAD Code Review
on:
  push:
    branches-ignore: [main]

jobs:
  call-central:
    uses: jcampb/mm-bmad/.github/workflows/bmad-code-review.yml@main
    secrets:
      BMAD_CENTRAL_TOKEN: ${{ secrets.BMAD_CENTRAL_TOKEN }}
```

### Configuring Skills per Project
In any project using these agents, you can define custom skills (e.g., `frontend-design`, `security-auditor`) by creating a file at `.bmad/skills.md` in that specific repository. The central agents are programmed to dynamically read that file and adopt the requested skills during execution!
