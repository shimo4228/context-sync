# context-sync

Audit and fix project documentation role overlaps. One command to keep all your context files healthy.

## Why This Matters

In LLM-driven development, organizing concepts _is_ implementation. Markdown carries the same weight as executable code — a stale number in CLAUDE.md, a misplaced design rationale in README, or a contradictory module count silently degrades every AI-assisted session that reads it. The agent works from what the documents say, not from what the code is.

Context consistency is not a housekeeping task. It is a prerequisite for the core concepts to reach the system without noise. This skill mechanically guarantees that consistency — freeing the human to focus on creating and refining the concepts themselves.

## The Problem

As projects grow, documentation sprawls across multiple files with overlapping responsibilities:

- CLAUDE.md becomes a dumping ground for architecture details, design decisions, and conventions
- README has internal implementation details that belong elsewhere
- Design decisions live in comments, PR descriptions, or nowhere at all
- Numbers in docs (module count, test count, version) silently go stale
- Domain entities and relationships are described in prose but never made machine-readable
- Data schemas appear in 3+ files and drift out of sync

## The Solution

`context-sync` enforces a **four-role model** where every piece of project knowledge lives in exactly one place:

| Role | Purpose | Examples |
|------|---------|---------|
| **Context** | How to work in this project | CLAUDE.md, .cursorrules, AGENTS.md |
| **Architecture** | What the code looks like (file-level) AND what concepts it defines (concept-level) | docs/CODEMAPS/, docs/architecture/, graph.jsonld |
| **Decisions** | Why the code is this way | docs/adr/ |
| **External** | What this project is | README.md |

### Architecture's two surfaces

The Architecture role spans two complementary surfaces that do not overlap:

- **Prose (CODEMAPS)** — "where is X implemented?" — file-level index with paths, function names, and line numbers for navigation
- **JSON-LD triples (graph.jsonld)** — "what is X / how does X relate to Y?" — domain entities and relationships encoded as machine-readable schema.org triples for LLM citation

The role boundary between CODEMAPS prose and `graph.jsonld` is owned by the [jsonld-knowledge-graph](https://github.com/shimo4228/jsonld-knowledge-graph) skill — `context-sync` defers to it for the concept-level surface.

## What It Does

0. **Codemap Freshness Pre-check** — Detects stale `docs/CODEMAPS/` against current source (timestamp lag, file-count drift, missing index) and automatically cascades to `codemap-writer` for regeneration before any other phase runs
1. **Discover** — Scans for context files, classifies them into four roles, identifies missing roles
2. **Overlap Detection** — Finds content in the wrong role (architecture detail in CLAUDE.md, decision rationale in README)
3. **Create / Migrate** — Creates missing docs (ADR, architecture), moves content, replaces with pointers
4. **Freshness Check** — Verifies numbers match code reality, flags stale files
5. **Report** — Summary of all changes

**Confirmation policy**: user confirmation is required only for creating a new file or a new directory. Edits to existing files apply automatically — `git diff` is the audit trail, and `git checkout -- <file>` is the undo.

## Results

**Large project** (6900 LOC, 30 modules):
- Migrated scattered design descriptions into CODEMAPS prose and a JSON-LD knowledge graph (Architecture's two surfaces)
- ADRs remain independent decision records
- All docs maintainable via context-sync

**Small project** (iOS app, 591 tests):
- Correctly identified project was already healthy
- Only action: created missing ADR index + fixed stale test count

## Recommended Workflow

### Solo developer

Run after milestones — feature complete, refactor done, architecture change. Not every commit; that's too frequent. A good trigger is "I just changed how something works, not just what it does."

### Team / PR workflow

Run as part of PR review, **before merge**. This catches doc drift introduced by the PR itself — a renamed module, a new pipeline stage, a changed threshold. Post-merge requires a follow-up PR just for doc fixes, which adds noise and often gets skipped.

### Scheduled hygiene

Monthly or per-sprint as a health check. Catches gradual drift that no single PR introduced — stale test counts, module counts that crept up, ADRs that were never created for decisions made in Slack.

## Installation

Copy the skill directory into your Claude Code skills tree:

```bash
git clone https://github.com/shimo4228/context-sync
cp -r context-sync/skills/context-sync ~/.claude/skills/context-sync
```

Then invoke with `/context-sync` in Claude Code.

## About this skill

This skill implements the **Maintain** phase of the [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle) — a Zenodo-citable six-phase bidirectional growth loop ([DOI 10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726)) for sustaining intent alignment between an AI agent and its operator over time. AKC is one of three research lines by [@shimo4228](https://github.com/shimo4228), alongside [Contemplative Agent](https://github.com/shimo4228/contemplative-agent) ([DOI 10.5281/zenodo.19212118](https://doi.org/10.5281/zenodo.19212118)) — autonomous agents grounded in four contemplative axioms — and [Agent Attribution Practice (AAP)](https://github.com/shimo4228/agent-attribution-practice) ([DOI 10.5281/zenodo.19652013](https://doi.org/10.5281/zenodo.19652013)) — harness-neutral ADRs on accountability distribution.

## License

MIT
