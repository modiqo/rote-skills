---
name: rote-adapter-create
description: >
  Create a rote adapter for any API — dry-run-first. Discovers the spec (built-in catalog →
  web search → local file), analyzes it without committing (`rote adapter new <id> <spec>
  --dry-run`), then pre-fills auth, base-url, and toolset choices from the analysis — asking
  you only where it's genuinely ambiguous. When auth is ambiguous or low-confidence, researches
  the provider's docs to recommend a scheme and produce setup steps (API key / OAuth2). Use
  when the user says "add an adapter", "connect
  to <API>", "create a stripe/notion/datadog adapter", "build an adapter from this OpenAPI
  spec". Never creates from a discovered spec without a clean dry-run
  first. Hands off to rote-adapter-config for tuning afterward.
---

# rote-adapter-create — dry-run-first adapter creation

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Build an adapter for any API with the same dry-run-first discipline as rote's guided setup:
**analyze the spec with no
side effects, then drive the choices from what the analysis found.** You hold the dry-run JSON
in context and ask the user only the questions the analysis can't answer.

Use the base `rote` companion skill when the full workflow graph or packet shape is needed; this
skill contains the active creation rules. On a
fresh run, clear any command/filesystem approvals the current environment requires before the first
`rote` command.

Core rules:
- **Never `--yes` create from a discovered spec without a successful `--dry-run` first.** The
  dry-run is the validation gate at every discovery branch.
- **Never stop at dry-run.** A dry-run emits analysis and creates no adapter. The adapter creation
  task is incomplete until the real create command succeeds, `rote adapter info <id>` verifies the
  install, and a readiness probe or explicit skip reason is recorded.
- **Determine facts from live commands, never from memory.** (If the rote binary isn't on
  PATH, resolve it via the **narrow probe** — check `$HOME/.local/bin/rote` then
  `$HOME/.cargo/bin/rote`, never a deep search of the home dir.)
- **One command at a time, strictly sequential — never parallel.** Each stage gates the
  next (dry-run result drives auth/toolsets); firing steps in parallel breaks the pipeline.
- Secrets (API tokens) are never captured in chat — hand off to the masked wizard (see Stage 5).

## Handoff Contract

- Use when: the user wants to create a new rote adapter, connect a provider API, or build from a
  catalog, web, MCP, GraphQL, OpenAPI, or local spec source.
- Preconditions: `rote` is runnable or the install blocker is known; the target adapter id and spec
  intent can be elicited; no adapter will be created until the dry-run or MCP-specific create gate is
  satisfied.
- Owns: spec discovery, dry-run analysis, auth research, base-url and toolset decisions, adapter
  creation, post-create credential/write-guard/sensitivity offers, and readiness probe options.
- Hands off to: `rote-adapter-config` for tuning; `rote-registry` for sharing; `rote` for
  day-to-day use after creation.
- Returns to: the caller with adapter id, spec source, dry-run summary, auth scheme, selected
  toolsets, create command, post-op choices, `rote adapter info` result, probe result or skip reason,
  and next recommended skill.
- Stop when: dry-run fails, spec/auth choice is ambiguous and needs the user, adapter creation
  succeeds or fails with a surfaced error, a credential/human browser action is pending, or another
  skill owns the next step.
- Completion signal: dry-run result recorded, create/probe status explicit, and a handoff packet
  emitted when returning to registry, config, or daily use.

## Handoff Packet

Return this packet when handing the created adapter to another skill:

- Origin skill: `rote-adapter-create`.
- Target skill: `rote-adapter-config`, `rote-registry`, or `rote`.
- Adapter id: `<adapter-id>`.
- Spec source: catalog id, spec URL, MCP URL, or local file path.
- Dry-run summary: title, version, base URL, operation/toolset count, auth recommendation.
- Decisions made: auth scheme, token env var, toolsets, base-url override, grouping.
- Commands run: dry-run command, create command, `rote adapter info <adapter-id>`, and probe
  command if any.
- Credential state: configured, terminal handoff pending, OAuth browser pending, or not required.
- Next owner: tuning, registry share, or daily rote use.

---

## Stage 0 — Discover the spec (catalog → web → local file)

**If the user hasn't already heard the adapter pitch this run**, explain that an adapter lets rote
talk to an API directly from the user's machine — no gateway/SDK middleman, no extra per-call fees,
and no new proxy holding their data. Skip it when the caller indicates the pitch was already given.

