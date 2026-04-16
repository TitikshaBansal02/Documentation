# Optimization: Reduce Caller Wait Time
 
| Field        | Details                              |
|--------------|--------------------------------------|
| **Branch**   | Titiksha/agent-start-optimization    |
| **Type**     | Optimization                         |
| **Status**   | In Progress                          |
| **Started**  | 2026-04-16                           |
| **Last Updated** | 2026-04-16                       |
 
---
 
## Overview
 
Reduce the caller-visible wait time (time from call initiation to first audio) to **under 1 second** across all four call types: In-app SDK (Embedded), Web Call, Inbound Phone, and Outbound Phone.
 
---
 
## Background / Context
 
Every call type shares a common setup pipeline (`create_call_log_and_get_config`) plus a worker entrypoint (`agent_entrypoint`) that together introduce 800ms–3s+ of silence before the caller hears anything. The goal of this optimization is to identify all blocking steps in this path and eliminate, parallelize, or defer them.
 
---
 
## Call Flow Analysis
 
### Shared Block — `create_call_log_and_get_config`
> Runs in every flow. Located at `call_service.py:273–647`
 
1. `fetch_agent_data_for_call` — DB query (prompt + KBs + configs)
2. `_handle_previous_outbound_call` (EMBEDDED only) — scans last 5 min of outbound logs
3. `_hydrate_agent_knowledge_bases` — fire-and-forget to RAG service (non-blocking, but racy with first `search_kb`)
4. `get_llm_config_details` — DB query
5. `get_stt_config_details` — DB query
6. `get_tts_config_details` — 3 sequential DB queries
7. `paygent_service.build_call_params` — DB/API call
8. `call_logs_repository.create` — DB INSERT
9. `_upload_call_config_to_s3` — serialize (50KB+) → S3 PUT → S3 GET (12-hr presigned URL)
10. `_acquire_org_concurrency_slot` — DB query + Redis HSET
**Observed cost:** ~300–800ms on caller-facing request.
 
---
 
### 1. In-app SDK (EMBEDDED)
> `embedded_agent_routes.py:45` → `EmbeddedAgentRuntimeService.create_token_and_save_config_for_embedded_call`
 
**HTTP request phase (caller-visible):**
1. `config_repo.get_by_api_key` — DB
2. `get_workspace_id_by_agent_id_from_db` — DB
3. Embedded context fetch: `user_data` — DB (last 30 min)
4. Embedded context fetch: `offer_state` — DB (last 30 min)
5. Embedded context fetch: `form_change` — DB
6. Variable translation / phone extraction — CPU
7. `create_call_log_and_get_config` — shared block above
8. `_acquire_org_concurrency_slot` — DB + Redis
9. `generate_livekit_token` — JWT signing (CPU)
**LiveKit phase:**
10. SDK opens WebRTC/WS to LiveKit
11. LiveKit auto-dispatches worker
12. Worker entrypoint (see Worker Runtime section)
 
---
 
### 2. Web Call (WEB / CHAT)
> `call_routes.py:31` → `CallService.create_web_call_token`
 
**HTTP request phase:**
1. `create_call_log_and_get_config` — shared block
2. `_acquire_org_concurrency_slot` — DB + Redis
3. `generate_livekit_token` — JWT signing
**LiveKit phase:**
4. Browser opens WebRTC (ICE + DTLS handshake)
5. LiveKit auto-dispatches worker
6. Worker entrypoint
 
---
 
### 3. Inbound Call (PHONE_INBOUND)
 
**Telephony phase:**
1. SIP INVITE → LiveKit SIP creates room
2. `room_started` webhook → Redis update
3. LiveKit dispatches worker
**Worker phase (caller already on the line — every step here is audible silence):**
4. `get_pipeline_metadata` — parses `InboundCallMetaData`
5. `_check_inbound_org_concurrency` — duplicate `fetch_agent_data_for_call` DB query + Redis write
6. **Lazy `create_call_log_and_get_config`** — entire shared block runs here (biggest inbound-only offender)
7. S3 GET of the config just uploaded in step 6
8. `ctx.connect()` to LiveKit
9. Rest of worker entrypoint
 
