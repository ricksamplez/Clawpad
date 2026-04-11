# Clawpad — Specification

## 1. Overview

**Clawpad** is a lightweight, DM-only, session-scoped working notebook for OpenClaw.
It stores **explicit working state** for the current interactive direct-message session only.

Clawpad is designed to behave like a small human notebook:

* hold the current objective
* hold the working plan
* hold relevant temporary facts
* hold open loops
* hold the next immediate actions
* park follow-up ideas
* record recently completed items

Clawpad is intentionally small.
It must not replace existing OpenClaw systems.

---

## 2. Compatibility and Runtime Assumptions

### 2.1 Target Runtime

This specification targets **OpenClaw `>= 2026.4.10`**.

Clawpad is implemented as a **native OpenClaw plugin**:

* package root contains `openclaw.plugin.json`
* runtime entry uses the current Plugin SDK entry model
* plugin registers tools and hooks through `register(api)`
* prompt mutation uses the current plugin hook surface, not legacy prompt-patching assumptions

### 2.2 Plugin Shape

Clawpad is a **non-provider native plugin** that registers:

* agent tools
* plugin hooks
* no model provider
* no channel provider
* no context engine
* no background indexing service

### 2.3 Session Model Assumption

Clawpad follows **OpenClaw session boundaries**, not raw transcript continuity assumptions.

Important runtime fact:

* OpenClaw sessions are bucketed by **`sessionKey`**
* the active transcript instance inside that bucket is identified by **`sessionId`**
* resets and expirations create a **new session instance** while preserving the broader routing bucket semantics

Clawpad therefore binds to:

* `agentId`
* `sessionKey`
* current active `sessionId`

### 2.4 DM Isolation Safety Assumption

OpenClaw defaults to **shared DM sessions** (`session.dmScope = "main"`) unless configured otherwise.
That default is acceptable for single-user installs, but unsafe for multi-user DM environments.

Clawpad must therefore treat DM scope explicitly:

* **recommended / safe mode**: `per-peer`, `per-channel-peer`, or `per-account-channel-peer`
* **single-user compatibility mode**: `main`

Clawpad should default to **fail closed** for shared-main DM notebooks unless the operator explicitly allows that mode.

---

## 3. Non-Goals

Clawpad is **not**:

* long-term memory
* semantic memory
* retrieval
* keyword search
* transcript storage
* context engine
* compaction logic
* Honcho replacement
* builtin memory replacement
* QMD replacement
* Lossless Claw replacement
* subagent state manager
* cron/task notebook system

Clawpad is a **working-state layer**, not a durable knowledge layer.

---

## 4. Core Principles

1. **DM-only**
   Clawpad exists only for eligible direct-message sessions.

2. **Session-scoped**
   A notebook belongs to one effective OpenClaw DM session lineage (`sessionKey`) and one current active session instance (`sessionId`).

3. **Ephemeral by default**
   Clawpad state is disposable unless explicitly promoted elsewhere.

4. **Explicit, not automatic**
   If something should survive the session, the agent must explicitly promote it to `MEMORY.md`.

5. **Simple over clever**
   No retrieval, embeddings, semantic matching, or hidden ranking.

6. **Small prompt footprint**
   The injected overlay must be compact, bounded, and predictable.

7. **Human-readable state**
   The notebook is stored as Markdown and remains manually inspectable.

8. **No responsibility overlap**
   Clawpad must coexist with builtin memory, `MEMORY.md`, daily notes, QMD, context engines, Honcho, and Lossless Claw without duplicating their jobs.

9. **Safe by deployment default**
   If OpenClaw is configured in a way that would merge unrelated DM users into one notebook, Clawpad should refuse to operate unless the operator knowingly opts in.

---

## 5. Operating Model

### 5.1 High-Level Model

Clawpad is a plugin-owned notebook system with four parts:

1. **Eligibility gate**
   Decide whether the current run is allowed to use Clawpad.

2. **Notebook store**
   Read/write the live notebook and archive snapshots.

