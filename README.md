# save-ticket

A Claude Code skill that saves the state of the ticket you're working on — status, open items,
files in play — to `.claude/tickets/<TICKET-KEY>.md`, so the next session can pick up where
this one stopped. The ticket key is read from the git branch name. One ticket = one file,
overwritten on each save.

## Install

In Claude Code:

```
/plugin marketplace add Crusaderrrr/save-ticket
/plugin install save-ticket@save-ticket
```

## Use

```
/save-ticket:save-ticket
```

or just ask Claude to "save this ticket".

On first run it adds `.claude/` to `.git/info/exclude` (local only, never `.gitignore`).
