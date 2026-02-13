# Parallel Subagent Execution Research Spike

**Created**: 2026-02-12
**Status**: Complete
**Objective**: Determine if pseudo-parallel subagent execution is achievable before Gemini CLI ships native support
**Decision**: Viable with caveats

---

## Background

Gemini CLI currently executes subagents sequentially — each agent is a tool call, and tool calls run one at a time. Parallel execution is tracked in:
- [Issue #17749: Parallel Execution of Subagents](https://github.com/google-gemini/gemini-cli/issues/17749) — 0 of 4 subtasks complete
- [Epic #17120: Parallel Tool Calling](https://github.com/google-gemini/gemini-cli/issues/17120) — broader initiative

Maestro's delegation and execution skills describe parallel dispatch patterns but the `delegate_to_agent` tool is inherently sequential. This research investigated workarounds.

---

## Phase 1: Investigation

### Question 1: Does `/subagent:run` or equivalent mechanism exist?

**Findings:**

No. Subagents are invoked via a unified `delegate_to_agent` tool (PR #14769). This is a Gemini function call tool registered in the tool registry using a Zod discriminated union keyed by `agent_name`. The tool executes subagents synchronously within a `SubAgentScope` context. There is no `/subagent:run` command or programmatic spawning mechanism outside the tool-call flow.

The `delegate_to_agent` tool is processed by `CoreToolScheduler` (`packages/core/src/core/coreToolScheduler.ts`) which explicitly enforces sequential execution:
```typescript
if (this.toolCalls.length > 0 || this.toolCallQueue.length === 0) { return; }
```
One tool executes while others wait in queue. This is the fundamental sequential bottleneck.

### Question 2: Can `run_shell_command` spawn independent Gemini CLI processes?

**Findings:**

Yes. The `gemini` CLI supports non-interactive execution via:
```bash
gemini -p "prompt" --yolo --output-format json
```

Key flags:
- `-p` / `--prompt`: Non-interactive (headless) mode with given prompt
- `--yolo` / `-y`: Auto-approve all tool calls
- `--output-format json`: Structured JSON output with stats and token usage
- `-e` / `--extensions`: Specify which extensions to load
- `-m` / `--model`: Override model selection

Multiple `gemini` processes CAN run concurrently as independent OS processes. Each spawned process:
- Inherits the project directory and linked extensions automatically
- Loads `.gemini/GEMINI.md` context file and agents from `.gemini/agents/`
- Has its own independent conversation context (no shared state with other processes)
- Requires `experimental.enableAgents: true` in `~/.gemini/settings.json`

Background process spawning via `&` in shell commands is supported but limited — "background shells were never fully supported" per maintainers (Issue #13594). The workaround is a wrapper script that manages spawning and waiting.

### Question 3: What are the constraints?

**Findings:**

| Constraint | Severity | Workaround |
|-----------|----------|------------|
| Token limits | Low | Each process has its own token budget from the API; no shared quota between concurrent processes |
| File locking | Medium | No built-in file locking — must enforce non-overlapping file ownership at the delegation level |
| State management | High | Session state (`active-session.md`) cannot be safely written by multiple processes — orchestrator must update state AFTER batch completes |
| Process management | Medium | Wrapper script uses `wait` and exit codes; `timeout` command handles runaway agents |
| Error handling | Medium | Exit codes + JSON output + stderr logs; orchestrator parses results after batch completes |
| Git conflicts | High | Parallel agents must NOT create git commits — orchestrator commits after validating the batch |
| No follow-up questions | Medium | Agents run non-interactively with `--yolo`; prompts must be fully self-contained |
| No shared context | Medium | Each agent is an independent CLI instance; downstream context from prior phases must be embedded in prompt files |

### Question 4: Are there other parallel patterns in the ecosystem?

**Findings:**

1. **Git worktrees + multiple CLI instances** (Discussion #3395, Dagger blog): Launch separate `gemini-cli` instances in different worktrees for file isolation. Overkill for Maestro since we enforce non-overlapping file ownership.

2. **Gemini API parallel function calls**: The underlying API supports returning multiple function calls in a single response. Gemini 3+ models handle this well. Gemini 2.x crashes on it (Issues #16135, #16144). This is what Issue #17749 will eventually leverage.

3. **Greedy tool scheduler** (#17750): First subtask of #17749 — replaces the queue-based sequential scheduler with one that can process multiple tools concurrently. When this ships, `delegate_to_agent` will support parallel invocation natively, making the shell-based workaround unnecessary.

---

## Phase 2: Prototype

### Gate Checklist

- [x] Question 1: Parallel mechanism identified — shell-based spawning via `gemini -p`
- [x] Question 2: Shell command spawn viability confirmed
- [x] Question 3: Constraints documented
- [x] Question 4: Ecosystem patterns reviewed

### Implementation

Built `scripts/parallel-dispatch.sh` — a shell script that:
1. Reads prompt files from a dispatch directory
2. Spawns one `gemini -p <prompt> --yolo --output-format json` process per prompt file
3. Runs all processes concurrently with PID tracking
4. Enforces `MAESTRO_AGENT_TIMEOUT` via the `timeout` command
5. Collects exit codes, JSON output, and stderr logs per agent
6. Writes `results/summary.json` with batch status

Built `scripts/test-parallel-dispatch.sh` — a test harness that:
1. Creates a temp directory with two files (file-a.txt, file-b.txt)
2. Writes two agent prompt files with non-overlapping tasks
3. Runs the parallel dispatch script
4. Verifies both files were modified and reports results

### Test Cases

| # | Test | Expected | Status |
|---|------|----------|--------|
| 1 | Both agents execute concurrently | Overlapping execution time (wall time < sum of individual times) | Built — requires manual validation with Gemini CLI |
| 2 | Both agents produce correct output | File contents match expected | Built — requires manual validation |
| 3 | No file conflicts | Each agent only touches its assigned file | Enforced by prompt design |
| 4 | Orchestrator detects completion | summary.json contains all agent results | Built — script writes summary |
| 5 | Error in one agent doesn't corrupt the other | Isolated failure with independent exit codes | Built — PID-based isolation |
| 6 | State update after batch | Orchestrator reads results and updates session state | Designed in execution skill |

### Prototype Files

- `scripts/parallel-dispatch.sh` — Production dispatch script
- `scripts/test-parallel-dispatch.sh` — Manual validation harness

---

## Phase 3: Decision

### Decision Matrix

Score: 1 = Poor, 2 = Below Average, 3 = Acceptable, 4 = Good, 5 = Excellent

| Criterion | Weight | Score (1-5) | Notes |
|-----------|--------|-------------|-------|
| Reliability | 30% | 3 | Independent processes are isolated but `gemini` CLI availability in `run_shell_command` PATH not guaranteed |
| Performance gain | 25% | 4 | N agents in ~1x time vs Nx time; significant for 3-4 agent batches |
| Implementation complexity | 20% | 4 | Simple shell script; prompts written to files; results read from files |
| Maintenance burden | 15% | 3 | Must maintain dispatch script; will become unnecessary when Issue #17749 ships |
| Future compatibility | 10% | 5 | Clean migration path: replace shell dispatch with native `delegate_to_agent` parallel calls |
| **Weighted Total** | | **3.55** | Above "Acceptable" threshold |

### Outcome

**Decision: Viable with caveats**

**Rationale:**

Shell-based parallel dispatch is technically sound and provides real performance benefits for batches of 2-4 independent agents. The implementation is straightforward (a single shell script) and the migration path to native parallel support is clean — when Issue #17749 ships, replace shell dispatch with parallel `delegate_to_agent` calls without changing the orchestration logic.

**Caveats:**
1. Requires `gemini` CLI to be available in the `run_shell_command` PATH (should be true in standard installations)
2. Agents cannot ask follow-up questions — prompts must be fully self-contained
3. No shared context between parallel agents — each is an independent CLI instance
4. Session state must NOT be written by parallel agents — orchestrator updates state after batch
5. Git commits must NOT be created by parallel agents — orchestrator commits after validation
6. Manual validation with Gemini CLI needed to confirm end-to-end behavior (test harness built)

### Integration Design

#### Files Modified

| File | Change |
|------|--------|
| `GEMINI.md` | Execution Mode updated to PARALLEL (shell-based) with dispatch protocol and decision logic |
| `skills/execution/SKILL.md` | Parallel Execution section replaced with concrete shell-based Parallel Dispatch Protocol |
| `skills/delegation/SKILL.md` | Parallel Delegation section replaced with prompt file construction protocol and dispatch invocation |
| `scripts/parallel-dispatch.sh` | Created — production dispatch script |
| `scripts/test-parallel-dispatch.sh` | Created — manual validation harness |

#### Limitations and Guards

1. **File ownership enforcement**: Delegation skill requires non-overlapping file sets; no runtime file locking
2. **No git commits in parallel**: Agents instructed via prompt to skip commits; orchestrator commits after batch
3. **Timeout enforcement**: `MAESTRO_AGENT_TIMEOUT` applied via `timeout` command per process
4. **Fallback**: If dispatch script fails, execution skill falls back to sequential `delegate_to_agent`
5. **Batch size limit**: Recommend 2-4 agents per batch to avoid overwhelming the system
6. **State isolation**: Orchestrator is sole writer of session state; parallel agents produce output files only