3. **Prompt overlay hook**
   Inject a compact working-state summary before prompt submission.

4. **Agent tools**
   Allow structured reads, writes, reset, and promotion.

### 5.2 Source of Truth

The **live notebook file** is the source of truth for Clawpad state.

The overlay is only a rendered view.
Tool reads reflect notebook file state.

### 5.3 Plugin Registration Contract

Clawpad registers through the native Plugin SDK entry surface and exposes:

* `clawpad_read`
* `clawpad_write`
* `clawpad_reset`
* `clawpad_promote`
* `before_prompt_build` hook
* `/new` handling hook
* `/reset` handling hook

Optional runtime helper modules may exist internally, but the public behavioral contract is defined by this spec.

---

## 6. Eligibility and Safety Gates

### 6.1 Eligible Runs

Clawpad is enabled only for **top-level, interactive, direct-message sessions**.

An eligible run must satisfy all of the following:

* chat/session kind is **direct / DM**
* run belongs to the main interactive agent loop
* current session is not a group, room, or channel session
* current session is not a slash-command-only session
* current session is not a cron/task/webhook session
* current session is not a spawned subagent session
* current DM scope passes the configured safety rule

### 6.2 Ineligible Runs

Clawpad is disabled for:

* group chats
* group DMs / MPIM-style sessions
* rooms
* channels
* guild/server channels
* slash-command helper sessions
* cron sessions
* webhook sessions
* subagent sessions
* detached background task sessions
* any session kind the plugin cannot confidently classify as eligible DM

### 6.3 DM Isolation Gate

Clawpad must evaluate the active OpenClaw DM scope.

#### Safe / recommended scopes

* `per-peer`
* `per-channel-peer`
* `per-account-channel-peer`

#### Shared-main scope

* `main`

When scope is `main`, Clawpad must assume that all DMs may share one session unless explicitly configured otherwise.

### 6.4 Default Safety Rule

Default behavior:

* if DM scope is isolated -> Clawpad may run
* if DM scope is `main` -> Clawpad is **disabled by default**
* operator may explicitly override this for trusted single-user setups

### 6.5 Detection Precedence

Eligibility should be decided in this order:

1. explicit session/run metadata from OpenClaw runtime
2. explicit route/chat type metadata
3. session tool/runtime flags that mark subagent, command, cron, webhook, or background runs
4. conservative session-key fallback classification only if richer metadata is unavailable

### 6.6 Conservative Fallback Rule

If Clawpad cannot confidently prove the run is an eligible DM run, it must disable itself for that run.

---

## 7. Session Binding Model

### 7.1 Binding Unit

A Clawpad notebook is bound to the effective OpenClaw DM session bucket:

* `agentId`
* `sessionKey`

and stores the currently active transcript instance:

* `sessionId`

### 7.2 Why This Binding Exists

OpenClaw routes chat continuity and concurrency through `sessionKey`, while the active transcript instance is represented by `sessionId`.

This means:

* archives for repeated sessions in the same DM lineage stay grouped together
* a new active session after `/new`, `/reset`, idle reset, or daily reset can be detected cleanly
* `last_session` can mean “latest archived notebook for this same DM lineage”

### 7.3 Shared-Main Scope Implication

If the operator allows `session.dmScope = "main"`, Clawpad becomes a notebook for the shared main DM session bucket, not for an individual peer.

That behavior is supported only as an explicit compatibility mode and must be considered unsafe for multi-user DM deployments.

---

## 8. Lifecycle

### 8.1 First Eligible Turn

On first eligible use for a session bucket:

* resolve notebook path from `agentId + sessionKey`
* if no live notebook exists, create one
* initialize notebook structure
* set frontmatter metadata

### 8.2 Normal Operation

During normal eligible session operation:

* live notebook file is current source of truth
* agent tools update notebook content
* `before_prompt_build` injects compact overlay
* live notebook remains attached to the current active `sessionId`

### 8.3 Session Instance Rollover

