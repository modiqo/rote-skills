---
name: rote
description: >
  Use only when the user explicitly invokes rote or a `rote:` hook line routes the request.
  API, adapter, MCP, browser, or registry work alone does not trigger this skill.
---

# rote

Rote is a CLI. Every instruction for using it is served by the installed binary, so it always
matches the version you run. Read it with your shell tool:

```sh
rote guidance root
```

If it reports "Unknown guidance topic", the rote binary is older than this skill.
Run `rote update`, then `rote guidance root` again.

Follow that document. It starts each request with one `rote journey next` call and says what to
do with each yield. `rote guidance index` lists every document.

Run one rote command per shell call and read its whole output. After compaction or a new user
message, run `rote guidance root` again before acting.

If `rote` is not on PATH, the installer may have just placed it off PATH. Check
`$ROTE_BIN/rote`, then `$HOME/.local/bin/rote`, and run the first one that exists by its absolute
path. If neither exists, tell the user and point them at https://getrote.dev to install it.
