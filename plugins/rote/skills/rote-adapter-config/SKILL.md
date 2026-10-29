---
name: rote-adapter-config
description: >
  Tune an existing rote adapter — auth, base URL, write guard, sensitivity, explore search data,
  OAuth re-auth, GraphQL field filter, policies, adapter guidance, grouping, versioning. A menu
  of small show → confirm → apply → re-show operations on an adapter you already created. Use
  when the user says "configure <adapter>", "change the base url", "re-auth notion", "rebuild
  the explore search data", "enable an auth scheme", or "set up sensitivity / write guard". For
  creating a NEW adapter, use rote-adapter-create instead.
---

# rote-adapter-config — tune an existing adapter

Iterative configuration of an adapter that already exists. Unlike creation (a linear
dry-run-first pipeline), this is a **menu of discrete operations**, each a small loop:
**show current state → confirm the change → apply → re-show**. Return to the menu after each.

`rote-adapter-config` is this skill's name, not a CLI command — there is no
`rote adapter config` subcommand. Every operation below runs through the specific `rote adapter …`
/ `rote token …` / `rote oauth …` commands shown.

When this skill is invoked from a failed workspace, play, or adapter call, it is a repair subroutine.
It must return to the owner after the config change; it does not complete the user's task by itself.
Repair is not complete until the owner reruns the failed probe/call or play gate.

Rules:

- Determine facts from live commands, never memory.
- Secrets never appear in chat. Use the **masked terminal handoff**: ask the user to run
  `rote token set <ENV> --stdin` themselves in their terminal and paste the value there, then
  continue the session once they confirm — never echo or capture the value in conversation.
- If rote isn't on PATH, resolve it via the **narrow probe** (check `$HOME/.local/bin/rote` then
  `$HOME/.cargo/bin/rote` — never a deep home-directory search).
- If the environment gates shell commands behind an approval prompt or allowlist, get the `rote`
  command pattern approved up front rather than stalling mid-loop.
- Run **one command at a time, strictly sequential — never parallel**.
- Use the base `rote` companion skill only when the caller needs the full workflow graph or return
  contract.

## Handoff Contract

- Use when: the user wants to tune an existing adapter's auth, base URL, write guard, sensitivity,
  OAuth session, GraphQL field filter, explore search data, policies, grouping, or version.
- Preconditions: the adapter already exists; the requested operation can be mapped to a live `rote`
  command; secrets will be handled only through masked terminal handoff or explicit opt-in.
- Owns: the show -> confirm -> apply -> re-show loop for one adapter configuration operation at a
  time, including honest limits when rote has no first-class command.
- Hands off to: `rote-adapter-create` when the requested change requires recreating the adapter;
  `rote` for day-to-day use after tuning; `rote-troubleshooting` if repeated unchanged failures
  prevent a configuration result.
- Returns to: `rote-adapter-create` or `rote` with adapter id, setting changed, command run,
  verification output, skipped operation, owner skill, failed command to rerun, and any credential
  or browser action still pending.
- Stop when: the requested setting is verified, the user declines or postpones the change, a missing
  credential/browser action blocks the operation, or adapter recreation is the correct path.
- Completion signal: affected state re-shown after each applied change, or a clear no-op/blocker
  reason plus next recommended skill.

---

## Open — show current state

Identify the adapter and show what's there now. Every command below takes the release reference
that `rote adapter list` prints (`<publisher>/<name>@<version>` or `@development/<name>`); the
version is always required.

```bash
rote adapter list
```
```bash
rote adapter info <release>
```
```bash
rote adapter keys <release> --json
```

Then present the operation menu. Run only what the user picks; re-show the relevant state after
each change.

---

## Auth Shape Matrix

Classify auth before changing credentials. Existing adapters may store OAuth tokens behind a
bearer env var, so never treat "Bearer" alone as proof of a static token.