If the current runtime `sessionId` differs from the live notebook `sessionId`, Clawpad must treat this as a **new session instance** within the same session bucket.

Required behavior:

* archive old notebook
* create fresh notebook
* update frontmatter with new `sessionId`
* do not carry over content automatically
* do not auto-promote to memory

This rule covers:

* daily reset
* idle reset
* manual reset where hooks did not already handle the transition
* any other session-instance change observed lazily on the next eligible turn

### 8.4 `/new`

When `/new` occurs for an eligible DM session:

* archive current notebook
* create fresh empty notebook
* bind fresh notebook to the new active `sessionId`
* do **not** carry over content
* do **not** run notebook handoff prompts
* do **not** auto-write memory

### 8.5 `/reset`

When `/reset` occurs for an eligible DM session:

* archive current notebook
* create fresh empty notebook
* bind fresh notebook to the new active `sessionId`
* do **not** carry over content
* do **not** run notebook handoff prompts
* do **not** auto-write memory

### 8.6 `clawpad_reset`

The explicit `clawpad_reset` tool:

* archives current notebook unless disabled for that call
* creates a fresh notebook
* may optionally carry over selected sections

This is the **only supported carry-over path**.

### 8.7 Empty-Notebook Policy

A freshly created notebook may be structurally empty except for headings and metadata.

The overlay may render a compact empty-state block.

### 8.8 Ephemeral Guarantee

If the agent does not explicitly promote an item before the session ends, that item is disposable.
Archived notebooks exist only as recent session snapshots, not as a long-term memory system.

---

## 9. Notebook Structure

### 9.1 Required Sections

Each notebook contains exactly these sections:

* Objective
* Plan
* Working Facts
* Open Loops
* Next Actions
* Parking Lot
* Done

### 9.2 Section Semantics

#### Objective

Current top-level objective for the session. Usually one main entry.

#### Plan

Ordered execution outline for the current task.

#### Working Facts

Temporary facts, constraints, findings, or state relevant to current work.

#### Open Loops

Questions, blockers, missing information, unresolved checks.

#### Next Actions

Immediate next moves. The next concrete action must be first.

#### Parking Lot

Useful but non-immediate follow-ups, later ideas, deferred points.

#### Done

Recently completed actions or closed loops.

---

## 10. Ordering Rules

Clawpad is not a random bullet dump. The agent must preserve logical ordering.

### Objective

* primary current objective first
* usually one active entry

### Plan

* ordered top-down in execution order

### Working Facts

* most relevant current facts and constraints first

### Open Loops

* most urgent / blocking / relevant unresolved loop first

### Next Actions

* immediate next action always at the top
* remaining actions follow in short execution order

### Parking Lot

* most likely future follow-up or most relevant parked item first

### Done

* newest completed item always at the top

### Global Rule

Operations must preserve or re-establish logical ordering after each write.

---

## 11. Internal Item IDs

Each notebook bullet must have a stable short internal ID.

Purpose:

* reliable updates
* reliable deletes
* safe `mark_done`
* safe promotion
* avoid raw text matching ambiguity

### 11.1 Example Bullet

```md
- [cp_a17f] Benchmark pricing model again later
```

### 11.2 Rules

* IDs are generated by Clawpad, not the model text itself
* IDs remain stable until the bullet is deleted or moved out of the notebook
* moving an item to `Done` keeps its ID
* promotion may strip the source ID in memory text

### 11.3 Display Rule

* compact overlay may omit IDs
* full reads should include IDs
* tool operations should use IDs whenever possible

---

## 12. File Format

### 12.1 Live Notebook Format

Markdown with compact YAML frontmatter.

Example:

