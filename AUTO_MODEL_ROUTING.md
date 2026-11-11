# OpenCode Automatic Multi-Model Routing & Adaptive Pipeline

> **Auto-Maintained Documentation**: This document explains the architecture, files, policies, and runtime behavior of the global OpenCode Automatic Model Router. Any updates made to `~/.config/opencode/model-routing.json` or `~/.config/opencode/profiles/*.json` automatically sync the **Live Configuration Snapshot** section at the bottom of this file via the `auto-model-router.ts` plugin and slash commands.

---

## 1. Overview & Purpose

By default, OpenCode uses a single static model for an entire session unless manually switched. This system upgrades OpenCode globally across **all projects** so that:

1. **Task Complexity & Risk Are Checked Automatically**: Every prompt is analyzed before execution for complexity (`simple`, `moderate`, `complex`, `critical`), category (`query`, `research`, `coding`, `automation`), and risk level (`low`, `medium`, `high`, `critical`).
2. **Simple Tasks Execute Immediately**: Quick questions, single commands, and minor edits are routed straight to a fast, low-cost `router` model (`glm-5.3-flash`) with **zero subagent overhead**.
3. **Complex Tasks Trigger a Policy-Governed Multi-Model Pipeline**: Multi-step engineering tasks (such as *"search for a new concept in blockchain and build a demo for understanding"*) are coordinated by a `primary` orchestrator model that delegates each phase (`researcher`, `architect`, `builder`, `tester`, `reviewer`, `escalation_reviewer`) to its own specialized model.
4. **Two Toggleable Profiles (`config-1` and `config-2`)**: You can maintain two complete model + policy configurations and flip between them live using `/toggle-config` without restarting OpenCode.

---

## 2. Complete File Inventory (What Changed, Why, How, and Use)

All configuration files live in `/home/sparkout/.config/opencode/` so they work globally in every repository.

### A. Configuration & Profile Files

#### 1. `/home/sparkout/.config/opencode/opencode.json`
- **What Changed**: Added top-level `"model"`, `"small_model"`, and `"agent"` model mappings (`build`, `plan`, `explore`, `general`, `researcher`, `architect`, `builder`, `reviewer`, `tester`, `escalation_reviewer`) while preserving all existing `"mcp"` servers (`gitnexus`, `graphify`, `github`, `context7`, `foundry`, `etherscan`).
- **Why It Was Done**: OpenCode validates its schema on startup and needs baseline model assignments for built-in and custom agents.
- **How It Works**: Serves as the startup baseline; the `auto-model-router.ts` plugin dynamically overrides models per turn based on the active profile.
- **Use**: Global OpenCode configuration file.

#### 2. `/home/sparkout/.config/opencode/model-routing.json`
- **What Changed**: Created master routing control file containing `"activeConfig"`, `"configs"` (file paths to `config-1.json` and `config-2.json`), and `"complexityRules"` (keywords and word-count thresholds).
- **Why It Was Done**: Separates the active profile pointer and complexity heuristics from individual profile definitions so you can switch profiles cleanly.
- **How It Works**: Read on every prompt by `auto-model-router.ts` to determine whether `config-1` or `config-2` is active.
- **Use**: Tracks which config profile is active (`"config-1"` or `"config-2"`) and stores complexity detection keywords.

#### 3. `/home/sparkout/.config/opencode/profiles/config-1.json`
- **What Changed**: Created Profile 1 with your speed/efficiency-optimized multi-model table and policy gates:
  - `router`: `glm-5.3-flash` | `primary`: `glm-5.3-flash`
  - `researcher`: `deepseek-v4.1-flash` | `architect`: `deepseek-v4.1-flash`
  - `builder`: `kimi-k2.7-code` | `reviewer`: `deepseek-v4.1-flash`
  - `tester`: `glm-5.3-flash` | `escalation_reviewer`: `deepseek-v4-pro`
  - `policy`: `research: "only_if_missing_context"`, `architecture: "only_if_multi_file_or_risky"`, `review: "diff_only"`, `max_retries: 2`.
- **Why It Was Done**: Provides a fast, cost-effective daily pipeline that skips unnecessary stages and runs `tester` + `reviewer` (`diff_only`) in parallel.
- **Use**: Default active profile (`config-1`) for fast day-to-day multi-model engineering.

#### 4. `/home/sparkout/.config/opencode/profiles/config-2.json`
- **What Changed**: Created Profile 2 with your high-rigor frontier model table and stricter policy gates:
  - `router`: `glm-5.3-flash` | `primary`: `kimi-k2.7-code`
  - `researcher`: `gemini-4-argon` | `architect`: `deepseek-v4-pro`
  - `builder`: `kimi-k2.7-code` | `reviewer`: `deepseek-v4-pro`
  - `tester`: `glm-5.3-flash` | `escalation_reviewer`: `gpt-6-luna`
  - `policy`: `research: "only_if_ambiguous_or_cross_repo"`, `architecture: "required_for_complex_or_risky_work"`, `review: "diff_and_test_output_only"`, `max_retries: 3`.
