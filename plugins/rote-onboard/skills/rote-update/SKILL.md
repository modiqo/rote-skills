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

Two layers most people forget to update together: the **binary** and the **installed skill
files**. This skill does both. Determine facts from live commands; if rote isn't on
PATH, resolve it via the **narrow probe** (check `$HOME/.local/bin/rote` then
`$HOME/.cargo/bin/rote` — never a deep `find`; see INDEX § 1b). **Follow the shared operating
rules in
[`../INDEX.md`](../INDEX.md) § "Shared operating rules"** — clear command access if the current
environment requires it, and run **one command at a time, sequentially — never parallel**.

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
rote install skill --force
```

If the user installed skills for a specific provider, preserve that provider choice:

```bash
rote install skill --provider <provider> --force
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
keep paying as it ages. Shared convention and rules in [INDEX.md](../INDEX.md). Skip it if
`self-update` errored or deferred to a package manager. e.g. "Updated to {version} — your
adapters still hit the real APIs directly, no proxy clipping a fee per call. Carry on."

**Related onboard skills** ([INDEX.md](../INDEX.md) is the full map):
- **First-run:** `rote-setup` · **New adapter:**
  `rote-adapter-create` · **Tune one:** `rote-adapter-config`