```md
---
schema_version: 1
plugin_id: clawpad
openclaw_min_version: 2026.4.10
agent_id: main
session_key_hash: abc123
session_id: sess_01ABCDEF
session_scope: dm
dm_scope_mode: per-channel-peer
created_at: 2026-04-09T12:00:00Z
updated_at: 2026-04-09T12:00:00Z
status: active
---

# CLAWPAD

## Objective
- [cp_0e3a] Ship Clawpad

## Plan
- [cp_36ab] Finalize spec
- [cp_be7c] Build scaffold

## Working Facts
- [cp_9d02] DM only

## Open Loops
- [cp_c340] Confirm hook placement

## Next Actions
- [cp_bab2] Implement file layer

## Parking Lot
- [cp_3daf] Add archive lookup tests

## Done
- [cp_b68e] Finalized plugin naming
```

### 12.2 Required Frontmatter Fields

* `schema_version`
* `plugin_id`
* `openclaw_min_version`
* `agent_id`
* `session_key_hash`
* `session_id`
* `session_scope`
* `dm_scope_mode`
* `created_at`
* `updated_at`
* `status`

### 12.3 Optional Frontmatter Fields

Optional fields may include:

* `archived_at`
* `archive_reason`
* `notes`

### 12.4 Archive Format

Archive files use the same Markdown structure as the live notebook, but frontmatter `status` must be `archived`.

---

## 13. Paths

Recommended plugin-owned workspace paths:

```text
<workspace>/.openclaw/plugins/clawpad/<agentId>/<sessionKeyHash>/CLAWPAD.md
<workspace>/.openclaw/plugins/clawpad/<agentId>/<sessionKeyHash>/archive/
<workspace>/.openclaw/plugins/clawpad/<agentId>/<sessionKeyHash>/.reset-lock.json
```

### 13.1 Live File

`CLAWPAD.md` is the current notebook for the effective DM session bucket.

### 13.2 Archive Directory

The archive directory stores past notebook snapshots for the same session bucket.

### 13.3 Lock File

A lock file is used to prevent duplicate archive/reset handling for `/new` and `/reset`.

### 13.4 QMD / Indexing Boundary

Plugin-owned notebook paths must be treated as **non-memory, non-QMD content**.
They must not be added to memory or QMD indexing inputs.

---

## 14. Overlay Injection

### 14.1 Injection Point

Clawpad injects a compact working-state overlay through the **`before_prompt_build`** plugin hook.

### 14.2 Injection Mode

Default injection mode is **dynamic per-turn context**, not stable workspace bootstrap text.

Clawpad should inject through the per-turn prompt addition path rather than pretending to be a permanent workspace file.

### 14.3 Why Dynamic Injection Is Required

Clawpad state changes frequently.
A notebook overlay is working-state, not durable operating policy.
Therefore it must be injected as bounded, current, per-turn context.

### 14.4 Overlay Content

The overlay includes a compact view of notebook sections.
Each section shows:

* section title
* total item count in square brackets
* visible truncated bullets
* `… (+N more)` if additional hidden items exist

Example:

```md
## Parking Lot [6]
- Entry 1
- Entry 2
- Entry 3
- … (+3 more)
```

### 14.5 Overlay Rules

* compact and predictable
* bounded by fixed per-section limits
* IDs omitted by default
* newest `Done` items visible first
* top `Next Action` visible first
* no full notebook dump
* no archive content included

### 14.6 Default Per-Section Limits

* Objective: 3
* Plan: 5
* Working Facts: 6
* Open Loops: 6
* Next Actions: 5
* Parking Lot: 4
* Done: 3

### 14.7 Overlay Purpose

The overlay is a **working-state reminder**, not a storage format and not a full notebook render.
If deeper inspection is needed, the agent uses `clawpad_read`.

---

## 15. Tools

Clawpad provides four tools.

### 15.1 `clawpad_read`

Reads notebook content.

#### Purpose

Used when the agent needs to inspect current or previous notebook state.

#### Parameters

* `source`: `"current" | "last_session"`
* `section?`
* `full?`

#### Behavior

* `source: "current"` reads the live notebook
* `source: "last_session"` reads the newest archive for the same session bucket
* if `section` is provided, return only that section
* if `full` is true, return the full notebook
* if no archive exists for `last_session`, return a clear empty result

#### Notes

