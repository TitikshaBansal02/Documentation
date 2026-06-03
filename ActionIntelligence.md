# Action Intelligence — Deep Dive

> **Audience:** anyone who wants to understand the embed-agent UI-action system end to end — not just the wire format or the file layout, but *what each piece does, why it exists, and how the bytes flow*.
>
> **Companion docs (read in this order):**
> - [`action-intelligence-backend-migration.md`](./action-intelligence-backend-migration.md) — historical: how the backend migrated from `ui_action`/`nodes_compact` to the mission protocol. Read for context on *why* the current shape exists.
> - [`action-intelligence.md`](./action-intelligence.md) — forward-looking gap analysis + roadmap. Read for *what's broken and what to build next*.
> - **This doc** — the explanation. Read to actually *understand* the system.
>
> **State this doc reflects:**
> - `revrag-core`: `app/modules/pipeline/` working tree, today.
> - `embed-flutter`: branch `Architectur_Change_Action_intelligence` (HEAD `fe03a3f`, 2026-06-01) — *not* `master`/v0.0.17. The action-intelligence SDK code is unmerged.

---

## 0. TL;DR (read this first)

Action Intelligence is the loop that lets the voice agent **act on the user's phone screen** instead of only talking about it. User says *"open my orders"*; the agent identifies a target in the live UI tree, dispatches a "mission" over LiveKit's data channel, and the Flutter SDK on the device finds-and-scrolls-to-and-taps the right widget — then reports back what happened.

There are two ends, one channel:

```
revrag-core (Python, in-pipeline LLM agent)   ←─ LiveKit reliable data channel (JSON) ─→   embed-flutter SDK (Dart, on device)
```

The current state is a **production-capable pipeline with one big architectural hole**: the SDK reports rich outcomes (status, phase, verification evidence) every step, but the backend only **logs** them — it never feeds them back to the LLM. So the agent confidently says "Opening orders!" even when the SDK already knows the mission failed. Closing that loop is Gap 1 / Roadmap R1 — the single highest-impact thing on the backlog.

This doc walks through both ends file-by-file, then lists every gap, drift, and bug found in the deep read.

---

## 1. What Action Intelligence is (the product)

When a user on a mobile app says *"open my orders"* or *"fill my name as Rohit"*, the agent has two choices:

1. **Talk about it.** "Sure, you'll find Orders in the menu." — useless if the user can't navigate.
2. **Do it.** Find the Orders tab in the live UI, scroll to it if needed, tap it, verify the screen changed. Then *narrate* — *"Opening orders…"* → *"Done."*

Option 2 is action intelligence. The hard parts:

