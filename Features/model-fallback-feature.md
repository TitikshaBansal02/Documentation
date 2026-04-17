# Feature: Independent Fallback Adapter Chains for LLM, TTS, and STT
 
| Field            | Details                                              |
|------------------|------------------------------------------------------|
| **Branch**       | revrag-core: `Titiksha/model-fallback-clean` · revrag-ui: `Titiksha/model-fallback` |
| **Merged Into**  | canary (both repos)                                  |
| **Type**         | Feature                                              |
| **Status**       | Completed                                            |
| **Started**      | 2026-04-08                                           |
| **Last Updated** | 2026-04-17 (follow-up fixes added)                   |
 
---
 
## Overview
 
Independent fallback chains for each of the three voice-agent services — LLM, TTS, and STT. Operators can configure up to 3 secondary models per service per agent. At call start, each service is wrapped in its own LiveKit `FallbackAdapter`, so an LLM provider outage does not trigger a TTS swap and vice versa.
 
Bundled alongside the failover wiring:
- A DB-driven model catalog replacing ~575 lines of hardcoded lists (`const.py`)
- New provider support: Amazon Nova 2 Lite, Gemini 3/3.1 previews, GPT-5, Cartesia Sonic 3, Sarvam Bulbul v3, ElevenLabs Scribe realtime, Deepgram Flux
- Expanded per-provider tuning knobs: `top_p`, `reasoning_effort` (LLM); `style`, `emotion`, `loudness` (TTS); `filler_words`, `interim_results`, `smart_format`, `diarize`, `detect_language` (STT)
---
 
## Problems Solved
 
| # | Problem | Impact |
|---|---------|--------|
| 1 | Single-model risk — one provider outage (OpenAI rate limit, ElevenLabs 5xx, Deepgram blip) took the whole call down | High |
| 2 | Fallover coupling — no independent chains meant a TTS failure could trigger a pointless LLM swap | High |
| 3 | Hardcoded catalog in `const.py` (~580 lines) duplicated against DB; every model update required a code change | Medium |
| 4 | Hardcoded provider parameters UI-side; adding a provider with a different range required frontend edits | Medium |
| 5 | AWS credentials hardcoded as literals in `models_util.py` | High (security) |
| 6 | Azure LLM always used `gpt-4o-mini` deployment and a hardcoded `api_version` | Medium |
| 7 | Google LLM hardcoded `temperature=0.4`, `max_output_tokens=256`, only supported one model | Medium |
 
---
 
## Changes in revrag-core
 
### Schema / DB
 
**`alembic/versions/b3c4d5e6f7a8_add_settings_jsonb_and_stt_models.py`** (new, 265 lines)
- Adds `settings JSONB` column to `llm_models` and `tts_models`; adds `avatar_url` to `tts_models`
- Creates new `stt_models` table with fields: `id`, `model_id` (unique), `model`, `display_name`, `provider`, `description`, `language_code`, `avg_latency_ms`, `is_default`, `is_active`, `settings`, `avatar_url`, timestamps
- Backfills `settings` for existing rows via `LLM_SETTINGS_BY_PROVIDER` and `TTS_SETTINGS_BY_PROVIDER` — each entry is a dict of param ranges `{min, max, step, default, min_label, max_label}`
- Seeds ~30 `stt_models` rows (Azure, Deepgram nova-3/nova-2/Flux, Google telephony/chirp, Sarvam, AWS Transcribe, ElevenLabs Scribe v2 + realtime)
- Seeds new TTS rows: `cartesia-sonic-3`, `cartesia-sonic-turbo`, `sarvam-bulbul-v3`, `sarvam-bulbul-v3-beta`
- `downgrade()` drops the `stt_models` table, new TTS rows, `avatar_url`, and both `settings` columns
**`app/models/agents/voices/stt_model.py`** (new) — SQLAlchemy `STTModel` matching the migration
 
**`app/models/agents/voices/llm_model.py`** — adds `settings: JSONB` column
 
**`app/models/agents/voices/tts_model.py`** — adds `settings: JSONB` and `avatar_url: String` columns
 
