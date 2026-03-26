# GSD Backend Mechanics and Project Enforcement

This report provides a deep-dive analysis into the backend architecture of **Get Shit Done (GSD)** and how its methodologies are enforced within a project context. GSD is not merely a collection of prompts; it is a **context engineering and state management system** designed to make AI-driven development reliable and scalable.

## 1. Backend Architecture Overview

The GSD backend is primarily built with **Node.js**, leveraging a modular architecture that centralizes core logic to ensure consistency across various AI runtimes (Claude Code, Gemini CLI, Copilot, etc.).

### 1.1 Core Components

| Component | Responsibility | Key File(s) |
| :--- | :--- | :--- |
| **CLI Router** | Entry point for all GSD operations, handling argument parsing and command routing. | `bin/gsd-tools.cjs` |
| **State Manager** | Manages `STATE.md`, `ROADMAP.md`, and `REQUIREMENTS.md`. Handles frontmatter CRUD and progress tracking. | `bin/lib/state.cjs`, `bin/lib/roadmap.cjs` |
| **Workflow Engine** | Orchestrates complex multi-step processes like project initialization, phase planning, and execution. | `bin/lib/init.cjs`, `workflows/*.md` |
| **Context Guard** | A suite of runtime hooks that monitor context usage and enforce workflow adherence. | `hooks/*.js` |
| **Model Resolver** | Maps abstract agent roles (e.g., `gsd-planner`) to specific model IDs based on user profiles. | `bin/lib/model-profiles.cjs` |

### 1.2 The `.planning/` Directory: The Source of Truth

The entire system revolves around the `.planning/` directory. This is the "brain" of the project where all state, configuration, and artifacts reside.

*   **`config.json`**: Project-specific settings (branching strategy, model profiles, feature flags).
*   **`STATE.md`**: The high-level heartbeat of the project, containing current focus, active workstreams, and progress metrics.
*   **`ROADMAP.md`**: The master plan, divided into phases and individual tasks (plans).
*   **`REQUIREMENTS.md`**: The definitive list of what is being built, used for verification.

---

## 2. Enforcement Mechanisms

GSD enforces its system through a combination of **Tool-based Guardrails**, **Runtime Hooks**, and **Git Integration**.

### 2.1 Workflow Guard (Soft Enforcement)

The `gsd-workflow-guard.js` is a `PreToolUse` hook that monitors Claude's actions. If the agent attempts to edit project files directly without using a GSD command (like `/gsd:fast` or `/gsd:quick`), the guard injects an advisory warning:

> ⚠️ **WORKFLOW ADVISORY**: You're editing file.ts directly without a GSD command. This edit will not be tracked in STATE.md...

This mechanism ensures that the agent remains "aware" of the system even when it tries to bypass it, preventing "context rot" and untracked changes.

### 2.2 Context Monitoring (Runtime Enforcement)

The `gsd-context-monitor.js` hook reads real-time metrics from the runtime. When context usage reaches critical levels (e.g., < 25% remaining), it injects instructions to the agent:

*   **Warning (35%)**: "Avoid starting new complex work."
*   **Critical (25%)**: "Do NOT start new complex work... Inform the user so they can run /gsd:pause-work."

This prevents the AI from hallucinating or losing track of the project goals as the context window fills up.

### 2.3 Git-Driven State Enforcement

GSD tightly integrates with Git to ensure that documentation and code stay in sync.

*   **Atomic Commits**: The `gsd-tools commit` command automates the staging and committing of planning artifacts.
*   **Branching Strategy**: Depending on the `config.json` (e.g., `branching_strategy: "phase"`), GSD automatically creates and switches to feature branches (e.g., `gsd/phase-1-setup`) when a phase begins.
*   **Contention Management**: In parallel execution modes, GSD uses `--no-verify` on intermediate commits to avoid pre-commit hook conflicts between multiple agents, performing a final verification only after all agents finish.

---

## 3. Project Context Enforcement Logic

When a GSD command is invoked, the system follows a strict initialization and enforcement flow:

### 3.1 Root Discovery
The `findProjectRoot` function in `core.cjs` ensures that GSD always operates relative to the `.planning/` directory, even if the agent is currently inside a sub-repo (e.g., `backend/`). This prevents fragmented state across monorepos.

### 3.2 Wave-Based Execution
In the `execute-phase` workflow, work is divided into **Waves**. Enforcement logic ensures:
1.  **Dependency Safety**: Wave $N+1$ cannot start until all plans in Wave $N$ are complete.
2.  **Verification Gates**: After each wave or phase, the `gsd-verifier` agent is spawned to audit the work against `REQUIREMENTS.md`.

### 3.3 The "Chain" Flag
GSD uses an ephemeral `_auto_chain_active` flag in the configuration to track automated workflows. If a user interrupts an automated chain and runs a manual command, the system detects the lack of the `--auto` flag and resets the chain state to prevent unwanted auto-advancement.

---

## 4. Summary of Backend Enforcement

| Feature | Enforcement Method | Outcome |
| :--- | :--- | :--- |
| **State Tracking** | `STATE.md` + `gsd-tools state` | Guaranteed "heartbeat" of project progress. |
| **Workflow Adherence** | `gsd-workflow-guard.js` | Soft-nudges agents back into tracked workflows. |
| **Context Reliability** | `gsd-context-monitor.js` | Prevents degradation by forcing state saves at limits. |
| **Multi-Agent Safety** | Wave-based logic + Git branching | Enables parallel work without state corruption. |
| **Verification** | `gsd-verifier` + `REQUIREMENTS.md` | Ensures the "What" matches the "How". |

GSD's power lies in its **invisibility to the user** but **omnipresence to the agent**. By embedding enforcement logic directly into the tool-use loop and the filesystem structure, it transforms a standard LLM into a disciplined engineering agent.
