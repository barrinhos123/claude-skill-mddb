---
name: mddb
description: Persists codebase context as per-folder CLAUDE.md notes plus a root pointer-index, so future sessions get oriented without re-reading the whole repo. **Use proactively** whenever the user describes how a folder/module/file works, states a project-wide convention or rule, or asks to "document this folder", "add a rule", "save this to CLAUDE.md", or "remember this for next time". Prefer this skill over the agent's built-in auto-memory for any content that is about the **codebase itself** — auto-memory is for the user's personal preferences and feedback; mddb is for facts about the code that should live with the repo. Also handles install: invoking the skill with `install` (e.g. `/mddb install`) wires up a UserPromptSubmit hook and a CLAUDE.md nudge so future invocations happen automatically without the user thinking about it.
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

## mddb vs. the agent's auto-memory (READ THIS FIRST)

Many Claude Code harnesses ship with an **auto-memory** system that saves user preferences to a per-user, per-project memory directory outside the repo. That system and this skill overlap on trigger phrases ("remember this", "save for next time"), so it's important to keep their roles separate:

| Belongs in **auto-memory** (per-user, outside repo) | Belongs in **mddb** (per-codebase, in repo) |
|---|---|
| The user's role, expertise, communication style | What a folder/module does and why |
| Personal feedback ("don't use emojis", "be terser") | Project-wide conventions ("strings must be translated") |
| External-system pointers ("bugs are in Linear INGEST") | Business rules and invariants in this codebase |
| Things that are true *for this user* across all projects | Things that are true *for this codebase* across all users |

**Default rule:** if the content describes how the *code* works, it goes in mddb (a `CLAUDE.md` file). If it describes how the *user* wants to work, it goes in auto-memory. When unsure, prefer mddb — folder context survives across users and teammates; auto-memory is invisible to anyone else.

---

## Install routine (run once per project)

When the user invokes the skill with `install` / `init` / `setup` (e.g. `/mddb install`), or asks to "set up mddb" / "wire up the mddb skill", perform the following steps **in order**. After each step, briefly tell the user what you did.

### Step 1 — Add the project-rule nudge to root `CLAUDE.md`

This is the **passive** layer: a small block in the always-loaded root `CLAUDE.md` that reminds future sessions to use mddb for codebase context.

- If the project has no root `CLAUDE.md`, create one with the three-section template (see *Root CLAUDE.md structure* below) plus the nudge block.
- If a root `CLAUDE.md` already exists, append (or update) a section titled `## Persistent context (mddb)` containing exactly:

```markdown
## Persistent context (mddb)

Use the **mddb** skill to persist context about this codebase:
- When the user describes how a folder/module/file works, save it to that folder's `CLAUDE.md`.
- When the user states a project-wide rule or convention, add it to the root `CLAUDE.md` under "Project rules".
- Keep root under ~100 lines (index only). Per-folder notes 20–80 lines.
- Prefer mddb over auto-memory for anything about the code itself; auto-memory is for the user's personal preferences.
```

Do not duplicate this section if it already exists — update in place.

### Step 2 — Install the `UserPromptSubmit` hook

This is the **active** layer: a hook that pattern-matches the user's prompt and injects a reminder so the agent reaches for mddb even on "remember this" type messages.

