# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- §9.5 Impact analysis and design decisions: a source-verified linkage matrix (system-prompt, agent-instructions, compaction, token-meter, spill, session persistence, session-query, session projections, model routing, agent-loop, checkpoint-policy, tool-result-pruner, intercept/isolate, events) plus a replacement decision checklist.
- "Knowledge Base Version Anchor" section pinning the KB to deepseek-harness commit `99f6f02fecdb7dff40c3fbc9470f5907c29f74ca` (2026-08-18) with an upstream-change comparison procedure.
- Project governance files: `LICENSE` (MIT), `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md` (Contributor Covenant 2.1).

## [0.1.0] - 2026-08-18

### Added
- Initial English-language skill: role and goal, anti-hallucination ground rules with a Reality-check table, quick start, four-phase development workflow, and a knowledge base spanning §1–§16, including the core pluggability map (§9), three replacement approaches (§10), and the custom compaction walkthrough (§11).

### Changed
- Knowledge base expanded from §1–§11 to §1–§16; Testing / Common pitfalls / Delivery checklist renumbered to §14 / §15 / §16.

### Fixed
- §8 service list deduplicated against §9; §11 example `compactRegion` marked `async`; §13 hot-path claims softened to candidate hot paths.
