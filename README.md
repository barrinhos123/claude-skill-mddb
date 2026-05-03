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

The skill is two steps: drop the file in, then run it once per project to wire up automation.

### 1. Drop the skill in

**Personal (available in every project):**

```bash
mkdir -p ~/.claude/skills/mddb
curl -o ~/.claude/skills/mddb/SKILL.md \
  https://raw.githubusercontent.com/barrinhos123/claude-skill-mddb/main/SKILL.md
```

**Project-scoped (only this repo; commits with your code):**

```bash
mkdir -p .claude/skills/mddb
curl -o .claude/skills/mddb/SKILL.md \
  https://raw.githubusercontent.com/barrinhos123/claude-skill-mddb/main/SKILL.md
```

### 2. Run the install routine once per project

Open a Claude Code session in the repo and run:

```
/mddb install
```

This wires up two things so the skill triggers itself going forward:

1. **A nudge block in your root `CLAUDE.md`** — always loaded by Claude Code, every conversation. Tells future sessions to prefer `mddb` over auto-memory for codebase context.
2. **A `UserPromptSubmit` hook in `.claude/settings.json`** — pattern-matches your prompts (`"this folder…"`, `"we always…"`, `"add a rule…"`, `"remember this…"`) and injects a system reminder so the skill kicks in even on phrasings that would otherwise hit the agent's built-in auto-memory first.

After this, you don't think about it. Talk normally about the codebase and the skill activates on its own.

The install is idempotent — running it again updates the existing nudge/hook in place rather than duplicating. If your root `CLAUDE.md` is already bloated (>150 lines, full of folder-specific detail), `/mddb install` will also offer a one-time split into per-folder notes — but only with your explicit yes.

## How it triggers (after install)

You don't invoke the skill explicitly. Phrases that wake it up via the hook + the root nudge:

- *"Document this folder."*
- *"Add a project rule: every user-facing string must be translated."*
- *"Save what we just figured out to CLAUDE.md."*
- *"We always use the repository pattern for DB access."*
- *"In this module we cache tokens for 5 minutes."*

It also triggers proactively: after Claude spends real effort understanding a folder, it will offer to save a `CLAUDE.md` so the next session doesn't redo the work.

### `mddb` vs. auto-memory

Many Claude Code harnesses ship with an auto-memory system that saves user preferences to a per-user directory outside your repo. `mddb` and auto-memory overlap on phrases like "remember this" — so the skill keeps a clear split:

| Auto-memory (per-user) | mddb (per-codebase) |
|---|---|
| Your role, expertise, communication style | What a folder/module does and why |
| Personal feedback ("don't use emojis") | Project conventions ("strings must be translated") |
| External-system pointers ("bugs are in Linear") | Business rules and invariants in this repo |

Default rule: if it's about the **code**, it goes in `mddb`. If it's about **you**, auto-memory.

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
