# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Changed

- Consolidated five documentation roles into four — Specification merged into Architecture as two surfaces (prose CODEMAPS + JSON-LD triples in `graph.jsonld`)
- Added Phase 0 (Codemap Freshness Pre-check) that cascades to the `codemap-writer` agent when CODEMAPS lag the source tree by seven or more days, drift in file count by twenty percent or more, or the index file is missing
- Refined confirmation policy: only new file / new directory creation requires user confirmation; edits to existing files apply automatically, with `git diff` as the audit trail
- Clarified role boundary with `claude-skill-jsonld-knowledge-graph`: context-sync defers to it for the concept-level Architecture surface (`graph.jsonld`)
- `origin` field on SKILL.md frontmatter switched from `ECC` to `shimo4228` to reflect post-ECC-withdrawal authorship (originally submitted as ECC PR #827)

### What it does

A Claude Code Agent Skill that audits project documentation for role overlap. Classifies every Markdown file into one of four roles, finds content in the wrong role, migrates content into the correct file, creates missing role files (ADR index, CODEMAPS), and verifies that numeric claims in documentation match code reality. Designed to keep AI-assisted development sessions from absorbing stale or contradictory context.

### Components

- `SKILL.md` — the skill body. Six-phase workflow (Codemap Freshness Pre-check, Discover, Overlap Detection, Create / Migrate, Freshness Check, Report). Confirmation is required only for new file / new directory creation; edits to existing files apply automatically.

### Scope

The skill assumes a Markdown-based documentation surface with file-role conventions a reader can recognize (CLAUDE.md / .cursorrules for Context, docs/CODEMAPS/ for Architecture prose, docs/adr/ for Decisions, README.md for External). Adopters using a different naming convention substitute the equivalent files; the four-role model itself is the load-bearing contribution.

### Requirements

- The skill is documentation-only; verification commands inside invoke `git`, `find`, `grep`, and language-specific count tools (e.g. `pytest --collect-only` for test counts) where applicable.

### Relationship to companion skills

| Skill / Agent | Role | When |
|---|---|---|
| `codemap-writer` agent | File-level architecture map regeneration | Phase 0 cascade target when CODEMAPS are stale |
| [`claude-skill-jsonld-knowledge-graph`](https://github.com/shimo4228/claude-skill-jsonld-knowledge-graph) | Concept-level Architecture surface | Owns the `graph.jsonld` boundary that context-sync defers to |
| [`claude-skill-llms-txt-writer`](https://github.com/shimo4228/claude-skill-llms-txt-writer) | AI-facing documentation prose | Used when `llms.txt` / `llms-full.txt` need regeneration |
| `adr-writer` agent | Six-section ADR body generation | Phase 3 invokes it when buried decisions are extracted to formal ADR files |
