---
name: rote-setup
description: >
  Guided, interactive first-run setup for rote. Installs the binary, signs the user in via
  Google/GitHub (every experience is identity-gated). Then forks: stop at just the CLI, pull
  curated powerpack adapters, or build adapters from the built-in catalog. Then offers a menu
  of remaining onboarding steps (adapters, credentials, OAuth, install skill, explore) and
  ends by running one live play to prove value. Use when the user says "set up rote", "rote
  setup", "onboard me to rote",
  "get rote working", "install and configure rote", or is a first-timer
  who needs hand-holding through `rote login`, API connection, and installing adapters.
  This skill drives the human through choices at each branch —
  it does NOT silently run the whole one-liner.
---

# rote-setup — Guided onboarding wizard

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Make installation feel like a breeze. You are the wizard: detect state, present clear
choices, run one step at a time, confirm, move on. Lead with the happy path: sign in,
then let power users connect APIs directly or take the step-by-step live tour.

Ask the user for every branch below. Run **one command at a time** unless a step explicitly
documents a single logical compound. After each step, give a one-line "what just happened" and
surface the next choice.

## Operating Rules

This skill is self-contained. Use the base `rote` companion skill only when you need the full
workflow graph or packet shape.

First, clear command access if the current environment requires it. The skills need to run `rote`
and enter paths under the active rote home. Do not assume a particular environment, permission file,
or allowlist syntax.

Prefer the exact workspace path printed by rote. If you must construct the path, use
`${ROTE_HOME:-$HOME/.rote}/rote/workspaces/<name>` rather than assuming `~/.rote`.

All commands are non-destructive to the filesystem; they touch `~/.rote/` config and the
registry.

**Self-contained — never rely on memory or prior session state.** This skill must work on a
fresh machine for a user the agent has never seen. Do **not** consult project memory,
recalled facts, or any "rote is active here" assumption to decide whether rote is
installed, where it lives, or whether the user is logged in. Determine every fact by running
the probe commands in the steps below and branching on their **actual output** — memory may
be stale, machine-specific, or simply wrong. If a recalled note contradicts a live probe,
the live probe wins.

## Handoff Contract

- Use when: the user needs guided first-run setup, installation repair, login, starter adapters,
  credentials, skill installation, or a first value-proof play.
- Preconditions: command access for `rote` can be requested or the binary/install blocker can be
  stated; every setup fact will be determined from live probes in this run.
- Owns: binary/state probing, install branch selection, login gate, starter adapter and credential
  setup, agent skill installation, and optional first value-proof play.
- Hands off to: `rote-adapter-create` for building a new adapter; `rote-adapter-config` for tuning an
  existing adapter; `rote-registry` when a newly useful artifact should be shared; `rote-update` when
  an existing binary should be refreshed before setup continues; `rote` for day-to-day use after
  onboarding.
- Returns to: `rote` with installed binary path, signed-in identity, installed adapters, credential
  state, skill-install target, value-proof result, and any skipped or blocked setup steps.
- Stop when: the user chooses to stop at CLI-only setup, a required browser/credential/human action is
  pending, a command fails and needs a branch decision, or the value-proof play completes.
- Completion signal: setup state summarized from live command output, next owner named, and no
  unverified credential or play proof presented as successful.

---

## Step -1 — Pre-flight gate: binary × state (the 2×2)

Determine this **only** from live probes — never from memory or a "rote is active here"
note. **Probe sequentially, one command at a time — never in parallel.** Each probe gates
the next; firing them as a batch (e.g. `rote whoami` alongside the `command -v rote` that's
supposed to decide whether `rote` even exists) is incoherent and wastes tokens. Do not run any
`rote <subcommand>` until the binary probe has resolved.

**Probe A — is the binary present?** (two sub-checks, in order, stop at the first hit)

1. ```bash
   command -v rote
   ```
   Prints a path → installed, invoke as `rote` for all later commands.

