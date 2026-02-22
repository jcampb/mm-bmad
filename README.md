# mm-bmad (BMAD Central Universal Wrapper)

This private repository is the central brain for running the **BMAD Method** autonomously via **GitHub Agentic Workflows**.

Instead of hardcoding out-of-date agent instructions, this repository uses a **Universal Wrapper Architecture**. The central agent dynamically reads your project's local `_bmad` installation to execute the official, up-to-date workflows. When BMAD updates upstream, your agents automatically inherit the new rules without requiring changes here.

## The Architecture
Please read the comprehensive design document explaining how this wrapper solves infinite loops, requests human intervention, and dynamically links to the BMAD core:
[BMAD Wrapper Design & Deep Research](architecture/BMAD_WRAPPER_DESIGN.md)

## How to Deploy to Other Projects

Because this is a **private repository**, your other projects will need a Personal Access Token (PAT) to read these reusable workflows.

### 1. Create a PAT
1. Go to your GitHub Settings -> Developer Settings -> Personal access tokens.
2. Create a token with `repo` scope (or fine-grained with read access to `jcampb/mm-bmad`).
3. In your target project repository, add this token as a Repository Secret named `BMAD_CENTRAL_TOKEN`.

### 2. Configure Caller Workflows
In the repository where you want the agents to operate, you map GitHub events (like labels or PR comments) to specific BMAD workflows by calling the universal runner.

See the [Caller Workflow Examples](examples/caller-workflows.md) for ready-to-use templates for Analysis, Planning, Implementation, and QA.
