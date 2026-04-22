# mddb

A [Claude Code](https://claude.com/claude-code) skill for maintaining a lightweight **"markdown database"** of context notes across a codebase.

Claude Code already loads `CLAUDE.md` files automatically. This skill is about **writing and maintaining** them — so that the next session gets the right context without re-reading the whole repo.

## The model

Two kinds of files, nothing else:

1. **Root `CLAUDE.md`** — an **index** of pointers plus a short list of project-wide rules. Under 100 lines. No detailed logic.
2. **Per-folder `CLAUDE.md`** — the actual content: purpose, business rules, gotchas. Lives next to the code it describes. 20–80 lines.

Think of the root as a table of contents. When Claude needs to know how an area works, the root tells it which folder note to open.

## When it triggers

The skill activates when you say things like:

- "document this folder"
- "save what we learned"
- "add this as a project rule"
- "remember this for next time"

It also triggers proactively when you state a project-wide convention ("every string must be translated", "always use the repository pattern") or explain the purpose of a folder in a way worth persisting.

## What it avoids

- Detail in the root (the root is an index, not documentation).
- Pointers to folders that don't have a `CLAUDE.md` yet.
- Listing every file in a folder — only the ones that matter.
- Restating code, signatures, or framework basics.
- Journal-style entries ("I refactored this today").

## Install

Clone into your Claude Code skills directory:

```bash
# User-level (available in every project)
git clone https://github.com/barrinhos123/claude-skill-mddb.git ~/.claude/skills/mddb

# Or project-level (only in this repo)
git clone https://github.com/barrinhos123/claude-skill-mddb.git .claude/skills/mddb
```

Claude Code will pick up the skill on next launch.

## Full spec

See [`SKILL.md`](./SKILL.md) for the complete skill definition — workflow, templates, and anti-patterns.
