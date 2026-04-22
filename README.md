# mddb

A Claude Code skill that maintains a lightweight "markdown database" of context notes across a codebase.

The idea is simple: instead of Claude re-reading your entire repo every session to figure out how things work, it reads a short root index (`CLAUDE.md`) that points it to per-folder notes describing the important bits. Context where you need it, nowhere else.

## The model

```
repo/
├── CLAUDE.md                    ← index: project rules + "where to look" pointers
├── src/
│   ├── <folder-a>/
│   │   ├── CLAUDE.md            ← what this folder does, key files, rules, gotchas
│   │   └── ...
│   ├── <folder-b>/
│   │   ├── CLAUDE.md
│   │   └── ...
│   └── <folder-c>/              ← no CLAUDE.md; not every folder needs one
│       └── ...
```

**Root `CLAUDE.md`** holds two things and nothing else:
- **Project rules** — short, universal conventions (e.g. "every user-facing string goes through `t()`").
- **Where to look** — one-line pointers to the per-folder notes (e.g. `Booking flow → src/booking/CLAUDE.md`).

**Per-folder `CLAUDE.md`** holds the real content for that area: purpose, key files, business rules, integrations, gotchas. Concise — 20 to 80 lines, not an essay.

Claude Code automatically loads `CLAUDE.md` files from the current working directory up to the repo root, so reading is free. This skill is about the discipline for **writing and maintaining** those files well.

## Install

### Personal (available in every project)

```bash
mkdir -p ~/.claude/skills/mddb
curl -o ~/.claude/skills/mddb/SKILL.md \
  https://raw.githubusercontent.com/barrinhos123/claude-skill-mddb/main/SKILL.md
```

### Project-scoped (only this repo; commits with your code)

```bash
mkdir -p .claude/skills/mddb
curl -o .claude/skills/mddb/SKILL.md \
  https://raw.githubusercontent.com/barrinhos123/claude-skill-mddb/main/SKILL.md
```

### Verify

```bash
ls ~/.claude/skills/mddb/
# SKILL.md
```

Open a Claude Code session and the skill will be available. It triggers on natural phrasing — you don't invoke it explicitly.

## Usage

Phrases that trigger the skill:

- *"Document this folder."*
- *"Add a project rule: every user-facing string must be translated."*
- *"Save what we just figured out to CLAUDE.md."*
- *"Remember this for next time."*
- *"We always use the repository pattern for DB access."*

The skill also triggers proactively: after Claude spends real effort understanding a folder, it will offer to save a `CLAUDE.md` so the next session doesn't redo the work.

## Example

You say: *"We always keep timestamps in UTC and convert at the presentation layer."*

The skill adds to the root `CLAUDE.md`:

```markdown
## Project rules
- Timestamps are stored in UTC; convert at the presentation layer only.
```

Later you say: *"Document this controllers folder."*

The skill reads the folder, writes `src/controllers/<area>/CLAUDE.md` with the folder's purpose, key files, business rules, integrations, and gotchas — then adds a pointer to the root:

```markdown
## Where to look
- <Area> → src/controllers/<area>/CLAUDE.md
```

Next session, Claude opens the project, sees the root index, and knows exactly where to look for the detail.

## What this skill does *not* do

- It doesn't read `CLAUDE.md` files — Claude Code already does that automatically. The skill is about writing them well.
- It doesn't document every folder. Only folders with meaningful logic worth persisting.
- It doesn't dump code into markdown. Notes are short: intent, invariants, gotchas. The code is right there.
- It doesn't auto-update on file changes. It triggers on your prompts.

## Contributing

Open an issue or PR. The skill is a single `SKILL.md` file — easy to hack on.

If you've used it on a real project for a while and have feedback on what was off (too verbose, too sparse, missing a section you always end up adding by hand), that's the most useful kind of issue to open.

## License

MIT — see [LICENSE](LICENSE).
