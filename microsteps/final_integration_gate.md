# Phase 4: Final Integration Gate — Passion Detection Engine

**Full system integration tests**
- [ ] `pytest tests/test_passion_engine.py` passes all 174 tests.
- [ ] `pytest tests/test_phase1.py tests/test_phase2.py tests/test_phase3.py` all pass (regressions).
- [ ] `python3 demo.py` runs without error (if applicable).
- [ ] All ~30 files compile/load without error: `python3 -m compileall .`
- [ ] No new console errors, warnings, or unhandled exceptions during standard pipeline run.

**Risk flag audit**
- [ ] `[SHARED STATE RISK]` (PipelineResult) → Defensive copies verified in `__post_init__`.
- [ ] `[INTERFACE BREAK RISK]` (TIP_CORPUS, PipelineResult) → All consumers updated.
- [ ] `[SECURITY SENSITIVE]` (HMAC, Secrets) → `INSIGHT_ENGINE_SECRET` requirement enforced at startup.
- [ ] `[CROSS FILE RIPPLE]` (schema.Col) → Ripple files consistent.
- [ ] `[ASYNC RISK]` (N/A) → Thread-safe initialization verified in tests.

**Plan vs reality reconciliation**
- [ ] `[PLAN CONFLICT #1]` → `test_passion_engine.py` Authoritative version deployed.
- [ ] `[PLAN GAP #1]` → `PipelineResult` constructor calls audited and keyword-only.
- [ ] Every block in Code Block Registry matches source planning file verbatim.

**Code fidelity final audit**
- [ ] Diff all introduced code against Code Block Registry.
- [ ] Every block matches source planning file verbatim.

**Deployment readiness**
- [ ] `INSIGHT_ENGINE_PASSION_ENABLED` kill switch verified (defaults to `false`).
- [ ] `INSIGHT_ENGINE_SECRET` min 32-byte requirement verified.
- [ ] All new code paths have structured log coverage.
- [ ] All new async paths (init) are observable and handle timeouts.
- [ ] Docs/comments updated (PipelineResult contract D4).