**`app/models/__init__.py`** — exports `STTModel`
 
---
 
### Pydantic Config Models
 
**`app/models/agents/agent/agent.py`**
- `LLMConfig`: adds `top_p: Optional[float]`, `reasoning_effort: Optional[str]` (`"low"` | `"medium"` | `"high"`)
- `TTSConfig`: adds `style` (ElevenLabs), `emotion` (Cartesia), `loudness` (Sarvam)
- `STTConfig`: default model changed `"default"` → `"nova-3"`; adds Deepgram flags (`filler_words`, `interim_results`, `smart_format`, `diarize`, `detect_language`) and ElevenLabs `language` BCP-47 override
- New classes: `FallbackAdapterSettings` (attempt_timeout=10.0, max_retry=1), `LLMFallbackConfig`, `TTSFallbackConfig`, `STTFallbackConfig` — all fields `Optional` so fallbacks can be registered with just `{model_id, provider}` and inherit tuning from the primary at runtime
- `RoomConfig`: adds `llm_fallbacks: List[LLMFallbackConfig] = []`, `tts_fallbacks`, `stt_fallbacks`, `fallback_adapter_settings: FallbackAdapterSettings`
---
 
### Agent API
 
**`app/modules/agents/agent/schema/agent_schema.py`**
- `LLMUpdateSchema`: validators for `top_p` (0–1) and `reasoning_effort` (enum)
- `TTSUpdateSchema`: speed range widened `0.5–2.0` → `0.25–4.0`; pitch range `-1..1` → `-20..20` (Sarvam scale); adds `style`, `emotion`, `loudness` validators
- `STTUpdateSchema`: default provider changed `"azure"` → `"deepgram"`, default model `"default"` → `"nova-3"`; adds Deepgram flag fields
- `UpdateAgentRequest`: adds `llm_fallbacks`, `tts_fallbacks`, `stt_fallbacks`, `fallback_adapter_settings`
- `TTSModelSchema` / `STTModelSchema` / `CommonModelSchema`: adds `settings: Optional[dict]`; `TTSModelSchema` adds `model_id`; `STTModelSchema` adds `language_code`
**`app/modules/agents/agent/routes/agent_routes.py`** — `/agent/models` becomes `async`, receives a `SessionDB`
 
**`app/modules/agents/agent/service/agent_service.py`**
- `get_models()` rewritten: now async, queries `AgentModelService.get_llm_models()` / `get_tts_models()` / `get_stt_models()` and the `agent_voices` table instead of the deleted `const.py` lists. Pipes each row's `settings` JSONB into the response
- `update_agent_config()`: persists `llm_fallbacks`, `tts_fallbacks`, `stt_fallbacks`, `fallback_adapter_settings` using the existing patch-or-keep pattern
- All imports of `LLM_MODELS`, `STT_LANGUAGES`, `TTS_MODELS`, `TTS_VOICES` from `const.py` removed
**`app/modules/agents/agent/service/const.py`** — 575 lines of hardcoded model/voice/language lists deleted; only `AgentUseCase` / `AgentUseCasesListingSchema` re-export remains
 
---
 
### New STT Model Lookup Subsystem
 
**`app/modules/agents/agent_models/repository/stt_model_repository.py`** (new)
- `get_stt_models(provider?, language_code?)`, `get_stt_model_by_id`, `get_default_stt_model_by_provider`
**`app/modules/agents/agent_models/schema/stt_model_schema.py`** (new)
- `STTModelResponseSchema`, `STTModelListResponseSchema`, Create/Update schemas
**`app/modules/agents/agent_models/service/agent_model_service.py`** — adds `stt_model_repo` and `get_stt_models()`
 
**`app/modules/agents/agent_models/routes/agent_model_routes.py`** — adds `GET /models/stt-models?provider=&language_code=`
 
**`app/modules/agents/agent_models/schema/llm_model_schema.py` / `tts_model_schema.py`** — response schemas add `settings: Optional[dict]`; TTS also adds `avatar_url`
 
---
 
### Core Wiring — FallbackAdapter Builders
 
