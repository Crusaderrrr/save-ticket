---
name: save-ticket
description: >
  Persist the current work session's state into a per-ticket Markdown file under
  .claude/tickets/. Use when the user runs /save-ticket, or asks to "save this ticket",
  "save session context", "checkpoint this ticket", or similar. The skill resolves the
  active ticket key (from the git branch or by asking), then OVERWRITES a single file per
  ticket with current status, open items, and files in play. It never creates a new dated
  file per session — one ticket = one file that evolves. On first use in a repo, it also
  ensures .claude/ is locally git-excluded.
---

# Save Ticket Skill

Persist per-ticket working state so it survives across Claude Code sessions and context
resets. One ticket maps to exactly one file: `.claude/tickets/<TICKET-KEY>.md`.

## What gets saved

The file holds "how things are right now" — status, what's left to do, files currently in
play. It is fully rewritten on every save; stale state is worse than no state. The file
is not a history log — don't accumulate old status or narrate what changed session to
session. If the user explicitly asks you to record something else (a decision, a note),
just add it as a normal edit; don't build a running log for it by default.

## Step 0 — Ensure .claude/ is git-excluded (first run per repo)

Before writing anything, guarantee the `.claude/` directory is locally excluded from git —
WITHOUT touching the shared `.gitignore` (that would push the rule onto teammates).

Use `.git/info/exclude`, which is local to this clone and never committed.

1. Confirm you are inside a git repo:
   ```
   git rev-parse --is-inside-work-tree
   ```
   If not a git repo, skip this whole step and just proceed to write the file.

2. Check whether `.claude/` is already excluded (either in `.gitignore` OR `.git/info/exclude`):
   ```
   grep -qE '^\.claude/?$' .gitignore .git/info/exclude 2>/dev/null
   ```
   If it matches, skip to Step 1.

3. If not excluded, append it to `.git/info/exclude` (NOT `.gitignore`):
   ```
   printf '\n# Local Claude Code config (not shared)\n.claude/\n' >> .git/info/exclude
   ```

4. Gotcha to check: `.git/info/exclude` only ignores files git is NOT already tracking. If
   `.claude/` was ever committed, the exclude has no effect until you untrack it. Detect:
   ```
   git ls-files --error-unmatch .claude/ >/dev/null 2>&1
   ```
   If this succeeds (meaning `.claude/` IS tracked), tell the user directly rather than
   silently failing:
   > `.claude/` is already tracked by git. `.git/info/exclude` won't hide a tracked path.
   > Run `git rm -r --cached .claude/` and commit that removal to stop tracking it.
   Do NOT run `git rm` yourself — untracking is the user's call.

## Step 1 — Resolve the ticket key

The key format is `PROJECT-NUMBER` (uppercase project code, hyphen, number), e.g. `D8A-142`.
Only the key is used — no trailing description, to keep filenames short and searchable.

1. Read the current branch:
   ```
   git rev-parse --abbrev-ref HEAD
   ```
2. Extract the FIRST match of `[A-Z][A-Z0-9]+-[0-9]+` from the branch name.
   Examples: `feature/D8A-142-add-sse` → `D8A-142`; `PARSE-9` → `PARSE-9`.
3. If exactly one key is found, use it but confirm in one line:
   > Saving to ticket **D8A-142** (from branch `feature/D8A-142-add-sse`). Correct?
4. If zero keys or multiple keys are found, ASK — don't guess:
   > I couldn't read a ticket key from the branch. Which ticket is this for? (format `PROJ-123`)

Never invent a key. A wrong key silently splits one ticket's history across two files.

## Step 2 — Gather session context

Assemble the state to persist. You are working from what's visible in THIS session, so if
the session is long and early context has dropped, prefer honesty over fabrication — mark
anything uncertain rather than inventing it.

Collect:
- **Status** — one or two sentences: where this ticket stands right now.
- **Open items** — concrete remaining work. Actionable, not vague ("write @DataJpaTest for
  the repository's date-range query", not "improve tests").
- **Files in play** — files created/modified for this ticket this session. Get the raw list
  from `git status --short` and `git diff --name-only`, then keep only the ones relevant to
  THIS ticket (ignore unrelated local edits).

## Step 3 — Confirm before writing

Show the user exactly what will be written BEFORE touching the file:

```
STATE (overwrites current):
  Status: ...
  Open items: ...
  Files: ...
```

Let the user correct or trim it. The user is the final filter, not a passive recipient. Only
proceed once they confirm.

## Step 4 — Write the file

Path: `.claude/tickets/<TICKET-KEY>.md`. Create `.claude/tickets/` if missing.

### If the file does NOT exist — create it from this template:

```markdown
# <TICKET-KEY>

**Status:** <one or two sentences>

**Last updated:** <YYYY-MM-DD>

## Open items
- [ ] <actionable item>

## Files in play
- `<path>` — <what/why>
```

### If the file DOES exist:

Overwrite the entire file with the refreshed Status, Last updated, Open items, and Files in
play. This is a full rewrite, not a merge — don't carry forward stale open items that are
now done, and don't preserve old status text.

## Step 5 — Report back

One line confirming what happened, e.g.:
> Updated `.claude/tickets/D8A-142.md` — refreshed state.

Do not re-print the whole file. The user can open it.

## Edge cases

- **No git repo:** skip Step 0, still write the file. Warn once that `.claude/` isn't
  git-excluded because there's no repo here.
- **Detached HEAD / no branch key:** fall back to asking (Step 1.4).
- **Nothing worth saving:** if nothing changed since the last save, say so instead of
  writing an empty/identical update. Don't create noise.
- **Multiple tickets touched in one session:** ask which ticket this save is for. One save =
  one ticket. Suggest running `/save-ticket` again for the other.
