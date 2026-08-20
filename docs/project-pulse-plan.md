# Project Pulse — Implementation Plan

## Summary

Project Pulse is a small static dashboard for Mona's contributors. It surfaces active projects, owners, status, recent activity, and priority as visual cards. The app is pure static assets (HTML + CSS + JSON) under `app/`, with a `.vscode/launch.json` entry named **Run Project Pulse Dashboard** that opens `app/index.html` with `cwd` set to `${workspaceFolder}/app`.

Work is split between two custom agents coordinated by the Orchestrator:

- **Designer** (Gemini 3.1 Pro) — UI/UX, information hierarchy, accessibility, visual styling. Owns `app/styles.css`.
- **Coder** (GPT-5.5) — Static markup, data shape, runnable-app support. Owns `app/index.html`, `app/project-data.json`, `.vscode/launch.json`.

The two agents share a **CSS hook contract** produced in step 1 so their parallel work in step 2 does not conflict.

## File assignments

| File | Owner | Purpose |
|---|---|---|
| `app/index.html` | Coder | Static markup + inline `<script>` that loads `project-data.json` and renders `.project-card` elements into `.dashboard`. |
| `app/project-data.json` | Coder | Top-level `projects` array. Each entry: `name`, `owner`, `status`, `recentActivity`, `priority`. |
| `app/styles.css` | Designer | Dashboard layout, card styling, status badges, priority treatment, spacing, typography, responsive breakpoints. |
| `.vscode/launch.json` | Coder | Strict-JSON launch config named **Run Project Pulse Dashboard**, `cwd` = `${workspaceFolder}/app`, opens `index.html`. |
| `docs/project-pulse-plan.md` | Planner (this file) | Implementation plan. |

No file is co-owned. Scopes are disjoint so Designer and Coder can work in parallel where noted.

## Designer responsibilities

1. Define the visual system: color palette, typography scale, spacing scale, corner radius, shadow tokens.
2. Design the dashboard layout: header/title area, responsive card grid (target ~1 col mobile, 2 col tablet, 3+ col desktop).
3. Design `.project-card` internals: name, owner, status badge, priority indicator, recent activity line, readable hierarchy.
4. Define status badge variants (e.g. `on-track`, `at-risk`, `blocked`, `complete`) and priority treatment (e.g. `high`, `medium`, `low`) with accessible contrast.
5. Publish the **CSS hook contract** (see step 1 below) so the Coder's markup matches.
6. Implement `app/styles.css` against that contract.
7. Verify accessibility basics: contrast ratios, focus states, no color-only signaling.

Designer must **not** edit `app/index.html`, `app/project-data.json`, or `.vscode/launch.json`.

## Coder responsibilities

1. Define the `app/project-data.json` schema and produce realistic seed entries (at least 4–6 projects covering multiple status and priority values).
2. Implement `app/index.html`:
   - Semantic structure with `<main class="dashboard">` root and a container the script populates.
   - Small vanilla-JS loader that `fetch`es `./project-data.json` and renders one `.project-card` per project.
   - Apply status and priority as modifier classes (e.g. `project-card--status-at-risk`, `project-card--priority-high`) per the CSS hook contract.
   - Link `styles.css` via `<link rel="stylesheet" href="./styles.css">`.
   - Graceful fallback message if fetch fails.
3. Create `.vscode/launch.json`:
   - Strict JSON, no comments, no trailing commas.
   - Configuration name: `Run Project Pulse Dashboard`.
   - `cwd`: `${workspaceFolder}/app`.
   - Opens `index.html` so learners see the dashboard, not a directory listing.
4. Validate: open via the launch config, confirm cards render, confirm responsive behavior.

Coder must **not** edit `app/styles.css`.

## Ordered implementation steps

### Step 1 — Contracts (parallel)

- **1a. Designer:** Publish CSS hook contract in the plan thread: class names for root (`.dashboard`), card (`.project-card`), and modifiers for each `status` and `priority` value. Include grid container class.
- **1b. Coder:** Publish `project-data.json` schema and enumerate allowed values for `status` and `priority`.

These two run in parallel. Both must complete before step 2 begins because the class-name modifiers depend on the enumerated status/priority values.

### Step 2 — Implementation (parallel)

- **2a. Coder:** Create `app/project-data.json` with seed data. Create `app/index.html` rendering `.project-card` elements with the agreed modifier classes.
- **2b. Designer:** Create `app/styles.css` implementing layout, cards, badges, priority, and responsive rules against the hook contract.

