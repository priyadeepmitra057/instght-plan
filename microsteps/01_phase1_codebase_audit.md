# Phase 1: Codebase Audit — Passion Detection Engine Integration

## 1.1 Dependency Map

| File | Role | Directly Touched? | Indirectly Affected? | Risk Level |
|------|------|-------------------|----------------------|------------|
| `schema.py` | Column Registry | YES | YES | HIGH (Schema change) |
| `config.py` | Configuration | YES | YES | HIGH (Schema change) |
| `contracts.py` | NEW - Facade | YES | YES | MEDIUM |
| `bootstrap.py` | NEW - Startup Checks | YES | YES | MEDIUM |
| `log_utils.py` | NEW - Logging Utilities | YES | YES | MEDIUM |
| `hash_utils.py` | Hashing (Deprecated) | YES | YES | MEDIUM |
| `recurring_detector.py` | Feature Logic | YES | NO | LOW |
| `pipeline.py` | Orchestration | YES | YES | CRITICAL |
| `insight_generator.py` | Insight Generation | YES | YES | HIGH |
| `banned_content.py` | NEW - Filter | YES | NO | LOW |
| `config_passion.py` | NEW - Config | YES | NO | LOW |
| `pipeline_result.py` | NEW - Container | YES | NO | LOW |
| `passion_models.py` | NEW - Models | YES | NO | LOW |
| `passion_utils.py` | NEW - Utils | YES | NO | LOW |
| `candidate.py` | NEW - Container | YES | NO | LOW |
| `marketplace_subcategory.py`| NEW - Logic | YES | NO | LOW |
| `passion_detector.py` | NEW - Detection | YES | NO | MEDIUM |
| `passion_insight_generator.py`| NEW - Generation | YES | NO | MEDIUM |
| `passion_pipeline.py` | NEW - Pipeline | YES | NO | MEDIUM |
| `tests/conftest.py` | Test Setup | YES | YES | MEDIUM |
| `tests/test_logging_safety.py` | Tests | YES | NO | LOW |
| `tests/test_passion_engine.py` | NEW - Tests | YES | NO | LOW |

## 1.2 Validate Plan Assumptions
1. **Assumption**: `PipelineResult` calls are mostly keyword-based.
   - **Reality**: Audit shows 4/4 calls in `pipeline.py` and `tests/test_phase3.py` are keyword-based. Conversion is safe.
2. **Assumption**: `TIP_CORPUS` can be migrated to a nested dict.
   - **Reality**: `config.py` currently uses a flat dict for values. Migration is necessary.
3. **Assumption**: `stable_hash` is only used for PII masking.
   - **Reality**: Used in `recurring_detector.py` for merchant references in logs. Replacement with `log_safe_merchant` is correct.
4. **Assumption**: `Col` constants are all lowercase.
   - **Reality**: Confirmed in `schema.py`.

## 1.3 Silent Failure Zones
- **Timeout guards**: `_StepBudgetGuard` is cooperative and cannot interrupt blocking calls (regex, pandas).
- **Mutable State**: `PipelineResult.stats` dict mutation — fixed by `dataclasses.replace` mandate.
- **Import Order**: `contracts.py` must be imported after `config.py` but before consumers.
- **PII Leakage**: `fallback_insight` in `passion_insight_generator.py` could leak merchant names if not handled (Fixed by plan).

## 1.4 Blast Radius Report
```
Directly modified:    schema.py, config.py, recurring_detector.py, pipeline.py, insight_generator.py, hash_utils.py, tests/test_logging_safety.py, tests/conftest.py
Indirectly affected:  All modules importing schema.Col or PipelineResult
External systems:     None (Local logic)
Risk per area:        Pipeline Integration (CRITICAL), Schema Migration (HIGH)
```
Blast radius is within plan scope. The plan explicitly covers all integration points and provides a comprehensive migration path.
