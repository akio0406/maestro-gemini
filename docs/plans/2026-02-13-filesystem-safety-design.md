# Filesystem Safety Architecture

**Date:** 2026-02-13
**Status:** Approved (revised after dual-agent review)

## Problem

Maestro operations that create or access directories (session state, plans, archives, parallel dispatch) produce jarring hard errors in Gemini CLI when the target directories don't exist. This affects session creation, resume/archive operations, and parallel dispatch across the entire orchestration lifecycle. The same class of error occurs when subagents attempt to write files to non-existent directories in the target project.

Current state: directory creation instructions are scattered as informal procedural steps across multiple skills with no enforcement, no centralization, and no coverage for target project operations.

## Solution

A three-layer filesystem safety architecture with clear separation of concerns:

| Layer | Component | Responsibility | Scope |
|-------|-----------|---------------|-------|
| Infrastructure | `scripts/ensure-workspace.sh` | Creates and validates all Maestro directories | Static directory tree |
| Behavioral | `protocols/filesystem-safety-protocol.md` | Defines ensure-before-write contract for all agents | Delegated subagents |
| Orchestrator | `GEMINI.md` Startup Check #4 | Invokes bootstrap script before any orchestration | Orchestrator startup |

**Layer boundary:** Layer 2 targets delegated subagents only. The orchestrator's own filesystem safety comes from Layer 1 (bootstrap) + Layer 3 (startup check) + defense-in-depth instructions retained in skills (see Integration Changes).

## Layer 1: Infrastructure — `scripts/ensure-workspace.sh`

Shell script that ensures the complete Maestro workspace directory tree exists.

### Behavior

- Accepts state directory path as argument (defaults to `.gemini`)
- Creates full directory tree idempotently via `mkdir -p`
- Silent on success (no output)
- Exits non-zero with actionable error on failure (permission denied, read-only filesystem)
- Validates created directories are writable

### Directory Manifest

```
<state_dir>/
├── state/
│   └── archive/
├── plans/
│   └── archive/
└── parallel/
```

### Interface

```bash
./scripts/ensure-workspace.sh [state-dir]
# Exit 0: all directories exist and are writable
# Exit 1: creation or validation failed (stderr has details)
```

Invoke as `./scripts/ensure-workspace.sh` (resolved relative to the extension root, consistent with the existing `parallel-dispatch.sh` invocation pattern in GEMINI.md).

### Extensibility

Adding a new directory = adding one line to the manifest array in the script. No other files need to change for infrastructure-level additions.

## Layer 2: Behavioral — `protocols/filesystem-safety-protocol.md`

Shared protocol injected into delegation prompts alongside `agent-base-protocol.md`. Defines the behavioral contract for safe filesystem operations covering both Maestro infrastructure and target project directories.

### Rules

**Rule 1 — Ensure Before Write:**
Before any file creation, write, or move operation, verify the target's parent directory exists. If it doesn't, create it using `mkdir -p`. Applies to every file operation that produces output at a path — writes, moves, copies, renames.

**Protocol Override:** This rule modifies the base protocol's Scope Verification Step 2, specifically the check "Files listed for creation have existing parent directories" (`agent-base-protocol.md` line 25). When parent directories are missing, agents must NOT stop and report the issue (as Step 2 line 30 instructs for verification failures). Instead, create the missing directory using `mkdir -p` and continue with the file operation. All other Step 2 verifications (file existence, tool permissions, scope boundaries) remain in effect.

**Rule 2 — Silent Success, Clear Failure:**
Directory creation is a precondition, not a noteworthy event. Don't report successful directory creation. Only report failures (permission denied, disk full) immediately as a blocker in the Task Report.

**Rule 3 — Never Assume Directory State:**
Treat every directory reference as potentially non-existent, even if a prior phase "should have" created it. Phases run independently (especially in parallel dispatch). Each agent ensures its own write targets exist.

**Rule 4 — Path Construction:**
Always construct full paths before writing. Never write to a path assembled from unverified components. For target project operations, verify the project root exists and is writable before creating subdirectories.

**Rule 5 — Scope:**
Applies to Maestro state directories, target project directories, and archive operations.

### Injection Point

The delegation skill prepends this protocol alongside `agent-base-protocol.md` — every delegated agent receives both protocols.

### Token Cost

Adding a second protocol increases each delegation prompt by approximately 300 tokens. This is acceptable because filesystem safety is critical infrastructure, the protocols may evolve at different rates (justifying separate files over a merge), and the overhead is amortized across the full phase context which dominates token usage.

## Layer 3: Orchestrator — `GEMINI.md` Startup Check

### New Startup Check #4: Workspace Readiness

After settings resolution (Step 2) and disabled agent check (Step 3):

> **Workspace Readiness**: Invoke `./scripts/ensure-workspace.sh` with the resolved `MAESTRO_STATE_DIR` value via `run_shell_command`. If the script exits non-zero, present the error to the user and do not proceed with orchestration.

Runs before any orchestration command. The script is idempotent so repeated invocations cost nothing.

### Parallel Dispatch Guard

Before writing prompt files for a parallel batch, the orchestrator must ensure the batch-specific dispatch directory exists:

```bash
mkdir -p <state_dir>/parallel/<batch-id>/prompts
```

