# rote-onboard — skill map

Five guided slash-command skills that walk a human through getting rote working, sharing what
they build, and keeping it all current. They're a sequence, not a grab-bag — each ends by
naming the next logical step, so the path becomes muscle memory. This file is the canonical
map; each skill links back here.

## The skills, in the order you meet them

| Order | Slash command | Use it when | Hands off to |
|------:|---------------|-------------|--------------|
| 1 | `/rote-onboard:rote-setup` | First run — install the binary, sign in, pull/​build your first adapters, prove value with one live flow. | adapter-create (build more) · main **rote** skill (daily use) |
| 2 | `/rote-onboard:rote-adapter-create` | Add an adapter for any API — dry-run-first, auth researched from the provider's docs. | adapter-config (tune it) · registry (share it) |
| 3 | `/rote-onboard:rote-adapter-config` | Tune an adapter you already have — auth, base URL, write guard, sensitivity, capability index, OAuth re-auth, grouping. | back to itself (menu) · adapter-create (new one) |
| 4 | `/rote-onboard:rote-registry` | Share an artifact — at the moment an adapter is minted or a flow crystallized, check if it's already in your orgs, push at chosen visibility, then surface members + invite. Also `/show usage` (quota across orgs). | rote-org (full admin) |
| 5 | `/rote-onboard:rote-update` | Keep current — update both layers (the `rote` binary **and** these skills), which version independently. | — |

**The day-to-day skill is separate:** once you're set up, the main **rote** skill takes over
(`rote flow search "<intent>"` before any direct adapter call). The onboard skills are for
*changing your setup*; the rote skill is for *using* it.

## How to build muscle memory

- **Discovery:** type `/rote` in the agent — the `rote-onboard:*` skills appear with their
  descriptions. This file gives them an order; the menu alone doesn't.
