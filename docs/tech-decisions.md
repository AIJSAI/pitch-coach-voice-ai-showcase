# Technical Decisions — Pitch Coach AI

This document contains excerpts from the project's Architecture Decision Records (ADRs).

---

## ADR-001: Two-Brain Architecture

**Status**: Accepted  
**Context**: Real-time voice AI coaching requires two fundamentally conflicting capabilities — sub-second conversational responsiveness and deep rubric-based performance grading that takes 5–15 seconds. No single model/pipeline can satisfy both constraints simultaneously.

**Decision**: Separate the system into two independent "brains":
- **Brain 1 (The Actor)**: `gpt-realtime-1.5` via LiveKit WebRTC → audio-to-audio conversational agent with <1,000ms glass-to-glass latency.
- **Brain 2 (The Grader)**: `o4-mini` triggered asynchronously via Azure Blob trigger → deep transcript analysis with Structured Outputs for schema-compliant scoring.
- **Air Gap**: Azure Blob Storage decouples Brain 1 from Brain 2. Transcript JSON is written at session end; Blob trigger fires Brain 2 independently.

**Consequences**:
- Latency budget met: voice loop never blocks on grading compute.
- Independent scaling: voice sessions and grading pipelines scale independently.
- Increased complexity: two deployment targets, event-driven coordination, eventual consistency between session state and grading results.

---

## ADR-005: Universal Envelope Multi-Rubric System

**Status**: Accepted  
**Context**: The platform supports two distinct scoring rubrics — ISR (In-Session Rating, 90 points) and OSR (Outside Sales Representative, 100 points) — each with different categories, scoring ranges, and weighting. A rigid schema per rubric type would create duplicate pipelines.

**Decision**: Implement a Universal Envelope schema using a `rubric_payload` VARIANT column in Snowflake:
- Common fields: `session_id`, `rubric_type`, `total_score`, `max_possible`, `timestamp`
- Rubric-specific scoring data stored as JSON in `rubric_payload`
- Grading prompts dynamically assembled based on `rubric_type`
- Validation gates check category count, score ranges, and mandatory fields per rubric type

**Consequences**:
- Single grading pipeline handles any rubric type.
- New rubrics require only prompt template + validation rules, not schema changes.
- Analytics queries require rubric-aware extraction (Snowflake JSON path syntax).

---

## ADR-008: Rubric Alignment with Official Sources

**Status**: Accepted  
**Context**: Calibration testing revealed significant drift between the implemented rubrics and official sources: 50% of OSR criteria were missing (10 implemented vs. 20 official), and ISR empathy weighting was inflated by 100% (22 pts vs. 11 pts official).

**Decision**:
- Conduct full rubric audit against official source documents.
- Expand OSR from 2-criterion to 4-criterion per category (20 criteria total across 5 categories).
- Correct ISR empathy weighting from 22 to 11 points.
- Create fair-band calibration transcripts for validation.
- All changes stakeholder-approved before implementation.

**Consequences**:
- Scoring now mirrors official rubric structure exactly.
- Calibration tests use verifiable fair-band transcripts.
- 199+ pytest tests protect against future drift.

---

## ADR-010: ServerVad over SemanticVad

**Status**: Accepted  
**Context**: LiveKit Agents SDK v1.4 introduced `SemanticVad` (model-powered speech endpoint detection) alongside the existing `ServerVad` (WebRTC-layer VAD). Initial migration to SemanticVad caused observably higher first-response latency — the model occasionally "waited" for more speech before returning endpointing decisions.

**Decision**: Revert to `ServerVad` for all voice sessions. ServerVad provides deterministic, millisecond-level turn boundary detection through configurable `vad_threshold` and `vad_silence_duration_ms` parameters.

**Consequences**:
- Consistent turn detection latency (no model inference in the VAD path).
- Per-persona VAD sensitivity becomes a simple parameter mapping (see ADR-012).
- Loses semantic understanding of speech boundaries (e.g., trailing "um..." might trigger premature endpointing), mitigated by per-persona silence duration tuning.

---

## ADR-012: Interrupt Sensitivity Mapping

**Status**: Accepted (5 addenda over 4 weeks)  
**Context**: Ghost triggers (false positive speech detections) were observed in production demos — 34 ghost triggers in a 7.7-minute session. Different personas require different turn-taking behavior: "Friendly Buyer" should be interruptible, "Skeptical Buyer" should hold the floor longer.

