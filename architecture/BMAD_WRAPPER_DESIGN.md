# BMAD Universal Wrapper Design & Deep Research

## The Problem with Static Agents
Traditional agent setups hardcode the agent's prompt in a central location. If the upstream BMAD methodology releases an update (e.g., changing the `create-story` checklist or adding a new `correct-course` step), the central agents fall out of sync.

## The Solution: Dynamic Execution
Instead of hardcoding the instructions, the GitHub Agentic Workflow running in the project is given a "Universal Prompt". This prompt acts as a strictly obedient executor. Its core directive is:
1. Receive a `bmad_workflow_path` (e.g., `4-implementation/dev-story`).
2. Read `_bmad/bmm/config.yaml` from the local project to get the user's preferred language and skill level.
3. Read the exact instructions from `_bmad/bmm/workflows/{bmad_workflow_path}/instructions.xml`.
4. Read the checklist from `_bmad/bmm/workflows/{bmad_workflow_path}/checklist.md`.
5. Execute the instructions blindly.

This means the agent's "brain" is always perfectly synchronized with the BMAD installation version running in that specific project.

---

## Autonomous Operations & Guardrails

When running agents autonomously on GitHub, two critical problems emerge: **Infinite Loops** and **Hallucination Blocking**. Here is how the Universal Wrapper solves both.

### 1. Closing Infinite Loops (The 3-Strike Rule)
**The Scenario:** 
1. The `code-review` agent leaves a comment: *"This function is inefficient."*
2. The `dev-story` agent attempts to fix it but fails the test.
3. `code-review` complains again.
4. `dev-story` tries the exact same broken fix.
This creates an endless loop consuming massive compute resources.

**The Solution:**
The Universal Wrapper injects a strict **3-Strike Protocol** into every agent:
*   Before writing code or responding to a PR review, the agent must read the comment history.
*   It counts how many attempts have been made to resolve the *exact same* issue.
*   If the count reaches 3, the agent triggers an emergency halt. It leaves a comment stating `🛑 [INFINITE LOOP DETECTED]` and applies a `needs-human-intervention` label. It refuses to write further code until the label is removed or a human explicitly clarifies the direction.

### 2. Requesting Human Input (The Blocker Protocol)
**The Scenario:**
An agent is asked to run the `correct-course` workflow, but the user's issue description simply says: *"The button doesn't work."* An unconstrained LLM might hallucinate a fix, guess the button, or rewrite the entire UI.

**The Solution:**
The Universal Wrapper includes a **Blocker Protocol**:
*   If critical information is missing (ambiguous issue, missing architecture documents, undefined UX).
*   The agent is strictly forbidden from guessing.
*   It halts execution and posts a comment: `⚠️ @username [BMAD BLOCKER] - I cannot proceed. I need clarification on [X].`
*   It applies the `needs-human-intervention` label.
*   All BMAD caller workflows are configured to ignore events on issues/PRs that carry the `needs-human-intervention` label. The loop only resumes when a human replies and removes the label.

---

## The BMAD Methodology Map

The Universal Wrapper can execute ANY of the official BMAD workflows. Here is the mapped directory structure it pulls from locally:

*   **1-analysis**
    *   `create-product-brief`
    *   `research`
*   **2-plan-workflows**
    *   `create-prd`
    *   `create-ux-design`
*   **3-solutioning**
    *   `check-implementation-readiness`
    *   `create-architecture`
    *   `create-epics-and-stories`
*   **4-implementation**
    *   `create-story` (Extracts epic requirements, formats BDD, creates Draft PR)
    *   `dev-story` (Implements the code based on the story file)
    *   `code-review` (Adversarial review requesting changes if needed)
    *   `correct-course` (Handles bugs or scope changes)
    *   `retrospective`
    *   `sprint-planning`
    *   `sprint-status`
*   **qa**
    *   `automate` (Generates automated test suites)