**`app/modules/pipeline/util/models_util.py`** (major file, +617 lines)
 
Provider fixes in existing builders:
- `get_llm_model()`: OpenAI/Google pass `top_p` and `reasoning_effort` as kwargs when set; Azure uses `azure_deployment=llm_config.model_value` (was hardcoded `gpt-4o-mini`) and `api_version` from config; Google uses `model=llm_config.model_value` and `temperature=llm_config.temperature` (were hardcoded); AWS credentials moved to `config.AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_BEDROCK_REGION`
- `get_stt_model()`: new AWS STT branch; Deepgram adds `filler_words`, `interim_results`, `smart_format`, `diarize`, `detect_language`; ElevenLabs adds `model_id` + language override; all providers use `model` from config with provider-specific fallback defaults
- `get_tts_model()`: ElevenLabs default fallback changed to `eleven_flash_v2_5`; Cartesia default changed to `sonic-3`; Sarvam pitch mapping restored; raw `speed` passed through without remapping; `style`, `volume`, `emotion`, `loudness` added per provider
New public methods `build_llm_with_fallback`, `build_tts_with_fallback`, `build_stt_with_fallback`:
1. Iterate fallback configs; skip entries missing `model_id` or `provider`
2. Inherit missing tuning fields from the primary via `_inherit(fb, name, default)`
3. Catch exceptions per-fallback with `logger.warning` — one bad fallback never fails the call
4. Delegate to `_wrap_fallbacks()` which instantiates the LiveKit `FallbackAdapter`, subscribes to `availability_changed` events, and forwards to `metrics_service.record_fallback_event()`
5. Return the primary unchanged if zero valid fallbacks remain
6. **Quirk:** `TTS FallbackAdapter` does not accept `attempt_timeout` — builder conditionally omits it for TTS only
New private helper `_ensure_google_credentials() -> bool` centralizes VertexAI creds check. Returns `False` so Google LLM falls back to the API-key path instead of raising. Evaluation LLM still hard-requires VertexAI.
 
**`app/modules/pipeline/util/agent_util.py`** — `load_agent_session()` now receives optional `metrics_service`. After building primary LLM/TTS/STT, reads `room_config.fallback_adapter_settings` and calls the corresponding builder per service
 
**`app/modules/pipeline/pipeline_service.py`** — passes `metrics_service=pipeline_plugins.metrics` to `load_agent_session`
 
**`app/modules/pipeline/tools/general_tools.py`** — on mid-call language switch, target STT/TTS are re-wrapped with `FallbackAdapter`s using the target agent's `room_config`, so fallback protection survives agent hand-off
 
---
 
### Metrics
 
**`app/modules/pipeline/schema/metrics_schemas.py`** — new `FallbackEvent(timestamp, service, provider, available)`; `MetricsCollection` gains `fallback_events: List[FallbackEvent]`
 
**`app/modules/pipeline/service/metrics_service.py`** — new `record_fallback_event(service, provider, available)` appends the event and logs a warning
 
---
 
### Other
 
**`app/modules/call/service/call_service.py`** — propagates `top_p` and `reasoning_effort` when building `LLMConfigDetails` from DB rows
 
**`app/modules/rag/service/vector_store_service.py`** — guards `hnswlib.add_items` with `if embedding_vectors:` check (empty list previously raised "Wrong dimensionality"). Collateral fix, unrelated to fallback feature.
 
**`app/core/config.py`** — adds `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_BEDROCK_REGION` (default `ap-south-1`), `AZURE_OPENAI_API_VERSION` (default `2024-10-01-preview`); removes hardcoded VertexAI project ID fallback
 
**`scripts/seed_db.py`** (+218 lines) — fixes AWS avatar URLs, normalizes Google display names, adds new LLM rows (GPT-5, Amazon Nova 2 Lite, Gemini 3/3.1 previews), adds `settings` dict to every LLM/TTS row, seeds new `stt_models` list
 
---
 
### Tests
 