2. Only if A.1 was empty: it may be installed but off this shell's PATH (the installer
   commonly drops it in `~/.local/bin`, which a non-interactive shell omits):
   ```bash
   ls -la "$HOME/.local/bin/rote" "$HOME/.cargo/bin/rote" 2>/dev/null
   ```
   - Either path exists → installed, off PATH. Use the **absolute path** for every later
     `rote` command, and tell the user their shell PATH is missing `~/.local/bin`.
   - Neither exists **and** A.1 empty → **binary absent**.

**Probe B — is there existing state?** One call:
```bash
test -d "$HOME/.rote" && echo "STATE" || echo "CLEAN"
```
`~/.rote` holds adapters, tokens, plays, workspaces — its presence means a prior install left
state behind.

**Branch on the 2×2** (this is the whole point — most cells skip the deep probing):

| Binary | `~/.rote` | Meaning | Action |
|:---:|:---:|---|---|
| ✗ | ✗ | **Clean slate** | Skip all state probing — there's nothing to probe. Go straight to the **install** choice below, then Step 0 (login). |
| ✓ | ✗ | **Binary, no state** | Ask if the binary should be updated first (`rote self-update`, or defer to **rote-update**), then begin setup fresh at Step 0. No `whoami` needed — there's no session to detect. |
| ✗ | ✓ | **Orphaned state** | The binary's gone but state remains (possibly stale / version-incompatible). **Propose backing up `~/.rote` and starting clean** — default to back-up-then-clean: move it aside, then install fresh. Confirm before moving anything. |
| ✓ | ✓ | **Existing install** | Only this cell justifies the deeper probing. Proceed to **Step 0 (login)** and branch on `rote whoami`'s real output. |

For the **orphaned-state** back-up (only after the user confirms): move, don't delete —
```bash
mv "$HOME/.rote" "$HOME/.rote.bak-$(date +%Y%m%d-%H%M%S)"
```
Tell the user where the backup is so they can restore adapters/tokens later if wanted. Then
treat the run as **clean slate** (install → login → fork).

**Install choice** (clean-slate and orphaned-after-backup) — render this block exactly:

```text
Choose how you want to install:

1. Full CLI installation (recommended, default)
   Installs the rote CLI and browser runtime for full rote and rote-browse capabilities.

   ROTE_YES=1 ROTE_FULL=1 bash -c "$(curl -fsSL https://getrote.dev/install)"

2. CLI without browser
   Installs the rote CLI without browser automation. rote-browse and full rote capabilities
   will not be available. Add them later with `rote setup --full`.

   ROTE_YES=1 ROTE_SKIP_BROWSER=1 bash -c "$(curl -fsSL https://getrote.dev/install)"

3. Editor extension
   I'll detect VS Code, Cursor, and Antigravity and offer only installed targets.

4. I'll install it myself
   I'll pause; after you install, run `rote-setup` again.

This downloads and runs remote install code, so I need your explicit approval.

Press Enter for the recommended full installation, or reply with 1, 2, 3, or 4.
```

Trim surrounding whitespace and match replies case-insensitively. Treat an empty reply as
Enter. Then interpret replies exactly:

- Enter, `1`, `full`, `full installation`, `Full CLI installation`, `install CLI full`, or
  `1 and full installation` → Full CLI installation
- `2`, `CLI-only`, `CLI without browser`, `install CLI without browser`, or
  `2 and CLI without browser` → CLI without browser
- `3` or `editor extension` → Editor extension
- `4` or `install myself` → I'll install it myself

For either CLI selection, show the selected command and ask once: "Approve running the
selected installer command?" Do not execute either remote command until the user explicitly
approves it. Choosing the default never counts as approval. Once approved, run the selected
command as the one allowed compound command. Do not repeat the remote-code warning in an
option or follow-up approval prompt.

After either CLI profile succeeds, resolve the installed binary before Step 0:

```bash
command -v rote
```

If that is empty, check the installer's usual destinations:

```bash
ls -la "$HOME/.local/bin/rote" "$HOME/.cargo/bin/rote" 2>/dev/null
```

