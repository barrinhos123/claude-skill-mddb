---
name: mddb
description: Maintains a lightweight "markdown database" of context notes across a codebase. The root CLAUDE.md acts as an index of pointers to per-folder CLAUDE.md files, plus a short list of project-wide rules. Use this skill whenever the user asks to "document this folder", "save what we learned", "add a rule", "remember this for next time", or describes the purpose/logic of a file or folder in a way worth persisting. Also use it proactively when the user states a project-wide convention ("every string must be translated", "always use the repository pattern") — those go in the root. The goal is that next time Claude opens the project, it reads the root index, jumps to the relevant folder note, and understands what it needs without reading the whole codebase.
---

# mddb

A skill for maintaining a small, pointer-based "markdown database" of context for a codebase.

Claude Code automatically loads `CLAUDE.md` files from the working directory up to the repo root. This skill is about **writing and maintaining** those files so that future sessions get the right context without reading the whole repo.

The model is deliberately simple:

1. **Root `CLAUDE.md`** — an **index** of pointers plus a short list of project-wide rules. It does not contain detailed logic.
2. **Per-folder `CLAUDE.md`** — the actual content: what this folder does, the business logic, the gotchas. Lives next to the code it describes.

Think of the root as a table of contents. When Claude needs to know how a particular area of the project works, the root tells it which folder note to open.

This applies to any kind of repo — application code, infrastructure, config, content, marketing assets, research notes. Anywhere a folder has meaningful context worth persisting, it can have a `CLAUDE.md`.

---

## When to use this skill

Trigger on any of these signals:

- User explicitly asks: "document this folder", "save this context", "add this to CLAUDE.md", "remember this for next time".
- User states a project-wide rule or convention: "every user-facing string must be translated", "we always use X pattern", "never import from Y directly". These go in the **root** under "Project rules".
- User explains the purpose or logic of a specific folder or file. Offer to save it to that folder's `CLAUDE.md` and add a pointer to it from the root.
- You (Claude) just spent real effort figuring out how a folder works. Proactively suggest writing a `CLAUDE.md` so the next session doesn't repeat the work.

Do NOT trigger for:
- Trivial folders (`assets/`, `node_modules/`, generated code).
- One-off notes that belong in a commit message or issue.

---

## The golden rule: concise, not exhaustive

These files exist to **save reading time**, not to reproduce the code. If a `CLAUDE.md` is longer than the code it describes, it's wrong.

- Root `CLAUDE.md`: **under 100 lines.** A short list of project rules + a pointer index. That's it.
- Per-folder `CLAUDE.md`: **20–80 lines.** One paragraph per important concept, maximum.

If you want to write more, you're explaining code when you should be pointing at it. The note explains **intent, invariants, and gotchas** — not mechanics.

---

## Root CLAUDE.md structure

The root is **two sections** and nothing else:

```markdown
# Project context

## Project rules
- Use `pnpm`, not `npm`.
- Every user-facing string goes through the i18n helper. No hardcoded copy.
- Data access goes through a repository/service layer. No raw queries in controllers or views.
- Timestamps are stored in UTC; convert at the presentation layer only.

## Where to look
- <Domain area A> → `<path>/CLAUDE.md`
- <Domain area B> → `<path>/CLAUDE.md`
- <Cross-cutting concern, e.g. auth, payments, i18n> → `<path>/CLAUDE.md`
- <Infrastructure, e.g. deploy scripts, IaC> → `<path>/CLAUDE.md`
```

The exact rules and pointers will be different for every project — a Go service, a React app, a Dart framework, and a marketing site each have their own vocabulary. The two-section shape stays the same; the content adapts.

### Rules for "Project rules"

- One to three sentences per rule.
- Universal — if it's not true across the whole project, it doesn't go here.
- No duplicates. Before adding, always read the existing file and check.
- No folder-specific content. "Bookings use a 10-minute hold" is **not** a project rule; it lives in `booking/CLAUDE.md`.

### Rules for "Where to look"

- One line per pointer: `<topic> → <path to CLAUDE.md>`.
- Sorted by domain, not by filesystem order.
- Only add a pointer when the folder actually has a `CLAUDE.md`. Don't pre-declare.
- If a folder's `CLAUDE.md` is deleted, remove the pointer.

---

## Per-folder CLAUDE.md

This is where the real content lives.

### What to include

- **Purpose** — one sentence. "Controllers for the public booking flow."
- **Key files** — one line each, only the important ones.
- **Business rules and invariants** — the stuff a code reader would miss.
- **Integration points** — external systems, sibling folders, events.
- **Gotchas** — non-obvious pitfalls.