This is a runtime operation not covered by the static bootstrap (batch IDs are generated dynamically). The execution skill's Parallel Dispatch Protocol Step 3 must be an imperative `mkdir -p` instruction, not documentation of the desired structure.

## Integration Changes

### `GEMINI.md`

Add Startup Check #4 (Workspace Readiness) after the existing Startup Check #3 (Disabled Agent Check). See Layer 3 above for the exact wording.

### `skills/delegation/SKILL.md`

Replace the "Injection Steps" subsection (lines 14-18) with the following complete sequence:

```markdown
### Injection Steps
1. Read `protocols/agent-base-protocol.md`
2. Read `protocols/filesystem-safety-protocol.md`
3. Prepend both protocols to the delegation prompt (base protocol first, then filesystem safety) — these appear before the task-specific content
4. For each phase listed in the current phase's `blocked_by`, read `phases[].downstream_context` from session state and include it in the prompt
5. If any required `downstream_context` is missing, include an explicit placeholder noting the missing dependency context (never omit silently)
```

Single integration point — all delegation prompts (sequential and parallel) automatically include filesystem safety rules.

Prompt structure after injection:
```
[agent-base-protocol.md content]

[filesystem-safety-protocol.md content]

Task: [...]
Progress: [...]
Files to modify: [...]
```

### `skills/session-management/SKILL.md`

**Defense-in-depth (belt and suspenders):** Retain the inline directory creation instructions as silent fallbacks rather than removing them. The workspace readiness startup check is the primary mechanism, but these inline instructions protect against bypass scenarios where session-management is activated directly by commands (`/maestro.resume`, `/maestro.archive`) without going through GEMINI.md startup checks.

Changes:
- Session Creation (line 29): Keep "Create `<state_dir>/state/` directory if it does not exist" — add note that this is a defense-in-depth fallback; the workspace readiness startup check is the primary mechanism
- Archive Protocol (lines 167-168): Keep "Create `<state_dir>/plans/archive/` ... Create `<state_dir>/state/archive/` ..." — same defense-in-depth note

### `skills/execution/SKILL.md`

In Parallel Dispatch Protocol, make Step 3 an imperative instruction:

```markdown
3. Ensure the batch-specific dispatch directory exists before writing prompt files:
   ```bash
   mkdir -p <state_dir>/parallel/<batch-id>/prompts
   ```
4. Write each agent's full delegation prompt [...] to its prompt file
```

This replaces the current documentation-style description of the desired directory structure with an enforceable action.

### No Changes

- `protocols/agent-base-protocol.md` — filesystem safety is a separate, composable protocol. The safety protocol's Rule 1 explicitly overrides the base protocol's Scope Verification Step 2 line 25 (parent directory check) while leaving all other Step 2 verifications intact.
- `scripts/parallel-dispatch.sh` — already has proper `mkdir -p` for results directory

## Known Limitations

### Hardcoded `.gemini/` Paths in TOML Commands

Several TOML command files reference hardcoded `.gemini/` paths that cannot be dynamically resolved via `MAESTRO_STATE_DIR`:

**File injection (`@{...}` syntax) — resolved at compile-time by Gemini CLI:**
- `commands/maestro.resume.toml` (line 9): `@{.gemini/state/active-session.md}`
- `commands/maestro.status.toml` (line 7): `@{.gemini/state/active-session.md}`

**Procedural instructions with hardcoded paths:**
- `commands/maestro.archive.toml` (lines 9, 15-18): multiple `.gemini/...` path references
- `commands/maestro.orchestrate.toml` (line 8): `Check .gemini/state/ for any existing active sessions`
- `commands/maestro.execute.toml` (line 11): `check .gemini/plans/ for the most recent implementation plan`

The `@{...}` syntax is resolved at compile-time by Gemini CLI before the prompt reaches the model, so these paths cannot be dynamically resolved. Custom `MAESTRO_STATE_DIR` values will cause the bootstrap script to create directories at the custom path, but these commands will still reference `.gemini/`. This is a pre-existing limitation not introduced by this design. A separate effort to address dynamic path resolution in commands would be needed.

### Defense-in-Depth Timing Gap for File Injection Commands

Commands that use `@{file-path}` syntax (`/maestro.resume`, `/maestro.status`) resolve file paths at compile-time, before any skill activates. This means the defense-in-depth `mkdir` instructions retained in session-management cannot execute before the file injection attempt.

**Affected scenario:** If a user invokes `/maestro.resume` or `/maestro.status` in a project where `.gemini/state/` does not exist, the `@{.gemini/state/active-session.md}` injection will fail or inject empty content before the session-management skill has a chance to create the directory.

**What defense-in-depth does protect:** Session creation (writing new `active-session.md`) and archive operations (moving files to archive directories) — these are skill-driven write operations that occur after skill activation, so the inline `mkdir` fallback can execute.

**Mitigation:** The Layer 3 startup check (invoked by `/maestro.orchestrate`) creates all directories on first run. Users who follow the normal workflow (`/maestro.orchestrate` first, then `/maestro.resume` or `/maestro.status` later) will never encounter this gap because directories already exist. The gap only manifests when `/maestro.resume` or `/maestro.status` is the very first Maestro command in a project.
