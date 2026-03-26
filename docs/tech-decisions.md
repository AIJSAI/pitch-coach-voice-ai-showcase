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