* no retrieval
* no semantic search
* no browsing beyond the newest archive

---

### 15.2 `clawpad_write`

Updates notebook content.

#### Purpose

Used for structured section edits.

#### Operations

* `replace`
* `append`
* `prepend`
* `clear`
* `remove_bullet`
* `mark_done`

#### Parameters

* `section`
* `operation`
* `content?`
* `items?`
* `item_ids?`
* `source_section?`

#### Behavior

##### `replace`

Replace section contents with provided items.
New items receive IDs if missing.

##### `append`

Append items to section, then normalize order if needed.
New items receive IDs if missing.

##### `prepend`

Prepend items to section, then normalize order if needed.
New items receive IDs if missing.

##### `clear`

Remove all items from section.

##### `remove_bullet`

Remove selected items from section.
Prefer ID-based selection.

##### `mark_done`

Move selected item(s) from a source section to `Done`.
Rules:

* insert at top of `Done`
* preserve newest-first order in `Done`
* remove from source section
* keep original item IDs

#### Notes

* all writes must be atomic
* ordering rules must be preserved after each write
* text-matching deletion should be a last resort if IDs are unavailable

---

### 15.3 `clawpad_reset`

Creates a fresh notebook through an explicit tool action.

#### Purpose

Manual notebook reset initiated by the agent.

#### Parameters

* `archive?`
* `carry_over?`

#### Behavior

* archive current notebook unless disabled
* create fresh notebook
* optionally copy selected allowed sections into the new notebook
* bind notebook to current `sessionId`

#### Carry-Over Rule

Optional carry-over is allowed **only here**.
This is not used by `/new`, `/reset`, daily reset, or idle reset.

---

### 15.4 `clawpad_promote`

Moves notebook items to `MEMORY.md`.

#### Purpose

Used when the agent decides that an item should outlive the current session.

#### Parameters

* `item_ids`
* `target_section?` default: `Clawpad Carryover`
* `create_section?` default: `true`
* `remove_from_clawpad?` default: `true`

#### Behavior

* read selected notebook items
* open or create `MEMORY.md`
* ensure target section exists
* append items under the target section
* perform simple exact dedupe
* write `MEMORY.md`
* remove items from notebook if configured and write succeeded

#### Notes

* all writes must be atomic
* promotion is explicit and manual
* no auto-promotion via hooks
* no direct write to Honcho
* no hidden memory behavior

---

## 16. Promotion Rules

### 16.1 When to Promote

Promote only when a notebook item should survive the session, such as:

* durable constraint
* recurring relevant fact
* persistent preference
* future-relevant follow-up worth keeping
* longer-lived open item that should persist outside the session

### 16.2 When Not to Promote

Do not promote:

* temporary work notes
* one-off tool findings with no future value
* transient planning scratch
* disposable open loops
* anything already captured elsewhere

### 16.3 Memory Target

Default target is:

* file: `MEMORY.md`
* section: `Clawpad Carryover`

### 16.4 Promotion Effect

After successful promotion, the promoted notebook items should be removed from Clawpad by default.

### 16.5 Why Manual Promotion Exists

This keeps Clawpad ephemeral and prevents it from becoming a hidden second memory system.

---

## 17. Reset and Archive Behavior

### 17.1 Archive Triggers

Clawpad archives the current notebook on any of the following:

* `/new`
* `/reset`
* explicit `clawpad_reset` with archive enabled
* observed `sessionId` rollover inside the same `sessionKey`

### 17.2 Archive Naming

Recommended archive filename pattern:

```text
YYYY-MM-DDTHH-mm-ssZ--<reason>.md
```

Examples:

```text
2026-04-09T12-00-00Z--manual-new.md
2026-04-10T04-00-03Z--daily-rollover.md
2026-04-10T16-42-12Z--idle-rollover.md
```

### 17.3 `last_session` Semantics

`clawpad_read(source="last_session")` returns the newest archive from the same `sessionKey` lineage, not a cross-route history search.