Ask what API the user wants (prose): "Which API? (e.g. notion, stripe, datadog — or paste a
spec URL / file path)".

**First, rule out an existing adapter.** Once you know which API, run `rote adapter list` as the
inventory pass. If any listed adapter plausibly matches the target API, run
`rote adapter info <id>` for each plausible match and use those details to decide whether it already
covers the target API. If one adapter already covers it, stop — use it, or hand off to
**rote-adapter-config** to tune it (auth, base URL, etc.); only continue to create a new one if the
user explicitly wants a second. If **several** installed adapters plausibly cover it (e.g. a REST
and an MCP adapter for the same provider), present them and let the user choose — do not silently
pick one. Re-creating over an existing adapter regenerates its fingerprint and churns config.

Then resolve a **spec source** through this chain. The spec source is either a catalog id, a URL, or
a local path.

### 0a. Catalog (built-in, ~872 specs) — ALWAYS try this first

```bash
rote adapter catalog search "stripe"
```

**Read the result count and act on it — do NOT skip to web search when the catalog has
matches** (this is a real bug seen on a fresh run: 2 matches were found but not shown, and the
skill jumped to web anyway):

- **1+ matches** → **present every match to the user** (id · category · provider) and let them
  **choose** — or confirm the single match.
  Never silently pick one, and never fall through to web search while matches exist. Only after
  the user picks (or explicitly says "none of these, search the web") do you leave the catalog.
  Include decision fields when available: kind/substrate, auth shape, spec-fetch auth, runtime auth,
  capability fit, installed/health state, install/write impact, and next command.
  When two matches are the **same product under different protocols** — e.g. a REST/OpenAPI spec
  *and* an MCP server for it (`Spec Type` "MCP") — do not treat either as a redundant duplicate:
  surface both and make the user choose, noting the tradeoff (a REST adapter and an MCP adapter
  differ in protocol, auth model, and which tools they expose).
- **0 matches** → only then go to **0b (web search)**.

Once the user picks a catalog entry, show its details (auth type, spec URL, notes):

```bash
rote adapter catalog info stripe
```

Always follow the catalog-provided install command — run the exact install command that
`catalog info` prints for the entry, rather than assembling your own. If `Spec Type = "MCP"`, use
`rote adapter new-from-mcp`. Do not substitute generic `rote adapter new`.

For non-MCP catalog entries, `rote adapter new <catalog-id>` and `--dry-run` resolve the spec from
the catalog:

```bash
rote adapter new <catalog-id> --dry-run
```

For MCP-type entries (catalog `info` Spec Type = "MCP", e.g. Notion), use the MCP path (Stage 5,
MCP note). If you need a dry run, use `rote adapter new-from-mcp ... --dry-run`.

If `catalog info`, MCP discovery, or dry-run returns `401/403` — or a `400` whose body reads as an
auth/token error — while fetching a spec, label it as spec-fetch auth required. That is separate
from runtime auth after install; 401/403 is not proof the adapter is bad. Ask whether to
authenticate/reauth or choose another catalog candidate.

### 0b. Web search (only when the catalog returned ZERO matches)

Reach this **only** if `rote adapter catalog search` returned no matches (or the user
explicitly rejected all matches). Web-search for the API's **machine-readable spec** — an OpenAPI
JSON/YAML URL or a GraphQL endpoint, **not** a human docs page. Propose the candidate URL to
the user, then **validate it by dry-run** before trusting it (Stage 1). If the dry-run fails
(404, HTML-not-a-spec, 403/401), say so plainly and try the next candidate or ask the user for
the right URL. Wrong-spec is the main failure mode here — never create from an unvalidated
web-found URL.

### 0c. Local file

If the user has a spec file, use the path directly: `rote adapter new <id> <path> --dry-run`.

Pick an adapter **id** with the user (lowercase, hyphens ok, e.g. `stripe`, `acme-api`).

---

## Stage 1 — Analyze (dry-run)

```bash
rote adapter new <id> <spec-source> --dry-run
```

This emits JSON and **creates nothing**. Keep the JSON in context. Its shape (verified):

- `spec`: `{ title, version, openapi_version, base_url, operation_count, spec_size_bytes }`
- `toolsets[]`: `{ name, tool_count, confidence, methods{GET,POST,…}, read_operations, write_operations }`
- `auth`: `{ type, ... }` — `type` is `none` | `bearer` | `api_key_header` | `api_key_query` |
  `basic` | `per_operation`. For `per_operation`: `schemes{}`, `default_scheme`.