### What to leave out

- Full code or full function signatures.
- Language/framework tutorials.
- Every file listed by name (only the ones that matter).
- Project-wide rules already in the root.

### Template

```markdown
# <folder-name>

<One sentence: what this folder is for.>

## Key files
- `<file>` — <what it does, one line>.
- `<file>` — <what it does, one line>.
- `<subfolder>/` — <role in this area>.

## Business rules
- <An invariant or rule a code reader would miss.>
- <A constraint enforced elsewhere that matters here — e.g. a DB constraint, a queue ordering guarantee.>
- <A state-machine or lifecycle note if relevant.>

## Integrations
- <Event this folder emits or consumes.>
- <Sibling folder this depends on — and what *not* to duplicate from there.>
- <External service, if any.>

## Gotchas
- <Non-obvious pitfall — the thing that caused a bug once.>
- <A "do not call X directly, go through Y" rule.>
```

Adapt the sections to the folder. A `utils/` folder might only need "Purpose" and "Key files". A domain folder typically needs all five. A folder wrapping an external SDK might just need "Purpose", "Integrations", and "Gotchas".

### A worked example (for reference — yours will look different)

For a controllers folder in a reservation-style app, a filled-in note might look like:

```markdown
# reservations

HTTP controllers for the public reservation flow.

## Key files
- `ReservationController` — handles create/cancel requests; enforces the hold window.
- `ReservationService` — the only place that should mutate reservation state.
- `validators/` — request shape and availability checks.

## Business rules
- New reservations hold the resource for 10 minutes before auto-releasing.
- Overlapping reservations for the same slot are rejected at the DB level.
- A reservation can exist in `pending` without a charge; payment is deferred.

## Integrations
- Publishes `reservation.confirmed` and `reservation.cancelled` to the event bus.
- Reads availability from `inventory/` — do not duplicate that logic here.

## Gotchas
- Never mutate a reservation directly; go through `ReservationService.commit()` so the audit log fires.
```

The shape is what matters — purpose, key files, rules, integrations, gotchas. Plug in whatever your project actually does.

---

## Workflow

### When the user asks to document a folder

1. Read the folder contents to understand what's there.
2. Check if a `CLAUDE.md` already exists in that folder.
   - If yes: read it, update only what's wrong or missing. Prefer surgical edits.
   - If no: write one from the template, adapted to the folder.
3. Keep it in the 20–80 line range. Cut aggressively.
4. **Update the root `CLAUDE.md`**: add a pointer under "Where to look" if one doesn't exist. Sort into the index sensibly.
5. Confirm with the user what you added.

### When the user states a project-wide rule

1. Read the existing root `CLAUDE.md` (create one with the two-section template if it doesn't exist).
2. Check for duplicates or near-duplicates under "Project rules".
   - Same rule → do nothing.
   - Clearer wording → replace, don't append.
   - Related but different → add near the existing one.
3. Add the rule in one to three sentences under "Project rules".
4. Tell the user what you added and where.

### When the user explains folder-specific logic during a task

1. Ask: "This seems worth saving to `<path>/CLAUDE.md`. Want me to?"
2. If yes: write the folder note and add a pointer to the root.
3. Do not silently write files unless the user has said "document as you work".

### When you (Claude) figured something out the hard way

Same as above — pause and propose a save. The point of this skill is that **the next session shouldn't have to do the same work again**.

---

## Updating vs. rewriting

Prefer **surgical updates** over rewrites. If a `CLAUDE.md` already exists and is mostly right, change the line that's wrong — do not regenerate. Rewrites lose context the user added by hand.

Full rewrite is appropriate only when:
- The folder has been substantially restructured.
- The existing note is clearly stale or misleading.
- The user asks for a rewrite.

---

## Anti-patterns to avoid

- **Putting folder-specific detail in the root.** The root is an index. If you're writing a paragraph about how bookings work, it belongs in `booking/CLAUDE.md`, not the root.
- **Pointing to folders that have no `CLAUDE.md` yet.** Pointers in the root must resolve. Add the folder note first, then the pointer.
- **Listing every file in a folder.** Only the important ones. A ten-file folder has two or three worth naming.
- **Explaining the language or framework.** Assume the reader knows the stack. Don't explain what a hook is, or what a goroutine is, or what a decorator does.
- **Restating function signatures.** The code is right there.
- **Vague filler.** "This folder contains various utilities." — delete and start over.
- **Duplicating root rules in folder notes.** If it's in the root, don't repeat it.
- **Journal entries.** "I refactored this today" does not belong. These are reference docs, not changelogs.
