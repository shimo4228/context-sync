# context-sync

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/context-sync)

An [Agent Skill](https://agentskills.io/specification) for Claude Code that audits and fixes the documents a coding agent reads. It reads a repository's context documents (CLAUDE.md and AGENTS.md, README, architecture decision records (ADRs), and, when present, a graph.jsonld concept graph and llms.txt for AI readers), finds facts that sit in the wrong document or have gone stale against the code, and moves each fact to one home. The author's other work is listed under [More from the author](#more-from-the-author).

It edits existing files without asking (documents, and a script's header comment when a pipeline's stage order moves there) and lists every edit at the end, so start from a committed working tree: `git diff` then shows what changed and `git checkout -- <file>` undoes one. It asks once before creating new files, and asks you for ADR details it cannot find in the documents, such as when a decision should be revisited. The project README is the exception: README findings are reported, not edited.

## Install

Clone this repository for context-sync alone, or install the [akc-cycle](https://github.com/shimo4228/akc-cycle) Claude Code plugin for it together with the other skills of the author's cycle, `adr-writer` among them; there it is called `/akc-cycle:context-sync`. This repository is synced one way from the same source, so between syncs it can trail the plugin.

```bash
git clone https://github.com/shimo4228/context-sync
mkdir -p ~/.claude/skills
cp -r context-sync/skills/context-sync ~/.claude/skills/context-sync
```

or, inside Claude Code:

```
/plugin marketplace add shimo4228/akc-cycle
/plugin install akc-cycle@akc-cycle
```

Then invoke with `/context-sync` in Claude Code, from the root of the repository you want to check, or ask in plain words, such as "the docs may have drifted from the code".

The clone alone runs every step. Its freshness check is a bundled Python script (Python 3.11 or later, standard library only) run from `~/.claude/skills/context-sync`, so keep the folder there. Two sibling skills are optional and widen what it covers:

- **`adr-writer`** writes ADRs, and the script imports its checker for the ADR index. Without it, the ADR index is reported as unverified rather than clean, and decisions that need an ADR are listed for you instead of written. It has no repository of its own; on the clone path, copy its folder from akc-cycle next to context-sync:

  ```bash
  git clone https://github.com/shimo4228/akc-cycle
  cp -r akc-cycle/skills/adr-writer ~/.claude/skills/adr-writer
  ```

- **[jsonld-knowledge-graph](https://github.com/shimo4228/jsonld-knowledge-graph)** creates and extends graph.jsonld, and lints it. context-sync has no concept writer of its own: it hands concept creation in step 3 to a knowledge-graph skill (the author's is this one) and names no other route, so without one, step 3 has no documented way to write concepts; step 1 still reports a missing graph.jsonld as concepts that live only in prose. The script itself only checks that graph.jsonld is valid JSON and prints the linter command at that skill's path, without checking that the skill is installed. Running that command needs the skill and `uv` (it fetches `pyld`).

## The four-role model

A stale number in CLAUDE.md, a misplaced design rationale in README, or a contradictory module count silently degrades every AI-assisted session that reads it. `context-sync` enforces a **four-role model** where every piece of project knowledge lives in exactly one place:

| Role | Purpose | Examples |
|------|---------|---------|
| **Context** | How to work in this project | CLAUDE.md, .cursorrules, AGENTS.md |
| **Architecture** | What concepts the code defines and how they relate (concept-level) | graph.jsonld, docs/architecture/ |
| **Decisions** | Why the code is this way | docs/adr/ |
| **External** | What this project is | README.md |

llms.txt and llms-full.txt sit beside the four roles as a fifth, AI-facing set: the AI-facing counterpart of README. They are the deliberate exception to one place per fact, since they describe the project again for AI readers; what keeps them from duplicating README is a check that llms.txt is not a README copy (the overlap of its first five H2 headings with README's), which flags a copy for regeneration by an llms.txt skill. The skill detects them, flags llms.txt as missing when the repository has graph.jsonld but no llms.txt, and checks that llms.txt links resolve.

context-sync belongs to the Maintain phase of the author's [Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle), where a coding agent's documents are kept to one home per fact after the code moves.

### Architecture is concept-level only

The Architecture role stores domain entities and relationships as machine-readable schema.org triples (`graph.jsonld`), plus at most a short hand-written overview. File-level structure ("where is X implemented?") is derivable from the code with LSP and grep, so it is **not stored**. A hand-maintained module map (`docs/CODEMAPS/` or similar) is reported as a finding, not treated as a role.

The concept-level surface is owned by the [jsonld-knowledge-graph](https://github.com/shimo4228/jsonld-knowledge-graph) skill, and `context-sync` defers to it for that boundary.

## What It Does

1. **Discover**: scans for context files, classifies them into the four roles and the AI-facing set, and identifies missing roles.
2. **Overlap Detection**: finds content in the wrong role, such as architecture detail in CLAUDE.md or decision rationale in README.
3. **Create / Migrate**: creates missing docs (ADRs through `adr-writer`, concepts in graph.jsonld through jsonld-knowledge-graph), moves content and replaces it with pointers, and deletes module or file lists inside context files such as CLAUDE.md when they can be derived from the code.
4. **Freshness Check**: runs the bundled script to check numbers, paths, the ADR index and llms.txt links against the code, and flags stale files.
5. **Report**: lists every file created and edited, and every finding handed elsewhere.

Only step 4 runs a script, and its checks are mechanical. Deciding which role a fact belongs to (steps 1 to 3) is left to the LLM, so what it detects can vary from run to run.

**README findings are flagged, not fixed**: each one is reported with its line and handed to a README skill (the author's is [readme-writer](https://github.com/shimo4228/readme-writer)), or to a release skill when it is a version or DOI.

## Results

**Large project** (6900 LOC, 30 modules):
- Migrated scattered design descriptions into a JSON-LD knowledge graph (Architecture role)

**Small project** (iOS app, 591 tests):
- Correctly judged that its documents needed no role split for its size (no separate Architecture docs)
- Only action: created missing ADR index + fixed stale test count

The first article under [More from the author](#more-from-the-author) has before/after tables for three projects, from an earlier version of the skill that asked before each phase; in one, CLAUDE.md went from 165 to 117 lines and its stale LOC and test counts were corrected.

## Recommended Workflow

### Solo developer

Run after milestones: feature complete, refactor done, architecture change. Not every commit; that's too frequent. A good trigger is "I just changed how something works, not just what it does."

### Team / PR workflow

Run as part of PR review, **before merge**. This catches doc drift introduced by the PR itself, such as a renamed module, a new pipeline stage, or a changed threshold. Post-merge requires a follow-up PR just for doc fixes, which adds noise and often gets skipped.

### Scheduled hygiene

Monthly or per-sprint as a health check. Catches gradual drift that no single PR introduced: stale test counts, module counts that crept up, ADRs that were never created for decisions made in Slack.

## More from the author

- **[Where to Put a Coding Agent's Knowledge — and How to Make It Stick](https://dev.to/shimo4228/where-to-put-a-coding-agents-knowledge-and-how-to-make-it-stick-161g)** ([日本語](https://zenn.dev/shimo4228/articles/coding-agent-memory-architecture)): where the four-role model came from, the first run on the author's own harness (which had no CLAUDE.md), and before/after numbers from three projects, including a small one where the right answer was to change almost nothing.
- **[ワークフロー象限と ReAct 象限の間のグラデーション](https://zenn.dev/shimo4228/articles/react-agent-business-quadrant-4)** (Japanese only): why context-sync leaves the role judgment to the LLM instead of a script, what that buys in portability, and the cost: what it detects can vary from run to run.
- **[Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle)**: the reasoning behind each phase of the cycle, Maintain among them, recorded as dated design decisions.
- **[repo-asset-stocktake](https://github.com/shimo4228/repo-asset-stocktake)**: another Maintain skill; instead of asking whether a document is in the right role, it asks whether a config, CI workflow or runbook is still used by anything at all.
- **[readme-writer](https://github.com/shimo4228/readme-writer)**: where context-sync's README findings go; it rewrites or reviews a README so first-time visitors can tell what the project is, and a fresh judge checks the page against the code.
- **[shimo4228](https://github.com/shimo4228/shimo4228)**: the author's hub, with AKC next to the other long-running practice lines and their DOIs.

## License

MIT

<details>
<summary>For tools and AI assistants</summary>

context-sync is an Agent Skill for Claude Code that audits a repository's context documents for role overlap and for staleness against the code, and fixes them, for developers whose coding agent reads CLAUDE.md, ADRs and similar files in every session. It classifies each document into one of four roles (Context: how to work here; Architecture: what concepts the code defines, as graph.jsonld; Decisions: why, as ADRs; External: what the project is, as README), treats llms.txt and llms-full.txt as a fifth, AI-facing set, and moves misplaced content to the role that owns it, leaving a pointer behind.

It exists because an agent works from what the documents say, not from what the code is: a stale count or a decision buried in CLAUDE.md degrades every session that reads it. It deliberately stores no file-level module map, because file structure can be derived from the code on demand; a hand-maintained one is reported as a finding.

Canonical facts: MIT license; the skill payload (`skills/context-sync/`) is a `SKILL.md` plus a standard-library Python 3.11+ evidence script, `scripts/context_evidence.py`, with tests in `tests/`; maintained by one author (@shimo4228). Status: active, synced one way from the author's Claude Code harness by `scripts/sync-from-local.sh` (it never commits), and also shipped in the akc-cycle plugin as `/akc-cycle:context-sync`, so this repository can trail the plugin between syncs. Requirements: Claude Code and `python3`; no keys and no network calls in the evidence run (URL liveness is reported as unverified). The graph.jsonld linter command it hands over additionally needs the jsonld-knowledge-graph skill and `uv`, and fetches `pyld`. Behavior: it edits existing files without confirmation and lists them in the final report, batch-confirms new files and directories once, asks for ADR inputs it cannot find in the source, and never edits the top-level README (findings go to a README skill such as readme-writer, release numbers to a release skill). ADRs are created through the `adr-writer` skill (without it, decisions that need an ADR are listed instead of written) and graph.jsonld through jsonld-knowledge-graph (no other route is named; without it, a missing graph.jsonld is still reported). Role classification is the LLM's judgment, and the evidence script covers only the mechanical checks: it imports adr-writer's ADR checker from the sibling skill folder and reports the ADR index under `degraded` (unverified) when it is absent, and for graph.jsonld it checks only JSON validity and emits the jsonld-knowledge-graph linter command rather than running it, without checking that the skill is installed.

Example: `python3 ~/.claude/skills/context-sync/scripts/context_evidence.py --root .` prints JSON evidence (it exits 0 whatever it finds, or 2 when `--root` is not a directory; `--gate` exits 3 when a gated check fails or could not run, which covers only the ADR index, graph.jsonld validity, llms.txt and llms-full.txt links and an unreadable llms-full.txt, plus context paths with `--gate-paths`; the other checks stay advisory) with keys such as `degraded`, `checks.context_paths.missing`, `checks.adr_index`, `checks.numeric_claims` and `checks.llms_txt.broken_links`; the skill reads it in its Freshness Check step and ends with a report block listing roles found, files created, sections moved, values updated, items deferred to the README or release skill, and stale files. The author reports one large project (6900 LOC, 30 modules) whose scattered design descriptions moved into a JSON-LD graph, and one small iOS project (591 tests) where the only actions were an ADR index and a corrected test count.

Links: [skills/context-sync/SKILL.md](skills/context-sync/SKILL.md) is the skill itself; [CHANGELOG.md](CHANGELOG.md) holds the release history; [llms.txt](llms.txt) and [llms-full.txt](llms-full.txt) are the machine-readable summary and reference. The skill implements part of the Maintain phase of the [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle), concept DOI [10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726); cite AKC by that DOI. The installable form of the whole cycle is [akc-cycle](https://github.com/shimo4228/akc-cycle).

</details>