**`tests/unit/modules/pipeline/util/test_fallback_adapter.py`** (new, 428 lines) — patches LiveKit's three `FallbackAdapter` classes with `_FakeAdapter` and asserts:
- Builder skips incomplete configs
- Missing fields inherited from primary
- Primary returned unchanged when no valid fallbacks
- Per-service retry kwarg (`max_retry_per_llm/_tts/_stt`) correctly wired
- `attempt_timeout` omitted for TTS only
- Availability callback forwarded through `metrics_service`
---
 
## Changes in revrag-ui
 
### Types
 
**`src/types/agents/config.types.ts`**
- `AgentConfig_LLM` gains `top_p`, `reasoning_effort`
- `AgentConfig_TTS` gains `style`, `emotion`, `loudness`, `gender`
- `AgentConfig_STT` gains `filler_words`, `interim_results`, `smart_format`, `diarize`, `detect_language`
- New types: `AgentConfig_LLM_Fallback`, `AgentConfig_TTS_Fallback`, `AgentConfig_STT_Fallback`, `FallbackAdapterSettings` — mirrors backend Pydantic classes field-for-field
- `AgentConfig` gains `llm_fallbacks?`, `tts_fallbacks?`, `stt_fallbacks?`, `fallback_adapter_settings?`
- Model-option shapes extended: `LLMModelOption+settings`, `STTModelOption+language_code`, `TTSModelOption+model_id`
**`src/types/models/model.type.ts`** — new `ParamRange` type; `LLMModel`/`TTSModel` gain `settings: Record<string, ParamRange> | null`; new `STTModel` type; `TTSModel` gains `avatar_url`
 
---
 
### Hooks / API
 
**`src/hooks/service/models/api/model.api.ts`** — new `getSTTModelList(provider?)`; `getTTSTextToSpeechModelList` provider param is now optional
 
**`src/hooks/service/models/model.hook.ts`** — new `useSTTModels(params?)` with same caching as `useTTSModels`; `useTTSModels` accepts `undefined` provider
 
**`src/hooks/service/agents/hooks/config/api/config.api.ts`** — `fetchAgentModels` now maps `tts.model_id` and `stt.language_code` from the expanded response shape
 
---
 
### Fallbacks UI (New)
 
**`src/components/molecules/AgentSettingsModal/content/FallbacksContent.tsx`** (new, 1114 lines)
 
Exports `LLMFallbacksTab`, `TTSFallbacksTab`, `STTFallbacksTab`, all backed by a shared `useFallbackTab()` hook managing local state + dirty flag. Server config syncs in only when `!dirty` to prevent clobbering unsaved edits.
 
Shared `FallbackSlot` component flow:
1. Provider dropdown — changing provider sets `pendingProvider` and clears model selection
2. Model dropdown filtered by provider (LLM/TTS) or STT model-family + language dropdowns (STT)
3. On model commit, tuning params reset to the new provider's `settings.default`
4. Sliders driven by `getLLMParams` / `getTTSParams` / `getSTTCapabilities` — only rendered when the provider's `settings` JSONB exposes the key
`getAvailableModels()` excludes the primary and other committed slots to prevent duplicate model selection. `MAX_FALLBACKS = 3`. `isFallbackComplete()` filters placeholder entries before save.
 
Only `LLMFallbacksTab` renders the **Adapter Settings** card (`attempt_timeout` 3–30s, `max_retry` 0–3) — these apply to all three chains but are only configurable here.
 
**`src/components/molecules/AgentSettingsModal/content/model-capabilities.ts`** (new) — `getModelSettings` / `getLLMParams` / `getTTSParams` / `getSTTCapabilities` read `ParamRange`-shaped entries from `model.settings` JSONB; snake_case labels normalized to camelCase; no per-provider hardcoded ranges anywhere
 
**`src/components/molecules/AgentSettingsModal/components/VoicePicker.component.tsx`** (new, 327 lines) — standalone voice selector for primary TTS and TTS fallbacks. Gender + language filters; "Browse all voices" opens `VoicesModal` with `lockedProvider`; `autoPickDefault` uses `findBestVoiceMatch` to pre-select a voice matching primary's gender + BCP-47 when none is set
 
