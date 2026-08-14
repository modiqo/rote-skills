---
name: rote-browse
description: "Use for browser automation through rote: browsing websites, attaching to active Chrome or Edge sessions, Gmail or other logged-in sites, headed/headless exploration, snapshots, readiness waits, page slices, ref rebasing, and crystallizing browser plays. Prefer rote browser primitives over direct Playwright calls."
---

# rote-browse

All `rote-<name>` references are companion skills, not CLI commands. Invoke them through the
runtime's skill mechanism; only literal `rote …` commands run in a terminal.

Use this skill for explicit browser intent: browse, open, inspect, snapshot, click, type, attach,
extract from a page, or use an existing logged-in browser. Do not replace browser work with native
web search, WebFetch, raw Playwright, `curl`, or shell scraping.

## Runtime Guidance

Load the smallest live guide that owns the current state:

```bash
rote guidance browser essential        # launch, observe, interact, recover refs
rote guidance browser auth             # login, CAPTCHA, MFA, saved sessions
rote guidance browser play-authoring   # crystallize and implement TypeScript replay
```

Runtime guidance is authoritative for commands and SDK examples. This skill chooses the route and
preserves the browser handoff.

## Decision Logic

1. **Choose launch shape.** If unspecified, ask headed/headless, attached/new session, then tab/window.
2. **Explore safely.** Follow the runtime guide's wait → snapshot → targeted query → action loop.
3. **Verify the result.** Read fresh page state or the requested artifact, not only command status.
4. **Classify replayability.** Decide one-off, private recall, or portable play candidate.
5. **Hand off reusable work.** Use `rote-flow-crystallization`; after its accepted or pre-approved
   handoff, `rote-flow-authoring` uses `rote guidance browser play-authoring` and materializes the
   play.

Do not start with generic play search when the user explicitly invoked browser automation. After the
launch shape is chosen, a reusable browser play may be offered as an option, but do not silently
replace the requested browser session with a saved play.

## Launch Choice

Use headed attach for existing login, Gmail, SSO/MFA, browser extensions, active tabs, or visual
debugging. Use an isolated headless session for public read-only work, CI, or replay.

Never silently choose headless for interactive work. Before personal browser, login, payment,
destructive, or transactional actions, confirm the intended scope and human gate.

## Browser Loop

Follow the selected browser runtime guide's current one-command-at-a-time loop. After navigation or
mutation, require fresh state and use targeted observations before full snapshots. Follow that guide
for current launch, wait, tab inspection, and stale-reference recovery commands. Do not guess when
current evidence finds zero or multiple safe targets.

## Attached-Tab Safety

When the attached target disappears, follow the essential runtime guide to inspect live inventory.
If live tab inventory shows only the Playwright extension connection or welcome tab, stop treating
it as the requested page and do not repeat the same attach attempt against the missing target.

- Activate another page tab only when its URL or origin matches the intended target exactly.
- If target identity or account context is ambiguous, ask the user to confirm or restore the
  intended page before activating anything.
- If no matching page tab is visible, ask the user to open or restore the intended page.
- If the target is missing twice for the same intended URL, stop retrying and return the blocker.

Before transactional actions, re-inspect through the live guide and verify the exact target URL or
origin, account context where relevant, and active lease.

## Human Gates

Stop and ask for human action for credentials, MFA, CAPTCHA, device approval, payment, destructive
actions, or account consent. Never store passwords, cookies, tokens, payment data, or authenticated
page contents in play code, notes, logs, or chat.

For authenticated replay, follow the auth runtime guide to save session state before
crystallization. Pending save infers the session dependency. For direct authoring, follow
`rote guidance browser play-authoring` instead of constructing frontmatter here.

## Reusable Browser Work

A browser procedure is reusable when it has parameterizable inputs, repeatable actions, readiness
conditions, a stable output, and likely future value. Before persistence or handoff, minimize and
redact browser artifacts: exclude PII, bearer tokens, payment data, credentials, and other secrets
from snapshots, slices, screenshots, actions, and notes. Preserve only what replay requires:

- Workspace name/path and launch shape.
- Sanitized intended origin and, only when safe, a URL with sensitive query or fragment data removed.
- Opaque response or page references when needed, without sensitive associated contents.
- Parameters that must replace one-off URLs, values, names, or dates.
- Sanitized expected output and verification evidence.
- Opaque saved-auth profile identifier and status, human gates, and portability limits.

Then hand off to `rote-flow-crystallization`; this skill never executes the emitted scaffold. After
crystallization resolves acceptance or pre-approval, `rote-flow-authoring` follows
`rote guidance browser play-authoring`, executes the scaffold or export, and owns tests, lint,
release, index/search verification, and pending cleanup.

## Handoff Summary

For subagent, compaction, or interruption boundaries, write a short markdown summary containing the
fields below in the workspace or artifact location named by the caller.

## Handoff Packet

Return:

- Origin and next skill.
- User-visible browser goal.
- Workspace path.
- Launch shape and lease/tab state.
- Only sanitized browser artifacts and opaque response/page references needed to resume.
- Opaque saved-auth profile identifier and status plus human-gate state.
- Sanitized verified result or artifact.
- Redaction confirmation covering PII, tokens, payment data, credentials, and other secrets.
- Replayability: one-off, private recall, or portable candidate.
- Blocker or exact next guidance/skill.

## Handoff Contract

- Use when: browser navigation, observation, interaction, attach, authentication state, or browser
  replay is required.
- Owns: launch choice, page lease, waits, snapshots, slices, ref recovery, and human gates.
- Hands off to: `rote-flow-crystallization` for reusable captured work; `rote-flow-authoring` through
  crystallization's accepted/pre-approved handoff; `rote-registry` only for an already released
  artifact that needs sharing; `rote` when substrate routing changes.
- Stop when: the result is verified, human action is required, the target cannot be recovered, an
  unsafe action lacks approval, or crystallization becomes the next owner.
- Completion signal: verified browser result plus browser state, replayability, and next owner.
