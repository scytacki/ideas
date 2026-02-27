# Sandbox Claude Code's Temporary Test Scripts

Allow Claude Code to run temporary code snippets (e.g. via `tsx`) in a sandboxed environment,
so they can be auto-approved without granting full system access.

## Problem

Claude Code sometimes generates throwaway scripts to test or exercise project code — running
them with `tsx` and a string passed via the Bash tool. Because `tsx` has full access to the
host system (filesystem, network, environment variables), blindly approving these executions
is risky. This means every invocation requires manual review and approval, which interrupts
flow and slows down iterative development.

## Goal

Find a way to run these temporary scripts in a sandbox so Claude Code can execute them freely
without requiring approval, while still giving the scripts access to the project's code and
dependencies.

## Possible approaches

- **Container-based sandbox** — Run `tsx` inside a lightweight Docker container with the
  project directory mounted read-only. Limits filesystem and network access. Could be
  wrapped in a shell script that Claude Code calls instead of raw `tsx`.
- **Claude Code hooks with allowlisting** — Use a pre-tool-call hook to inspect Bash commands
  and auto-approve only `tsx` invocations that match a safe pattern (e.g. no filesystem
  writes outside `/tmp`, no network calls). Fragile — hard to guarantee safety via pattern
  matching alone.
- **Node.js `--experimental-permission`** — Node 20+ has a permissions model
  (`--experimental-permission`) that can restrict filesystem reads/writes and child process
  spawning. Could pass these flags through to the underlying Node process that `tsx` uses.
- **Firecracker / gVisor microVM** — Heavier-weight but stronger isolation. Probably overkill
  for throwaway test scripts.
- **Deno instead of tsx** — Deno runs sandboxed by default (no filesystem/network access
  unless explicitly granted). If the project's TypeScript can be loaded by Deno (or the
  snippet is simple enough), this gives sandboxing for free. Would require Deno-compatible
  import paths.
- **Claude Code permission model enhancement** — Feature request: let Claude Code define
  per-command permission policies (e.g. "auto-approve Bash commands matching this pattern
  when running in sandbox mode"). This would be the cleanest solution but depends on
  upstream support.

## Key considerations

- The sandbox needs read access to the project's source files and `node_modules` to be
  useful — pure isolation without project access defeats the purpose.
- Write access should be heavily restricted (ideally only `/tmp` or a scratch directory).
- Network access is probably not needed for most test scripts and should be blocked by
  default.
- The solution should be easy to configure per-project and not require significant setup
  overhead.

## Open questions

- Does Node's `--experimental-permission` work reliably with `tsx` and its loader hooks?
- Could a Claude Code hook inspect the script content (not just the command) to make smarter
  approval decisions?
- Is there a way to configure Claude Code's permission model to auto-approve sandboxed
  commands without a hook?