**`src/components/molecules/AgentSettingsModal/utils/voiceMatch.ts`** (new) — `findBestVoiceMatch(voices, provider, prefs)` with a strict → loose matching ladder: (1) provider+gender+bcp47, (2) provider+bcp47, (3) provider+iso639, (4) provider+gender, (5) first provider voice
 
---
 
### Primary Config Updates
 
**`src/components/molecules/AgentSettingsModal/content/LLMModelContent.tsx`** — sliders now driven by `getLLMParams(selectedModel)`. Adds Top P and Reasoning Effort sliders, only rendered when model exposes them. Temperature range and labels read from `settings`
 
**`src/components/molecules/AgentSettingsModal/content/VoiceSettingsContent.tsx`** (+703/−274 lines) — reworked: Provider → Model → Language → Gender → Voice selection order. All parameter sliders wrapped in `hasSettingKey(key)` guard; `getSliderConfig(key, fallback)` reads `min/max/step/default` from `model.settings`. Adds Style (ElevenLabs) and Loudness (Sarvam) sliders
 
**`src/components/molecules/AgentSettingsModal/content/AdvanceSettingsSection.tsx`** — tab order updated (LLM first); labels renamed; each primary-tab pane now renders its matching `<LLMFallbacksTab>` / `<STTFallbacksTab>` / `<TTSFallbacksTab>` below primary controls
 
**`src/components/molecules/AgentSettingsModal/AgentSettingModal.component.tsx`** — adds a dedicated "Fallbacks" top-level nav item (case 8) with LLM/TTS/STT sub-tabs. `handleProviderChange` auto-selects first model + matching voice for the provider
 
**`src/components/atoms/Modal/VoicesModal.tsx`** — new props `onVoiceSelect`, `lockedProvider`, `initialLanguage`, `initialGender` enabling select-mode; provider dropdown hidden when `lockedProvider` is set
 
---
 
## Cross-repo Relationship
 
| Flow | revrag-core | revrag-ui |
|------|-------------|-----------|
| Fallback config persistence | `RoomConfig.{llm,tts,stt}_fallbacks` + `fallback_adapter_settings` in `agent.py`; merged in `update_agent_config` | `AgentConfig.{llm,tts,stt}_fallbacks` in `config.types.ts`; edited in `FallbacksContent.tsx`; saved via existing `/agent/:id/config` endpoint |
| Model catalog | `stt_models` table + `settings` JSONB on LLM/TTS models; `GET /models/stt-models`, `GET /models/tts-models?provider=` (provider now optional) | `useSTTModels()`, `useTTSModels()` hooks pulling full catalogs for the Fallback tab |
| Parameter ranges | Stored in `settings` JSONB per row; returned verbatim by `/agent/models` and `/models/*` | `model-capabilities.ts` reads JSONB and renders sliders dynamically — no per-provider hardcoding |
| Runtime failover | `load_agent_session` wraps primary + fallbacks in LiveKit `FallbackAdapter` per service, inheriting missing fields from primary | Invisible to UI |
| Observability | `MetricsService.record_fallback_event()` appended to `MetricsCollection.fallback_events` | Not surfaced in UI yet |
 
The feature is **end-to-end complete** for config and runtime — fallbacks can be configured in the UI, they persist, and the pipeline wraps each service independently at call start. Mid-call language switch (`change_language`) also re-wraps with the target agent's fallbacks.
 
---
 
## Edge Cases & Fallback Behavior
 
