# Agent team

This is the custom agent team I will orchestrate with GitHub Copilot CLI (running in a Codespace) to build Mona's **Project Pulse** dashboard. The agent definitions live in `.github/agents/`.

## Team overview

| Agent | Model | Responsibility | Definition file |
| --- | --- | --- | --- |
| Orchestrator | Claude Opus 4.7 | Breaks the request into phases and delegates to the specialist agents. Coordinates work but does not implement it. | `.github/agents/orchestrator.agent.md` |
| Planner | Claude Opus 4.7 | Researches the repo, docs, and edge cases to produce an implementation plan with ordered steps, file ownership, and dependencies. Plans only; writes no code. | `.github/agents/planner.agent.md` |
| Coder | GPT-5.5 | Implements the static Project Pulse app files with clear, deterministic, testable code, plus support files like `.vscode/launch.json`. | `.github/agents/coder.agent.md` |
| Designer | Gemini 3.1 Pro | Owns UI/UX, accessibility, information hierarchy, and visual polish for the dashboard (project cards, status badges, responsive layout, CSS hooks like `.dashboard` and `.project-card`). | `.github/agents/designer.agent.md` |

## How the team builds Project Pulse

1. The **Orchestrator** receives the request to build the Project Pulse dashboard and asks the **Planner** for a plan.
2. The **Planner** researches the repository and returns ordered steps, per-file ownership, dependencies, and work that can run in parallel vs. sequentially.
3. The **Orchestrator** parses the plan into phases and assigns explicit file scopes so the **Coder** and **Designer** never edit overlapping files at the same time.
4. The **Designer** shapes the dashboard experience (layout, cards, badges, styling), while the **Coder** implements the static `app/` files and any runnable support such as `.vscode/launch.json`.
5. The **Orchestrator** verifies the integrated result hangs together and reports the outcome.

Throughout, the learner controls all git operations (stage, commit, push) through GitHub Copilot CLI prompts — the agents do not run git themselves.