| Shape | How to recognize | Correct repair |
|------|------------------|----------------|
| Static bearer/API key | Health shows missing static env var or user is rotating a known API key | Use masked terminal handoff or `rote token set <ENV> --stdin`; never echo the value. |
| OAuth client id/secret | Scheme metadata is OAuth2 or OpenAPI `oauth2_schemes` is present | From a terminal, run `rote adapter reauth <release>` for an installed single-scheme adapter, or `rote adapter reauth <release> --scheme <scheme>` for a multi-scheme adapter's OAuth2 scheme. Use `rote adapter auth scheme add <release> <scheme> [descriptor] --oauth2` when adding a scheme. On a scheme declaring both authorization-code and client-credentials, reauth re-runs the held grant's flow, or authorization-code when none is held. If `rote adapter auth scheme list` shows `oauth2 (authorization_code)` for a scheme holding a client-credentials grant, the adapter predates two-flow support; recreate it. |
| OAuth DCR / MCP PRM | Token metadata says OAuth/DCR, MCP auth redirects, or provider registration was dynamic | Use `rote adapter reauth <release> [--scheme <name>]`; use `rote adapter reauth <release> --force-reregister` only when the provider pruned the dynamic client. |
| Google Discovery | Google API adapter, or the manifest declares `google_discovery` setup | Gmail, Google Calendar, and Google Drive: install the managed release (`modiqo-hosted/gmail`, `modiqo-hosted/google-calendar`, `modiqo-hosted/google-drive`) and follow the Managed OAuth app row. Any other Google Discovery adapter: `rote oauth setup google --adapter <release> --scopes ...`. Never ask for a pasted bearer token. |
| Managed OAuth app | The manifest declares `managed_oauth` setup with an `app` and `scopes` | Requires `rote login`. Use `rote adapter reauth <release>` for both first acquisition and repair; the OAuth broker runs browser consent. Do not ask for a client id/secret or a pasted token. A manifest may name a third-party broker with `"broker": "<id>"` (absent means `modiqo`). If the user has not trusted that broker, reauth fails and prints `rote oauth broker trust <broker> <url>`. Only the user decides to trust a broker operator, because that broker sees every token it issues; never run `rote oauth broker trust` on your own. Third-party brokers do not need `rote login`. |
| Unknown bearer | Installed adapter reports bearer but provenance is unclear | Run `rote adapter list <release> --json --health` before advising token set vs reauth. |

---

## Operations (all verified against the live CLI)

### Settable keys — base URL, name, description, tags, sensitivity tier

```bash
rote adapter set <release> base_url <url>
```
Other settable keys (same command): `name`, `description`, `tags` (csv),
`sensitivity_tier` (`low|medium|high`), `additional_headers.<name>` (`""` deletes),
`enable_parameter_cleaning` (bool). Check current values first with `rote adapter keys <release>`.
A registry release refuses `name`, `description`, `tags`, and `bump`, and keeps the other
settings in its config overlay (`rote guidance adapters essential`). To change a refused field,
fork the release into an editable development adapter, then re-show `rote adapter info` for the
returned reference and edit that:

```bash
rote adapter fork <publisher>/<name>@<version> [name]   # -> @development/<name>; name defaults to the release's
```
The fork's manifest takes the release's settings, `info` shows `Forked from:`, and the release is
left unchanged. A fork under another name records its own fingerprint, so only the release's
bundled default policy applies to it; re-add other policies with `rote policy add`. Fork refuses
when `@development/<name>` already exists; pass another name.

### Auth — single scheme

```bash
rote adapter auth update <release> --bearer-token <ENV_VAR>
```
Variants: `--api-key-header <name=ENV>`, `--api-key-query <name=ENV>`, `--basic <USER_ENV>`
(password read from `<USER_ENV>_PASSWORD`, or `--basic <USER_ENV>=<PASS_ENV>` to name both),
`--none` (remove auth).
`rote adapter auth scheme list <release>` also works here: it shows the spec's declared schemes,
whether this binding carries them, and how each OAuth2 credential is acquired on this
machine. The other `scheme` subcommands are multi-scheme only.
The token **value** is a secret — set it via the masked `rote powerpack credentials` (user's
own terminal) or `rote token set <ENV> --stdin` as an explicit terminal opt-in. Never echo a
token.

### Auth — multi-scheme (per-operation adapters)

```bash
rote adapter auth scheme list <release>
```
```bash
rote adapter auth scheme enable <release> <scheme>      # idempotent; descriptor optional
rote adapter auth scheme add <release> <scheme> [descriptor]   # (re-)binds + runs credential flow
rote adapter auth scheme disable <release> <scheme> [--force]  # --force to disable the default
```