- `auth_scoring`: `{ recommended{auth_type,confidence,source,detail}, all_schemes[], has_ambiguity }`
- `summary`: `{ total_toolsets, total_tools, get/post/put/delete_operations, read/write_operations }`
- `dry_run`: `{ created:false, message, next_commands[] }` — if this says no adapter installed,
  that is the current state. Run the real install command before probing or answering.

If the dry-run errors, report the error verbatim and return to discovery (the URL/spec is the
problem, not the adapter).

---

## Auth Shape Matrix

Classify the auth shape before giving setup instructions. `auth.type` is the transport shape;
OAuth/DCR often still appears as a bearer token at runtime.

| Shape | Signals | Direction |
|------|---------|-----------|
| Static bearer/API key | `auth.type` is `bearer`, `api_key_header`, `api_key_query`, or `basic` with no OAuth metadata | Use masked credential handoff. If the user explicitly opts into a terminal command, use `rote token set <ENV> --stdin`. |
| OAuth client id/secret | OpenAPI `oauth2_schemes`, dry-run `auth_scoring`, or provider docs list authorization-code/client-credentials | Configure the OAuth scheme during create; surface client id/secret, redirect URI, scopes, and token URL requirements. |
| OAuth DCR / MCP PRM | Catalog entry is MCP, provider advertises dynamic client registration, or MCP handshake redirects to auth | Use `rote adapter new-from-mcp`, then `rote adapter reauth <id>` if needed. Do not ask for a static bearer token. |
| Google Discovery | Spec source is Google Discovery or adapter is a Google API | Use `rote oauth setup google --scopes ...`; do not ask for a pasted token. |
| Unknown installed bearer | Existing adapter reports bearer but the token provenance is unclear | Inspect `rote adapter list <id> --json --health` before advising. OAuth-backed bearers recover with reauth; static bearers use token set. |

---

## Stage 2 — Review

Show the user the headline facts from `spec` + `summary`: title, version, base_url,
`total_tools` and read/write split. **Only ask for a base-url override if `spec.base_url` is
empty** (some specs omit `servers`). Offer to set a custom display name/description only if the
user wants to.

---

## Stage 3 — Auth (resolve, don't punt)

Drive from the Auth Shape Matrix plus `auth` + `auth_scoring`. Do not assume `bearer` means
static token. Pick a branch from two fields — `has_ambiguity` and `recommended.confidence` —
then **resolve ambiguity yourself**: sharpen the guess from the provider's docs *when web tools
are available*, but the **probe → real call is the final arbiter**. Never hand the user a blind
multiple-choice, and never stall waiting on docs you can't fetch:

```
auth.type == "none"                                          → skip, no auth
has_ambiguity == false  AND  recommended.confidence >= 0.8   → CONFIRM (no research)
has_ambiguity == false  AND  recommended.confidence  < 0.8   → RESEARCH-LITE (verify the lone default)
has_ambiguity == true                                        → RESEARCH-FULL (disambiguate + setup)
auth.type == "per_operation"                                 → RESEARCH-FULL scoped to default_scheme
```

Low confidence is itself a trigger — a single-scheme guess at 0.5 (common for GraphQL specs:
`detail` says "no directives found") is rote *guessing*. Sharpen it up front when you can; when
you can't (no web access, ambiguous docs), install the most-likely scheme and let the first
real call correct it — a live auth error from the provider is the most authoritative signal
there is, no web round-trip needed.

GraphQL adapters are a canonical low-confidence case: GraphQL SDL often carries no security
directives, and some providers expect a raw `Authorization: <token>` header without the `Bearer `
prefix. When auth scoring is that ambiguous, research provider docs and assemble `--config-json`
explicitly instead of accepting the dry-run guess — and if research isn't possible, rely on the
probe → real call arbiter to correct the header form after install.

- **CONFIRM** → state the scheme and proceed, pre-filling the token env var from
  `auth.token_env` / `key_env`. e.g. "Detected bearer auth (0.95 confidence, from the spec).
  I'll use `STRIPE_TOKEN` — sound right?"
- **RESEARCH-LITE / RESEARCH-FULL** → run the **Auth research sub-routine** below, then present
  a *researched recommendation with setup steps inline* (not a blind menu).
- **`per_operation`** → research the `default_scheme`; you can create with it now and enable
  other `schemes` later via **rote-adapter-config** (`auth scheme`).

### Auth research sub-routine (read-only, pre-create)