- **Why It Was Done**: Provides a deep-reasoning configuration for heavy architecture, cross-repo work, and high-stakes verification where the reviewer inspects both git diffs and test outputs.
- **Use**: High-rigor profile (`config-2`) for complex architecture and critical production/smart-contract work.

---

### B. Core Routing Engine Plugin

#### 5. `/home/sparkout/.config/opencode/plugins/auto-model-router.ts`
- **What Changed**: Created an OpenCode plugin implementing four hooks (`tool`, `config`, `tool.execute.before`, and `chat.message`) plus automatic Markdown documentation synchronization.
- **Why It Was Done**: OpenCode's static config cannot inspect prompt complexity or switch models mid-session on its own. This plugin brings the adaptive intelligence from `/home/sparkout/Hook/Multi_agent_pipeline` (`TaskAnalyzer`, `WorkflowPlanner`, `AgentCapabilityRegistry`, and `AdaptiveResourceAllocator`) into every OpenCode session.
- **How It Works**:
  1. **`config(cfg)` hook**: On startup, maps the active profile's models into OpenCode's agent definitions and normalizes short model names (e.g., `"glm-5.3-flash"` -> `"opencode-go/glm-5.3-flash"`).
  2. **`chat.message(input, output)` hook**:
     - When a message runs inside a subagent (`researcher`, `architect`, `builder`, `reviewer`, `tester`, `escalation_reviewer`), it dynamically sets `output.message.model` to the model assigned in the active profile.
     - When a user sends a prompt to the primary agent, it runs `analyzeTask()` and `detectProjectStack()`:
       - **If Simple**: Switches `output.message.model` to `profile.router` and instructs the model to answer or execute immediately with **0 subagents**.
       - **If Moderate / Complex / Critical**: Switches `output.message.model` to `profile.primary`, computes the **minimal required subagent roster** based on `profile.policy`, checks `policy.escalate_on` triggers, and injects a structured pipeline execution plan into `output.message.system`.
  3. **`tool.execute.before` hook**: When the orchestrator calls `task`, the plugin automatically appends the detected workspace tech stack (e.g. Foundry `forge test`, Anchor `anchor test`, `pnpm test`, `pytest`) and review policy (`diff_only` vs `diff_and_test_output_only`) to the subagent's prompt so subagents don't waste tokens rediscovering project commands.
  4. **`tool.execute.after` hook & `toggle_model_config` tool**: Whenever configs are toggled or edited, automatically updates the **Live Configuration Snapshot** in this Markdown file (`OPENCODE_AUTO_MODEL_ROUTING.md` and `~/.config/opencode/AUTO_MODEL_ROUTING.md`).

---

### C. Specialized Subagent Definitions

Each file in `/home/sparkout/.config/opencode/agents/` defines a dedicated specialist with strict tool permissions so read-only agents can never accidentally modify code:

| File | Role | Permissions | Purpose & Output |
| :--- | :--- | :--- | :--- |
| `agents/researcher.md` | `researcher` | `edit: deny`, `bash: deny`, `websearch: allow`, `webfetch: allow` | Uses `websearch`, `webfetch`, `context7`, and `read`/`grep`/`glob` to gather external concepts, protocol specs, and repo layout. |
| `agents/architect.md` | `architect` | `edit: deny`, `bash: deny` | Designs component boundaries, file layout, interfaces, smart-contract security invariants, and step-by-step build plans. |
| `agents/builder.md` | `builder` | `edit: allow`, `bash: allow` | Implements source code, demos, contracts, and unit tests following the architect's plan, and applies targeted fixes during repair loops. |
| `agents/reviewer.md` | `reviewer` | `edit: deny`, `bash: git diff/status/log only` | Audits git diffs (and test output in `config-2`) for bugs, missing requirements, and security flaws (`BLOCKER`, `IMPORTANT`, `NIT`). |
| `agents/tester.md` | `tester` | `edit: deny`, `bash: allow` | Runs compilers, linters, test suites, and demo scripts; reports `PASS`/`FAIL` with exact command output. |
| `agents/escalation_reviewer.md` | `escalation_reviewer` | `edit: deny`, `bash: git diff/status/log only` | Invoked automatically when `policy.escalate_on` triggers match (`security`, `payments`, `auth`, `database_migration`, `production`, `repeated_test_failure`). |

---

### D. Slash Commands

