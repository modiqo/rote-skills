---
name: rote-update
description: >
  Update rote to the latest version — both the binary and the skill distribution. Runs
  `rote self-update` and refreshes generated adapter guidance templates. Use when the user says
  "update rote", "upgrade rote", "is there a new rote version", or asks to refresh the shipped
  skills. Quick and guided — covers the binary and generated guidance people forget to refresh
  together.
---

# rote-update — keep rote current

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Two layers most people forget to update together: the **binary** and the **installed skill
files**. This skill does both. Determine facts from live commands; if rote isn't on
PATH, resolve it via the **narrow probe** (check `$HOME/.local/bin/rote` then
`$HOME/.cargo/bin/rote` — never a deep home-directory search). Clear command access if the current
environment requires it, and run **one command at a time, sequentially — never parallel**. Use the
base `rote` companion skill only when a caller needs the full workflow graph.

## Handoff Contract

- Use when: the user wants to check, update, upgrade, or refresh rote, including the binary and the
  bundled skill distribution.
- Preconditions: `rote` is runnable or the install/path/package-manager blocker can be stated; the
  user accepts that long-lived agent sessions may need restart after skill refresh.
- Owns: update availability check, self-update invocation, version confirmation, bundled skill
  refresh, provider-specific skill refresh guidance, and restart/manual-refresh advice.
- Hands off to: `rote-setup` if update reveals a broken or incomplete install; `rote` after the
  refreshed binary/skills are ready for normal use.
- Returns to: the caller with old/new version when known, update result, skill refresh command run,
  provider target, restart requirement, and any package-manager/manual-update blocker.
- Stop when: update/check completes, self-update defers to another installer, skill refresh finishes,
  or restart/manual action is required before active skills can reload.
- Completion signal: version and skill-refresh status reported from live commands, plus explicit
  restart or next-skill guidance.

---

## 1. Check first (optional)

```bash
rote self-update --check
```
Shows whether a newer version is available without installing. If already current, say so and
skip to step 3 (the skills can still drift).

## 2. Update the binary

```bash
rote self-update --yes
```
`--yes` skips the confirmation prompt, which is safer for non-interactive command runners. After
it completes, confirm:

```bash
rote --version
```

## 3. Refresh the shipped skills

rote ships the skills as part of its own distribution. After the binary update, refresh the
installed skill copy through rote instead of assuming an external installer:

```bash
rote install skill --package '*' --force
```

If the user installed skills for a specific provider, preserve that provider choice:

```bash
rote install skill --provider <provider> --package '*' --force
```

Use `rote install skill --help` when the supported provider names are unclear. If the current
agent session cached an older skill copy, tell the user to restart that session so the refreshed
files are reloaded.

---

## Notes

- `rote self-update` updates the `rote` binary in place (typically `~/.local/bin/rote`).
- If `self-update` reports the binary was installed via a package manager (brew) or an editor
  extension, follow its guidance — it may defer to that installer rather than self-replacing.
- The binary and installed skill files can drift when a long-lived session has an older copy
  loaded; refresh the installed skills after updating the binary.

---

## Closing line + related skills

**Closing line** (only after a clean update of both layers): one dry one-liner keyed to the
version just landed on — still a local binary hitting providers directly, no middleman proxy to
keep paying as it ages. Skip it if `self-update` errored or deferred to a package manager. e.g. "Updated to {version} — your
adapters still hit the real APIs directly, no proxy clipping a fee per call. Carry on."

**Related setup skills:**
- **First-run:** `rote-setup` · **New adapter:**
  `rote-adapter-create` · **Tune one:** `rote-adapter-config`