Runs between Stage 1 (dry-run) and Stage 5 (create). Creates nothing, touches no secrets — it
only sharpens *which* `config-json` you assemble and *what instructions* the user gets.

1. **Derive search terms from context (never memory):** provider = `spec.title` / adapter id;
   candidate schemes = `auth_scoring.all_schemes[].auth_type.type`. Query templates:
   - `"<provider> API authentication" (api key OR oauth OR bearer)`
   - `"<provider> create API key"` / `"<provider> oauth2 client credentials"`
   - Prefer the provider's own docs domain (developer.x.com, docs.x.com) over blogs.

2. **Fetch the top authoritative hit** and extract a structured profile **per candidate scheme**:
   ```
   { scheme: bearer | api_key_header | api_key_query | oauth2_authorization_code |
             oauth2_client_credentials | basic,
     header_format: "Authorization: Bearer {token}" | "Authorization: {token}" |
                    "X-Api-Key: {token}" | ...,
     token_env:    <suggested env var>,
     how_to_obtain: [ ordered steps ],
     doc_url:      <source>,
     caveats:      [ "personal keys omit Bearer prefix", "key scoped per-workspace", ... ] }
   ```

3. **Reconcile research against the dry-run** — provider docs win over the dry-run guess, but
   cite `doc_url` for every claim:
   - **Confirms** the recommended scheme → proceed with it, now carrying the correct
     `header_format` + setup steps.
   - **Contradicts** it (dry-run guessed bearer, docs say `X-Api-Key`) → present both choices,
     lead with the doc-backed one, cite the source.
   - **Inconclusive, or research skipped/unavailable** → don't punt the scheme to the user.
     Install the most-likely scheme and let **Stage 6's probe → real call arbitrate**: on an
     auth-indicating `4xx` (read the body, not just the status), switch the header format
     (`Bearer` ↔ raw `Authorization`) and re-call. Ask the user only to obtain or paste a token.

### Present a researched recommendation, not a blind menu

Lead with the doc-backed scheme; put the setup steps in the option description so the user can
act without leaving the play.

**API-key shape:**
```
Recommended: API key in header  (developer.linear.app)
  Header:  Authorization: <key>     ← raw, no "Bearer " prefix
  Get one: linear.app/settings/api → "Create key" → Personal API key
  Env var: LINEAR_API_TOKEN
[ Use this ]  [ Use OAuth2 instead ]  [ Paste a different scheme ]
```

**OAuth2 shape** (the case people get stuck on — surface what they must register up front):
```
Recommended: OAuth2 authorization_code  (provider docs URL)
  Need: client_id, client_secret, redirect_uri, scopes [...]
  Register the app: <console URL from docs>
  rote runs: new-from-mcp / OAuth-DCR opens a browser
[ Walk me through OAuth setup ]  [ Use a static token if supported ]
```

### Guardrails

- **Read-only and pre-create.** Research never creates an adapter, never asks for or echoes a
  token value. It only changes the config-json and the instructions.
- **Docs are authoritative over the dry-run guess**, but every claim cites `doc_url`. The
  dry-run's `detail` ("no directives found") is the signal that rote is guessing — trust docs
  more there.
- **The token value is a secret** — see Stage 5 for the masked handoff. Research produces
  *how to obtain* it and the *header format*, never the value.
- **The probe → real call is the final gate — and the fallback arbiter.** Research raises
  confidence and yields correct setup, but `<id>_probe` only *finds* operations (local
  capability search — it comes back green even with broken auth). What actually proves auth is
  Stage 6's real `<id>_call` on the probe's top hit. When auth went in unverified (low
  confidence, no doc-backed confirmation), that call is **mandatory, not offered**: any `4xx`
  whose body reads as an auth/token problem (usually `401`/`403`, but some providers signal a
  wrong header scheme with `400`) means adjust the scheme (`rote adapter auth update`) and
  re-call — do not stop or ask the user until the call is green or the failure is provably
  token-side.

---

## Stage 4 — Toolsets (the "choosing tools" step)

The dry-run `toolsets[]` is the menu. **This is the only place toolsets are chosen** — there
is no post-creation toolset toggle.

- **Small APIs** (a handful of toolsets) → present them all (name · tool_count · methods) and
  let the user include/exclude.
- **Large APIs** (Stripe has 128 toolsets / 587 tools) → do NOT dump all of them. Summarize:
  "This API has {total_toolsets} toolsets / {total_tools} tools. Want all of them, or shall I
  narrow to specific areas?" Then let the user name areas (search `toolsets[].name`) or take
  all. Default to **all** if they don't care.