#### 12. `/home/sparkout/.config/opencode/commands/toggle-config.md` (`/toggle-config`)
- **What It Does**: Calls `toggle_model_config` to switch between `config-1` and `config-2` (or `/toggle-config 1`, `/toggle-config 2`), displays the newly active model table, and refreshes this Markdown documentation file.

#### 13. `/home/sparkout/.config/opencode/commands/models-pipeline.md` (`/models-pipeline`)
- **What It Does**: Displays the active configuration and both profile tables, or updates any role's model/policy in `config-1.json` / `config-2.json` and syncs the changes back into this Markdown documentation file.

---

## 3. End-to-End Execution Flow

```text
User Prompt
    │
    ▼
[auto-model-router.ts: "chat.message" Hook]
    ├─► Detects Workspace Stack (Foundry / Anchor / Node / Rust / Python / Go)
    ├─► Evaluates Complexity, Category & Risk (analyzeTask)
    │
    ├──► Case 1: SIMPLE Task (e.g., "what is ERC-4337?", "check git status", "fix typo in README")
    │       └─► Model switched to `router` (glm-5.3-flash)
    │       └─► Executed immediately in 1 turn (0 subagents spawned)
    │
    └──► Case 2: MODERATE / COMPLEX / CRITICAL Task
            (e.g., "search for a new concept in blockchain and build a demo for understanding")
            │
            ├─► Primary Orchestrator switched to `primary` model
            ├─► Policy Gates Checked:
            │     • Research gate (`only_if_missing_context` / `only_if_ambiguous_or_cross_repo`)
            │     • Architecture gate (`only_if_multi_file_or_risky` / `required_for_complex_or_risky_work`)
            │     • Escalation gate (`security`, `payments`, `auth`, `database_migration`, `production`)
            │
            ├─► Stage 1: task(subagent_type="researcher")   [if needed by policy]
            ├─► Stage 2: task(subagent_type="architect")    [if needed by policy]
            ├─► Stage 3: task(subagent_type="builder")      [writes code & tests]
            ├─► Stage 4: task(subagent_type="tester") + task(subagent_type="reviewer" | "escalation_reviewer")
            │     • In config-1 (`diff_only`): tester & reviewer run in PARALLEL
            │     • In config-2 (`diff_and_test_output_only`): tester runs first, output passed to reviewer
            └─► Stage 5: Bounded Repair Loop (up to `max_retries`)
                  • If identical error repeats twice (`repeated_test_failure`), halts blind retries
                    and escalates to `escalation_reviewer`
```

---

## 4. How to Use & Customize

- **Toggle between Config 1 and Config 2**:
  ```text
  /toggle-config
  /toggle-config 1
  /toggle-config 2
  ```
- **View or change models/policies via command**:
  ```text
  /models-pipeline
  /models-pipeline config-1 builder kimi-k2.7-code
  ```
- **Force simple or complex mode on any prompt**:
  - Add `[simple]` to force direct single-model execution.
  - Add `[complex]` or `[pipeline]` to force the full multi-model pipeline.

---

<!-- AUTO-SYNC-START -->
## 5. Live Configuration Snapshot (Auto-Updated)

- **Last Synced**: `2026-11-11`
- **Currently Active Profile**: `config-1`
- **Default Provider Prefix**: `opencode-go`

### Active Profile Comparison (`config-1` vs `config-2`)

| Role / Setting | `config-1` *(Active)* | `config-2` |
| :--- | :--- | :--- |
| **`router`** *(Simple tasks)* | `glm-5.3-flash` | `glm-5.3-flash` |
| **`primary`** *(Orchestrator)* | `glm-5.3-flash` | `kimi-k2.7-code` |
| **`researcher`** | `deepseek-v4.1-flash` | `gemini-4-argon` |
| **`architect`** | `deepseek-v4.1-flash` | `deepseek-v4-pro` |
| **`builder`** | `kimi-k2.7-code` | `kimi-k2.7-code` |
| **`reviewer`** | `deepseek-v4.1-flash` | `deepseek-v4-pro` |
| **`tester`** | `glm-5.3-flash` | `glm-5.3-flash` |
| **`escalation_reviewer`** | `deepseek-v4-pro` | `gpt-6-luna` |
| **`policy.research`** | `only_if_missing_context` | `only_if_ambiguous_or_cross_repo` |
| **`policy.architecture`** | `only_if_multi_file_or_risky` | `required_for_complex_or_risky_work` |
| **`policy.review`** | `diff_only` | `diff_and_test_output_only` |
| **`policy.max_retries`** | `2` | `3` |
| **`policy.escalate_on`** | `security, payments, auth, database_migration, production, repeated_test_failure` | `security, payments, auth, database_migration, production, repeated_test_failure` |
<!-- AUTO-SYNC-END -->