**Backend `build_*_with_fallback`:**
- Skips entries missing `model_id` or `provider`
- Exceptions while building a fallback instance are caught per-entry — the call still starts with whatever fallbacks did build successfully
- Zero valid fallbacks → primary returned unwrapped (no adapter overhead)
- Missing tuning fields inherit from primary config, then from hard defaults
- STT `id` field coerces free-form string to UUID: tries `uuid.UUID(str(stt_id))` → primary's UUID on failure → `uuid.uuid4()`
- `attempt_timeout` is only passed to LLM and STT adapters; TTS adapter does not support it and it is silently omitted
**Frontend:**
- `isFallbackComplete()` filters incomplete entries (provider picked, model not yet chosen) before save
- Dirty flag prevents server→local sync from overwriting unsaved edits
- `pendingProvider` hides the previously committed model while the user is switching provider, forcing a fresh selection
- `getAvailableModels()` prevents the same model appearing in multiple slots
- On model switch within a slot, all tuning params reset to `settings.default` to prevent out-of-range carryover
- `VoicePicker.autoPickDefault` only triggers when no voice is selected — user choice is never overwritten
**Validation changes:**
- `TTSUpdateSchema.speed`: `0.5–2.0` → `0.25–4.0`
- `TTSUpdateSchema.pitch`: `-1..1` → `-20..20`
- New validators: `top_p` (0..1), `reasoning_effort` (enum), `style` (0..1), `loudness` (0.5..2.0)
---
 
## Known Gaps / TODOs
 
| # | Gap | Severity |
|---|-----|----------|
| 1 | ~~`settings` dict inconsistency between migration backfill and `seed_db.py`~~ — **resolved in follow-up** | ~~Medium~~ |
| 2 | Adapter Settings UI is LLM-tab-only but applies to all three chains — user on TTS tab has no visibility or control | Medium |
| 3 | `attempt_timeout` for TTS is silently dropped; no UI warning | Low |
| 4 | `onGenderChange={() => {}}` no-op in `AdvanceSettingsSection.tsx` and `AdvanceConfigModal.tsx` — gender prop exists but is functionally dead | Low |
| 5 | Deployment-ordering comment in `config.api.ts`: `// populated once backend is restarted with new schema` — not a permanent solution | Low |
| 6 | `STTFallbackConfig.id` is `str` while `STTConfig.id` is `uuid.UUID` — no schema-level validation, runtime `try/except` only | Medium |
| 7 | `ModelsResponseSchema.voices.language_name` is lossy — `", ".join(v.languages)` produces `"hi-IN, en-IN"` instead of human-readable names | Low |
| 8 | Old `stt_languages` table still populated; `seed_db.py` writes both old and new tables; old `/models/stt-languages` endpoint still active | Low |
| 9 | `vector_store_service.py` empty-embedding guard is collateral — fixes an hnswlib crash but has no test and no issue reference | Low |
| 10 | `MAX_FALLBACKS=3` enforced only in UI — backend API accepts any list length | Medium |
| 11 | `FallbackEvents` captured in `MetricsCollection` but no UI surface (call logs / analytics) yet | Medium |
 
---
 
## Follow-up Fixes & Improvements
 
> Same branches: `revrag-core: Titiksha/model-fallback-clean` · `revrag-ui: Titiksha/model-fallback`  
> Applied after initial feature merge. Includes settings range corrections, data integrity fixes, and a default value change.
 
---
 
### 1. Wrong Slider Ranges (Per-Model Settings)
 
**Problem:** The initial migration seeded settings per-provider (bulk `UPDATE WHERE provider = X`), so every ElevenLabs model got the same `speed: 0.25–4.0` range. The ElevenLabs API rejects speed outside `[0.7, 1.2]` for all non-v3 models — calls with speed > 1.2 crashed with a TTS error. The same pattern of per-provider-instead-of-per-model caused wrong ranges across Cartesia, Sarvam, and OpenAI as well.
 
**Root cause:** Architecture was per-provider not per-model. Same provider, different models, different valid ranges — this wasn't representable.
 
**Audit findings across all providers:**
 