### Re-authorize OAuth

```bash
rote adapter reauth <release> [--scheme <name>] [--access read-write | --scope <s>...] [--force-reregister]
```
For an installed MCP adapter with a missing DCR credential, this registers and authorizes in place;
it does not recreate the adapter. `--force-reregister` re-runs DCR when the provider pruned the
client (rare). `--scope` (repeatable) adds scopes to a DCR, spec-declared OAuth2, or
client-credentials grant's requested list; managed apps refuse it and use `--access`, and Google
Discovery setups refuse it. A scope the result lists in `not_granted` was declined by the
provider; do not repeat `--scope` for it. A call refused with `kind: missing_scopes` sent no
request; run its `@@next` command, then retry. Without `@@next`, its `fix` field says why no reauth
grants the scopes; report it and do not retry.

### Write guard

```bash
rote adapter guard show <release>
rote adapter guard init <release>
```
`init` generates `write_guard.json` from the adapter's tools (allow/confirm/deny gradient).
**There is no first-class "disable" command** — say so; disabling means removing the guard
files manually.

### Sensitivity classification

```bash
rote sensitivity upgrade            # download/update the index (first time)
rote sensitivity apply <release> --json  # classify this adapter's fields
rote sensitivity lookup <release> --json # show current classification
```

### Explore search data

`rote explore` reads each adapter's tool index and the IDF cache; `status` reports which are
missing and the fix for each. `rebuild` rebuilds the IDF cache and development tool indexes; a
registry release keeps the tool index it shipped with, so restore it with
`rote registry adapter pull <release>`.

```bash
rote adapter capability status
rote adapter capability rebuild
```

### GraphQL field filter (GraphQL adapters only)

```bash
rote adapter tool-spec <release> --show
```
Then `--add` / `--rm <INDEX>` / `--action <skip|keep>` with `--tool`/`--type`/`--field`/
`--desc-prefix`/`--desc-contains` to scope which fields the selection set requests. Skip
entirely for non-GraphQL adapters.

### Policies (rate limit, cost, retry, cache, circuit breaker)

```bash
rote adapter policies <release>
```
Generate from a preset: `rote adapter policies <release> --generate --preset <github|conservative|custom>`.

### Group / version

```bash
rote adapter group set <release> <group>     # also: group unset <release>, group list
rote adapter bump <release> [--minor|--major]
```
`bump` takes a development adapter only; a registry release's version is fixed as published. Fork
it first (`rote adapter fork <release> [name]`) and bump the returned `@development/<name>`.

---

## Honest limits — state these when relevant

- **No post-creation toolset toggle.** Toolset selection is create-time only (via the
  dry-run + `toolset_filters` in rote-adapter-create). To change which toolsets are active,
  recreate the adapter, or — for GraphQL — use `tool-spec` for field-level filtering. Don't
  offer a toolset-toggle command; it doesn't exist.
- **No write-guard disable command** — only `init` / `show`.

---

## Pattern (per operation)

1. **Show** current value (`info` / `keys` / `guard show` / `auth scheme list` / `tool-spec --show`).
2. **Confirm** the change with the user.
3. **Apply** the verified command.
4. **Re-show** the affected state so the user sees the effect.
5. If this was a repair, return to the owning skill with the exact failed command or gate to rerun.
6. Return to the menu, or exit.

## Repair Return Packet

When called because an adapter probe/call, play run, or workspace gate failed, return:

- Owner skill and workspace, if known.
- Adapter id and setting changed.
- Failed command or gate that triggered the repair.
- Verification command that re-showed the changed setting.
- Exact next command to rerun in the owner.

Do not summarize success to the user until the owner reruns the failed command and resumes artifact
composition. Updating auth, a base URL, or keys is only a changed precondition.

---

## Closing line + related skills

**Closing line** (optional, on a clean exit): one dry one-liner keyed to whatever was just
tuned — still the provider's own API, still no proxy taking a cut per call. Keep it lower-key than
setup or create — config is housekeeping, so don't force it; skip it if a step errored.

**Related skills:**
- **New adapter:** `rote-adapter-create`
