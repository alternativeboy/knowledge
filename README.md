# Knowledge

A personal knowledge base — notes, summaries, and references kept for long-term use.

## Structure

```
.
├── notes/       # Daily notes / things just learned
├── topics/      # Summaries organized by subject
├── references/  # Cheatsheets, snippets, frequently used links
└── assets/      # Images and attachments
```

> These folders are a starting point, not a requirement. Add them as the need arises.

## Writing Conventions

- One file = one topic, named in `kebab-case.md`
- Always start a file with `# Title` so it is easy to search and preview
- Link between notes with relative paths, e.g. `[Topic](../topics/topic.md)`
- Always cite the source when summarizing from external material
- Write for yourself six months from now — capture the context, not just the conclusion

### Frontmatter (optional)

```markdown
---
title: Topic Title
date: 2026-07-18
tags: [tag1, tag2]
source: https://example.com
---
```

## Searching

```bash
# Search all notes for a term
grep -ri "search term" --include="*.md" .

# Find notes by tag
grep -rl "tags:.*rust" --include="*.md" .

# Recently modified files
git log --name-only --pretty=format: -20 | sort -u
```

## Workflow

```bash
git add .
git commit -m "notes: topic added"
git push
```

## Notes

The `private/` and `drafts/` folders are ignored in `.gitignore` — use them for anything not ready to publish or not meant to be pushed to the remote.