**Decision**: Introduce a single `interrupt_sensitivity` enum per persona that maps to two underlying VAD parameters:
- `interrupt_sensitivity` → `vad_threshold` (0.0–1.0) + `vad_silence_duration_ms` (200–800ms)
- Tiered presets: `low`, `medium`, `high`
- ISR and OSR personas get differentiated profiles (ISR buyers more interruptible than OSR gatekeepers)
- Ghost trigger analytics stream to Snowflake for data-driven threshold calibration

**Evolution (5 addenda)**:
1. Initial mapping table
2. Re-differentiated ISR vs. OSR profiles after accidental flattening
3. Reduced silence duration minimums for more natural conversation flow
4. Added ghost trigger analytics to Snowpipe pipeline
5. Final calibrated values after production testing

**Consequences**:
- Persona authors set one parameter (`interrupt_sensitivity`) instead of coordinating two VAD knobs.
- Data-driven calibration possible via Snowflake analytics.
- 34-ghost-trigger problem reduced to near-zero in production testing.

---

## ADR-020: Half-Cascade Architecture (DragonHD TTS for Voice Gender Drift Fix)

**Status**: Accepted
**Supersedes**: Portions of ADR-001 — audio synthesis ownership shifts from `gpt-realtime` to Azure Speech.

**Context**: Voice gender drift was reproduced repeatedly through Phase 5–9.3 sessions and confirmed via telemetry (`voice.item.snapshot` per-conversation-item `voice_id`) — the configured voice stayed pinned to its persona's setting throughout each session, but listeners reported the assistant's voice wobbling between gender presentations mid-conversation. The drift originated downstream of every config knob the application controlled, inside `gpt-realtime`'s audio synthesis layer. No prompt-side fix could address it because the model itself decides the final waveform.

LiveKit documents a half-cascade pattern for exactly this class of problem: the realtime model emits text only (`modalities=["text"]`), and a separate TTS engine synthesizes the audio. With a deterministic voice ID, voice gender cannot drift by construction — identical text in the same voice produces identical audio bytes.

**Decision**: Switch Brain 1 to half-cascade for all personas.

1. `RealtimeModel.with_azure(modalities=["text"], ...)` — `gpt-realtime` emits text only.
2. `livekit-plugins-azure` `azure.TTS` wired into `AgentSession(tts=...)` with per-persona DragonHD voice IDs (e.g., `en-US-Ava:DragonHDLatestNeural`, `en-US-Andrew:DragonHDLatestNeural`).
3. Managed-identity auth via a new `utils/speech_auth.py` module that returns the `aad#{resourceId}#{aadToken}` wrapped token with a 5-min expiry-refresh cache.
4. Volume-boost code retired (~412 lines including tests) — Azure TTS emits at standard amplitude.
5. Bicep provisions a SystemAssigned-MI Speech account with the custom subdomain required for Entra auth; the agent ACA is granted `Cognitive Services Speech User` scoped to that Speech resource.
6. Telemetry: `TTSMetrics.ttfb` dispatch feeds first-byte latency to KQL Q8 (p50/p95); Q9 enforces the `dcount(azure_tts_voice) == 1` per-session invariant as a regression guard.

**Consequences**:
- **(+)** Deterministic per-persona voice — voice gender cannot drift; pilot launch criterion #1 cleared.
- **(+)** 600+ Azure voices available; swapping voices or adopting a future Custom Neural Voice is a single config field.
- **(+)** ~98% per-session cost reduction versus full audio-to-audio (Azure TTS at $15/1M chars vs. `gpt-realtime` audio output at $0.20/1K tokens).
- **(+)** LLM-quality and TTS-quality decoupled — model upgrades no longer regress voice; voice upgrades no longer regress turn handling.
- **(+)** No plaintext Speech key — managed-identity auth via custom-subdomain Entra token wrapping; rotation is automatic at the Azure side.
- **(−)** +~150–300ms TTS hop. Steady-state glass-to-glass remains under 1000ms with ~150–450ms headroom; first-turn variance is tighter due to the plugin's lack of pre-connect pooling. Mitigation: synthesize a brief warmup utterance on session start if Q8 first-turn p95 exceeds 800ms post-pilot.
- **(−)** Adds Azure Speech as a runtime dependency, deployed in the same region and managed-identity scope as the agent.
- **(−)** Non-Omni DragonHD voices (Ava, Emma, Andrew, Adam) support only the `temperature` knob; full `<mstts:express-as>` style support is reserved for Omni voices. Acceptable for pilot.

**Pre-merge measurement (stg)**: Q1/Q9 invariants held across 6/6 persona sessions. Q8 TTS TTFB direct measurement p50 ~237ms, p95 ~316ms — well under the 1000ms HARD gate.