Build `toolset_filters` from the selection: `{"<toolset-name>": "<filter>", ...}` where
`<filter>` is `all` | `read-only` | `write-only` | `exclude` (omit the map to include
everything). For a read-oriented task, `read-only` on the selected toolsets is the safe default.

---

## Stage 5 — Create + post-ops

Assemble `--config-json` from the auth + toolset decisions and create:

```bash
rote adapter new <id> <spec-source> --yes --config-json '{"auth":{...},"toolset_filters":{...}}'
```

(Add `--base-url <url>` if Stage 2 required it, `--group <name>` if grouping.) For the
config-json shape, mirror the dry-run `auth` block for the chosen scheme.

**MCP servers** (Stage 0 flagged Spec Type "MCP") use a different create command — no spec
dry-run, OAuth/DCR runs during creation:
```bash
rote adapter new-from-mcp <id> <mcp-url> --headless
```
For OAuth-DCR servers (Notion) this opens a browser; tell the user to complete it.

### Post-create (offer each; run what the user wants)

- **Credentials** (static-token auth): hand off to the masked wizard — tell the user to run
  `rote powerpack credentials` in their own terminal (masked), or set one with
  `rote token set <ENV> --stdin` only as an explicit terminal opt-in. Never echo a token.
  (Same secrets discipline as the setup skill.)
- **Write guard**: `rote adapter guard init <id>`
- **Sensitivity**: `rote sensitivity upgrade` (if needed) then `rote sensitivity apply <id> --json`
- **Capability index**: `rote adapter capability rebuild`

> Adapter helper templates are opt-in. Do not generate or install one unless the user explicitly
> asks for a reusable adapter-specific helper.

---

## Stage 6 — Confirm + hand off

```bash
rote adapter info <id>
```

Show it's ready (tools/toolsets/auth). Offer to:
- **Prove it works** — probe, then a real call, inside a workspace (creation alone runs
  neither). The probe only *finds* the operation (local capability search — green even with
  broken auth); the **call's response is the auth verdict**. If auth went in unverified
  (Stage 3 RESEARCH branch, or research was skipped), both are **mandatory and you run them
  yourself** — don't just offer them:
  ```bash
  rote init proof --seq --force
  ```
  ```bash
  cd ${ROTE_HOME:-$HOME/.rote}/rote/workspaces/proof && rote <id>_probe "<query>"
  ```
  ```bash
  cd ${ROTE_HOME:-$HOME/.rote}/rote/workspaces/proof && rote <id>_call <top-probe-hit> '<minimal-args>'
  ```
  Read the call's verdict by failure class — judge auth by the response *body*, not the status
  code alone. Any whole-response `4xx` whose body points at the token, key, or header format is
  auth (usually `401`/`403`, but some providers return `400` for a wrong header scheme) — fix
  the scheme (`rote adapter auth update`) and re-call. A single *field* erroring inside an
  otherwise-green GraphQL response is **not** auth — don't touch the scheme; invoke the
  **rote-adapter-config** skill and use its GraphQL field filter
  (`rote adapter tool-spec <id> --add --action skip --field <Type.field>`), then re-call.

Then offer to:
- **Tune it** — invoke **rote-adapter-config** for base-url, auth schemes, sensitivity, etc.
- **Share it** — this is the schelling point for the registry. Hand off to **rote-registry**
  with the new adapter id: it checks whether the adapter already exists in the user's orgs,
  tells them if a push is needed, pushes at their chosen visibility, then surfaces org members
  and offers to invite others for review/use. Only offer this on a clean create.

**Closing line** (only on a clean create + green call): land one dry one-liner keyed to this
run's `total_tools`, e.g. "512 tools talking straight to the provider's API — no metered middleman
quietly billing you per call." Skip it if the run was rocky.

---

## Notes

- `--dry-run` creates nothing — safe to run repeatedly while discovering the right spec.
- A provider-side `HTTP 401/403` on dry-run means spec-fetch auth is required; runtime auth may still
  be separate after install. This is not proof the adapter is bad.
- After a successful create, suggest the main **rote** skill for day-to-day use
  (`rote play search "<intent>"` before any direct adapter call).

---

## Related skills

- **Next (tune):** `rote-adapter-config` — auth, base URL, write guard, sensitivity.
- **Next (share):** `rote-registry` — check/push to your orgs, then invite others.