| Provider | Parameter | Issue |
|----------|-----------|-------|
| ElevenLabs (non-v3) | `speed` | Was `0.25–4.0`; correct range is `0.7–1.2`. Only `eleven_v3` family supports the wider range |
| ElevenLabs | `style` | Defined in migration but not exposed in `getTTSParams` — orphaned, never shown |
| ElevenLabs | `stability` default | Was `0.5`; should be `0.75` to match the code fallback in `models_util.py:507` |
| ElevenLabs | `temperature` | Legacy alias for `style` — drop it, expose `style` directly |
| Cartesia | `volume` | `sonic-2` had no volume in migration; prod DB had it and it was working — restored |
| Sarvam | `pitch` | Was `-20..20` (Sarvam-native scale); UI should show `-1..1` (server maps to native) |
| Sarvam | `temperature` | Existed on `bulbul:v2`; only valid on v3/v3-beta — removed from v2 |
| Google TTS | `speed` | Was `0.25–4.0`; UI should show `0.5–2.0` (server maps up) |
| OpenAI | `reasoning_effort` | Showed on `gpt-4o` / `gpt-4.1` — OpenAI API rejects it on non-gpt-5/o1/o3 models |
| Gemini | `reasoning_effort` | Kept — maps to `thinking_budget` via LiveKit plugin |
 
**Fix:** Switched from per-provider to per-model settings. New migration `c4d5e6f7a8b9` replaces the bulk provider `UPDATE` with per-`model_id` `UPDATE`s. `seed_db.py` updated to match, eliminating the step/default inconsistency from gap #1.
 
**Files changed:**
- `alembic/versions/c4d5e6f7a8b9_per_model_settings_for_llm_tts.py` (new, consolidated migration — see below)
- `scripts/seed_db.py` — `LLM_SETTINGS_BY_MODEL_ID` and `TTS_SETTINGS_BY_MODEL_ID` dicts replace the old per-provider maps; stability default corrected to `0.75`
**UI fix:** `model-capabilities.ts` — `getTTSParams` now returns `style` and `loudness`. `FallbacksContent.tsx` — Style and Loudness sliders added; `temperature` removed from TTS fallback defaults; both sliders reset to `settings.default` on model change.
 
---
 
### 2. ElevenLabs Speed Safety Clamp (Backend)
 
**Problem:** Even after the catalog fix, agents that had already saved an out-of-range speed (e.g. `1.5`) would still crash on the next call until they updated their config.
 
**Fix:** Added a runtime clamp in `models_util.py` `get_tts_model()`. Before passing speed to the ElevenLabs plugin:
- Detects model family: `"v3" in model_id` → allowed range `[0.25, 4.0]`; all others → `[0.7, 1.2]`
- Clamps `speed` to the allowed range and logs a warning with the original and clamped values
- Acts as permanent defense-in-depth so future out-of-range values never crash the pipeline
---
 
### 3. STT Cross-Provider Model Bleed (`_resolve_stt_model`)
 
**Incident:** Agent `8cfca55f` (`Inprime_PostDD_collection_More_Then7Days__Kannada_Advanced_Gemini`) had `provider=sarvam` with `model=nova-3` (a Deepgram model). Sarvam's API returned HTTP 400 on every recognize call → STT stream-adapter gave up after 3 retries → session closed → 54-second dead call.
 
**Fix:** New `_resolve_stt_model(provider, requested)` helper in `models_util.py`:
- Maintains a `_STT_VALID_MODELS` whitelist per provider
- If `requested` is not in the provider's whitelist, logs a warning and returns the provider's safe default
- All four STT branches (deepgram, sarvam, google, elevenlabs) route through it
**Whitelist:**
 
| Provider | Valid models |
|----------|-------------|
| deepgram | `nova-2`, `nova-3`, `flux-general-en`, `flux-general-multi` |
| sarvam | `saarika:v1/v2/v2.5/flash`, `saaras:v2.5/v3/v3-realtime` |
| google | `long`, `telephony`, `chirp`, `chirp_2` |
| elevenlabs | `scribe_v1`, `scribe_v2`, `scribe_v2_realtime` |
 
---
 
### 4. `ignore_interruption_turns` Default Changed: `0` → `1`
 
**Change:** The number of initial agent turns during which user interruptions are suppressed now defaults to `1` instead of `0`.
 
**Rationale:** Agents were being interrupted on their very first utterance (the welcome message). Defaulting to `1` gives the agent one clean turn before interruptions are allowed.
 
**Files changed (both repos):**
 