Parallel-safe: disjoint file ownership. Both must complete before step 3.

### Step 3 — Runnable app support (sequential)

- **3a. Coder:** Create `.vscode/launch.json` with the **Run Project Pulse Dashboard** configuration. Requires `app/index.html` to exist so the launch target is real.

### Step 4 — Validation (sequential, Orchestrator-led)

- Run the launch config, verify rendering, responsiveness, and accessibility spot checks.

## Dependencies between steps

- Step 2 depends on Step 1 (shared class-name + data-value contract).
- Step 3 depends on Step 2a (launch target must exist).
- Step 4 depends on Steps 2 and 3.

## Parallel vs sequential

**Can run in parallel:**
- Step 1a and 1b (different artifacts).
- Step 2a and 2b (disjoint file ownership; both bound by step-1 contract).

**Must run sequentially:**
- Step 1 → Step 2 (contract before implementation).
- Step 2 → Step 3 (need `index.html` before wiring launch config).
- Step 3 → Step 4 (need launch config to validate via it).

## Validation expectations

1. **Launch config works:** VS Code shows a **Run Project Pulse Dashboard** entry. Running it opens `app/index.html` in a preview/browser with `cwd` = `${workspaceFolder}/app`. No directory listing appears.
2. **Cards render from JSON:** Opening the dashboard shows one `.project-card` per entry in `app/project-data.json`. Editing `project-data.json` (adding a project) and reloading reflects the change.
3. **Required fields visible:** Each card shows `name`, `owner`, `status` (as a badge), `recentActivity`, and `priority` (as a visual treatment).
4. **CSS hooks present:** DOM contains `.dashboard` and `.project-card`. Status and priority modifier classes are applied per the contract.
5. **Responsive layout:** Grid collapses to a single column at narrow widths and expands at wider widths. No horizontal scroll at ~360px width.
6. **Accessibility spot checks:** Text has sufficient contrast against card background. Status is not conveyed by color alone (badge has a text label). Interactive elements (if any) have visible focus.
7. **Strict JSON:** `.vscode/launch.json` parses cleanly (no comments, no trailing commas).
8. **Graceful failure:** If `project-data.json` is missing or malformed, `index.html` shows a readable error message instead of a blank page.

## Edge cases to handle

- **`file://` fetch restrictions:** Some browsers block `fetch('./project-data.json')` when opening `index.html` directly from disk. The launch config should open the page in a way that avoids this (e.g. via VS Code's Simple Browser / Live Preview, or a lightweight static server). Coder must confirm the chosen launch mechanism serves the file over HTTP or uses a preview that permits local fetch.
- **Unknown `status` or `priority` values in JSON:** Renderer should fall back to a neutral default class rather than crash or leave the card unstyled.
- **Empty `projects` array:** Show an empty-state message, not a blank dashboard.
- **Long `name` / `owner` / `recentActivity` strings:** CSS must handle wrapping without breaking the card grid.
- **Missing optional visual assets:** No external image or font dependencies required; keep everything local so the app runs offline.
- **Case sensitivity on paths:** Repository is on Windows but may run in Linux dev containers — keep filenames lowercase and reference them exactly (`./styles.css`, `./project-data.json`).
- **Duplicate CSS hook drift:** If Designer renames a class, Coder's markup breaks silently. The step-1 contract prevents this; any later rename must be coordinated by the Orchestrator.

## Open questions

1. **Preview mechanism:** Should the launch config use VS Code's built-in Simple Browser / Live Preview extension, or spin up a static server (e.g. `python -m http.server`)? This affects whether `fetch` works and what `type`/`program`/`url` fields belong in `launch.json`. Recommend the Coder pick the simplest option that satisfies the fetch requirement and that works in the provided devcontainer.
2. **Status and priority vocabularies:** Brief does not enumerate allowed values. Recommend Coder proposes (`on-track`, `at-risk`, `blocked`, `complete`) and (`high`, `medium`, `low`) in step 1b; Designer confirms before styling.
3. **Recent activity format:** Free-text string, timestamp, or structured `{ text, timestamp }`? Recommend free-text for v1 to keep scope small.
4. **Number of seed projects:** Not specified. Recommend 4–6 to demonstrate multiple status/priority variants.
5. **Dark mode / theme:** Not required by the brief. Recommend deferring unless the Designer flags it as low-cost.