> ⚠️ Unlike other flows, the full config build runs **after** the caller is already on the line.
 
---
 
### 4. Outbound Call (PHONE_OUTBOUND)
> `call_routes.py:45` → `CallService.create_outbound_call`
 
**HTTP request phase:**
1. Agent resolution — up to 3 sequential DB queries (CohortTrigger / PhoneNumber / fallback)
2. SIP trunk resolution — extra DB query
3. `create_call_log_and_get_config` — shared block
4. `_acquire_org_concurrency_slot` — DB + Redis
5. `agent_dispatch.create_dispatch` — LiveKit API HTTP call
6. `agent_dispatch.list_dispatch` — **extra HTTP call for debug logging only**
**LiveKit / SIP phase:**
7. Worker entrypoint up through `load_initial_prompt`
8. `_create_outbound_sip_participant` — SIP dial
9. Carrier ringing (0–60s)
10. Callee answers → `wait_for_participant`
11. Session start + `on_enter` welcome
 
---
 
### 5. Worker Runtime (Common to All Four)
> `worker_service.py` + `pipeline_service.py:agent_entrypoint`
 
**Per-worker (amortized):** `prewarm_fnc` loads WebRTC VAD + Silero VAD.
 
**Per-call sequential awaits before first audio:**
1. `get_pipeline_metadata` — S3 GET of CallConfigData
2. `_handle_language_switch` — conditional DB query
3. `ctx.connect()` — LiveKit WS handshake (~200–500ms)
4. `initialize_agent_plugins` — Egress + Metrics objects
5. `run_pre_call_processing` — optional external API (blocking, can be seconds)
6. `load_agent_session` — STT + TTS + LLM plugins instantiated **sequentially**, fresh every call
7. `load_initial_prompt` — variable substitution + `fetch_dependent_agent_memories` (DB/API)
8. `_create_pipeline_agent` — reads `system_prompt_v2_ai_calling_hinglish.txt` **from disk every call**
9. `_handle_participant_connection` — SIP dial for outbound; `wait_for_participant` for others
10. `agent_session.start()` — STT/TTS/LLM WebSocket connections
11. `PipelineAgent.on_enter` — `session.say(welcome)` or `session.generate_reply()` (LLM round-trip)
---
 
## Identified Delay Sources
 
### Hotspot Table
 
| Hotspot | Typical Cost | Affects |
|---|---|---|
| 3 sequential config-detail DB queries (LLM/STT/TTS) | 50–150ms | All 4 |
| S3 PUT + presigned URL GET in request path | 100–300ms | Web / Embedded / Outbound |
| S3 GET of CallConfigData inside worker | 100–300ms | All 4 |
| Full lazy `create_call_log_and_get_config` inside worker | 350ms–1.2s | Inbound only |
| Fresh STT/TTS/LLM plugin instantiation, sequential | 150–300ms | All 4 |
| `agent_session.start` WS handshakes | 300ms–1s | All 4 |
| `session.generate_reply` for dynamic welcome (LLM TTFT) | 300ms–1s | Agents without static welcome |
| Outbound SIP dial ring time | 0–60s | Outbound |
| Redundant `list_dispatch` debug RPC | 30–100ms | Outbound |
 
### Full Delay Inventory (43 sources)
 