- The voice LLM doesn't see the screen. It needs a textual ground truth — a "viewport block" describing what's currently on the device.
- That viewport may not contain the target (it's below the fold, on another screen, behind a tab swap). The SDK must scroll and search.
- The agent can't *trust* the UI hierarchy — it must re-verify that the tap *actually changed something* (route changed? button disappeared? new dialog appeared?).
- Mobile UI text changes language/casing/spelling. The match algorithm has to be tolerant — "Welcome Back" should match "welcome\nback".
- Stable references matter. A tap on a row whose ID changes between captures must not silently target the wrong row.

The mission-protocol architecture (current state) solves all of this:

- The SDK captures every visible widget and assigns a **stable ID** to each. The viewport block lists these IDs.
- The LLM picks an action verb + a target (stable ID *or* free-text description) via the `perform_ui_action` tool.
- The backend wraps that into a **mission** (one or more `MissionStep`s) and ships it over LiveKit's reliable data channel.
- The SDK's `MissionRunner` executes each step in three phases — **searching → executing → verifying** — and emits a wire event at each phase boundary plus terminal status.

---

## 2. The two repos and how they link

| Repo | Role | Lives in |
|---|---|---|
| **`revrag-core`** | Python voice pipeline (LiveKit worker). Holds the LLM agent, the tool, the dispatcher, the typed snapshot state. | `app/modules/pipeline/` |
| **`embed-flutter`** | Dart SDK embedded in the host mobile app. Captures the live Flutter SemanticsNode tree, runs missions, verifies outcomes. | `lib/core/` *(on the action-intelligence branch — NOT on `master`)* |

**The branch caveat.** The published SDK (`master`, v0.0.17) does **not** contain any of the action-intelligence code. The entire pipeline (`lib/core/protocol/`, `lib/core/tree/`, `lib/core/runtime/`, etc.) lives only on the branch `Architectur_Change_Action_intelligence` (last commit 2026-06-01). When you read this doc and the SDK files don't exist on disk, you're on the wrong branch — `git checkout Architectur_Change_Action_intelligence`.

**The transport.**

```
Backend side (Python)                                  SDK side (Dart)
─────────────────────                                  ────────────────────

LiveKit Server API:                                    LiveKit Client:
  async with api.LiveKitAPI() as lk_api:                 room.localParticipant
      await lk_api.room.send_data(                          .publishData(bytes, reliable:true)
          api.SendDataRequest(                              … and …
              room=room_name,                            room.onData(callback)
              data=mission_json.encode("utf-8"),
              kind=api.DataPacket.Kind.RELIABLE,
          )
      )
```

Both sides serialize UTF-8 JSON with a top-level `"type"` discriminator field. The reliable channel guarantees ordered, exactly-once delivery (we never have to worry about a `mission_status: succeeded` arriving before `mission_status: started`).

---

## 3. End-to-end flow — walking through *"open my orders"*

Concrete trace of what happens when the user speaks that phrase. Numbers correspond to the diagram further down.

```
                                       ┌───── MOBILE APP (Dart SDK on the device) ─────┐
USER (speaks)                          │                                                │
    │                                  │  (1) SnapshotPump already pushed a baseline    │
    │                                  │      on LiveKit connect, and diffs every       │
    │                                  │      route/scroll change since.                │
    ▼                                  │                                                │
LiveKit audio track                    │                                                │
    │                                  └────────────────────────────────────────────────┘
    ▼
STT → LLM (in the pipeline)
    │
    │  (2) Each LLM turn, the backend agent injects the current
    │      viewport block into the system prompt — see §4.4.
    │
    ▼
LLM calls perform_ui_action(
   action="navigate",
   target_description="Orders",
   target_stable_id="main.tabs.orders",   ← LLM copied this from the viewport block
   target_role="tab"
)
    │
    │  (3) Backend's _dispatch_mission runs its 5-stage funnel:
    │      action coercion → snapshot resolution → override guards →
    │      blind dispatch fallback → empty-target fallback.
    │
    │  (4) Build mission JSON via build_mission_message(steps=[step]).
    │      Send over LiveKit reliable data channel.
    ▼
{"type":"mission","mission_id":"m_3f2e1a","goal":"Orders",
 "steps":[{"action":"navigate","target_stable_id":"main.tabs.orders",
           "target_text":"Orders","target_role":"tab",
           "bounds":[…]}]}
    │
    ▼
┌────────────────────── SDK: VoiceLoop.handlePayload(bytes) ─────────────────────┐
│                                                                                 │
│  (5) Switch on type → "mission" → _startMission(MissionMessage)                 │
│                                                                                 │
│  (6) MissionRunner.run(mission):                                                │
│       emit mission_status: started                                              │
│       for each step:                                                            │
│         emit mission_status: step_started                                       │
│         for attempt in [0..2]:                                                  │
│           ┌─ PHASE: searching ────────────────────────────────────────────┐    │
│           │   emit mission_phase: searching                              │    │
│           │   Explorer.find(snapshot, target) →                          │    │
│           │       5-rung ladder: stable_id → identifier →                │    │
│           │       text → role → bounds-IoU                               │    │
│           │   if not found, Explorer.findOrScroll(... direction=both)    │    │
│           └──────────────────────────────────────────────────────────────┘    │
│           ┌─ PHASE: executing ────────────────────────────────────────────┐    │
│           │   emit mission_phase: executing                              │    │
│           │   TapExecutor.execute(node) — 4-rung tap fallback:           │    │
│           │       semantic tap → ancestor walk → text-twin → synth pointer│   │
│           │   settle 400ms (550ms for back/navigate)                     │    │
│           └──────────────────────────────────────────────────────────────┘    │
│           ┌─ PHASE: verifying ────────────────────────────────────────────┐    │
│           │   emit mission_phase: verifying                              │    │
│           │   Verifier.verify(step, execResult, before, after):          │    │
│           │     collect evidence: route_changed? node_disappeared?       │    │
│           │                       bounds_moved? text_changed? appeared?  │    │
│           │   action-specific verdict (navigate needs non-empty evidence)│    │
│           │   emit verification_result                                   │    │
│           └──────────────────────────────────────────────────────────────┘    │
│           if success or terminal_find_failure → break                          │
│           else delay 450ms and retry                                           │
│         emit mission_status: step_completed                                    │
│       emit mission_status: succeeded (or failed/cancelled)                     │
│       emit mission_exploration (consolidated audit, one packet at end)         │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
    │
    ▼
Backend pipeline_service._process_data_message handles each inbound:
    mission_status   → logged only       (Gap 1)
    mission_phase    → logged only       (Gap 1, Gap 4)
    verification_result → logged only    (Gap 1)
    mission_exploration → in-mem + local ephemeral disk (Gap 2)
```

Look at what's NOT in this flow: **nothing flows the result back to the LLM.** The LLM dispatched the mission, got `{"status": "dispatched"}` back instantly, and the next turn it will say "Opening orders!" without knowing whether the SDK actually succeeded. That's the open loop.

---

## 4. Backend side (`revrag-core`) — file by file

### 4.1 `app/modules/pipeline/pipeline_service.py` — inbound packet router

The LiveKit worker entrypoint. Routes data-channel messages from the SDK into the agent.

**Key method: `_process_data_message`** — switch on top-level `type`:

| Inbound `type` | What happens | Status |
|---|---|---|
| `ui_snapshot` | Parse via `UiSnapshot.from_ui_snapshot_payload` → write to `agent._agent.ui_snapshot`. This is the modern path; SDK sends one of these on first connect. | LIVE |
| `ui_diff` | If `baseline: true` or no prior snapshot, rebuild via `UiSnapshot.from_baseline_diff`; otherwise sparse `UiSnapshot.apply_diff`. | LIVE |
| `ui_tree` *(legacy)* | Store as `agent._agent.ui_tree_compact` + `ui_tree_screen`. Only used by older Android SDKs still on `nodes_compact`. | legacy |
| `mission_status` | Pretty-print to log with `[ActionIntel ◀ RECV]` block. **Nothing else.** | ⚠️ log-only |
| `mission_phase` | Pretty-print to log. **Nothing else.** | ⚠️ log-only |
| `verification_result` | Pretty-print to log. **Nothing else.** | ⚠️ log-only |
| `mission_exploration` | `setattr(agent, "last_mission_exploration", parsed_data)` (in-memory) + `_persist_mission_exploration` writes JSON to local disk. | partial — ephemeral disk |
| `user_context_updated` | Routed into `agent_session.userdata.embedded_context_events` (40-item cap, dedup by event ID) and `proactive_context_handler.on_event_received` if present. | (not action-intel, but worth knowing) |
| `end_call` | Saves transcript, interrupts agent, speaks farewell, shuts down room. | (not action-intel) |

**`_persist_mission_exploration`** writes to `<REVRAG_EXPLORATION_LOG_DIR>/mission_<mission_id>_<status>_<ts>.json` — defaults to `./exploration_logs/` (CWD-relative). **On EKS pipeline pods this directory is ephemeral** — every restart/deploy wipes it. This is Gap 2.

**`_early_data_buffer`** — a critical race fix. LiveKit's `onData` callback can fire **before** the agent is fully constructed (frontend sends `ui_snapshot` instantly on join; backend agent assembly takes ms). A pre-connect handler registers a buffer that captures these early messages; once `agent_session.start(...)` returns, the full handler is registered and the buffer is replayed. Messages that still can't apply (agent not yet ready) get re-buffered for a second replay pass; excess gets dropped to avoid stale-data hazards.

### 4.2 `app/modules/pipeline/util/ui_snapshot.py` — typed in-memory tree state

Backend's source-of-truth for what's on the device's screen, kept in sync with the SDK via `ui_snapshot` + `ui_diff` packets.

**`UiNode`** — typed dataclass mirroring the SDK's node wire shape:

| Field | Source on wire | Notes |
|---|---|---|
| `stable_id` | `id` | Backend renames the field. |
| `role`, `text`, `bounds`, `scrollable`, `visible`, `has_action`, `identifier`, `actions` | identical | |
| `path` | `path` (per-node) | **Bug:** Dart doesn't emit `path` per node — paths live in the top-level `paths` map. So `node.path` is always empty for snapshot-derived nodes; only `moved` diff entries populate it. |
| ~~`selected`~~ | `selected` | **Drift:** SDK emits `selected: true` on the wire, backend silently drops it. Active-tab and checkbox state are lost. |

**`UiSnapshot`** — typed snapshot:
- `from_ui_snapshot_payload(payload)` parses the full `ui_snapshot` packet. **Ignores** top-level `paths` map and `visible` (visibleIndex) array.
- `from_baseline_diff(payload)` — treats a `ui_diff` with `baseline: true` as a fresh snapshot.
- `apply_diff(payload)` — handles all five diff sections:
  - **Screen-swap detection.** If `screen` field changes, treat the whole diff as a baseline regardless of the `baseline` flag.
  - `removed` — delete by stable_id.
  - `added` — parse via `UiNode.from_wire`.
  - `updated` — sparse field application via `UiNode.apply_update` (only changed fields).
  - `moved` — updates `path` field only (one of the few places `path` ever has a value).
  - `visibility_changed` — sparse `visible` flip only.
- `find_by_stable_id` / `find_by_identifier` — O(n) / O(1) lookups.
- **`render_for_prompt(max_per_section=30)`** — the secret sauce: groups nodes into four labelled sections for the LLM:

  ```
  ── NAVIGATION ──   (role in {tab, link}, OR stable_id contains tab/tabs/nav/menu/bottomnav/topbar/appbar)
  ── INTERACTIVE ──  (visible + has_action, NOT nav)
  ── CONTENT ──      (visible label-only, NOT nav, NOT actionable)
  ── OFFSCREEN ──    (not visible, NOT nav)
  ```

  Each line format: `[role] "text" id=stable_id (visible|off-screen, tappable|label)`. Text truncated to 60 chars, newlines replaced with `⏎`. Why sectioned: in the original "open account page" → `home.recent.arjun_05_10` bug, the flat alphabetical list made the LLM pick the first text match regardless of role. Sectioning forces the LLM to scan NAVIGATION first for nav intents.

### 4.3 `app/modules/pipeline/util/mission_wire.py` — outbound builders

Pure functions that build wire JSON. No state.

- **`MISSION_ACTIONS`** — canonical action vocabulary: `{click, navigate, scroll_to, highlight, scroll_and_highlight, set_text, type, wait, back}`.
- **`build_mission_step(action, target_stable_id=None, target_identifier=None, target_text=None, target_role=None, bounds=None, params=None)`** → one step dict. Drops None fields.
- **`build_mission_message(steps, goal=None, mission_id=None)`** → wire-ready JSON string. Auto-generates `mission_id` (`m_<6 hex>`) if not provided.
- **`build_cancel_message(mission_id)`** → built but **never emitted** by any backend code path.
- **`build_highlight_message(target_stable_id=None, target_identifier=None, bounds=None)`** → built but **never emitted**.
- **Missing:** no `build_speak_message`. The SDK consumes `type: speak` (text + interrupt) for deterministic TTS narration — but Python has nothing that produces it. This is the missing piece for Gap 4 narration.

### 4.4 `app/modules/pipeline/agents/ui_action_mixin.py` — the agent's tool + dispatcher

The brain of the backend side. 1252 lines. Composed as a mixin: `class EmbedPipelineAgent(UiActionMixin, EmbeddedContextMixin, Agent)`.

#### The `perform_ui_action` tool (the LLM's interface)

A `@function_tool`-decorated coroutine that's auto-discovered by LiveKit's `find_function_tools()` when MRO walks the class.

```python
async def perform_ui_action(
    self,
    context: RunContext[Dict[str, Any]],
    action: str,
    target_description: str,
    target_stable_id: Optional[str] = None,
    target_role: Optional[str] = None,
    value: Optional[str] = None,
) -> Dict[str, Any]:
```

**Docstring length matters.** OpenAI's `function.description` field has a hard ~1024 char cap. An earlier version of this docstring grew to 4040 chars; the LLM call returned 400 errors *silently* — the agent shipped, the tool registered, but the LLM never called it. That was the "no greeting / no mission" incident. The docstring is now ~700 chars; the long rules live in the system prompt (no per-message cap).

**The opt-in flag is now opt-out.** `enable_ui_actions` is derived from the agent's `function_list` DB column. The historical doc says "agents opt-in via a `perform_ui_action` entry"; the current code (line 64, line 212–228) says: enabled if *any* entry has `type=="perform_ui_action"` AND `enabled != False`. Absence of the entry means enabled (the tool is auto-discovered). Explicit opt-out requires an entry with `enabled: false`. This means **every embedded agent gets viewport injection on every LLM turn** — see Gap 5 (token cost).

#### Viewport injection (`render_viewport_ground_truth`)

Called inside `llm_node` each turn. Builds a system message with three parts:

1. **The viewport block** — `self.ui_snapshot.render_for_prompt()` output (the four-section list).
2. **The Intent → Action lookup table** (deterministic mapping — see §7).
3. **The MANDATORY DISPATCH rule** — the long instruction block telling the LLM:
   - You MUST dispatch for any UI request.
   - You must NOT say "I don't see X" instead.
   - Off-screen targets are fine; the SDK Explorer will scroll-search.
   - If target IS in the viewport block → pass its `id=…`. If not → still dispatch (blind), passing `target_description` + `target_role`.
   - The only exception is pure information questions ("what does the screen say?") — answer from CONTENT text directly.

This block is the result of every failure mode the migration hit. Earlier versions let the LLM honestly report "नहीं दिख रहा" ("not visible") — now it's explicitly forbidden.

If `enable_ui_actions` is True but no snapshot exists yet, returns `None` and the LLM proceeds without viewport context (it waits for the SDK to push one).

#### Intent → Action determinism (3 layers)

Same Intent → Action mapping is implemented in three places, all consistent. See §7 for the full table. The layers:

1. **System-prompt table** — the LLM reads the table and copies the action value verbatim.
2. **`_ACTION_SYNONYMS` dict** (~50 entries) — second-chance synonym mapping when the LLM emits something close but not canonical (e.g., `tap` → `click`, `खोलो` → `navigate`).
3. **Action coercion in `_dispatch_mission`** — last-resort substring keyword fuzzy-match for entirely unknown verbs (`*scroll*` → `scroll_to`, etc., default `click`).

#### `_dispatch_mission` — the 5-stage funnel

This is where every mission gets built. **Never returns early without dispatching** — that's the "never reject" principle. The only failures that skip a mission are infrastructure (no LiveKit room, send error).

**Stage 1: Action coercion** (lines 499–540 of ui_action_mixin.py)

```python
if action not in MISSION_ACTIONS:
    lower = action.lower().strip()
    if   any(k in lower for k in ("scroll","swipe","pan","slide")):           action = "scroll_to"
    elif any(k in lower for k in ("back","previous","return","pop","पीछे","वापस")): action = "back"
    elif any(k in lower for k in ("fill","type","write","input","set_value","enter")): action = "set_text"
    elif any(k in lower for k in ("highlight","show","find","locate","where","point","दिखा","ढूंढ")): action = "scroll_and_highlight"
    elif any(k in lower for k in ("navigate","go","open","switch","launch","जा","खोल")): action = "navigate"
    else:                                                                     action = "click"   # safe default
    logger.warning("coerced …")
```

**Stage 2: Target resolution (snapshot path)**

If `self.ui_snapshot` exists:
- Primary: `snap.find_by_stable_id(target_stable_id)` → `snap.find_by_identifier(target_stable_id)`.
- If not found and `target_description` provided: fuzzy text scoring. Per-node score combines:
  - Exact text match: +100
  - Substring match: +60 + min(len(needle), 30)
  - Reverse substring (needle ⊂ haystack): +40
  - Exact stable_id-tail match: +70
  - Substring on stable_id-tail: +30
  - Has-action bonus: +5
  - Visible bonus: +5
  - Role bias: if action is nav-like, prefer nav primitives (+50) and demote content (-40).

**Stage 3: Override guards**

Two guards run *after* a snapshot resolution succeeds:

- **Nav-intent content-row override.** If `action in {navigate, click}` AND the resolved node looks like a content row (stable_id contains `recent`/`activity`/`feed`/`row`/`card`/`item`/`tile`/`message`, OR role is `text`/`card`), drop the node, fall through to blind dispatch with `target_role='tab'`. Reason: prevents the "tap Arjun's row when user said 'open account page'" failure.
- **Generic text-overlap.** If the resolved node's text shares zero word-overlap (after stopword removal) with the provided `target_description`, drop the node — the LLM hallucinated a stable_id whose UI text doesn't match the user's spoken target.

**Stage 4: Blind dispatch (fallback when snapshot doesn't resolve)**

Ship `target_text=target_description` + `target_role`. SDK's Explorer scroll-searches.

- **Role-omit fallback:** if `target_role in {tab, link}` but no node in the snapshot has that role *and* a text overlap with `target_description`, drop the role constraint so the SDK can find content cards the LLM mis-typed. Prevents the "Welcome Back" + `target_role=tab` failure.

**Stage 5: Empty-target fallback**

If we have neither stable_id nor description, still build a step with `action` + `target_role` only. SDK tries role-only match. Philosophy: never swallow the user's intent silently.

**`set_text` value plumbing.** A separate branch handles `value`:
- If `value` is non-None → ship verbatim as `params["text"]`.
- If description contains "clear"/"erase" and no value → ship empty string.
- ASCII-only field guard: if the field looks ASCII-only (hints: "email"/"phone"/"code"/"OTP"/"url") but value has non-ASCII chars, log warning (transliteration bug likely) but still ship.
- Email TLD truncation guard: warn if email-shaped value has no `.com`/etc.

**Dispatch.** Final step: `build_mission_message(steps=[step], goal=target_description)` → `await lk_api.room.send_data(...)`. Logged with `[ActionIntel ▶ SEND]` prefix and full JSON. Returns `{"status": "dispatched", "mode": "snapshot"|"blind"|"empty", "mission_id": …, …}` immediately — does NOT wait for SDK confirmation.

**Stale stable-id telemetry.** If the LLM passed a `target_stable_id` that doesn't exist in the live snapshot, log the downgrade path (`text_match_recovered`, `blind_dispatch`, `empty_dispatch`). Pure observability — no behavior change.

### 4.5 `app/modules/pipeline/agents/embed_pipeline_agent.py` — concrete agent class

Inherits `UiActionMixin, EmbeddedContextMixin, Agent`. MRO guarantees the mixin's `@function_tool` methods are auto-discovered. This class adds:

- Per-call state: `agent_id`, `agent_name`, `room_name`, `welcome_message`, `app_user_id`, `current_screen`, `previous_screen`.
- Tools registry that dynamically composes static tools (`end_call`, `generative_ui`), `perform_ui_action` (from mixin, auto-discovered), and agent transfer tools (from `AgentTransferToolBuilder`).
- Number conversion (English/Hindi/dynamic-per-turn) for TTS.
- `llm_node` — overrides Agent.llm_node, calls `render_viewport_ground_truth()` and injects the system message + assistant context (RAG, custom events, screen-change diffs) before delegating to the LLM.

### 4.6 `app/modules/pipeline/util/ui_action_resolver.py` — legacy Android path

Kept for older Android SDKs still publishing `ui_tree`/`nodes_compact` instead of the mission protocol. A separate `gpt-4o-mini` resolver call. Pre-filtered by a 187-keyword `needs_ui_action()` check so 85–90% of utterances don't even hit the LLM. Has its own 11-action vocab and own Intent → Action table.

This is Gap 8 — the **two parallel code paths**. Every rule update has to be applied twice or the two LLMs disagree. Slated for sunset once Android SDKs ship the mission protocol.

---

## 5. SDK side (`embed-flutter`) — file by file

All paths relative to `lib/core/` on branch `Architectur_Change_Action_intelligence`.

### 5.1 Tree pipeline: capture → normalize → stable_ids → snapshot → diff_engine

The chain that turns Flutter's live `SemanticsNode` tree into a wire-format snapshot.

#### `tree/capture.dart` — raw capture

- **Long-lived `SemanticsHandle`** retained for the entire SDK session so Flutter doesn't deallocate/renumber SemanticsNode IDs between captures (in-process refs stay stable).
- **Physical → logical pixel conversion** at the capture seam (using device pixel ratio). Everything downstream (overlap detection, synthetic pointer taps, overlay positioning) expects logical pixels.
- **Depth-first pre-order walk** producing a flat list with parent indices always pointing backward — invariant that ancestors precede descendants.
- **Output:** `RawTree.nodes` — list of `RawNode` (`rawId`, `SemanticsData`, `bounds`, `parentIndex`, `siblingIndex`, `depth`).

#### `tree/normalize.dart` — role inference + filtering

- **Drops low-signal nodes** with: zero bounds, OR (empty text AND no action AND not scrollable AND empty identifier AND no recognized landmark role). Recognized roles: `button, link, textField, image, tab, header, checkbox, radio, progressbar, slider, switch`.
- **Role inference ladder** (priority order):
  1. Explicit `SemanticsRole` (e.g. `SemanticsRole.button`).
  2. `scrollExtentMax != scrollExtentMin` → `scrollable_container`.
  3. Semantic flags → `heading` / `input_field` / `checkbox` / `toggle` / `image` / `link` / `slider` / `button`.
  4. Has tap + has text → `card`.
  5. Has tap + no text → `button`.
  6. No action + has text → `text`.
  7. Fallback → whatever explicit role string was (even empty).
- **Text merging**: combines `attributedLabel + attributedValue + tooltip` (joined with space, trimmed, capped at 100 chars).
- **Visibility**: `viewport.overlaps(node.bounds)`.
- **Identifier promotion**: walks the kept tree to push `Semantics(identifier:)` from a non-actionable leaf onto its nearest tappable ancestor that doesn't already have an identifier. Fixes the Flutter idiom of `IconButton(icon: Semantics(identifier: 'app_bar.menu', …))` where the identifier ends up on the icon instead of the tappable parent.
- **NOTE: the 150-node priority cap is NOT in this file.** It's enforced downstream (in the snapshot pump or socket bridge), scoring nodes by visibility, has-action, identifier presence, role type, text presence.

#### `tree/stable_ids.dart` — stable-id minting

Produces dotted, human-readable IDs like `home.quick_stats.see_all`.

**Slug source ladder** (in priority order):
1. **Identifier** → slugify (e.g., `Semantics(identifier: 'quick_stats')` → `quick_stats`).
2. **Text** → slugify (e.g., "See All" → `see_all`).
3. **Role + sibling position** (e.g., role `button`, position 2 → `button_2`; defaults to `node_2` if role is empty).

`slugify()`: lowercase, collapse non-`[a-z0-9]` runs to `_`, trim outer `_`, **cap at 24 chars**.

**Collision resolution**: if two siblings produce the same slug, append `_<N>` (e.g., `home_2`, `home_3`). The assigner tracks used slugs per parent and retries with incrementing disambiguator.

**Output:** parallel arrays `(stableIds[], paths[])` aligned with the normalized node list. `paths[i]` is the dotted structural path of sibling indices (e.g. `0.2.1.4`).

#### `tree/snapshot.dart` — `UiNode` + `UiSnapshot`

**`UiNode`** fields: `stableId, role, text, bounds, scrollable, visible, hasAction, identifier, actions, selected, rawId` (rawId is on-device-only, never serialized).

**`UiNode.toWire()`**:
```dart
{
  'id': stableId,            // ← backend reads as stable_id
  'role': role,
  if (text.isNotEmpty) 'text': text,
  'bounds': [l, t, r, b],
  if (scrollable) 'scrollable': true,
  'visible': visible,
  if (hasAction) 'has_action': true,
  if (identifier.isNotEmpty) 'identifier': identifier,
  if (actions.isNotEmpty) 'actions': actions,
  if (selected) 'selected': true,   // ← Python drops this silently (Gap)
}
```

**`UiSnapshot.toWire()`**:
```dart
{
  'type': 'ui_snapshot',
  'screen': screen,
  'captured_at': iso8601,
  'viewport': [width, height],
  'nodes': [/* node.toWire() for each */],
  'paths': {stableId: dotted-path-string, ...},   // ← Python ignores
  'visible': [/* indices of visible nodes */],     // ← Python ignores
}
```

#### `tree/diff_engine.dart` — delta computation

**Baseline trigger:** if `oldSnapshot == null` OR `screen` changed → return all new nodes as `added` (i.e., a fresh baseline disguised as a diff).

**Diff categories:**
- `added` — new stableId not in old.
- `removed` — list of stableId strings (old has it, new doesn't).
- `updated` — sparse field changes (role/text/bounds/scrollable/has_action/actions).
- `moved` — `path` changed only.
- `visibility_changed` — *only* `visible` flipped, nothing else. Separate list to keep scroll-heavy events tiny.

**Bounds comparison** uses rounded ints (wire-format precision) so sub-pixel animation jitter doesn't trigger spurious updates.

**`toWire()`:**
```dart
{
  'type': 'ui_diff',
  'screen': screen,
  'captured_at': iso8601,
  'previous_captured_at': iso8601,
  if (isBaseline) 'baseline': true,
  if (added.isNotEmpty) 'added': [...],
  if (removed.isNotEmpty) 'removed': [...],
  if (updated.isNotEmpty) 'updated': [...],
  if (moved.isNotEmpty) 'moved': [...],
  if (visibilityChanged.isNotEmpty) 'visibility_changed': [...],
}
```

#### `tree/active_screen_detector.dart` — screen-name detection in shells

For apps that use `IndexedStack` or `PageView` instead of pushing routes, the active "screen name" can't come from `Navigator`. This detector walks the widget tree and infers it.

**Two signals, priority order:**

1. **Pushed-screen detection** wins: any widget whose `runtimeType.toString()` ends in `Screen`, `Page`, or `Detail` (excluding names starting with `_`). Snake-cased. Last match (depth-first) wins because pushed routes appear later in the walk.
2. **Tab-content detection** as fallback: walks the first `IndexedStack` (reads `widget.index`, recurses only into the active child) or `PageView` (reads `controller.page`, looks up child from `SliverChildListDelegate`).

`_toSnakeCase()` converts `HomeTab` → `home_tab`, `PlansScreen` → `plans_screen`. Idempotent.

### 5.2 `runtime/snapshot_pump.dart` — when snapshots get pushed

Wraps the capture pipeline + `SnapshotDiffEngine` to decide when to push.

**Two push modes:**

| Mode | Trigger | Debounce | Empty diff |
|---|---|---|---|
| `forcePush()` | LiveKit connect, route change, legacy `request_ui_tree` | none | always pushed (heartbeat) |
| `schedulePush()` | scroll (noisy stream) | 500ms | skipped |

After each push, calls `ExplorationLog.instance.observe(snap)` for audit deduping. Updates `_lastPushed = snap`. Send errors are swallowed (`.catchError((_) => null)`).

### 5.3 `runtime/voice_loop.dart` — LiveKit data-channel wrapper

The bidirectional bridge between the data channel and the `MissionRunner`.

**Outbound:** `_sendJson(json)` — UTF-8 encodes, fire-and-forget via the `SendPayload` callback. Logs to `WireLog` if enabled.

**Inbound:** `handlePayload(Uint8List bytes)` — JSON-decode, switch on `type`:

```dart
switch (type) {
  case MissionMessage.kType:    _startMission(MissionMessage.fromJson(json));
  case CancelMessage.kType:     _cancelMission(CancelMessage.fromJson(json));
  case SpeakMessage.kType:      /* TTS rides the audio track; no-op here */
  case HighlightMessage.kType:  standaloneHighlightDispatcher?.call(...);
  case 'request_ui_tree':       onRequestSnapshot?.call();   // legacy pull
}
```

**Multi-mission safety:** `_active: Map<missionId, MissionCancelToken>`. Re-entry with an in-flight `mission_id` cancels the prior, starts fresh. Finally-block on cleanup checks `identical(_active[id], token)` so a re-entry's later cleanup doesn't kill the new mission.

### 5.4 `runtime/mission_runner.dart` — the per-step state machine

The actual engine. ~891 lines.

**Defaults:**
- `stepTimeout: 30s`
- `missionTimeout: 3min`
- `retriesPerStep: 2` (so 3 total attempts)
- `retryDelay: 450ms`
- `postExecuteSettle: 400ms` base, +150ms for `back`/`navigate` (Material pop animations run ~300ms).

**Top-level flow** (`run(MissionMessage, cancel)`):

```
emit mission_status: started
for each step:
  check cancellation + mission deadline
  emit mission_status: step_started
  outcome = _runStepWithRetries(step)
  emit verification_result
  emit mission_status: step_completed
  if outcome.failure → emit failed, rollback, return
  if cancelled → emit cancelled, return
emit mission_status: succeeded
ExplorationLog.emitToBackend(...)
```

**`_runStepWithRetries`** loops attempts 0..retriesPerStep. Each attempt runs through three phases (`_runSingleAttempt`):

- **Phase 1 — searching**: emit `mission_phase: searching` with hints; capture "before" snapshot; for non-target-less actions, call `_resolveStepTarget`. Fast path: `Explorer.find(snap, target)`. Slow path: `Explorer.findOrScroll(target, ..., direction: both)`.
- **Phase 2 — executing**: emit `mission_phase: executing`; dispatch to the appropriate executor; sleep `postExecuteSettle`; capture "after" snapshot.
- **Phase 3 — verifying**: emit `mission_phase: verifying`; `verifier.verify(step, execResult, before, after)`; return outcome.

**Terminal find failures short-circuit retries** — `_isTerminalFindFailure()` checks for `target_not_found`, `scrollEndReached`, `maxAttemptsReached`, `alreadyVisited`, `noScroller`, `emptyTarget`. These don't retry (no point — the answer won't change).

### 5.5 `explore/explorer.dart` — 5-rung match + scroll-search

~1128 lines. The heart of "make the LLM's target description actually resolve to a widget."

**The 5-rung match ladder** (`_match(snapshot, target)`):

1. **`stableId`** — direct lookup.
2. **`identifier`** — linear scan for `Semantics(identifier:)` match.
3. **`text`** — **scored fuzzy match**, by far the most complex rung:
   - **Tier 1 (substring)** scores: exact 100, startsWith 60, word-boundary 40, substring 15.
   - **Tier 2 (token-prefix fuzzy)** if substring fails and `fuzzySim >= 0.75`: score 8..14 scaled across similarity. (Capped strictly below substring's 15 so substring always wins.)
   - **Bonuses**: length proximity (max +40), tappable (+25), role match (+20), selected (+5).
   - **Visibility partition**: ANY visible candidate wins unconditionally over offscreen. Prevents picking pre-built carousel pages that AABB-overlap but aren't "live".
   - **Tiebreaker within visible**: strength > score > viewport-center proximity. Critical for PageView/carousels with multiple text copies — only the center one is "live".
   - **Text promotion**: bare text-leaf hits get walked UP (max 8 ancestors) to the nearest tappable / container-role / 1.5×-size node that doesn't exceed 80% of viewport area. Lifts a `<Text>Home</Text>` match to the tappable `Card` wrapper.
4. **`role`-only** — skipped if any stronger signal (stableId/identifier/text) was provided and missed. Otherwise first visible node with matching role (fallback to offscreen).
5. **`approximateBounds`** — IoU spatial match. Tie-break (within 1e-3 eps): prefer visible.

**Stopwords** stripped before tokenization: `option, options, item, items, button, card, tab, tile, section, screen, page, row, list, the, a, an, this, that, these, those, please, just, now, here, there`.

**Number-word ↔ digit**: `twenty` ↔ `20`. **Bidirectional prefix**: `saving` ↔ `savings`. **`%` expansion**: `%` ↔ ` percent `.

**`findOrScroll`** — scroll-search loop with three escape hatches:
- `maxAttempts = 106` (per direction; not shared in `ScanDirection.both`).
- `scanTimeout = 6s` wall-clock (shared in `both` mode).
- `offsetTolerance = 8px` for cycle detection (NaN/±∞ treated as null).

`ScanDirection.both`: forward pass first, then backward pass with a *fresh* attempt budget. Critical because "user at bottom, target at top" with shared budget would silently fail.

### 5.6 Executors — what actually fires the gesture

- **`tap_executor.dart` — 4-rung tap fallback:**
  1. **Semantic tap**: if node advertises `SemanticsAction.tap`, call `performAction(rawId, tap)`.
  2. **Neighbour walk (structural)**: walk UP (nearest tappable ancestor) then DOWN (tappable descendant closest to original bounds-center). Covers "inner icon's tap belongs to the InkWell ancestor" and "I targeted a container; the tappable children are the real targets."
  3. **Tappable-by-text twin**: scan the snapshot for a tappable node whose text matches the original's (mutual substring match after normalization). Prefer visible; prefer shorter text. Covers `<Text>Home</Text>` heading vs `BottomNav` tab with same label.
  4. **Synthetic pointer**: fabricate `PointerDown` + `PointerUp` at the bounds center as last resort.

- **`scroll_executor.dart`**: Explorer find/scroll + optional `SemanticsAction.showOnScreen` + edge-padding alignment.
- **`input_executor.dart`**: `SemanticsAction.setText`. If `setText` not in actions mask, tap first to focus, wait 50ms for focus + semantics rebuild, re-read mask. Atomic replace.
- **`nav_executor.dart`**: delegates to TapExecutor with `expects_route_change: true` diagnostic.
- **`back_executor.dart` — 3-rung:**
  1. No host handler → fail (`no_back_handler`).
  2. Host handler returned `false` → fail (`no_back_target`).
  3. Handler threw → fail (`back_handler_threw`).
- **`wait_executor.dart`**: fixed duration, OR poll-until-appears, OR poll-until-disappears. 200ms poll interval, 5s timeout default.
- **`executor_result.dart`**: shared result type — `success, strategy, elapsed, error, diagnostics`.

### 5.7 `verify/verifier.dart` — was it real?

Post-action evidence collection. Three steps:

1. **Compute evidence set** from before/after snapshots:
   - `routeChanged` — `before.screen != after.screen`.
   - `nodeDisappeared` — target in before, not in after.
   - `boundsMoved` — same target moved >4px (default `boundsMovedPx`).
   - `textChanged` — same target's text differs.
   - `nodeAppeared` — new stableId in after that wasn't in before.

2. **Action-specific verdict**:
   | Action | Success criterion |
   |---|---|
   | `navigate` | non-empty evidence (route change preferred but not required) |
   | `tap`/`click` | non-empty evidence (the fake-success killer) |
   | `back` | trust executor (don't penalize race-condition empty evidence mid-animation) |
   | `set_text`/`type` | target still resolves + text contains expected value |
   | `scroll_to` | target resolves + visible (strict — partially clipped = fail) |
   | `scroll_and_highlight` | same as scroll_to |
   | `highlight`/`wait`/unknown | trust executor |

3. Returns `VerificationOutcome { success, evidence, failureReason? }`.

### 5.8 `runtime/exploration_log.dart` — audit log

Singleton stream collector with `(screen, stableId)` dedup. Every node the agent observes (via SnapshotPump pushes, Explorer mid-scan captures, MissionRunner snapshots) gets logged once per `(screen, stableId)`.

**`emitToBackend(missionId, status)`** — called once per mission at every terminal path (succeeded / failed / cancelled / mission_timeout). Builds a `MissionExplorationMessage` containing every unique node seen during that mission, grouped by screen. Wire-sent fire-and-forget; also logged locally in chunks (8 entries/line) to defeat logcat's ~4KB-per-line truncation.

### 5.9 `overlay/` — visual feedback

- `highlighter.dart` — pulse overlay for `highlight` / `scroll_and_highlight`.
- `active_border.dart`, `awakening_overlay.dart` — additional visual cues.

These run client-side; backend only triggers via the `highlight` mission action (or, in theory, the standalone `highlight` message — currently never emitted).

---

## 6. The wire protocol contract (the linkage)

### Backend → SDK

| `type` | Python emitter | Dart handler | Status |
|---|---|---|---|
| `mission` | `build_mission_message` | `VoiceLoop` → `_startMission` → `MissionRunner.run` | LIVE |
| `cancel` | `build_cancel_message` | `_cancelMission` (cancels token) | **built, NEVER emitted by backend** |
| `highlight` | `build_highlight_message` | `standaloneHighlightDispatcher` | **built, NEVER emitted by backend** |
| `speak` | ❌ **missing in Python** | reads `text` + `interrupt`; deterministic TTS | **SDK ready, Python missing — blocks R1 narration** |
| `request_ui_tree` (legacy) | (legacy) | `onRequestSnapshot` → `SnapshotPump.forcePush()` | legacy |

### SDK → Backend

| `type` | Dart emitter | Python handler | Status |
|---|---|---|---|
| `ui_snapshot` | `SnapshotPump` (first push, route change) | `UiSnapshot.from_ui_snapshot_payload` → `agent.ui_snapshot` | LIVE |
| `ui_diff` | `SnapshotPump` (subsequent pushes) | `apply_diff` / `from_baseline_diff` | LIVE |
| `ui_tree` (legacy) | n/a (Android) | `agent.ui_tree_compact` | legacy |
| `mission_status` | `MissionRunner` (lifecycle: started/step_started/step_completed/succeeded/failed/cancelled) | **log only** | ⚠️ open loop |
| `mission_phase` | `MissionRunner` (per-step: searching/executing/verifying) | **log only** | ⚠️ open loop |
| `verification_result` | `Verifier` (per step, terminal outcome) | **log only** | ⚠️ open loop |
| `mission_exploration` | `ExplorationLog.emitToBackend` (one per mission terminal) | in-mem stash + ephemeral local-disk JSON | partial |

### Field shapes (the parts where ends agree)

**`mission` step** — clean match both sides:
```json
{
  "action": "click|navigate|scroll_to|highlight|scroll_and_highlight|set_text|type|wait|back",
  "target_stable_id": "main.tabs.account",
  "target_identifier": "host-supplied-tag",
  "target_text": "Account",
  "target_role": "tab|link|button|text|card|input_field",
  "bounds": [left, top, right, bottom],
  "params": { /* action-specific extras */ }
}
```

**`mission_status`**:
```json
{
  "type": "mission_status",
  "mission_id": "m_3f2e1a",
  "status": "started|step_started|step_completed|succeeded|failed|cancelled",
  "timestamp": "2026-06-02T…Z",
  "step_index": 0,        // only on step events
  "error": "…"            // only on failed
}
```

**`mission_phase`**:
```json
{
  "type": "mission_phase",
  "mission_id": "m_3f2e1a",
  "step_index": 0,
  "phase": "searching|executing|verifying",
  "timestamp": "…",
  "hints": {
    "action": "click",
    "attempt": 0,
    "target_stable_id": "main.tabs.account",
    "target_text": "Account",
    "target_role": "tab"
  }
}
```

**`verification_result`**:
```json
{
  "type": "verification_result",
  "mission_id": "m_3f2e1a",
  "step_index": 0,
  "success": false,
  "evidence": ["route_changed", "node_disappeared", "bounds_moved", "text_changed", "node_appeared", "no_observable_effect"],
  "timestamp": "…",
  "failure_reason": "target_still_offscreen"  // only on failures
}
```

**`mission_exploration`**:
```json
{
  "type": "mission_exploration",
  "mission_id": "m_3f2e1a",
  "status": "succeeded|failed|cancelled|mission_timeout",
  "captured_at": "…",
  "screens": {
    "home_tab": [
      {"id":"home.cta", "role":"button", "text":"Get started",
       "bounds":[l,t,r,b], "visible":true, "has_action":true}
    ],
    "account_tab": [/* ... */]
  },
  "stats": {"screens": 2, "unique_nodes": 117}
}
```

### Field-level drift (the parts where ends disagree)

| Field | SDK emits | Backend handles | Impact |
|---|---|---|---|
| `selected` (on UiNode) | `if (selected) 'selected': true` | parsed into nothing — `from_wire` doesn't read it | **silent loss of active-tab/checkbox state** |
| `paths` (top-level on snapshot) | `{stableId: dotted-path}` | ignored by `from_ui_snapshot_payload` | path-based ancestor logic on backend never works |
| `visible` (top-level visibleIndex on snapshot) | `[indices]` | ignored | wasted bandwidth or missing optimization |
| `path` (per-node) | NOT emitted (lives in top-level `paths` map) | `from_wire` reads `raw.get("path")` | **always empty for snapshot-derived nodes; only `moved` diff entries set it** |
| `tap_bounds` action | SDK has executor for it | not in `MISSION_ACTIONS` | backend can't emit, SDK can't show value |
| `speak` message | full consumer in VoiceLoop | NO builder in `mission_wire.py` | **blocks R1 narration outright** |

These drifts exist because the schema is **hand-mirrored**, not codegen'd or contract-tested. The original migration (which produced the docs in this folder) was driven by exactly this kind of drift (`ui_action` ↔ `nodes_compact` divergence). Gap 6 is asking for either a shared schema or a fixture round-trip test, neither of which exists yet.

---

## 7. Intent → Action determinism (the verb table)

Source of truth — the same mapping is implemented in three places to keep the LLM, the synonyms dict, and the dispatcher consistent.

```
USER INTENT (any language)                                        → ACTION

find / search / search_for / where_is / show / show_me /
locate / point / point_to / highlight /
दिखाओ / ढूंढो / go to <section/card/banner/tile>                  → scroll_and_highlight

jump / navigate / open / switch / switch_to / go / go_to /
goto / launch / visit / खोलो / जा / पेज                            → navigate

click / tap / press / hit / push / touch / submit / select /
pick / choose / activate / toggle / check / tick / uncheck /
untick / mark / unmark / focus                                    → click

scroll / scroll down / scroll up / swipe /
page_down / page_up                                               → scroll_to

fill / type / write / input / enter / set / set_value /
populate / clear / erase / delete_text                            → set_text
   (set_text REQUIRES value=… verbatim from utterance,
    NOT the field label, NEVER translated)

back / previous / previous_screen / pop / return /
पीछे / वापस                                                       → back
```

**Layer 1: System prompt.** The LLM reads the table in the viewport block and copies the action value verbatim.

**Layer 2: `_ACTION_SYNONYMS` dict** in `ui_action_mixin.py`. If the LLM emits a synonym, it's mapped to the canonical action *before* `_dispatch_mission` looks at it.

**Layer 3: Fuzzy coercion** in `_dispatch_mission` for anything the synonyms dict didn't catch. Substring keyword match → coerce. Default to `click` (safest default — almost every utterance about touching something maps to click).

This three-layer redundancy is why the system never rejects the user. Even if the LLM hallucinated an action verb in Klingon, fuzzy coercion would catch the closest canonical action and dispatch.

---

## 8. Gaps, bugs, and improvements

This is the meat. Ordered by impact. Each entry: what it is, evidence, user-visible symptom, what would fix it.

### Critical

#### G1. The feedback loop is open — agent claims success it can't see
- **Where:** `pipeline_service.py:_process_data_message` — `mission_status` / `mission_phase` / `verification_result` are pretty-printed and dropped. `_dispatch_mission` returns `{"status": "dispatched"}` the moment bytes hit LiveKit.
- **Symptom:** User: *"open Welcome Back."* Agent: *"Opening it now!"* — but the SDK Explorer exhausted its scroll budget and emitted `verification_result {success: false, failure_reason: "target_still_offscreen"}`. The user stares at an unchanged screen while the agent claims victory. Agent **cannot self-correct, retry differently, or even apologize**, because it never learns the mission failed.
- **Fix shape:** add `agent.active_missions: dict[mission_id, MissionState]` accumulating inbound status/phase/result, feed terminal outcomes into the next LLM turn (function-call-output style: "mission m_xxx failed: target_still_offscreen"). Roadmap R1.

#### G2. `speak` message is missing on the Python side
- **Where:** `util/mission_wire.py` has no `build_speak_message`. SDK's `VoiceLoop.handlePayload` has `case SpeakMessage.kType` ready to consume.
- **Symptom:** even if you wire status events back to the agent (G1), you can't deterministically narrate phase transitions ("Looking for that…" → "Opening it now…" → "Done"). `session.generate_reply(instructions=...)` is non-deterministic at this latency budget; `session.say(text)` is the right primitive. But the *channel for the SDK to actually play the text via its TTS* is the `speak` wire message — and we have no builder.
- **Fix shape:** add `build_speak_message(text: str, interrupt: bool = False) -> str` to `mission_wire.py` mirroring `embed-flutter/lib/core/protocol/event.dart:SpeakMessage`. Wire `mission_phase` → debounced send. Phase 4 of original migration.

#### G3. `selected` field is silently dropped
- **Where:** `ui_snapshot.py:UiNode.from_wire` (line 55–64) — no `selected` field. SDK's `UiNode.toWire` emits `'selected': true` for active tabs, checked checkboxes, selected chips.
- **Symptom:** backend has no way to know which tab is currently active. The LLM's "you're already on Orders" replies are fabrications.
- **Fix shape:** add `selected: bool = False` to `UiNode`, parse from `raw.get("selected", False)`, render in viewport block (`(active)` marker on selected items).

### High impact

#### G4. No durable mission outcome store
- **Where:** `_persist_mission_exploration` writes to `./exploration_logs/` — pod-local, ephemeral on every restart/deploy/scale-down. `agent.last_mission_exploration` is in-memory, dies with the session.
- **Symptom:** can't answer "what's our mission success rate per agent / screen / action?" without grepping CloudWatch line by line. Mission outcomes never reach post-call extraction or analytics.
- **Fix shape:** DynamoDB sink (`PK=workspace_id#call_id`, `SK=mission_id`) capturing action, target, status, evidence, attempts, latency. Optionally S3 for full audit blob. Roadmap R2.

#### G5. Single-step missions only — backend never plans multi-step
- **Where:** `_dispatch_mission` builds exactly one `MissionStep` per `perform_ui_action` call. Wire + SDK `MissionRunner` fully support `steps[]` with per-step verify/retry.
- **Symptom:** "Go to orders and open the first one" = two LLM turns, two round-trips. Because of G1, the LLM fires the second before knowing the first landed. Compound tasks are fragile and slow.
- **Fix shape:** allow `perform_ui_action` to accept an ordered list, OR add a separate `perform_ui_mission(steps=[...])` tool. Roadmap R3.

#### G6. `cancel` and `highlight` outbound are dead
- **Where:** `mission_wire.py` has the builders but no code path calls them. SDK fully supports both (`_cancelMission` cancels the token; `HighlightDispatcher` paints the pulse).
- **Symptom:** user mid-mission interrupts ("no wait, stop") — mission keeps running. Agent can't proactively point at something ("see this button here?") without a full mission.
- **Fix shape:** detect user interrupt → `await room.send_data(build_cancel_message(mission_id))`. Expose a lightweight `highlight` tool. Roadmap R6.

#### G7. Repos linked only on an unmerged SDK branch
- **Where:** SDK `master` (v0.0.17) doesn't have any action-intelligence code. The pipeline lives only on `Architectur_Change_Action_intelligence`.
- **Symptom:** backend changes can't be integration-tested against the published SDK; consumer apps can't pull a release. Cycle time for any wire-format fix requires SDK release coordination.
- **Fix shape:** product/eng decision needed — what's the merge + release plan? Until then, every doc claim about "the SDK does X" is technically about an unreleased branch.

### Medium

#### G8. Token cost of viewport injection (default-on)
- **Where:** `render_viewport_ground_truth` injects ~40-150 nodes × ~30 tokens into the system prompt every LLM turn. Default-on for *every* embedded agent (Gap 0 in `action-intelligence.md`).
- **Symptom:** chatting agents pay viewport token cost even when no UI request is in flight. Offscreen nodes are included.
- **Fix shape (options):**
  (a) inject visible (`v=1`) nodes only; let SDK's blind-scroll handle offscreen;
  (b) strip non-semantic container nodes;
  (c) convert to a `read_screen()` tool the LLM calls only when needed (cleanest but changes determinism guarantees — needs design doc).
  Roadmap R4. Measure first.

#### G9. DynamicAgent (workflow agents) lacks UI tree
- **Where:** `EmbedPipelineAgent` receives `ui_snapshot`/`ui_diff` via `pipeline_service`. `DynamicAgent` (multi-agent / workflow) doesn't.
- **Symptom:** action intelligence silently doesn't work inside workflow flows even though the user expects it.
- **Fix shape:** unify the UI-state interface. Either move the snapshot-holding fields up the inheritance chain, OR add a `UiStateHolder` protocol both classes implement. Roadmap R6.

#### G10. Top-level `paths` and `visible` ignored on the backend
- **Where:** `UiSnapshot.from_ui_snapshot_payload` only reads `nodes/screen/captured_at/viewport`. SDK emits `paths: {stableId→dotted-path}` and `visible: [indices]` at the top level.
- **Symptom:** any backend logic that wants ancestor relationships (e.g. "is this a child of a scroll container?") can't get them. Wire bandwidth wasted.
- **Fix shape:** parse top-level `paths` into a `paths: dict[str, str]` field on `UiSnapshot`. Use it for path-based override guards (already implemented loosely via stable_id token tests — can be tightened).

#### G11. Per-node `path` is always empty (silent parsing bug)
- **Where:** `UiNode.from_wire` reads `raw.get("path", "")`. SDK doesn't emit `path` on individual nodes (it's in the top-level `paths` map). So `node.path` is `""` always for snapshot-derived nodes; only `moved` diff entries — which DO carry per-node `path` — populate it.
- **Symptom:** subtle bug. Looks like the field works (it doesn't crash). But any code that reads `node.path` assuming it has data will silently get empty strings.
- **Fix shape:** either delete the `path` field from `UiNode.from_wire` and pull paths from the top-level `paths` map (combine with G10), or document explicitly that `path` only populates on `moved` diffs.

### Low / doc-vs-code drift

#### D1. Docs say "opt-in"; code is "default-on with explicit opt-out"
- The migration doc §7 (`action-intelligence-backend-migration.md`) says "Opt-in per agent. A row in `agents.room_config.function_list` with `type: 'perform_ui_action'` is what enables UI actions."
- The current code (lines 64, 212–228 of `ui_action_mixin.py`) enables UI actions **unless** there's an entry with `enabled: false`. Absence enables. This is the source of Gap 8 (viewport cost on every embedded agent).
- `action-intelligence.md` Gap 0 acknowledges this; the migration doc is now stale on this point.

#### D2. Docs reference paths that don't exist
- Migration doc: `embed-flutter/lib/protocol/mission.dart`.
- Reality: `embed-flutter/lib/core/protocol/mission.dart` (on the action-intelligence branch).
- Multiple references throughout both docs follow the wrong `lib/protocol/` prefix.

#### D3. `perform_ui_action` / `_dispatch_mission` moved
- Docs say they're in `embed_pipeline_agent.py`.
- They're in `agents/ui_action_mixin.py` (which `EmbedPipelineAgent` inherits as a mixin).
- Line-number references in the migration doc are stale.

#### D4. Two parallel code paths (legacy resolver)
- `util/ui_action_resolver.py` (gpt-4o-mini) is maintained alongside `_dispatch_mission`. Intent → Action table exists in BOTH prompts; rules can drift.
- **Fix shape:** extract the shared Intent → Action table into a constant both prompts import. Plan Android mission-protocol rollout to delete the legacy path. Roadmap R5.

#### D5. Action vocab mismatch
- Dart `MissionStep` docstring lists `tap_bounds` as an action. Python `MISSION_ACTIONS` doesn't include it. Backend can never emit, SDK can never receive from backend. Probably dead surface — confirm and remove from Dart, or add to Python.

### Specific code observations worth knowing

- **`MISSION_ACTIONS` lists `type` as a canonical action** but the Intent → Action table doesn't reference it (all the synonyms route to `set_text`). It exists for parity with the Dart executor vocab (`set_text` replaces a field; `type` adds to current focus). Not a bug, but unclear whether the agent will ever pick it.
- **SDK's `forcePush` always sends even on empty diffs** (as a heartbeat); `schedulePush` skips empty. Backend can use the heartbeat to detect "we're still connected and the screen hasn't changed" — currently no code uses it.
- **`_early_data_buffer` re-buffers messages** that can't apply during replay (agent still not fully ready). The second-pass replay drops excess to avoid stale data. If the backend ever scales to support multiple concurrent missions during agent assembly, this drop is the bound.
- **Mission `params` is loose schema** by design (Dart comment in mission.dart). Both sides agree to treat it as a bag. Action-specific keys: `text` (set_text), `duration_ms` (wait), `scroll_within_stable_id` (scroll). Backend currently uses only `text`.

---

## 9. Roadmap (where to go from here)

From `action-intelligence.md` §4, with priority + dependencies:

| ID | What | Depends on | Priority |
|---|---|---|---|
| **R1** | Close the feedback loop (consume status/phase/result) + `speak` narration | nothing (start here) | ⭐ |
| **R2** | Durable mission telemetry (DynamoDB) | R1 (so terminal outcomes are reliable) | high |
| **R3** | Multi-step missions | R1 (so each step's failure is observable) | medium |
| **R4** | Viewport cost reduction | nothing; measure first | medium |
| **R5** | Contract hardening + legacy resolver sunset | Android mission-protocol rollout | medium |
| **R6** | Cancel, standalone highlight, DynamicAgent parity | nothing | low–medium |
| **R0** *(implicit)* | SDK branch merge + release | product/eng decision | blocker for all |

R1 is the keystone: every other roadmap item compounds on it. If you can only build one thing this quarter, it's R1.

---

## 10. Quick reference

### File map

**Backend (`revrag-core/app/modules/pipeline/`)**
| File | Role |
|---|---|
| `pipeline_service.py` | LiveKit data-channel inbound router; 7 message types; `_persist_mission_exploration`; `_early_data_buffer` |
| `agents/embed_pipeline_agent.py` | Concrete agent class; `llm_node` viewport injection; tool composition |
| `agents/ui_action_mixin.py` | The brain: `perform_ui_action`, `_dispatch_mission`, `_ACTION_SYNONYMS`, `render_viewport_ground_truth` |
| `util/ui_snapshot.py` | Typed in-memory tree; `apply_diff`; `render_for_prompt` sectioning |
| `util/mission_wire.py` | Outbound builders: `mission`/`cancel`/`highlight`; `MISSION_ACTIONS` |
| `util/ui_action_resolver.py` | Legacy Android path; gpt-4o-mini resolver; 187-keyword pre-filter |

**SDK (`embed-flutter/lib/core/` on branch `Architectur_Change_Action_intelligence`)**
| Dir | Files | Role |
|---|---|---|
| `tree/` | capture, normalize, stable_ids, snapshot, diff_engine, active_screen_detector | Live Flutter tree → wire-format snapshot/diff |
| `runtime/` | mission_runner, voice_loop, snapshot_pump, exploration_log, nav_stack, agent_runtime | State machine + transport + audit |
| `protocol/` | event, mission, ui_diff, result, mission_exploration | Wire envelopes (source of truth for field names) |
| `explore/` | explorer, scroller_impls | 5-rung match + scroll-search |
| `executors/` | tap, scroll, input, nav, back, wait + executor_result | Action dispatch |
| `verify/` | verifier | Post-action evidence + verdict |
| `overlay/` | highlighter, active_border, awakening_overlay | Visual feedback |

### Wire-message glossary

| Direction | Type | Purpose |
|---|---|---|
| ← (SDK→backend) | `ui_snapshot` | Full screen state (on connect / route change baseline) |
| ← | `ui_diff` | Sparse delta (per scroll / state change) |
| ← | `mission_status` | Lifecycle: started/step_started/step_completed/succeeded/failed/cancelled |
| ← | `mission_phase` | Per-step phase: searching/executing/verifying |
| ← | `verification_result` | Per-step outcome with evidence list |
| ← | `mission_exploration` | One-shot terminal audit (all nodes seen during mission) |
| → (backend→SDK) | `mission` | Execute these steps |
| → | `cancel` | Stop this mission (currently never emitted) |
| → | `highlight` | Standalone visual pulse (currently never emitted) |
| → | `speak` | TTS narration (Python missing builder) |

### Magic numbers worth knowing

| Where | Value | Meaning |
|---|---|---|
| MissionRunner | `stepTimeout: 30s` | per-step wall clock |
| MissionRunner | `missionTimeout: 3min` | per-mission wall clock |
| MissionRunner | `retriesPerStep: 2` | 3 total attempts per step |
| MissionRunner | `retryDelay: 450ms` | between attempts |
| MissionRunner | `postExecuteSettle: 400ms` (+150ms for back/navigate) | animation settle |
| Explorer | `maxAttempts: 106` | scroll budget per direction (not shared in `both`) |
| Explorer | `offsetTolerance: 8px` | cycle detection threshold |
| Explorer | `scanTimeout: 6s` | wall clock per scan (shared in `both` mode) |
| SnapshotPump | `debounce: 500ms` | scroll-event coalesce window |
| InputExecutor | 50ms | post-tap focus wait before re-reading actions mask |
| WaitExecutor | 200ms poll / 5s timeout | default conditions |
| Verifier | `boundsMovedPx: 4.0` | bounds-moved threshold |
| ui_snapshot.py | `max_per_section: 30` | viewport block per-section cap |
| stable_ids.dart | 24 chars | slug length cap |
| normalize.dart | 100 chars | text merge length cap |

### Useful grep tags

```bash
# Backend
grep -rn "[ActionIntel ▶ SEND]"     # outbound missions
grep -rn "[ActionIntel ◀ RECV]"     # inbound SDK events
grep -rn "PAYGENT_UNMETERED"        # unmetered AI calls (not action-intel but worth knowing)

# SDK
grep -rn "WireLog.enabled"          # wire trace toggle
grep -rn "ExplorationLog"           # audit log paths
```

### Operational quick refs

- Restart pipeline after backend code changes: `uv run pipeline dev`.
- Per-agent opt-out: insert/update `agents.room_config.function_list` entry with `type: "perform_ui_action", enabled: false`.
- Exploration logs land in `./exploration_logs/` by default (CWD-relative, ephemeral on pods). Override via `$REVRAG_EXPLORATION_LOG_DIR`.
- SDK branch checkout: `cd embed-flutter && git checkout Architectur_Change_Action_intelligence`.

---

## Appendix A — Why a mission "fails" the way it does

Reading `verification_result.failure_reason` you'll see strings like `target_still_offscreen`, `no_observable_effect`, `field_disappeared_after_input`, `typed_text_not_present_in_field`. These are SDK-side strings produced by `Verifier`. Map them back to user experience:

| Failure reason | Most likely cause | What you'd tell the user (if R1 were done) |
|---|---|---|
| `target_not_found` | Target doesn't exist in any reachable scroll position | "I couldn't find that — can you describe it differently?" |
| `target_still_offscreen` | Scroll didn't bring it into view (clipped by drawer/overlay) | "I scrolled but couldn't bring it into view. Can you scroll there yourself?" |
| `scrollEndReached` | Hit the bottom/top of every scrollable; target isn't here | "I don't see it in this section." |
| `alreadyVisited` | Scroll cycled (carousel / rubber-band list) | "This is a carousel — let me try a different angle." |
| `no_observable_effect` | Tap fired but nothing changed (disabled button? race?) | "I tapped but nothing happened. Did the screen change?" |
| `route_changed` is missing for navigate | The tap didn't actually navigate (maybe opened a sub-menu) | "I tried, but the screen didn't change as expected." |
| `field_disappeared_after_input` | Input field went away mid-type | "The field went away while I was typing." |
| `typed_text_not_present_in_field` | Field rejected the input | "The field didn't accept what I typed." |
| `back_handler_threw` / `no_back_target` | Host's back handler isn't wired or has no history | "I can't go back from here." |

This table is what an R1-completed agent should be able to *honestly* produce instead of saying "Done!" regardless of outcome.

---

## Appendix B — A worked example of every guardrail firing

Hypothetical request: *"open my account page"* on a screen where:
- the viewport has a content card `home.recent.arjun_05_10` with text "Arjun 05:10",
- a real tab `main.tabs.account` (offscreen, not in viewport block),
- the LLM hallucinates the target as `home.recent.arjun_05_10` because it text-matched on "account" somewhere in the row.

What happens:

1. LLM calls `perform_ui_action(action="navigate", target_description="account", target_stable_id="home.recent.arjun_05_10", target_role="tab")`.
2. `_dispatch_mission`:
   - Stage 1 (action coercion): `navigate` is in `MISSION_ACTIONS`. Pass-through.
   - Stage 2 (snapshot resolution): `home.recent.arjun_05_10` IS in the snapshot. Resolves.
   - Stage 3 (override guards):
     - **Nav-intent content-row guard fires**: action is `navigate`, stable_id has `recent` token, role would be content. → drop the node.
     - `target_description` derived from `target_description` (or stable_id tail) — already "account".
     - `target_role` promoted to `tab` (already was, in this case).
   - Stage 4 (blind dispatch): build step with `target_text="account"`, `target_role="tab"`.
     - Role-omit check: are there any nodes in snapshot with role=tab AND text matching "account"? No → drop role constraint.
3. Mission ships with just `target_text="account"`.
4. SDK Explorer:
   - Rung 1 (stable_id): empty.
   - Rung 2 (identifier): empty.
   - Rung 3 (text): scans for "account". Finds nothing visible matching "account" as substring → does a scroll search.
   - `findOrScroll` scrolls down 6s budget, doesn't find. Tries backward, finds `main.tabs.account` in the bottom navigation.
5. TapExecutor:
   - Rung 1 (semantic tap): node has `tap` in actions → fires `SemanticsAction.tap`. Returns `strategy: 'semantics_tap'`.
6. Settle 550ms (navigate gets +150ms).
7. Verifier sees `routeChanged` (screen changed from `home_tab` to `account_tab`). `navigate` action accepts non-empty evidence → success.
8. SDK emits `mission_status: succeeded` + `mission_exploration` with both screens worth of nodes.
9. Backend logs both. LLM never learns the mission *actually worked* — it just optimistically said "Opening account!" two seconds earlier. (Same blindness for the failure case.)

Every override guard, every fallback ladder, every retry budget — they all exist because each was once the only thing standing between a working mission and a broken one. The current open loop is the next thing standing between this system and being truly self-correcting.
