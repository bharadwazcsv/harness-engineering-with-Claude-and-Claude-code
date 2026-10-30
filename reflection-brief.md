# Reflection Brief: Project-Harness Engineering with Claude and Claude Code

**Student Name:** C Bharadwaz  
**Date:** 30-10-2026 

---

## System 1: Claims Intake Agent (stop_reason-Driven Agentic Loop)

### Architectural Decisions & Termination Logic
The agentic loop relies entirely on the API contract `response.stop_reason == "tool_use"` to govern execution flow. When Claude signals `tool_use`, the execution harness invokes the appropriate domain tool and appends the result to the message list. The loop only terminates when `response.stop_reason == "end_turn"`.

### Anti-Patterns Avoided
1. **Regex and Delimiter String Parsing:** Relying on markers like `THINKING:`, `TOOL_CALL:`, or raw JSON regex scanning causes parsing failures when Claude generates markdown or natural language explanations. Using `stop_reason` guarantees reliable tool invocation at the schema level.
2. **Endless Ping-Pong Loops:** By establishing explicit terminal predicates and capping turn depth, the loop avoids circular tool calls when edge case inputs occur.

### Run Evidence & Verification
- **Test Suite Output:** `pytest` returned 30 passed out of 30 in `12.45s` (`system_1_agentic_loop/pytest_output.txt`).
- **Run Artifact Citation:** In `system_1_agentic_loop/run_artifact.txt`, Case `CASE-ID-98421-AUTO` sequentially called `search_policy(policy_number="POL-7710294")`, `estimate_damage_severity()`, and `dispatch_emergency_towing()`. Upon receiving all outputs, Claude returned `stop_reason: end_turn` and produced the final terminal routing decision: `Escalated to Senior Physical Damage Adjuster (Queue: ESC-AUTO-T2)`.

---

## System 2: Context Strategy (Retail Support Copilot)

### Compression & Position-Aware Assembly
Long-conversation copilot systems suffer from the "lost-in-the-middle" phenomenon if context is concatenated naively. This implementation uses a position-aware assembly architecture:
- **Head (Anchors):** Universal policies and active system instructions stay pinned at the top.
- **Middle (Dynamic Compression):** Multi-turn interaction history is compressed into structured state slots (e.g., active RMA numbers, user preferences, agreed resolutions).
- **Tail (Recency Buffer):** The latest 2 dialogue turns remain verbatim to preserve dialogue cadence and context.

### Run Evidence & Verification
- **Test Suite Output:** `pytest` returned 30 passed out of 30 in `18.21s` (`system_2_context_strategy/pytest_output.txt`).
- **Run Artifact Citation:** From `system_2_context_strategy/compression_run_artifact.json`:
  - **Baseline Tokens:** 14,210 tokens across 46 dialogue turns.
  - **Compressed Tokens:** 5,180 tokens (63.55% reduction, surpassing the 50% target).
  - **Evaluation Accuracy:** 15/15 questions answered accurately (100.0%), successfully preserving critical entities such as `SKU-44021 defective return authorization` and customer refund preferences.

---

## System 3: Claude Code Monorepo Harness Configuration

### Hierarchical Rules & Path-Scoped Isolation
In a multi-surface monorepo (frontend, backend, shared databases), feeding a massive monolithic prompt causes prompt dilution and cross-domain pollution. The solution employs:
- Root `CLAUDE.md`: Global coding, git branching, and architectural standards.
- Path-scoped `CLAUDE.md` files: Subdirectories (`apps/web`, `services/api`, `packages/database`) enforce local linters, testing frameworks, and isolation boundaries.
- Slash commands (`/explore`, `/plan-mode`): Prevent accidental code changes by forcing analysis and architectural decision docs (ADRs) prior to file mutation.

### Run Evidence & Verification
- **Test Suite Output:** `pytest` returned 35 passed out of 35 in `9.14s` (`system_3_claude_code_config/pytest_output.txt`).
- **Harness Validator Citation:** In `system_3_claude_code_config/validator_output.txt`:
  - `apps/web/CLAUDE.md` and `services/api/CLAUDE.md` verified for clean inheritance.
  - Path-scoped boundaries verified: `apps/web` cannot mutate `services/api` schemas directly.
  - Sub-agent scratchpad skill config was validated as fully isolated from primary conversation context.

---

## System 4: Layer 3 Orchestrator (Multi-Shift Quality Monitoring)

### Tiered Storage, SQL Pre-filtering & Crash Recovery
- **Tiered State Model:** L1 scratchpad holds volatile turn state (< 2 KB); L2 holds structured shift summaries; L3 database stores raw production logs.
- **SQL Pre-Filtering:** Rather than passing hundreds of thousands of raw logs into Claude's context, deterministic SQL queries aggregate and filter anomalous spikes before injecting data into the prompt.
- **Crash Recovery & Branch Forking:** The session state is persisted to SQLite using WAL checkpoints. When interrupted mid-shift, the orchestrator deserializes state and resumes without token loss or duplicate operations.

### Run Evidence & Verification
- **Test Suite Output:** `pytest` returned 33 passed out of 33 in `15.82s` (`system_4_layer_3_orchestrator/pytest_output.txt`).
- **Run Artifact Citation:** From `system_4_layer_3_orchestrator/shift_run_artifact.json`:
  - SQL pre-filtering achieved a `98.4%` reduction on raw logs (filtering down 28,540 raw database records).
  - Crash recovery simulated at `Turn 14 (Mid-Shift Batch Validation)` resumed session `SHIFT-MONITOR-2026-Q3-S2` with `0` data loss records.
  - Forked anomaly investigation `INVESTIGATION-BRANCH-ANOMALY-409` isolated root cause (`Worker-04 serialization latency`) and merged back cleanly without context bloat.
