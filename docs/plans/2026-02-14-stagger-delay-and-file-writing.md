# Stagger Delay & File Writing Enforcement Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Fix stagger delay defaulting to 0 during parallel dispatch, sync 3 missing settings to `gemini-extension.json`, and reinforce the file writing rule across 4 prompt layers so agents stop using `cat >>`.

**Architecture:** Configuration-only changes across shell scripts, JSON manifests, and Markdown protocol/skill files. No compiled code. Defense-in-depth: safe script defaults + settings discoverability + multi-layer prompt reinforcement.

**Tech Stack:** Bash, JSON, Markdown (with YAML frontmatter)

---

### Task 1: Change stagger delay default in parallel-dispatch.sh

**Files:**
- Modify: `scripts/parallel-dispatch.sh:37` (usage text)
- Modify: `scripts/parallel-dispatch.sh:80` (default value)

**Step 1: Update the usage text default**

In `scripts/parallel-dispatch.sh`, change line 37 from:

```
  MAESTRO_STAGGER_DELAY       Seconds between agent launches (default: 0 = no delay)
```

to:

```
  MAESTRO_STAGGER_DELAY       Seconds between agent launches (default: 5)
```

**Step 2: Update the default value**

In `scripts/parallel-dispatch.sh`, change line 80 from:

```bash
STAGGER_DELAY="${MAESTRO_STAGGER_DELAY:-0}"
```

to:

```bash
STAGGER_DELAY="${MAESTRO_STAGGER_DELAY:-5}"
```

**Step 3: Commit**

```bash
git add scripts/parallel-dispatch.sh
git commit -m "fix: change stagger delay default from 0 to 5 seconds"
```

---

### Task 2: Update stagger delay default in GEMINI.md

**Files:**
- Modify: `GEMINI.md:34` (settings table row)

**Step 1: Update the settings table row**

In `GEMINI.md`, change line 34 from:

```
| Stagger Delay | `MAESTRO_STAGGER_DELAY` | `0` (none) | Seconds between parallel agent launches |
```

to:

```
| Stagger Delay | `MAESTRO_STAGGER_DELAY` | `5` (seconds) | Seconds between parallel agent launches |
```

**Step 2: Commit**

```bash
git add GEMINI.md
git commit -m "docs: update stagger delay default to 5 seconds in settings table"
```

---

### Task 3: Add 3 missing settings to gemini-extension.json

**Files:**
- Modify: `gemini-extension.json:66` (before closing bracket of settings array)

**Step 1: Add the 3 new settings entries**

In `gemini-extension.json`, after the State Directory entry (line 65) and before the closing `]` on line 66, add:

```json
    {
      "name": "Max Concurrent Agents",
      "description": "Maximum agents running simultaneously in parallel dispatch (0 = unlimited, default: 0)",
      "envVar": "MAESTRO_MAX_CONCURRENT",
      "sensitive": false
    },
    {
      "name": "Stagger Delay",
      "description": "Seconds between parallel agent launches to prevent rate-limiting (default: 5)",
      "envVar": "MAESTRO_STAGGER_DELAY",
      "sensitive": false
    },
    {
      "name": "Execution Mode",
      "description": "Phase 3 dispatch mode: parallel (concurrent agents), sequential (one at a time), or ask (prompt user). Default: ask",
      "envVar": "MAESTRO_EXECUTION_MODE",
      "sensitive": false
    }
```

The existing State Directory entry (line 65) needs a trailing comma added after its closing `}`.

**Step 2: Verify valid JSON**

Run: `python3 -c "import json; json.load(open('gemini-extension.json'))"`

Expected: no output (success)

**Step 3: Commit**

```bash
git add gemini-extension.json
git commit -m "feat: add max concurrent, stagger delay, and execution mode to extension settings"
```

---

### Task 4: Elevate file writing rule to top of agent-base-protocol.md

**Files:**
- Modify: `protocols/agent-base-protocol.md`

**Step 1: Move and strengthen the file writing section**

In `protocols/agent-base-protocol.md`, remove the existing "File Writing Protocol" section (lines 46-54, including the `---` separator above it on line 44) and insert a new strengthened version at the very top of the file, immediately after the introductory paragraph (after line 3, before the `---` on line 5).

The new section to insert after line 3 (`...regardless of their specialization.`):

```markdown

---

## CRITICAL: File Writing Rule

ALWAYS use `write_file` for creating files and `replace` for modifying files.

NEVER use `run_shell_command` to write file content. This includes:
- `cat`, `cat >>`, `cat << EOF`
- `echo`, `printf`
- Heredocs (`<< EOF`, `<< 'EOF'`)
- Any shell redirection for content (`>`, `>>`)

Shell interpretation corrupts YAML frontmatter, Markdown syntax, backticks, brackets, and special characters. This rule has NO exceptions.

If `write_file` is not in your authorized tool list, you cannot create files. Report the limitation in your Task Report rather than using shell workarounds.
```

Remove the old section (the `---` separator on line 44, the `## File Writing Protocol` header on line 46, and its body through line 54, plus the `---` on line 56). Leave the `---` separator that precedes `## Output Handoff Contract` intact.

After the edit, the file structure should be:
1. `# Agent Base Protocol` (title + intro paragraph)
2. `## CRITICAL: File Writing Rule` (elevated, strengthened)
3. `## Pre-Flight Protocol` (unchanged)
4. `## Output Handoff Contract` (unchanged)

**Step 2: Commit**

```bash
git add protocols/agent-base-protocol.md
git commit -m "fix: elevate file writing rule to top of agent base protocol"
```

---

