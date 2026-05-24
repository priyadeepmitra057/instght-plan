# Phase 0: Plan Synthesis — Passion Detection Engine Integration

## 0.1 Master Intent
1. Integrate a new Passion Detection Engine sidecar into the existing insight pipeline to identify deep lifestyle engagement.
2. Update schema constants to include inferred subcategory and confidence fields for enhanced transaction classification.
3. Migrate `TIP_CORPUS` and `INSIGHT_TEMPLATES` to a structured, immutable schema in `contracts.py` with strict import-time validation.
4. Replace insecure `stable_hash` with keyed HMAC-based PII masking (`log_safe_merchant`) across the codebase.
5. Implement `PassionSignal` and `PassionResult` immutable models to carry passion-specific data through the pipeline.
6. Add specialized merchant resolution and subcategory enrichment for large marketplaces like Amazon and Flipkart.
7. Orchestrate the passion pipeline with timing budget guards (`_StepBudgetGuard`) and cooperative hard timeouts.
8. Wire the passion engine into `run_pipeline` and `run_inference` with a runtime kill switch and crash dump support.
9. Deploy a 174-test regression and integration suite to ensure system stability and contract compliance.
10. Enforce defensive-copy integrity for all DataFrames to prevent unintended mutation across pipeline stages.

## 0.2 Detect Plan Conflicts
[PLAN CONFLICT #1]
Files involved: `passion_plan_part1.md`, `passion_plan_part7.md`, `passion_plan_part8.md`
Conflict: `tests/test_passion_engine.py` is mentioned as receiving B5 tests in Part 1, but Part 7 explicitly states "The existing test_passion_engine.py file is completely replaced by this new suite" and provides a new implementation spanning Part 7 and Part 8.
Decision: The version in Part 7/8 is authoritative and supersedes any earlier mentions of tests for this file. Part 1's B5 tests are integrated into the final authoritative suite in Part 8.
STATUS: RESOLVED (Authoritative version in P7/P8 to be used).

## 0.3 Detect Plan Gaps
[PLAN GAP #1]
Missing: Explicit conversion of all `PipelineResult` calls to keyword-only.
Implied by: `passion_plan_part1.md` Section 0 (Item 2) and `passion_plan_part2.md` Section 8.
Has code: YES (Audit table provided in Part 2).
STATUS: RESOLVED (Executor will follow Part 2 Audit Table).

## 0.4 Code Block Registry

| Block ID | Planning File | Section | Target File | Target Function |
|----------|---------------|---------|-------------|-----------------|
| CB-P1-01 | Part 1 | 1. schema.py Update | schema.py | Col |
| CB-P1-02 | Part 1 | 2. contracts.py | contracts.py | Module Level |
| CB-P1-03 | Part 1 | 3. bootstrap.py | bootstrap.py | Module Level |
| CB-P1-04 | Part 1 | 4. log_utils.py | log_utils.py | Module Level |
| CB-P1-05 | Part 1 | 5. hash_utils.py Update | hash_utils.py | Module Level |
| CB-P1-06 | Part 1 | 5. hash_utils.py Update | recurring_detector.py | Imports/Logging |
| CB-P1-07 | Part 1 | 5. hash_utils.py Update | recurring_detector.py | stable_hash replacement |
| CB-P1-08 | Part 1 | 5. hash_utils.py Update | tests/test_logging_safety.py | Imports/Tests |
| CB-P1-09 | Part 1 | 6. banned_content.py | banned_content.py | Module Level |
| CB-P1-10 | Part 1 | 7. config_passion.py | config_passion.py | Module Level |
| CB-P1-11 | Part 1 | 7a. config.py — TIP_CORPUS Schema Migration | config.py | TIP_CORPUS / SPECIFIC_MERCHANT_ALIASES |
| CB-P1-12 | Part 1 | 7b. insight_generator.py — TIP_CORPUS Import Migration | insight_generator.py | Imports |
| CB-P1-13 | Part 1 | 7b. insight_generator.py — TIP_CORPUS Import Migration | insight_generator.py | TIP_CORPUS access |
| CB-P1-14 | Part 1 | B5 — TIP_CORPUS Generic Prefix Tests | tests/test_passion_engine.py | TestTipCorpusGenericPrefix (Integrated in P8) |
| CB-P2-01 | Part 2 | 9. pipeline.py PipelineResult Extension | pipeline.py | PipelineResult |
| CB-P2-02 | Part 2 | 10. pipeline_result.py | pipeline_result.py | PassionResult |
| CB-P3-01 | Part 3 | 11. passion_models.py | passion_models.py | PassionSignal |
| CB-P3-02 | Part 3 | 12. passion_utils.py | passion_utils.py | Module Level |
| CB-P3-03 | Part 3 | 13. candidate.py | candidate.py | Candidate |
| CB-P3-04 | Part 3 | 14. marketplace_subcategory.py | marketplace_subcategory.py | Module Level |
| CB-P4-01 | Part 4 | 15. passion_detector.py | passion_detector.py | Module Level |
| CB-P4-02 | Part 4 | 16. passion_insight_generator.py | passion_insight_generator.py | Module Level |
| CB-P5-01 | Part 5 | 17. passion_pipeline.py | passion_pipeline.py | Module Level |
| CB-P5-02 | Part 5 | pipeline.py — PipelineResult Contract (D4) | pipeline.py | PipelineResult Docstring |
| CB-P6-01 | Part 6 | 18. pipeline.py — Integration Hook (Step 1) | pipeline.py | Imports |
| CB-P6-02 | Part 6 | 18. pipeline.py — Integration Hook (Step 2) | pipeline.py | _write_crash_dumps |
| CB-P6-03 | Part 6 | 18. pipeline.py — Integration Hook (Step 2) | pipeline.py | _attach_passion_results |
| CB-P6-04 | Part 6 | 18. pipeline.py — Integration Hook (run_pipeline) | pipeline.py | run_pipeline hook |
| CB-P6-05 | Part 6 | 18. pipeline.py — Integration Hook (run_inference) | pipeline.py | run_inference hook |
| CB-P6-06 | Part 6 | 18. pipeline.py — Integration Hook (crash handler) | pipeline.py | run_pipeline crash handler |
| CB-P7-01 | Part 7 | 19. Infrastructure (deploy.yml) | .github/workflows/deploy.yml | Secret validation |
| CB-P7-02 | Part 7 | 19. Infrastructure (pyproject.toml) | pyproject.toml | Pytest config |
| CB-P7-03 | Part 7 | 19. Infrastructure (requirements.txt) | requirements.txt | Dependencies |
| CB-P7-04 | Part 7 | 19. Infrastructure (conftest.py) | tests/conftest.py | Fixtures |
| CB-P7-05 | Part 7 | 20. tests/test_passion_engine.py (first half) | tests/test_passion_engine.py | Module Level / First Half |
| CB-P8-01 | Part 8 | 21. tests/test_passion_engine.py (second half) | tests/test_passion_engine.py | Second Half |

## 0.5 Unified Plan Summary
1. Update `schema.py` with new passion-related column constants (Part 1).
2. Audit and convert all `PipelineResult` constructor calls to keyword-only (Part 1/2).
3. Update `config.py` with new `TIP_CORPUS` schema and mandatory marketplace aliases (Part 1).
4. Update `contracts.py` to enforce `TIP_CORPUS` and `INSIGHT_TEMPLATES` immutability and schema (Part 1).
5. Update `bootstrap.py` with comprehensive startup checks, template validation, and dry-rendering (Part 1).
6. Implement HMAC-based PII masking in `log_utils.py` and deprecate `stable_hash` in `hash_utils.py` (Part 1).
7. Migrate `recurring_detector.py` and `tests/test_logging_safety.py` to use `log_safe_merchant` (Part 1).
8. Create `banned_content.py` for text normalization and obfuscated word detection (Part 1).
9. Create `config_passion.py` for passion-specific thresholds and alias validation (Part 1).
10. Update `insight_generator.py` to import `TIP_CORPUS` from `contracts` and handle the new schema (Part 1).
11. Extend `PipelineResult` in `pipeline.py` with passion fields and defensive copy logic (Part 2/5).
12. Create `pipeline_result.py` defining the `PassionResult` immutable container (Part 2).
13. Create `passion_models.py` defining the `PassionSignal` dataclass (Part 3).
14. Create `passion_utils.py` with bool coercion, mask sanitization, and numeric parsing (Part 3).
15. Extend `Candidate` in `candidate.py` with passion signal support and deterministic sorting (Part 3).
16. Create `marketplace_subcategory.py` for vectorized merchant resolution and enrichment (Part 3).
17. Create `passion_detector.py` with distress gate and anomaly suppression logic (Part 4).
18. Create `passion_insight_generator.py` for lifestyle opportunity rendering and banned content filtering (Part 4).
19. Create `passion_pipeline.py` to orchestrate the sub-stages with budget guards (Part 5).
20. Wire the passion engine into `pipeline.py` hooks and crash handlers (Part 6).
21. Update infrastructure files (`deploy.yml`, `pyproject.toml`, `requirements.txt`) and `tests/conftest.py` (Part 7).
22. Deploy the full `tests/test_passion_engine.py` suite (Part 7/8).
