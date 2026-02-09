---
name: validation
description: Cross-cutting validation methodology for verifying phase outputs and project integrity
---

# Validation Skill

Activate this skill when validating phase outputs during orchestration execution or when running standalone validation checks. This skill provides the pipeline, heuristics, and interpretation rules for verifying that changes meet quality standards.

## Validation Pipeline

Execute validation steps in this order. Stop on the first blocking failure unless the user explicitly requests continuing.

### Step 1: Build / Compile
Verify the project compiles without errors.

| Project Type | Command |
|-------------|---------|
| Node.js (TypeScript) | `npx tsc --noEmit` |
| Node.js (JavaScript) | N/A (skip) |
| Rust | `cargo build` |
| Go | `go build ./...` |
| Python | `python -m py_compile [files]` |
| Java (Maven) | `mvn compile` |
| Java (Gradle) | `./gradlew compileJava` |

### Step 2: Lint / Format
Verify code meets style and quality standards.

| Project Type | Command |
|-------------|---------|
| Node.js | `npx eslint . && npx prettier --check .` |
| Rust | `cargo clippy && cargo fmt --check` |
| Go | `go vet ./... && gofmt -l .` |
| Python | `ruff check . && ruff format --check .` |
| Java | `mvn checkstyle:check` or `./gradlew checkstyleMain` |

### Step 3: Unit Tests
Run unit tests to verify behavior preservation.

| Project Type | Command |
|-------------|---------|
| Node.js (Jest) | `npx jest` |
| Node.js (Vitest) | `npx vitest run` |
| Rust | `cargo test` |
| Go | `go test ./...` |
| Python (pytest) | `python -m pytest tests/` |
| Java (Maven) | `mvn test` |
| Java (Gradle) | `./gradlew test` |

### Step 4: Integration Tests
Run integration tests if available and applicable.

Detect integration test presence by looking for:
- `tests/integration/`, `test/integration/`, or `**/integration_test*` directories/files
- Test files with `integration` in the name
- Test scripts in package.json (e.g., `test:integration`)

### Step 5: Manual Verification
For changes that cannot be automatically validated, present a checklist to the user.

## Project Type Detection

Detect the project type by checking for the presence of these files in the project root:

| Indicator File | Project Type |
|---------------|-------------|
| `package.json` | Node.js (check for `typescript` dep for TS) |
| `Cargo.toml` | Rust |
| `go.mod` | Go |
| `pyproject.toml` or `setup.py` | Python |
| `pom.xml` | Java (Maven) |
| `build.gradle` or `build.gradle.kts` | Java (Gradle) |
| `Gemfile` | Ruby |
| `*.csproj` or `*.sln` | .NET |

When multiple indicators are present, validate each project type independently.

## Validation Result Interpretation

### Pass
All executed validation steps completed with exit code 0. No errors or warnings that indicate broken functionality.

### Fail (Blocking)
Any of the following constitute a blocking failure:
- Build/compile errors
- Lint errors (not warnings, unless the project treats warnings as errors)
- Test failures
- Type errors

### Warn (Non-Blocking)
The following are recorded but do not block progression:
- Lint warnings (when not configured as errors)
- Deprecation notices
- Coverage decreases (unless coverage threshold is configured)
- Format-only issues (can be auto-fixed)

## Post-Phase Validation

### When to Validate
Run validation after:
- Every phase that creates or modifies source code
- Every parallel batch completion (validate the combined result)
- Before marking any phase as `completed`

### When to Skip Validation
Skip validation when:
- The phase only modified documentation files
- The phase only produced read-only analysis (architect, code-reviewer reports)
- The user explicitly requests skipping validation

Record `skipped` with rationale in the phase validation result.

## Manual Verification Checklist

For changes that cannot be automatically validated, present this checklist template:

```
### Manual Verification Required

The following changes require manual verification:

- [ ] [Description of what to verify]
- [ ] [Visual/UI changes look correct]
- [ ] [Integration with external service works]
- [ ] [Environment-specific behavior confirmed]

Please confirm these items are verified before I mark this phase as complete.
```

Use manual verification for:
- UI/visual changes
- External service integrations
- Environment-specific configurations
- Performance improvements (require load testing)
- Security remediations (require penetration testing)
