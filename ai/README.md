# AI

Knowledge base for everything AI — models, tooling, prompting, and applied work.

## Structure

```
ai/
├── models/      # Model families, capabilities, pricing, benchmarks
├── prompting/   # Prompt patterns, techniques, evaluated results
├── tools/       # Claude Code, SDKs, MCP servers, agent frameworks
├── papers/      # Paper summaries with takeaways
└── projects/    # Notes from things actually built
```

Add folders as they are needed — none of these have to exist up front.

## Conventions

Same rules as the root [README](../README.md), plus:

- **Date everything.** AI facts go stale fast. Every note gets a `date:` in its frontmatter, and any claim about pricing, limits, or model behavior gets the date it was verified.
- **Record the version.** Name the exact model or tool version a note applies to (e.g. `claude-opus-4-8`, not "the latest Opus").
- **Link the primary source.** Prefer official docs, model cards, and papers over blog posts summarizing them.
- **Separate observation from conclusion.** Note what was actually seen in a run, then what was inferred from it.

### Frontmatter

```markdown
---
title: Topic Title
date: 2026-07-19
tags: [ai, models]
model: claude-opus-4-8
source: https://docs.claude.com/...
verified: 2026-07-19
---
```

## Searching

```bash
# All notes about a specific model
grep -rl "model:.*opus" --include="*.md" ai/

# Notes not verified this year — candidates for a refresh
grep -rL "verified: 2026" --include="*.md" ai/
```