If either binary exists, retain its absolute path for every later command and continue to
Step 0. Tell the user it is off PATH and suggest opening a new terminal, sourcing their shell
profile, or adding its parent directory to PATH. Block only if neither `command -v rote` nor
an absolute-path check resolves the binary.

**Command convention for the rest of this run:** every step below writes `rote …`. If Step
-1 or the post-install check resolved rote on PATH, run it verbatim. If either resolved only
an absolute path (off PATH), substitute that path for the leading `rote` in **every** command
(e.g. `$HOME/.local/bin/rote whoami`). Pick this once and stay consistent.

For **Editor extension**, go to **Step -1a (detect editors)**. It installs the rote extension
into a VS Code-family editor and preserves the existing editor-specific setup path. For
**I'll install it myself**, pause until the user installs rote and runs `rote-setup` again.

### Step -1a — Detect installed editors and install the extension

The rote extension ships to two registries: the **VS Code Marketplace** (VS Code only)
and **Open VSX** (the open registry that Cursor, Antigravity, and other VS Code forks pull
from). Detect which editors are actually installed before offering them — never present an
editor the user doesn't have.

Probe each editor's CLI (and fall back to the macOS app bundle):

```bash
command -v code
```
```bash
command -v cursor
```
```bash
command -v antigravity
```

If a `command -v` is empty, fall back to checking the app bundle on macOS, e.g.:

```bash
ls -d "/Applications/Visual Studio Code.app" "/Applications/Cursor.app" "/Applications/Antigravity.app" 2>/dev/null
```

Build the option list from **only the editors that resolved**. For each:

| Editor | CLI | Install command | Registry |
|---|---|---|---|
| **VS Code** | `code` | `code --install-extension Modiqo.rote` | VS Code Marketplace |
| **Cursor** | `cursor` | `cursor --install-extension Modiqo.rote` | Open VSX |
| **Antigravity** | `antigravity` | `antigravity --install-extension Modiqo.rote` | Open VSX |

Run the chosen editor's install command as its own command, e.g.:

```bash
code --install-extension Modiqo.rote
```

If the editor's CLI isn't on PATH but the app bundle exists, point the user at the
marketplace page to install manually instead:
- VS Code → **https://marketplace.visualstudio.com/items?itemName=Modiqo.rote**
- Cursor / Antigravity (Open VSX) → **https://open-vsx.org/extension/Modiqo/rote**

If **none** of the three editors are detected, say so and return to the **install choice**
(defaulting to Full CLI installation), or let the user install an editor first. Don't offer
an editor that isn't there.

After install, the extension's setup wizard provisions the CLI on first launch — tell the
user to reload/open the editor, then re-verify the binary with `command -v rote` before
continuing to **Step 0 (login)**.

---

## Step 0 — Detect login state (always — login is mandatory)

Login runs for **every** user, right after the binary is confirmed installed — the whole
experience is identity-gated so usage is attributable. Detect current login state silently:

```bash
rote whoami
```

Read the output:
- `ok: <email>` → **already logged in.** Say "You're signed in as `<email>`." and go to the
  fork (**Step 2.5**).
- Anything else (error, `not logged in`, empty) → **not logged in.** Go to **Step 1**.
  Sign-in is required before the fork — do not skip it.

Do not assume — branch on the actual output of `rote whoami`.

---

## Step 1 — Sign in (detect configured providers)

Present the sign-in options. The CLI supports `--provider google`, `--provider github`,
and `--email <addr>`. Show provider choices:

- **Google** → `rote login --provider google`
- **GitHub** → `rote login --provider github`

Run the chosen one as its own command:

```bash
rote login --provider google
```

This opens a browser for OAuth. Tell the user to complete it in the browser, then
re-verify:

```bash
rote whoami
```

When `whoami` returns `ok: <email>`, say "Signed in as `<email>`." and go to the fork
(**Step 2.5**). If it still fails, show the login output and offer to retry with a different
provider.

---

## Step 2.5 — How far do you want to go? (fork, after login)

The binary is installed **and the user is signed in** (login in Steps 0–1 always runs first —
every experience is identity-gated, so usage is attributable). Now ask how far they want to
take setup.