### 17.4 Why Archive Exists

The archive exists so the agent can re-check the immediately previous session notebook if needed.
It does not function as a searchable notebook history system.

---

## 18. Write Safety and Robustness

### 18.1 Atomic Writes

Notebook and memory writes must be atomic:

* write temp file
* fsync when appropriate
* rename temp file to target file

### 18.2 Locking

`/new` and `/reset` archive handling should use a lock file or equivalent guard to avoid duplicate execution.

### 18.3 Partial Failure Rule

If promotion to `MEMORY.md` fails, the source notebook items must not be deleted.

### 18.4 Deterministic Operations

Use stable IDs wherever possible instead of raw text matching.

### 18.5 Corruption Handling

If the notebook file is malformed:

* never silently discard content
* prefer safe parse failure
* preserve the broken file to a recovery archive
* create a fresh notebook only after preserving the damaged state

---

## 19. Configuration Surface

Clawpad should define a small plugin config surface under:

```json
plugins.entries.clawpad.config
```

Recommended fields:

```json
{
  "enabled": true,
  "requireIsolatedDmScope": true,
  "allowMainDmScope": false,
  "injectWhenEmpty": false,
  "maxArchiveFiles": 20,
  "overlay": {
    "objective": 3,
    "plan": 5,
    "workingFacts": 6,
    "openLoops": 6,
    "nextActions": 5,
    "parkingLot": 4,
    "done": 3
  },
  "promotion": {
    "defaultTargetSection": "Clawpad Carryover",
    "removeFromClawpadByDefault": true
  }
}
```

---

## 20. Interoperability

### 20.1 OpenClaw Session Store

Clawpad must not modify OpenClaw session transcripts or `sessions.json` directly.
It is an adjacent plugin-owned state layer.

### 20.2 `MEMORY.md`

Clawpad may explicitly write to `MEMORY.md` only through `clawpad_promote`.
It must not rely on hidden auto-capture.

### 20.3 Daily Notes / Memory Search

Clawpad is not a replacement for:

* `MEMORY.md`
* `memory/YYYY-MM-DD.md`
* memory search
* memory promotion pipelines

### 20.4 QMD

Clawpad must coexist with QMD, but notebook files must not be indexed as QMD inputs.

### 20.5 Honcho

Clawpad must work alongside Honcho.
It must not write directly to Honcho.
It must not assume Honcho exists.

### 20.6 Lossless Claw / Context Engines

Clawpad must work alongside Lossless Claw and other context engines.
It must not duplicate context-engine responsibilities.
It must not assume a specific context engine exists.

### 20.7 Subagents

Clawpad is disabled for subagent sessions by default.
Subagents may still call Clawpad tools only if the parent/top-level session remains the notebook owner and the runtime explicitly supports that safely.
Default v1 behavior will keep Clawpad attached to top-level DM runs only.

### 20.8 Minimal Dependency Assumption

Clawpad should remain fully usable with only:

* local Markdown files
* plugin tools
* prompt overlay hook
* session boundary handling

### 20.9 Active Memory

Clawpad must coexist with the optional Active Memory plugin.

It must not:

- register a memory capability
- override the active memory slot
- depend on active memory being present
- assume recall/promotion/dreaming ownership

Clawpad remains a separate working-state layer.
If Active Memory is enabled, Clawpad overlay injection and tools must still behave independently and predictably.

---

## 21. Agent Policy

The agent must understand the following rules.

### 21.1 Core Rule

Clawpad is temporary.
If a point should survive the session, the agent must explicitly move it to `MEMORY.md`.
If not promoted, it dies with the session.

### 21.2 Usage Rule

Use Clawpad for:

* active working state
* short plans
* temporary facts
* open loops
* next actions
* parked follow-ups
* recently done items

Do not use Clawpad for:

* durable knowledge store
* hidden user profiling
* transcript logging
* large historical dumps
* auto-memory accumulation

### 21.3 Promotion Rule

If the agent notices that a notebook item has become persistently relevant, it should promote it to `MEMORY.md` and remove it from Clawpad.