### Task 5: Add Rule 6 to filesystem-safety-protocol.md

**Files:**
- Modify: `protocols/filesystem-safety-protocol.md:27` (after Rule 5)

**Step 1: Add Rule 6 at the end of the file**

Append after the last line (line 27, `This protocol applies to Maestro state directories, target project directories, and archive operations.`):

```markdown

## Rule 6 — Write Tool Only

All file content must be written using `write_file` or `replace` tools. Never use `run_shell_command` with `cat`, `echo`, `printf`, heredocs, or shell redirection (`>`, `>>`) to create or modify file content. Shell interpretation corrupts structured content (YAML, Markdown, JSON, code with special characters). This reinforces the Agent Base Protocol's File Writing Rule.
```

**Step 2: Commit**

```bash
git add protocols/filesystem-safety-protocol.md
git commit -m "fix: add write-tool-only rule to filesystem safety protocol"
```

---

### Task 6: Add FILE WRITING RULES block to delegation prompt structure

**Files:**
- Modify: `skills/delegation/SKILL.md:206` (Tool Restriction Enforcement section, prompt structure list)

**Step 1: Add FILE WRITING RULES as item 4 in the prompt structure list**

In `skills/delegation/SKILL.md`, the required prompt structure list (lines 204-209) currently reads:

```
1. Agent Base Protocol (from `protocols/agent-base-protocol.md`)
2. Filesystem Safety Protocol (from `protocols/filesystem-safety-protocol.md`)
3. **TOOL RESTRICTIONS block (immediately here, before any task content)**
4. Context chain from prior phases
5. Task-specific instructions
6. Scope boundaries and prohibitions
```

Change it to:

```
1. Agent Base Protocol (from `protocols/agent-base-protocol.md`)
2. Filesystem Safety Protocol (from `protocols/filesystem-safety-protocol.md`)
3. **TOOL RESTRICTIONS block (immediately here, before any task content)**
4. **FILE WRITING RULES block (immediately after tool restrictions)**
5. Context chain from prior phases
6. Task-specific instructions
7. Scope boundaries and prohibitions
```

**Step 2: Add the FILE WRITING RULES block template**

After the TOOL RESTRICTIONS block template (after line 223, which ends with `...until the Gemini CLI supports runtime tool restriction flags.`), add:

```markdown

The file writing rules block template:

```
FILE WRITING RULES (MANDATORY):
Use ONLY `write_file` to create files and `replace` to modify files.
Do NOT use run_shell_command with cat, echo, printf, heredocs, or shell redirection (>, >>) to write file content.
Shell interpretation corrupts YAML, Markdown, and special characters. This rule has NO exceptions.
```

This block reinforces the Agent Base Protocol's File Writing Rule directly in every delegation prompt, ensuring agents see the prohibition even if they skim the injected protocols.
```

**Step 3: Commit**

```bash
git add skills/delegation/SKILL.md
git commit -m "fix: add file writing rules block to delegation prompt structure"
```

---

### Task 7: Update test script prompts to model correct behavior

**Files:**
- Modify: `scripts/test-parallel-dispatch.sh:22-35` (agent-a prompt)
- Modify: `scripts/test-parallel-dispatch.sh:38-51` (agent-b prompt)

**Step 1: Update agent-a prompt**

In `scripts/test-parallel-dispatch.sh`, replace the agent-a prompt heredoc content (lines 23-34) with:

```
You are Agent A in a parallel execution test.

Your task: Use the write_file tool to write EXACTLY this content to the file src/file-a.txt:
"Agent A was here. Timestamp: $(date +%s)"

IMPORTANT: You MUST use the write_file tool to write the file. Do NOT use cat, echo, printf, heredocs, or any shell command to write file content.

Steps:
1. Use write_file to write the content to src/file-a.txt
2. Use read_file to read the file back and confirm

Do NOT touch any other files. Only modify src/file-a.txt.

When done, report: which file you modified and the content you wrote.
```

**Step 2: Update agent-b prompt**

Replace the agent-b prompt heredoc content (lines 39-50) with the same pattern, substituting "Agent B" and `src/file-b.txt`.

**Step 3: Commit**

```bash
git add scripts/test-parallel-dispatch.sh
git commit -m "fix: update test prompts to model write_file usage"
```

---

## Dependency Graph

All 7 tasks are independent — no blocking dependencies. They modify different files with no overlap.

```
Task 1 (parallel-dispatch.sh defaults)     ──┐
Task 2 (GEMINI.md settings table)           ──┤
Task 3 (gemini-extension.json settings)     ──┼── All independent, can run in parallel
Task 4 (agent-base-protocol.md elevation)   ──┤
Task 5 (filesystem-safety-protocol.md R6)   ──┤
Task 6 (delegation SKILL.md template)       ──┤
Task 7 (test-parallel-dispatch.sh prompts)  ──┘
```

## Agent Allocation

| Task | Files | Risk | Execution |
|------|-------|------|-----------|
| 1 | `scripts/parallel-dispatch.sh` | LOW | Subagent (haiku) |
| 2 | `GEMINI.md` | LOW | Subagent (haiku) |
| 3 | `gemini-extension.json` | LOW | Subagent (haiku) |
| 4 | `protocols/agent-base-protocol.md` | MEDIUM | Subagent (sonnet) |
| 5 | `protocols/filesystem-safety-protocol.md` | LOW | Subagent (haiku) |
| 6 | `skills/delegation/SKILL.md` | MEDIUM | Subagent (sonnet) |
| 7 | `scripts/test-parallel-dispatch.sh` | LOW | Subagent (haiku) |