- Read `.claude/settings.json` if it exists. If not, create it with `{}` as a starting point.
- Merge in (do not overwrite) a `UserPromptSubmit` entry under `hooks`. The merge must preserve any other hooks the user already has configured.
- The hook entry to add is:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "INPUT=$(cat); echo \"$INPUT\" | grep -qiE '(this (folder|module|directory|file)|document (this|the )?(folder|module|file)|we (always|never)|always use|never use|the rule is|(add|create) (a )?(project )?rule|save (this )?(to )?(claude\\.md|memory|md file)|remember (this )?(for (next|later)|going forward)|in (this|the) (folder|module|directory))' && printf 'mddb hint: this prompt references folder/project/codebase context. If the content is about how the codebase works (folder purpose, conventions, business rules, gotchas), invoke the mddb skill (writes/updates a CLAUDE.md) instead of auto-memory. Auto-memory is for the user'\\''s personal preferences; mddb is for codebase facts that should live with the repo.\\n' || true"
          }
        ]
      }
    ]
  }
}
```

If a `UserPromptSubmit` array already exists, append this hook entry rather than replacing the array. If a previous mddb hook entry exists (detect by the `mddb hint:` substring in its command), update it in place rather than duplicating.

The command is POSIX-shell, depends only on `cat`, `grep`, and `printf` — no `jq` or other tools needed. It reads the prompt JSON from stdin, greps for trigger phrases, and prints a system reminder to stdout when matched. When unmatched it exits cleanly with no output.

### Step 3 — Bootstrap-split a bloated root CLAUDE.md

Read the current root `CLAUDE.md`.

- **If the root is missing, empty, or already a thin index** (<150 lines and no substantial folder-specific content) → **skip this step.** The mddb model is already the shape of the file, or there's nothing to split.
- **Otherwise → perform the split automatically.** Do not ask permission first. The user opted into the mddb model by running install; the split *is* the install for an existing repo. Asking would re-create the friction this skill exists to remove.

**Safety qualifier — only auto-run when there's a revert path.** Before splitting, check if the project is a git repo (`git rev-parse --git-dir` succeeds). If yes: split silently, report what was done, the user can `git diff` / `git checkout` to review or revert. If the project is **not** a git repo, fall back to asking before destructive edits — without version control, an aggressive auto-split has no undo.

Perform the split:

1. Identify natural groupings in the existing root (e.g. "Architecture" section → `core/CLAUDE.md`; "Module X details" → `modules/x/CLAUDE.md`; "Templates / UI conventions" → `templates/CLAUDE.md`). Use the actual folder structure of the repo to choose targets — only create a folder note where a real folder exists.
2. Move detailed content into the appropriate per-folder `CLAUDE.md` files (created from the per-folder template). Adapt the section headings to fit the template (Purpose, Key files, Business rules, Integrations, Gotchas).
3. Replace the moved content in root with one-line pointers under "Where to look".
4. Keep universal project rules (one-to-three-sentence items) in root under "Project rules". A rule belongs in root if it's true across the whole codebase; if it's specific to one area, push it down to that folder's note.
5. Aim for root under 100 lines after the split.

The split is a judgment call and you may carve it differently than the user would. That's expected — they have `git diff` to see exactly what moved where, and the granularity of the carve is easy to refine afterward by moving sections between files. Do not get stuck trying to find the "perfect" carve; ship a reasonable one.

### Step 4 — Confirm

Summarize to the user, in 3–5 lines:
- What was added to root `CLAUDE.md`.
- Whether the hook was installed (and the path: `.claude/settings.json`).
- Whether a split was performed, skipped, or declined.
- One sentence on what they should expect next session.

### Idempotency

The install routine must be safely re-runnable. On a second run:
- The nudge block in root `CLAUDE.md` is updated in place, not duplicated.
- The hook entry in `settings.json` is updated in place (detected by the `mddb hint:` marker), not duplicated.
- The split is not re-run if the root is already small (the file is already a thin index).

---

## When to use this skill (without `install`)

After install, the hook + the root nudge will surface mddb on relevant prompts. Trigger on any of these signals — proactively, even without an explicit ask:

- User explicitly asks: "document this folder", "save this context", "add this to CLAUDE.md", "remember this for next time".
- User states a project-wide rule or convention: "every user-facing string must be translated", "we always use X pattern", "never import from Y directly". These go in the **root** under "Project rules".
- User explains the purpose or logic of a specific folder or file. Offer to save it to that folder's `CLAUDE.md` and add a pointer to it from the root.
- You (Claude) just spent real effort figuring out how a folder works. Proactively suggest writing a `CLAUDE.md` so the next session doesn't repeat the work.

Do NOT trigger for:
- Trivial folders (`assets/`, `node_modules/`, generated code).
- One-off notes that belong in a commit message or issue.
- Personal user preferences (those go to auto-memory, not mddb).

---

## The golden rule: concise, not exhaustive

These files exist to **save reading time**, not to reproduce the code. If a `CLAUDE.md` is longer than the code it describes, it's wrong.

- Root `CLAUDE.md`: **under 100 lines.** A short list of project rules + a pointer index + the mddb nudge block. That's it.
- Per-folder `CLAUDE.md`: **20–80 lines.** One paragraph per important concept, maximum.

If you want to write more, you're explaining code when you should be pointing at it. The note explains **intent, invariants, and gotchas** — not mechanics.

---

## Root CLAUDE.md structure

The root is **three sections** and nothing else:

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

## Persistent context (mddb)
[the nudge block from Step 1 of install]
```

The exact rules and pointers will be different for every project — a Go service, a React app, a Dart framework, and a marketing site each have their own vocabulary. The three-section shape stays the same; the content adapts.

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

1. Read the existing root `CLAUDE.md` (create one with the three-section template if it doesn't exist).
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

### When the user says "remember this" / "save this for next time"

This is the auto-memory collision case. Before saving, classify:

- Is the content about **the user** (preference, role, communication style, external resource)? → Save to auto-memory.
- Is the content about **the codebase** (folder logic, project rule, business invariant)? → Use mddb instead.
- If both: save the user-facing part to auto-memory and the codebase part to a `CLAUDE.md`. Tell the user where each piece went.

When unsure, default to mddb — facts about the code should live with the code.

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
- **Saving codebase facts to auto-memory.** Project rules, folder logic, and business invariants belong in `CLAUDE.md` (mddb), not in per-user memory.