- **The chain is the teacher:** every skill ends by naming the next step in context ("you just
  made an adapter → `rote-adapter-config` tunes it"). Following the chain a few times is what
  makes it stick — you stop needing this table.
- **One entry point to remember:** `/rote-onboard:rote-setup` is the front door. From a clean
  machine it routes to everything else.

## Shared operating rules (apply in EVERY skill)

These are load-bearing — a skill that ignores them fumbles the run. Each SKILL.md references
this section; keep them identical across skills.

### 1. Permissions — clear the prompts up front

Every `rote …` / `cd …` / `rote deno …` call goes through the agent's Bash tool, which prompts
for permission unless allowlisted. On a fresh machine that means a prompt on *every step* —
death by a thousand confirmations. **At the very start of any skill, before the first rote
command, tell the user to allowlist these** (or do it via the `update-config` skill if
available):

There are **two independent permission layers** — you need both, or you'll still be prompted:

**(a) Bash-command allowlist** — `permissions.allow`:
- `Bash(rote:*)` — all rote commands (this already covers `rote deno run …`, so a separate
  deno rule is redundant; include it only as an explicit teaching aid if you like)
- `Bash(cd:*)` — the `cd <dir> && rote …` compounds used for workspace/flow execution

**(b) Directory-access allowlist** — `permissions.additionalDirectories` (THIS is the one
most people miss). The skills `cd` into several paths **under `~/.rote/` — all outside the
user's project directory**: `~/.rote/rote/workspaces/<name>` (flow runs / probes),
`~/.rote/flows/<org>/<name>` (deno flow execution), and they read `~/.rote/adapters/…`. Each
`cd` outside the project triggers a *separate* filesystem prompt ("allow reading from <dir> …")
that the Bash allowlist does **not** cover. The simplest cover-all is to add the whole
`~/.rote` tree:
- **`~/.rote`** — covers workspaces, flows, adapters, tokens — everything the skills touch by
  path. (Use a narrower `~/.rote/rote/workspaces` + `~/.rote/flows` pair if you want to scope
  it tighter, but `~/.rote` is the one-entry answer.)

So the complete `~/.claude/settings.json` shape is:

```json
{
  "permissions": {
    "allow": ["Bash(rote:*)", "Bash(cd:*)"],
    "additionalDirectories": ["~/.rote"]
  }
}
```

Offer to add **both** layers once at the start; don't re-ask. Without `additionalDirectories`,
every flow run / probe inside a workspace re-prompts even though the Bash rules are present —
the exact bug seen on a fresh machine. (Codex: the equivalent is its approved-commands /
sandbox-writable-roots config.)

### 1b. Resolving the rote binary — narrow probe, NEVER a deep search

If `command -v rote` is empty, rote may be installed but off this shell's PATH. Check the
**two known install locations directly** — do **NOT** run `find ~` / a recursive search of the
home directory (it's slow, triggers a scary read-everything permission prompt, and is exactly
the overreach to avoid):

```bash
ls -la "$HOME/.local/bin/rote" "$HOME/.cargo/bin/rote" 2>/dev/null
```

- Either path exists → use that **absolute path** for every later `rote` command in the run.
- Neither exists **and** `command -v rote` was empty → the binary is genuinely **not
  installed** (route to install, or tell the user). Do not go hunting for it elsewhere.

### 1c. Shell expansion defeats the allowlist — prefer literal paths

Even with `Bash(rote:*)`, `Bash(cd:*)`, and `~/.rote` all allowlisted (#1 above), a command
will **still** prompt if it contains **parameter expansion with a default** (`${VAR:-fallback}`)
or **command substitution** (`$(…)`). The agent's Bash tool flags these as "Contains expansion"
and asks for confirmation **regardless of the allow rules** — it can't statically verify what
the expansion resolves to. This is the most common "I allowlisted everything and it *still*
prompts every step" trap.

The single worst offender is prefixing workspace commands with
`cd ${ROTE_HOME:-$HOME/.rote}/rote/workspaces/<name>` — the `${ROTE_HOME:-…}` default-expansion
triggers a prompt on **every** command that uses it.

Fixes, in order of preference:

- **Use a literal absolute path** for the `cd`: `cd ~/.rote/rote/workspaces/<name>` or
  `cd /Users/<you>/.rote/rote/workspaces/<name>`. (Plain `~` / `$HOME` and simple redirects
  like `2>/dev/null` are fine — it's `${VAR:-default}` and `$(…)` that get flagged.)
- **Lean on workspace persistence** — the rote workspace stays selected across Bash calls, so
  after the first `cd` you usually don't need to `cd` again at all. Just run `rote …` directly
  on subsequent steps.
- **Run discrete single commands** — don't chain a rote call with `$(…)` substitution or an
  `echo "… $(rote …)"` wrapper. Use `rote is-error @N` on its own line, then read the value
  with a separate `rote @N '…' -r`, instead of `rote is-error @N && echo "$(rote @N …)"`.

rote's embedded jq has two quirks worth knowing while inspecting responses: a string slice
like `.body[0:200]` returns `null`, and a `slice | map({…})` chain errors — index a single
element with `.[idx]` instead.

### 2. Step-wise, NEVER parallel

Run **one command per Bash call, strictly in sequence** — never batch independent probes into a
single parallel tool block, and never fire the next step before the current one's result is in.
Each step's output gates the next (an empty `whoami`, a 401 on a dry-run, an unset token all
change what comes next). Parallel probing skips that gating and the run fumbles — the exact bug
seen on a fresh machine. If you catch yourself about to send two Bash calls at once: don't.

### 3. Required-state gates block the next step

When a step establishes a precondition the next step needs — a token set, a login succeeded, a
dry-run passed — **verify it actually happened before proceeding.** Do not advance on an
adapter whose required credential is unset, a login that still reports "not logged in", or a
create whose dry-run errored. Re-read the real output; if the precondition isn't met, stop and
resolve it (or ask the user) rather than marching to the next step on a broken foundation.

### 4. Running a flow — explore, read frontmatter, run the right way

There are two execution modes and you must pick by the flow's frontmatter:

1. **Find the flow** for the intent (don't guess the name):
   `rote explore "<intent>"` (ranks installed adapters/flows) and/or `rote flow search "<q>"`.
2. **Read the flow's frontmatter** before running it: look at
   `~/.rote/flows/<org>/<name>/main.ts`. Does it have a `steps:` block?
   - **Has `steps:` (DAG flow)** → `rote flow run <name> key=value …` works (needs a workspace cwd).
   - **No `steps:` (legacy/sequential)** → `rote flow run` may misfire (it can fall back to a
     bash invocation instead of Deno). Run it the reliable way instead:
     ```bash
     cd ~/.rote/flows/<org>/<name> && rote deno run --allow-all main.ts [args…]
     ```
     `rote deno run` uses the bundled Deno at `~/.rote/bin/deno`. Pass the flow's positional
     args (read them from the frontmatter `parameters:`). This `cd && rote deno run` compound is
     one logical step (see the `Bash(cd:*)` allowlist above).

When unsure which mode, prefer the `cd … && rote deno run` form — it works for both and matches
how the flows are authored.

## Primitive intros — explain the *what* and *why*, the first time

A first-timer meets three rote primitives during onboarding — **adapter**, **flow**, **hub** —
and the skills used to just *use* them without ever saying what they are or why they're worth
it. **The first time a primitive appears in a run, give its What/Value beat before the
AskUserQuestion** (~3–4 lines — the "what" with a bit of dry humor, then the concrete value).
On repeat appearances, skip it — they've got it.

These are the canonical beats. Keep the voice consistent (dry, confident, no hype, no invented
numbers); adapt the wording to the moment but keep the *claims* exactly these. The three map
onto one motion — **reach → record → replay** — so the set reads as a single idea, not a
grab-bag. Lead each with the *what*, land it on the *tax it removes*. The recurring dig is the
honest one: you are charged a toll to call tools you already pay for. Never name a competitor,
never invent a price.

### Adapter — *Reach* — introduced at the "build an adapter" fork (setup), and in `rote-adapter-create`

> **What:** An adapter lets your agent reach any API directly — talk its protocol from your own
> machine, no gateway in the middle, no SDK to wire. rote reads the API's own spec and exposes
> it as tools, locally.
> **Value:** You already pay for these tools. An adapter means you don't pay a *second* toll to
> call them — no per-tool-call fee, no extra box to run, no new attack surface. Your traffic
> goes straight to the provider, not through someone else's meter.

### Flow — *Record → Replay* — introduced at the value-proof closer (setup Step 5), and when a flow is first saved

> **What:** A flow is how your agent learns the way you did. As you work, rote *records* what
> worked and what failed, then crystallizes the wins into reusable memory — so next time it
> *replays* instead of re-reasoning from zero.
> **Value:** Determinism and token savings, with nobody renting it back to you — no
> workflow-vendor subscription, no memory provider metering your own recall. A proven flow runs
> like muscle memory: you don't re-read the manual to ride the bike.

### Hub — *Replay, shared* — introduced in `rote-registry` before the first push

> **What:** When a flow or adapter works, the hub lets your team and community replay it too —
> the way a teacher's lesson saves every student from rediscovering it. One person's mastery
> becomes everyone's.
> **Value:** The same determinism, now shared, with no registry tax on knowledge you authored —
> one proven flow is the whole org's, no re-derivation, no per-seat toll to reuse what your own
> team already figured out.

## Shared closing-line convention (dry humor)

All skills end a **clean** run on one dry one-liner, keyed to what actually just happened.
The running joke: rote wired you straight to the provider's own API — no metered middleman
proxy billing per call, no extra network hop forwarding your request for a markup.

Rules (identical across all skills — keep them consistent):
- **Grounded in this run's facts**, never a canned template — the flow name, adapter count, or
  tool count from *this* session.
- **Honest claims only.** "Per-call metering" and "an extra network hop" are the safe, true
  jabs. Never invent a dollar figure or name a competitor's pricing you didn't verify.
- **Read the room.** Skip the quip if the run was rocky (login failed, a step errored, a probe
  401'd, `self-update` deferred). A victory lap after a half-finished run reads as oblivious.
- **One line.** If it needs a second sentence to explain the joke, cut it.

Per-skill hook (so it lands on different facts each time, not a hardcoded tagline):
- **setup** → the live flow that just proved value
- **adapter-create** → the `total_tools` count just indexed
- **adapter-config** → whatever was just tuned (lower-key; config is housekeeping)
- **registry** → the artifact + org it just landed in, now shareable without a registry tax
- **update** → the version just landed on