### 21.4 Session-Boundary Rule

The agent must assume that Clawpad is reset whenever the current OpenClaw session instance changes.
Manual resets are only one subset of session boundaries.

### 21.5 Safety Rule

If the runtime is in shared-main DM mode and Clawpad safety override is not enabled, the agent must behave as if Clawpad does not exist for that run.

---

## 22. Plugin Tree

```text
clawpad/
├── package.json
├── tsconfig.json
├── openclaw.plugin.json
├── index.ts
├── docs/
│   └── clawpad-spec.md
├── src/
│   ├── plugin.ts
│   ├── constants.ts
│   ├── types.ts
│   ├── config.ts
│   ├── eligibility.ts
│   ├── binding.ts
│   ├── ids.ts
│   ├── paths.ts
│   ├── frontmatter.ts
│   ├── parser.ts
│   ├── render.ts
│   ├── normalize.ts
│   ├── notebook-store.ts
│   ├── archive-store.ts
│   ├── memory-store.ts
│   ├── hooks/
│   │   ├── before-prompt-build.ts
│   │   ├── on-command-new.ts
│   │   ├── on-command-reset.ts
│   │   └── ensure-session-rollover.ts
│   └── tools/
│       ├── read.ts
│       ├── write.ts
│       ├── reset.ts
│       └── promote.ts
└── tests/
    ├── parser.test.ts
    ├── render.test.ts
    ├── write.test.ts
    ├── promote.test.ts
    ├── archive.test.ts
    ├── eligibility.test.ts
    └── rollover.test.ts
```

---

## 23. Implementation Order

1. finalize this spec
2. scaffold native plugin package and manifest
3. define config schema and constants
4. implement eligibility and DM-safety gating
5. implement session binding and path resolution
6. implement parser, frontmatter, and renderer
7. implement notebook I/O and atomic writes
8. implement rollover detection by `sessionId`
9. implement `clawpad_read`
10. implement `clawpad_write`
11. implement `clawpad_reset`
12. implement `clawpad_promote`
13. implement `before_prompt_build` overlay injection
14. implement `/new` and `/reset` hooks
15. write tests
16. run DM smoke tests on OpenClaw `2026.4.10+`

---

## 24. Acceptance Criteria

Clawpad is complete when all of the following are true:

* native plugin loads on OpenClaw `2026.4.10+`
* plugin exposes four tools
* plugin injects overlay through `before_prompt_build`
* overlay stays bounded and compact
* DM-eligible sessions get a live notebook
* group/channel/cron/webhook/subagent sessions do not get a notebook
* unsafe shared-main DM mode is blocked by default
* isolated DM scope works correctly
* live notebook path is derived from `agentId + sessionKey`
* session rollovers are detected via `sessionId`
* `/new` archives and clears without carry-over
* `/reset` archives and clears without carry-over
* `clawpad_reset` works with optional carry-over
* section counts render correctly
* `Done` shows newest-first
* `Next Actions` keeps next step on top
* `clawpad_read(source="last_session")` returns newest archive in the same session lineage
* `clawpad_write(mark_done)` works correctly
* `clawpad_promote` writes to `MEMORY.md` and removes promoted items after success
* promotion failure does not delete source notebook items
* Clawpad does not require Honcho, QMD, or Lossless Claw
* Clawpad does not clash with them when present

---

## 25. Short Summary

Clawpad is a **DM-only, session-scoped, explicit working-state notebook** for OpenClaw.
It is a small native plugin that uses local Markdown files, bounded prompt overlay injection, and explicit tools.

It follows OpenClaw’s current runtime model:

* native plugin registration
* hook-based prompt injection
* session-key lineage
* session-id rollover detection
* DM-scope safety handling

Clawpad is intentionally simple.
It helps the agent stay organized during the current DM session without turning into another hidden memory system.

If something matters beyond the session, the agent must explicitly move it into `MEMORY.md`.
If not, it dies with the session.
