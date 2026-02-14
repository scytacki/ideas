# Keep Git Commit Messages Updated During Claude Code Sessions

Get Claude Code to maintain a running commit message summary as it works, so you always have
a persistent, readable overview of what changed.

## Problem

When Claude Code makes changes across multiple files, there's no persistent summary of what
it's done until you ask it to commit. The conversation scrolls, and it's easy to lose track
of the full scope of changes before reviewing them.

Interestingly, when Claude uses `git cherry-pick <sha> --no-commit`, the commit message is
pre-populated in the VSCode source control UI. This is really handy — you get a persistent,
at-a-glance summary of the changes right in the sidebar without having to scroll through the
conversation. The question is how to get this benefit more broadly during normal Claude Code
sessions.

## Possible approaches

- **Hook into Claude Code's tool calls** — Use a Claude Code hook (e.g. post-tool-call) to
  detect file edits and automatically stage + update a draft commit message in
  `.git/MERGE_MSG` or `.git/SQUASH_MSG`, which VSCode's source control UI reads and displays.
- **Periodic `git commit --amend` with `--no-verify`** — Have Claude periodically commit and
  amend with an updated message as it works. Downsides: pollutes history if not squashed,
  and amending can be surprising.
- **Use `git stash` messages** — Less natural, but `git stash` messages show up in the UI.
- **Write to a scratch file** — Claude could maintain a `CHANGES.md` or similar file with a
  running summary. Lower-tech but visible in the editor. Downside: it's another file to
  clean up.
- **Custom VSCode extension** — A small extension that watches Claude Code's output and
  extracts/displays a summary in a panel or the SCM view.

## Key insight

The VSCode git UI already knows how to display pre-populated commit messages from
`.git/MERGE_MSG` and `.git/SQUASH_MSG`. If Claude Code (or a hook) writes to one of these
files as it works, the summary would appear automatically in the source control sidebar —
no extension needed.

## Open questions

- Which `.git/*_MSG` file is safest to write to without interfering with actual git
  operations? `SQUASH_MSG` might be the least likely to conflict.
- Can a Claude Code hook reliably summarize changes incrementally, or does it need to
  re-summarize from scratch each time?
- Should this be a Claude Code skill/hook, a standalone tool, or a feature request to
  Claude Code itself?