**This fork is where "adapter" first appears — give the adapter What/Value beat before the
question** (the user may not know what an adapter buys them). Explain that an adapter lets rote talk
to a provider API directly from the user's machine: no gateway SDK middleman, no extra per-call fees,
and no new proxy quietly holding their data. Then ask how far they want to go:

- **Just the CLI — stop here** — confirm rote is installed and signed in, print `rote how`
  for next steps, and **end the wizard cleanly**. No adapters. They can run `rote-setup`
  anytime to go further.
- **Pull the powerpack** (curated starter adapters) — github, gmail, calendar, linear from
  the registry. Go to the powerpack picker in **Step 3a**.
- **Build an adapter** (from the 872-spec catalog, a web-found spec, or a local file) —
  **delegate to the `rote-adapter-create` skill** (see Step 2.6). It does the dry-run-first
  pipeline: discover spec → analyze → pick auth/toolsets → create.

Login is **not** part of this fork — it has already happened. All three branches run under
the signed-in identity; do not skip or defer sign-in for any of them.

### Step 2.6 — Build an adapter (delegated)

**Invoke the `rote-adapter-create` skill** rather than inlining adapter creation here. That
skill owns the full dry-run-first play — spec discovery (catalog → web → local), `--dry-run`
analysis, auth + toolset selection driven by the analysis, create, and post-create options
(write guard, sensitivity, credentials). It's the same skill a user invokes directly later when they add more
adapters, so the logic lives in one place.

After `rote-adapter-create` finishes, continue the setup flow: offer **Step 3c** (credentials,
if the new adapter needs a static token), the **install-skill** step, and the value closer
(**Step 5**). For ongoing tuning of an adapter, point the user at the `rote-adapter-config`
skill.

> **Future direction (not yet wired):** a created adapter should be **pushed to the user's
> private org hub** so the team shares it — `rote registry adapter push <path> <org-slug>`,
> gated on identity + org membership. For now, created adapters stay local.

---

## Step 3 — Post-login menu (menu-driven)

