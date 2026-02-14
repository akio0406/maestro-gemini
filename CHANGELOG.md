# Changelog

All notable changes to Maestro will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.0] - 2026-02-13

### Added

- Extension settings with 10 configurable parameters via environment variables
- Maestro branded dark theme with warm gold accents
- Shell-based parallel dispatch for concurrent subagent execution (`scripts/parallel-dispatch.sh`)
- Agent base protocol with pre-flight procedures and structured output formatting
- Settings references in delegation, execution, session-management, and validation skills
- TechLead orchestrator startup checks with settings resolution

### Fixed

- Hardened `parallel-dispatch.sh` against shell injection and edge cases

### Changed

- Execution mode upgraded from sequential-only to PARALLEL (shell-based) as default strategy

## [1.0.0] - 2026-02-09

### Added

- TechLead orchestrator with 12 specialized subagents
- Guided design dialogue with structured requirements gathering
- Automated implementation planning with phase dependencies and parallelization
- Parallel execution of independent phases via subagent invocation
- Session persistence with YAML+Markdown state tracking
- Least-privilege security model per agent
- Standalone commands: `maestro.orchestrate`, `maestro.resume`, `maestro.execute`
- Standalone commands: `maestro.review`, `maestro.debug`, `maestro.security-audit`, `maestro.perf-check`
- Session management: `maestro.status`, `maestro.archive`
- Design document, implementation plan, and session state templates
- Skill modules: code-review, delegation, design-dialogue, execution, implementation-planning, session-management, validation
