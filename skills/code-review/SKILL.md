---
name: code-review
description: Standalone code review methodology for structured, severity-classified code assessment
---

# Code Review Skill

Activate this skill when performing standalone code reviews via the `/maestro.review` command or during Phase 3 code quality gates. This skill provides the methodology for scoping, executing, and reporting code reviews.

## Scope Detection Protocol

Determine review scope using the following priority order:

1. **User-specified paths**: If the user provides file paths or glob patterns, review those exclusively
2. **Staged changes**: If `git diff --staged` produces output, review staged changes
3. **Last commit diff**: If no staged changes exist, review the last commit via `git diff HEAD~1`
4. **Fallback**: If none of the above yield content, ask the user to specify scope

Always confirm the detected scope with the user before proceeding.

## Review Orchestration

### Delegation Flow

1. Detect review scope using the protocol above
2. Gather the full diff content for the detected scope
3. Delegate to the `code-reviewer` agent with:
   - The full diff content
   - File paths involved
   - Any user-provided focus areas or concerns
4. Process the agent's Task Report
5. Present findings to the user in the structured output format below

### Context Enrichment

When delegating to the code-reviewer agent, include:
- The diff content (not just file names)
- Surrounding context for modified sections (10 lines before/after when available)
- Project language and framework information (detected from package.json, Cargo.toml, go.mod, etc.)

## Severity Classification

### Critical
Issues that could cause security vulnerabilities, data loss, or system crashes:
- SQL/NoSQL injection vectors
- Authentication/authorization bypasses
- Unvalidated user input at system boundaries
- Resource leaks (unclosed connections, file handles)
- Race conditions with data corruption potential

### Major
Issues that cause bugs, design flaws, or significant maintainability problems:
- Logic errors in business rules
- Missing error handling on external calls
- SOLID principle violations that impact extensibility
- Incorrect API contracts or type mismatches
- Missing null/undefined checks on external data

### Minor
Issues related to style, naming, or minor convention violations:
- Naming inconsistencies
- Code style deviations from project conventions
- Suboptimal but correct implementations
- Missing type annotations where inference is insufficient

### Suggestion
Optional improvements that enhance readability or maintainability:
- Alternative patterns that improve clarity
- Performance optimizations with marginal impact
- Structural improvements for future extensibility

## Output Format

Present findings in a structured table followed by a summary:

```
## Code Review Results

**Scope**: [description of what was reviewed]
**Files Reviewed**: [count]
**Total Findings**: [count by severity]

### Findings

| # | Severity | File | Line | Description | Suggested Fix |
|---|----------|------|------|-------------|---------------|
| 1 | Critical | path/to/file.ts | 42 | [description] | [fix] |
| 2 | Major | path/to/file.ts | 87 | [description] | [fix] |

### Summary

[1-2 paragraph summary of overall code quality, patterns observed, and priority actions]
```

## Verification Rule

Every finding **must**:
- Reference a specific file and line number
- Be verified against the actual code (not assumed from patterns)
- Include a concrete suggested fix or action
- Be classified with a severity that matches the classification criteria above

Do NOT report:
- Speculative issues based on assumptions about runtime behavior
- Style preferences not established by the project's conventions
- Issues in code outside the review scope