Now logged in. Present a **menu** of remaining setup steps (per design — let the user
pick, don't force the full sequence). Ask which setup steps they want, allowing multiple
choices when the environment supports it:

| Option | What it does | Command |
|---|---|---|
| **Pull powerpack adapters** (recommended) | Curated registry adapters, à la carte. **See 3a.** | `rote registry adapter pull bootstrap/<name> --yes` |
| **Build an adapter** | Catalog / web / local spec → dry-run-first create. **Delegates to `rote-adapter-create`.** | invoke the `rote-adapter-create` skill |
| **Configure an adapter** | Tune auth, base-url, sensitivity, guard, etc. **Delegates to `rote-adapter-config`.** | invoke the `rote-adapter-config` skill |
| **Set up credentials** | Token wizard for installed adapters. **Secrets — see 3c.** | `rote powerpack credentials` (run in user's own terminal) |
| **Connect Google (OAuth)** | Browser OAuth for Google adapters (gmail/calendar). **Non-interactive — see 3d.** | `rote oauth setup google --adapter <id> --scopes <list>` |
| **Install the agent skill** | Installs the rote skills for supported local agents. | `rote install skill` |
| **Explore / learn** | Onboarding tree + human-friendly help. | `rote how`, then `rote human` |

Recommend the order **adapters → credentials → OAuth → install skill → explore**, but run
only what they pick, one at a time, confirming after each. Once their picks are done, offer
the **value-proof closer (Step 5)** — run one live play so setup ends with real output.

### Step 3a — Install adapters (à la carte, from the live registry)

**Do not use `rote pull powerpack` presets.** The presets list adapters that aren't
published to the reachable registry (`notion`, `gemini-api`, `googledrive`, `googledocs`),
so they partially or fully fail (`ai` fails entirely; `default`/`dev` choke on notion;
`google` loses drive+docs). Install à la carte instead — every pick either installs or
gives its own clear error, with no phantom adapters.

**Query the registry at runtime** — never hardcode the list, it grows:

```bash
rote registry adapter list --community bootstrap
```

Build the checklist from **only the adapters that command returns**. As of writing the
bootstrap community publishes:
`calendar, cloudflare-api, elevenlabs, exasearch, github, gmail, linear, parallelweb,
polymarket-data, polymarket-gamma, stripe` — but trust the live output, not this list.

**Cap the selection at 2 adapters per run** (fewer than 3). If the user wants more, they
re-run this step. Keep the first pass small so credential setup stays manageable.

Install each picked adapter as its own command:

```bash
rote registry adapter pull bootstrap/github --yes
```

`--yes` keeps it non-interactive. After each, note whether it needs a credential and carry
that into Step 3c. Then reindex once at the end if needed (`rote adapter list` to confirm
ready state). Adapters that need an API token will surface their env var name; Google
adapters (gmail/calendar) use OAuth via Step 3d, not a static token.

**Credential gate — do NOT proceed past an unset required token** (real bug: an adapter was
installed from a preset/bootstrap, its token was never set, and the wizard marched on to the
next step / tried to run a play against it anyway). After installing adapters, **verify** which
required tokens are actually configured before relying on any of them:

```bash
rote powerpack tokens
```

For each installed adapter whose token shows **✗ not configured**, do **not** treat it as
usable — route the user to **Step 3c (credentials)** to wire it (or to **Step 3d** for Google
OAuth) *before* the value-proof play in Step 5. A play run against an adapter with an unset
credential will fail; don't attempt it. If the user declines to set a token now, that adapter
is simply skipped for the live-play proof — pick a play whose adapter IS credentialed instead.

**Pull each adapter's plays right after installing it.** The adapter pull installs only the
adapter — its curated plays are separate, and the value-proof closer (Step 5) needs at
least one play present. For each adapter just installed, find and pull its plays:

```bash
rote registry play find-by-adapter github
```

That lists `<org>/<name>` plays associated with the adapter (e.g.
`modiqo/list-top-committers`, `modiqo/retrieve-recent-emails`,
`modiqo/check-calendar-meetings`). Pull them non-interactively — `--yes` is **required**
because the pull confirmation prompt may not work in an agent-run command:

```bash
rote registry play pull modiqo/list-top-committers --yes
```

Pull a couple of the most useful per adapter (don't bulk-pull all of them). These land in
`${ROTE_HOME:-$HOME/.rote}/flows/<org>/<name>/` and become the menu for Step 5.

### Step 3b — Install skill provider

If they pick "Install the agent skill", ask which supported provider to install for. Then:

```bash
rote install skill --provider <provider>
```

Use `rote install skill --help` if the supported provider names are unclear, or `--provider all`
when the user wants every supported target. `--force` overwrites an existing SKILL.md without
prompting — only add it if the user confirms an overwrite.

## Auth Shape Matrix

These credential shapes are handled differently:

| Shape | Examples | Setup direction |
|------|----------|-----------------|
| Static bearer/API key | github `GITHUB_TOKEN`, linear `LINEAR_API_TOKEN`, stripe | Use masked terminal handoff; `rote token set <ENV> --stdin` only as explicit terminal opt-in. |
| OAuth client id/secret | OAuth2 OpenAPI adapters with `oauth2_schemes` | Register the app, collect client id/secret through the OAuth flow, and avoid pasted bearer tokens. |
| OAuth DCR / MCP PRM | MCP adapters with automatic browser redirect and dynamic registration | Pull/create the adapter, then run `rote adapter reauth <name>` if auth is not already complete. |
| Google Discovery | gmail, calendar, drive | Use `rote oauth setup google --adapter <id> --scopes ...`; no static token. |
| Unknown installed bearer | Existing adapter says bearer but provenance is unclear | Run `rote adapter list <id> --json --health` before telling the user to set a token. |

- **Browser-OAuth adapters** → no static token at all; the browser redirect keeps the secret out
  of chat and command history. Always prefer this shape when the adapter supports it. For Google
  Discovery adapters (gmail, calendar) skip to **Step 3d**; for OAuth DCR / MCP PRM adapters skip
  to **Step 3e** for reauth/DCR.

### Step 3c — Static-token credential handoff

**State this constraint up front, before offering any static-token option.** Do not collect
static tokens in chat. Most agent conversations and command histories are not a masked
secret-entry surface, and the exact retention/redaction behavior depends on the environment.
The terminal wizard is the safe default because it uses masked input outside the conversation.
Say this plainly so the user understands the handoff isn't friction, it's the secure choice.

- **Static-token adapters** → the user holds a bearer string that has to be conveyed. Hand off
  to the masked wizard:

	  **Default — hand off to the terminal wizard.** Tell the user to run, **in their own
	  terminal**:
  ```
  rote powerpack credentials
  ```
  It prompts per-adapter and **masks** input (rote uses `rpassword`), storing to
	  `~/.rote/secrets/` (perms 600). The secret never passes through chat. This is the
	  recommended path — frame the handoff as the secure one. The user
  runs it, returns, and you verify (below). Note: the wizard lists Google credentials under the
  shared `GSUITE_TOKEN` name whatever the adapter declares — tell the user to **skip them**;
  Google is wired and verified in Step 3d, not here.

	  **`rote token set` is a last-resort opt-in only.** Offer it only if the user explicitly
		  refuses the terminal handoff and insists on setting a token through the current session.
		  State plainly: "this may put the token in chat and shell history — rotate it
		  afterward if it's long-lived." Only on explicit go-ahead:
	  ```bash
	  read -rsp "Token: " TOKEN; echo
	  printf %s "$TOKEN" | rote token set GITHUB_TOKEN --stdin
	  unset TOKEN
	  ```

- **Never print, echo, or re-quote a token back** to the user once set. Don't `cat` the
  secrets dir.

**Verify with cwd-independent checks only.** Do not run a play or `rote ready` merely to test a
token: both exercise provider capabilities rather than reporting credential state. Direct adapter
checks such as `rote ready` require an active workspace, while a `steps:` play must run outside
every active workspace so its runner can create the DAG workspace. Verify the token itself with:

```bash
rote powerpack tokens
```

(or `rote token get <name>` for one) — both work from any directory and show
configured/not-configured without exercising the API.

### Step 3d — Google OAuth (non-interactive)

The bare `rote oauth setup google` opens an interactive multi-select ("Which Google APIs
do you need?"). Drive it non-interactively with **`--adapter` and `--scopes`** — one run per
adapter, passing only that adapter's scopes:

```bash
rote oauth setup google --adapter gmail --scopes gmail.readonly
rote oauth setup google --adapter calendar --scopes calendar.calendarlist.readonly,calendar.calendars.readonly,calendar.events.readonly,calendar.events.freebusy
```

`--adapter` stores the grant in the credential that adapter declares (`GMAIL_TOKEN`,
`CALENDAR_TOKEN`, or the shared `GSUITE_TOKEN` on installs predating per-adapter names) and
binds it there. Without it the grant always lands in `GSUITE_TOKEN`, which an adapter
declaring its own credential never reads — the flow reports success and the adapter stays
unauthenticated.

Scopes are comma-separated, no spaces, and belong to the adapter being authorized: sending
Gmail scopes with `--adapter calendar` stores a Gmail grant in the calendar credential without
error. Each run opens a **browser** for the Google consent screen; tell the user to complete it
there, then confirm with `rote adapter list <adapter-id> --json --health`. That proves a
refreshable credential exists, not that its scopes fit the adapter — the report carries no scope
field, so a wrong-scope grant still reads `healthy: true`.

### Step 3e — OAuth DCR adapters (reference)

Some MCP adapters authenticate via **OAuth Dynamic Client Registration (DCR)** rather than
a static token — for those, after `rote registry adapter pull bootstrap/<name> --yes`, run:

```bash
rote adapter reauth <name>
```

The first run registers a DCR client (RFC 7591) and opens a PKCE browser play; tell the
user to complete it in the browser. If a later reauth fails because the provider pruned the
client (rare), add `--force-reregister`. Note: **Notion is not currently in the reachable
registry** (`bootstrap` / `modiqo`), so don't offer a Notion install — the bootstrap
community list in Step 3a is the source of truth for what's actually installable.

---

## Step 4 — Wrap up

After the menu items they chose are done, confirm and orient:

```bash
rote how
```

Summarize what's now set up (signed-in email, adapters pulled, credentials wired, skill
installed) and point at `rote human` for friendly help and `rote how` for the agent
onboarding tree. Then offer the value-proof closer (Step 5).

---

## Step 5 — Prove value: run one live play (optional, recommended)

End on a win — run a real play against the user's own data so setup ends with *output*,
not just "complete." Only offer this if at least one credential is wired (a play with no
working credential will just error).

**This is where "play" first appears — give the play What/Value beat before the question.** Explain
that a play is a reusable workflow the agent can run locally: rote crystallizes what worked into
deterministic steps, saving tokens and repetition without a workflow-vendor subscription. Then ask:
"Want me to run a quick play to see it work?" — yes / skip.

Use the `rote-flow-run` execution rules when a matched play owns the proof. The short version,
applied here:

**1. Find a play for a credentialed adapter.** Use `rote explore "<intent>"` to discover what
can handle the intent, and/or list adapter-matched plays:

```bash
rote registry play find-by-adapter github
```

**Only pick a play whose adapter's credential is actually configured** (you verified this with
`rote powerpack tokens` in the credential gate). A read-only play is the safest first run
(`list-top-committers`, `retrieve-recent-emails`, `check-calendar-meetings`) — avoid
write/create plays for the proof.

**2. Let the user pick one.**

**3. Ensure it's pulled.** If not already local, pull it (`--yes` required):

```bash
rote registry play pull modiqo/list-top-committers --yes
```

**4. Read the frontmatter — for BOTH the params AND the execution mode.** Read the path returned
by `rote play search "<intent>" --json` or `rote play list --json`. It tells you two things:
- the `parameters:` block — name + description per param (ask the user for each, using the
  description as the hint; offer a sensible default like `modiqo/rote`);
- whether there's a `steps:` block — this decides **how** you run it (next).

**5. Run it the right way (this is the fix for the "ran via bash, not Deno" bug).**

- **Frontmatter has `steps:` (DAG play)** → run it from a directory outside every active rote
  workspace. The play runner creates and owns its DAG execution workspace:

  ```bash
  cd /tmp && rote play run <name> param=value …
  ```

- **No `steps:` (legacy/sequential play — most curated plays)** → do **NOT** use
  `rote play run` (it can fall back to a plain bash invocation instead of Deno). Run it via
  the bundled Deno from the play's own directory:

  ```bash
  cd <play directory from rote output> && rote deno run --allow-all main.ts [args…]
  ```

  Pass the play's positional args (from the `parameters:` block), e.g.
  `… main.ts modiqo/rote`. `rote deno run` uses the bundled Deno from the active rote home.
  The `cd && rote deno run` compound is one logical step. This `cd`s into the play directory
  outside the project, so
  make sure the current environment has access if it requires filesystem approval.

When unsure which mode, inspect frontmatter or run `rote play info <name-or-path> --json` first.
Do not use direct Deno for any play with frontmatter `steps:`.

Show the play's output to the user — that's the payoff.

### Step 6 — Day-to-day use

Suggest the main **rote** skill for day-to-day use (`rote play search "<intent>"` before any direct
adapter call). For single-adapter delegated work, spawn a subagent and tell it to use
**rote-using-adapters** with the adapter id and task.

---

## Behavior notes

- **One command at a time.** No `&&`, `|`, or `;` chains. Keep each
  success/failure visible on its own. Four allowed compounds are single logical steps: the
  selected canonical profile installer from the **Install choice**; `cd <workspace> && rote …`
  for adapter probes or calls that require a workspace cwd (see below);
  `cd /tmp && rote play run <name> param=value` for a `steps:` play whose runner must own the DAG
  workspace (never point that `cd` at an active rote workspace); and
  `cd <play directory> && rote deno run --allow-all main.ts [args…]` for a legacy no-steps play
  (step 5 above).
- **Prefer non-interactive commands in agent-run shells.** Many agent command runners cannot
  answer terminal prompts. The non-interactive switch differs per command: installer →
  `ROTE_YES=1`; powerpack picker → `--yes`; `rote adapter new` →
  `--yes`; `rote adapter new-from-mcp` → **`--headless`** (it rejects `--yes`); Google OAuth →
  `--adapter` + `--scopes …`. Don't guess the flag — if unsure, check `<command> --help` first. The one
  prompt you must NOT automate away is credential entry — see secrets below.
- **Never handle secrets in chat.** Agent conversations and command histories are not a masked
  secret-entry surface unless the current environment explicitly says otherwise. Static tokens
  should hand off to `rote powerpack credentials` in the user's own terminal (masked via
  rpassword). Prefer OAuth (browser redirect) over any static token where the adapter supports
  it. `rote token set <name> <value>` is a last-resort opt-in — only on explicit insistence,
  with a rotate-afterward warning. Never echo a token back or read the secrets dir.
- **Verify without a workspace where possible.** To check a credential, use
  `rote powerpack tokens` / `rote token get` — these work anywhere. But **anything that probes
  an adapter or calls an API directly needs a workspace cwd** (`rote <adapter-id>_probe`,
  `rote ready`, `rote POST`) — outside one they fail with
  `not in a workspace directory` / `Permission denied (os error 13)`.
- **Running a workspace-scoped command (the correct pattern).** Do not assume the command runner
  preserves cwd between separate command invocations. `rote cd` / `--enter` only affect an
  interactive shell, so `eval $(rote cd …)` usually won't carry over. Instead, create the
  workspace, then `cd` into its real directory **in the same command invocation** as the command:

  ```bash
  rote init proof --seq --force
  cd ${ROTE_HOME:-$HOME/.rote}/rote/workspaces/proof && rote <adapter-id>_probe "<intent>"
  ```

  The workspace lives at `${ROTE_HOME:-$HOME/.rote}/rote/workspaces/<name>`. This `cd && rote …` compound is a
  necessary exception to the one-command-at-a-time rule for direct adapter work (the cwd must hold
  for the command). It does not apply to a DAG play: use
  `cd /tmp && rote play run <name> param=value` so the runner can own its execution workspace
  outside every active rote workspace.
  `--force` skips the "search for existing plays first" gate. Clean up later with
  `rote workspace clean` if desired.
- **Detect before offering.** Resolve the rote binary through `command -v rote` or the
  supported absolute-path check before any rote command, and detect installed editors
  (`command -v code|cursor|antigravity`) before offering the extension path. Never present an
  install target that isn't there.
- **Detect, don't assume — and never from memory.** Every fact (installed? where? logged
  in?) comes from a live probe in this run, not from recalled notes or a "rote is
  active here" assumption. A stale memory must never short-circuit a probe; if memory and a
  live probe disagree, the probe wins. Once rote is present, start from `rote whoami` and
  branch on its real output.
- **Lead with the happy path.** Sign-in first; then connect APIs or take the live tour.
- **Browser steps are async.** After `rote login --provider ...` or `rote oauth setup
  google`, tell the user to finish in the browser, then re-run `rote whoami` /
  re-check before declaring success.
- **Show errors verbatim.** Never paper over a failed step; offer a retry or alternate
  branch.
- **Don't fabricate URLs or codes.** Determine every setup fact from live CLI output.
- After a successful first run, suggest the main **rote** skill takes over for day-to-day
  use (`rote play search` before any direct adapter call).

---

## Closing line + related skills

**Closing line** (only after a clean first run): land one dry one-liner keyed to the live play
that just proved value, e.g. "And that play ran straight against the provider's API — no proxy in
the middle quietly metering you per call. Welcome to rote." Skip it if any step errored.

**Related setup skills** (this is the front door):
- **Build more adapters:** `rote-adapter-create`
- **Tune one:** `rote-adapter-config` · **Keep current:** `rote-update`