| # | Source | Type | Affects |
|---|---|---|---|
| 1 | API-key auth DB lookup | DB | Embedded |
| 2 | Workspace / agent resolution DB queries | DB | Embedded, Outbound |
| 3 | Embedded context fetches ×3 | DB | Embedded |
| 4 | Previous outbound call scan | DB | Embedded |
| 5 | `fetch_agent_data_for_call` with eager joins | DB | All |
| 6 | KB hydration fire-and-forget — racy with first search | Network | All |
| 7 | LLM config detail DB query | DB | All |
| 8 | STT config detail DB query | DB | All |
| 9 | TTS config detail — 3 sequential DB queries | DB | All |
| 10 | Call-transfer SIP lookup | DB | Conditional |
| 11 | PhonePe / MagicBricks branch extra queries | DB | Conditional |
| 12 | Paygent billing DB/API | DB/API | All |
| 13 | `CallLogs` INSERT | DB | All |
| 14 | Config serialization + S3 PUT | S3 | All |
| 15 | S3 presigned URL generation (12hr for one-shot use) | S3 | All |
| 16 | Org concurrency slot DB + Redis | DB + Redis | All |
| 17 | LiveKit JWT signing | CPU | Embedded, Web |
| 18 | `create_dispatch` HTTP call | Network | Outbound |
| 19 | Redundant `list_dispatch` debug HTTP | Network | Outbound |
| 20 | LiveKit room / dispatch queue wait | Queue | All |
| 21 | Full lazy config build inside worker | DB + S3 | Inbound |
| 22 | Duplicate agent fetch in inbound concurrency check | DB | Inbound |
| 23 | S3 GET of config inside worker | S3 | All |
| 24 | `ctx.connect()` LiveKit WS handshake | Network | All |
| 25 | Pre-call processing external call | External API | Optional |
| 26 | STT plugin instantiation — fresh per call | Init | All |
| 27 | TTS plugin instantiation — fresh per call | Init | All |
| 28 | LLM plugin instantiation — fresh per call | Init | All |
| 29 | Sequential plugin instantiation (no `asyncio.gather`) | Init | All |
| 30 | Initial prompt build + variable substitution | CPU | All |
| 31 | Dependent-agent memories fetch | DB/API | Conditional |
| 32 | System prompt disk read per call | Disk I/O | All |
| 33 | Language-monitoring prompt build | CPU | All |
| 34 | `InterruptionHandlerLLM` construction | Init | All |
| 35 | Session listeners + tracking setup | Init | All |
| 36 | STT WebSocket connect | Network | All |
| 37 | TTS WebSocket connect | Network | All |
| 38 | LLM client connect / pool warm | Network | All |
| 39 | `wait_for_participant` | Network | All |
| 40 | SIP outbound dial + ring timeout | PSTN | Outbound |
| 41 | `session.generate_reply` LLM TTFT for dynamic welcome | LLM | No static welcome |
| 42 | TTS time-to-first-audio for welcome | TTS | All |
| 43 | Logger / Redis / boto3 cold-start on fresh worker | Cold start | All |
 
---
 
## Open Questions (Pending Validation)
 
- [ ] Is the worker co-located with Postgres/Redis? (Determines if DB cost is LAN RTT or internet RTT — changes priority ordering)
- [ ] Are static `welcome_messages` used in production, or mostly `agent_speaks_dynamic`? (Dynamic forces a full LLM turn before first audio)
- [ ] For inbound — can `CallConfigData` be pre-baked into Redis at phone-number-assign time so the worker skips the full lazy build?
- [ ] Are STT/TTS/LLM plugins safe to pool across calls, or intentionally built fresh per call for isolation/config reasons?
---
 
## Approach
 
> _To be filled once open questions above are validated._
 
Planned grouping:
 
**Easy wins (low risk, high impact):**
- Parallelize LLM/STT/TTS config DB queries with `asyncio.gather`
- Remove redundant `list_dispatch` debug RPC (outbound)
- Cache system prompt file in memory (not re-read from disk per call)
- Eliminate S3 round-trip: pass config via Redis or inline metadata instead of S3 PUT → worker S3 GET
- Fix duplicate `fetch_agent_data_for_call` in inbound concurrency check
**Structural (higher effort):**
- Pre-bake inbound `CallConfigData` into Redis at phone-number-assign time
- Pool STT/TTS/LLM clients across calls
- Move welcome-audio queuing before `wait_for_participant`
- Convert 12-hr presigned URL to a short-TTL or eliminate entirely
---
 
## Implementation Details
 
> _To be filled as changes are made._
 
---
 
## Testing
 
> _To be filled._
 
---
 
## Outcome / Results
 
> _To be filled on completion._
 
---
 
## Notes / Learnings
 
> _To be filled as work progresses._