| File | Change |
|------|--------|
| `app/models/agents/agent/agent.py:709` | `Optional[int] = 1` |
| `app/models/agents/agent/agent.py:792` | Schema example updated |
| `app/modules/pipeline/schema/pipeline_schema.py:67` | Runtime default `= 1` |
| `app/modules/pipeline/util/agent_util.py:531` | `getattr(..., 1)` + explicit `None` → `1` guard |
| `app/modules/agents/agent/service/agent_service.py:899` | Service-layer fallback updated |
| `AdvanceSettingsSection.tsx:351` | `?? 1` |
| `AdvanceConfigModal.tsx:167` | `?? 1` |
| `advanceSettings.shared.ts:158` | `?? 1` |
 
> **Note:** Existing agents with an explicitly saved `ignore_interruption_turns: 0` keep their value. Only new agents and configs missing the field get the new default.
 
---
 
### 5. Consolidated Migration `c4d5e6f7a8b9` + Production Data Fixes
 
After discovering the per-provider settings issue, three separate migrations were initially created (`c4d5e6f7a8b9`, `d5e6f7a8b9c0`, `e6f7a8b9c0d1`). These were consolidated into a single migration to reduce complexity.
 
**What the single migration does (in one pass):**
 
**Part 1 — Catalog:** Per-model `settings` JSONB for all `llm_models` and `tts_models` rows.
 
**Part 2 — Avatar backfill:**
- `tts_models.avatar_url` populated by provider (from the `providers` table avatar URLs)
- `stt_models.avatar_url` populated by language prefix (`en` → English.png, `hi` → Hindi.png, `kn` → Kannada.png, `ta` → Tamil.png, `te` → telugu.png)
**Part 3 — Agent data fixes (discovered via diagnostic queries on prod DB):**
 
A full audit of the `agents` table revealed several classes of config bleed beyond the Sarvam/nova-3 incident. Queries run, findings, and fixes:
 
| Class | Count | Pattern | Fix |
|-------|-------|---------|-----|
| ElevenLabs speed out-of-range | 154 agents | Primary `tts_config.speed` outside `[0.7, 1.2]` | Clamped to model's allowed range |
| STT cross-provider bleed | 4 entries | `provider=sarvam, model=nova-3` and similar in primary + fallbacks | `model` reset to `"default"` |
| LLM provider mismatch | 49 agents | `provider=azure, model_id=gpt-4o-mini` — OpenAI's catalog row resolves at runtime; agents were silently running on OpenAI | `provider` corrected to `openai` (no behavior change, honest config) |
| TTS orphan `model_id` | 35 agents | Five patterns: voice-id stored as model_id, provider name as model_id, UUID as model_id, legacy model name | Mapped to correct canonical `model_id` |
 
**TTS orphan mapping:**
 
| Provider | Bad `model_id` | Correct `model_id` | Count |
|----------|---------------|-------------------|-------|
| azure | `hi-IN-AartiNeural` | `default_azure` | 15 |
| cartesia | `cartesia` | `sonic-2` | 11 |
| sarvam | `sarvam-bulbul-v2` | `bulbul:v2` | 7 |
| elevenlabs | `d428ef60-be0d-4111-8781-9d76b8d43dff` (UUID) | `eleven_multilingual_v2` | 1 |
| google | `google_neural2_a_female` | `google` | 1 |
 
**Deployment on stage:**
 
The stage DB had the schema from `b3c4d5e6f7a8` applied manually (out-of-band) without recording it in `alembic_version`. To reconcile:
1. `INSERT INTO alembic_version VALUES ('b3c4d5e6f7a8')` — stamped existing state
2. Changed `down_revision` of `c4d5e6f7a8b9` to `('a1c4f92b7e31', 'b3c4d5e6f7a8')` — made it a merge revision
3. `alembic upgrade head` — ran cleanly, collapsed two heads into one
**Dry-run verified before committing.** All row counts matched expectations.
 
**Commit messages:**
- `revrag-core`: `fix(models): per-model LLM/TTS settings, avatar backfill, clamp legacy agent configs (migration c4d5e6f7a8b9)`
- `revrag-ui`: `feat(fallbacks): expose TTS style/loudness sliders and bump ignore_interruption_turns default to 1`
