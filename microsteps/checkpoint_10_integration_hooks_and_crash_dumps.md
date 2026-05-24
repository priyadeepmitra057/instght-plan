# CHECKPOINT 10: Integration Hooks and Crash Dumps

Directly modified:   pipeline.py
Indirectly affected: Pipeline execution, Crash handling
Code blocks used:    CB-P6-01, CB-P6-02, CB-P6-03, CB-P6-04, CB-P6-05, CB-P6-06
Risk:                CRITICAL
Depends on:          CHECKPOINT 09

---
EXECUTOR DIRECTIVE
You are an executor. Not a decision maker.
Follow each step exactly as written.
Do not infer. Do not improvise. Do not skip.
Do not reformat or alter any code block.
If a step is unclear → STOP and ask.
If a pre-condition fails → STOP and report.
If any validation fails → STOP. Do not continue.
Never modify files not listed in the current step.
Confirm each step complete before moving to next.
Treat every silent success as a potential silent failure until validation proves otherwise.
Never self-fix a failed test. HALT and wait for instruction.
---

CONTEXT
Wiring the passion engine into the main pipeline flows (`run_pipeline`, `run_inference`) and implementing enhanced crash dump support.

PRE-CONDITIONS
[ ] Checkpoint 09 passed.

STEPS

  STEP [10.1]
  File:           pipeline.py
  Action:         MODIFY
  Target:         Top of file
  Source file:    passion_plan_part6.md
  Source section: 18. pipeline.py — Integration Hook (Step 1)
  Block ID:       CB-P6-01
  Flags:          NONE

  Instruction: Replace the vague Before with an exact insertion rule:
  Find this exact line once:
  ```python
logger = logging.getLogger(__name__)
  ```

  Insert immediately after it:

  After:
  ```python
# FIX #1: passion_logger MUST be at module scope before _attach_passion_results.
# Do NOT shadow the existing pipeline-level logger — use a distinct name.
from logger_factory import get_logger
passion_logger = get_logger("passion.engine")
  ```

  Rollback: Remove added lines.

  STEP [10.2]
  File:           pipeline.py
  Action:         MODIFY
  Target:         Above run_pipeline
  Source file:    passion_plan_part6.md
  Source section: 18. pipeline.py — Integration Hook (Step 2)
  Block ID:       CB-P6-02, CB-P6-03
  Flags:          NONE

  Instruction: Add `_write_crash_dumps`, `_resolve_passion_crash_fields`, and `_attach_passion_results` verbatim above `run_pipeline`.

  After:
  ```python
# ── Passion Engine Integration Helpers ────────────────────────────────
# FIX-01: Shared helper for both run_pipeline and run_inference.
# FIX-33: Checks PASSION_ENGINE_ENABLED kill switch at runtime.
# FIX C9/C10: Lazy-import patching to prevent circular dependencies.

def _write_crash_dumps(
    *,
    debits,
    credits,
    crash_dump_dir,
    passion_debits=None,
    passion_insights=(),
    passion_signals=(),
    run_id=None,
) -> None:
    """Safely write crash dump files containing debits, credits, and passion data."""
    import os
    import json
    import numpy as np
    import pandas as pd
    import config
    from logger_factory import get_logger, pipeline_run_id_ctx
    from schema import Col

    logger = get_logger(__name__)

    if not run_id:
        run_id = pipeline_run_id_ctx.get() or "unknown"

    os.makedirs(crash_dump_dir, exist_ok=True)

    if debits is not None and not isinstance(debits, pd.DataFrame):
        logger.warning(
            "Unexpected type for debits: %s",
            type(debits).__name__,
            extra={"event_type": "data_corruption", "stage": "crash_handler"}
        )
    safe_debits = debits.head(1000) if isinstance(debits, pd.DataFrame) else pd.DataFrame()

    if credits is not None and not isinstance(credits, pd.DataFrame):
        logger.warning(
            "Unexpected type for credits: %s",
            type(credits).__name__,
            extra={"event_type": "data_corruption", "stage": "crash_handler"}
        )
    safe_credits = credits.head(1000) if isinstance(credits, pd.DataFrame) else pd.DataFrame()

    # Safely handle new passion fields
    safe_passion_debits = pd.DataFrame()
    passion_summary = {}

    if passion_debits is not None and isinstance(passion_debits, pd.DataFrame):
        safe_passion_debits = passion_debits.head(1000)

    if passion_insights is not None:
        _raw_insights = list(passion_insights)[:100]
        if getattr(config, "ENABLE_PII_DEBUG_LOGS", False):
            passion_summary["insights"] = _raw_insights
        else:
            from log_utils import log_safe_text
            passion_summary["insights"] = [log_safe_text(str(x)) for x in _raw_insights]

    if passion_signals is not None:
        signals_serial = []
        from log_utils import log_safe_merchant
        for sig in list(passion_signals)[:100]:
            masked_merchants = [
                log_safe_merchant(m) if not getattr(config, "ENABLE_PII_DEBUG_LOGS", False) else m
                for m in sig.merchant_list
            ]
            signals_serial.append({
                "category": sig.category,
                "total_spend": sig.total_spend,
                "merchant_count": sig.merchant_count,
                "spend_share": sig.spend_share,
                "trend_direction": sig.trend_direction,
                "is_suppressed": sig.is_suppressed,
                "suppression_reason": sig.suppression_reason,
                "latest_ts": sig.latest_ts,
                "merchant_list": masked_merchants,
            })
        passion_summary["signals"] = signals_serial

    # Atomicity Note: Guaranteed on POSIX systems; best-effort on Windows.
    if not safe_debits.empty:
        tmp_path = os.path.join(crash_dump_dir, f"{run_id}_debits.csv.tmp")
        final_path = os.path.join(crash_dump_dir, f"{run_id}_debits.csv")
        safe_debits.to_csv(tmp_path, index=False)
        os.replace(tmp_path, final_path)

    if not safe_credits.empty:
        tmp_path = os.path.join(crash_dump_dir, f"{run_id}_credits.csv.tmp")
        final_path = os.path.join(crash_dump_dir, f"{run_id}_credits.csv")
        safe_credits.to_csv(tmp_path, index=False)
        os.replace(tmp_path, final_path)

    if not safe_passion_debits.empty:
        tmp_path = os.path.join(crash_dump_dir, f"{run_id}_passion_debits.csv.tmp")
        final_path = os.path.join(crash_dump_dir, f"{run_id}_passion_debits.csv")
        from log_utils import log_safe_merchant
        pii_safe_passion_debits = safe_passion_debits.copy()
        if Col.CLEANED_REMARKS in pii_safe_passion_debits.columns and not getattr(config, "ENABLE_PII_DEBUG_LOGS", False):
            pii_safe_passion_debits[Col.CLEANED_REMARKS] = pii_safe_passion_debits[Col.CLEANED_REMARKS].map(log_safe_merchant)
        pii_safe_passion_debits.to_csv(tmp_path, index=False)
        os.replace(tmp_path, final_path)

    if passion_summary:
        tmp_path = os.path.join(crash_dump_dir, f"{run_id}_passion_summary.json.tmp")
        final_path = os.path.join(crash_dump_dir, f"{run_id}_passion_summary.json")

        def _json_safe(obj):
            import math
            import pandas as _pd2
            if isinstance(obj, np.generic):
                return obj.item()
            if isinstance(obj, float) and (math.isnan(obj) or math.isinf(obj)):
                return None
            try:
                if _pd2.isna(obj):
                    return None
            except (TypeError, ValueError):
                pass
            if isinstance(obj, np.ndarray):
                return obj.tolist()
            return str(obj)

        with open(tmp_path, "w") as f:
            json.dump(passion_summary, f, indent=2, default=_json_safe)
        os.replace(tmp_path, final_path)


def _resolve_passion_crash_fields(result=None, locals_snapshot=None):
    """
    Resolve passion fields for crash dumps without relying on inline locals()
    expressions in the exception handler.
    """
    snap = locals_snapshot or {}

    if "passion_debits" in snap:
        passion_debits = snap.get("passion_debits")
    else:
        passion_debits = getattr(result, "passion_debits", None) if result is not None else None

    if "passion_insights" in snap:
        passion_insights = snap.get("passion_insights") or ()
    else:
        passion_insights = getattr(result, "passion_insights", ()) if result is not None else ()

    if "passion_signals" in snap:
        passion_signals = snap.get("passion_signals") or ()
    else:
        passion_signals = getattr(result, "passion_signals", ()) if result is not None else ()

    return passion_debits, passion_insights, passion_signals


def _attach_passion_results(
    result: "PipelineResult",
    process_fn=None,
    replace_fn=None,
    fields_fn=None,
    rng=None,
    strict_attach: bool = False,
) -> "PipelineResult":
    """
    Attempt to run the passion pipeline and attach results to PipelineResult.

    P2-3: MemoryError and RecursionError always propagate.
    TimeoutError cancels the entire Passion sidecar.
    Core PipelineResult is returned unchanged on any other failure.
    strict_attach=True (set via env INSIGHT_ENGINE_PASSION_STRICT_ATTACH=true in CI)
    makes all exceptions propagate.

    Args:
        result: The existing PipelineResult from the core pipeline.
        debits: The debits DataFrame to analyze (will be deep-copied internally).
        strict_attach: If True, all non-fatal exceptions propagate. Default False.

    Returns:
        A new PipelineResult with passion fields populated, or the original
        result unchanged if the passion engine fails or is disabled.
    """
    def _with_passion_status(res: "PipelineResult", status: str) -> "PipelineResult":
        import dataclasses as _dc
        r_fn = replace_fn or _dc.replace
        current_stats = getattr(res, "stats", {})
        if not isinstance(current_stats, dict):
            current_stats = {}
        new_stats = {**current_stats, "passion_status": status}
        return r_fn(res, stats=new_stats)

    # FIX-33: Runtime kill switch — check on every call so env var changes
    # take effect without restart. Default is now false!
    import os as _os
    enabled = _os.environ.get(
        "INSIGHT_ENGINE_PASSION_ENABLED", "false"
    ).lower() == "true"

    if not enabled:
        passion_logger.info("passion_engine_config", extra={"enabled": "false", "reason": "INSIGHT_ENGINE_PASSION_ENABLED=false"})
        return _with_passion_status(result, "disabled")

    if not strict_attach:
        strict_attach = _os.environ.get(
            "INSIGHT_ENGINE_PASSION_STRICT_ATTACH", "false"
        ).lower() == "true"

    try:
        import dataclasses as _dc
        import random
        import pandas as pd
        from schema import Col

        if process_fn is None:
            from passion_pipeline import process_pipeline as process_fn

        replace_fn = replace_fn or _dc.replace
        fields_fn = fields_fn or _dc.fields
        rng = rng or random.Random(0)

        # FIX 15: Extract debits from result to make contract robust to misuse
        debits = result.debits if hasattr(result, "debits") else pd.DataFrame()

        # P2-3: Preflight checks before calling process_fn
        if not isinstance(debits, pd.DataFrame):
            passion_logger.warning("passion_skip", extra={"reason": "non_dataframe"})
            return _with_passion_status(result, "skipped")
        # FIX-21: Configurable max rows limit to prevent memory blowup (default 100,000 rows)
        max_rows = int(_os.environ.get("INSIGHT_ENGINE_PASSION_MAX_ROWS", "100000"))
        if len(debits) > max_rows:
            passion_logger.warning("passion_skip", extra={"reason": "exceeds_max_rows", "row_count": len(debits), "max_rows": max_rows})
            return _with_passion_status(result, "skipped")
        if debits.empty:
            passion_logger.info("passion_skip", extra={"reason": "empty_debits"})
            return _with_passion_status(result, "skipped")
        if debits.index.duplicated().any():
            passion_logger.warning("passion_skip", extra={"reason": "duplicate_index"})
            return _with_passion_status(result, "skipped")
        # FIX 8: IS_KNOWN_PERSON is required
        required = {Col.DATE, Col.AMOUNT, Col.PREDICTED_CATEGORY, Col.CLEANED_REMARKS, Col.IS_KNOWN_PERSON}
        missing = required - set(debits.columns)
        if missing:
            passion_logger.warning("passion_skip", extra={"reason": "missing_columns", "missing": sorted(missing)})
            return _with_passion_status(result, "skipped")
        if debits.columns.duplicated().any():
            passion_logger.warning("passion_skip", extra={"reason": "duplicate_columns"})
            return _with_passion_status(result, "skipped")

        # FIX C3 / L3: Preflight check — verify PipelineResult has the passion fields and stats
        required_fields = {
            "passion_debits", "passion_insights", "passion_signals", "stats"
        }
        actual_fields = {f.name for f in fields_fn(result)}
        if missing := required_fields - actual_fields:
            passion_logger.warning("passion_fields_missing", extra={"missing_fields": list(missing)})
            # Fix #4a: Return with passion_status="missing_fields" — not bare result.
            # A bare return loses the status audit trail needed for kill-switch monitoring.
            return _with_passion_status(result, "missing_fields")

        # A2 / L2: pass result and rng
        passion_result = process_fn(
            df_raw=debits,
            strict_mode=True,
            rng=rng,
        )

        # Blocker 14: Improve structured logs with required metrics
        passion_logger.info("passion_engine_success", extra={
            "outcome": "success",
            "row_count": len(debits),
            "candidate_count": len(passion_result.candidates),
            "insight_count": len(passion_result.insights),
            "signal_count": len(passion_result.passion_signals),
        })

        # Fix #4b: Single replace_fn call — _with_passion_status already calls replace_fn
        # internally, so wrapping its result in a second replace_fn is a double-replace.
        # Merge stats directly and replace all passion fields in one atomic call.
        new_stats = {**getattr(result, "stats", {}), "passion_status": "success"}
        return replace_fn(
            result,
            stats=new_stats,
            passion_debits=passion_result.debits,
            passion_insights=tuple(passion_result.insights),
            passion_signals=tuple(passion_result.passion_signals),
        )

    except MemoryError:
        raise
    except RecursionError:
        raise
    except SystemExit:
        raise
    except KeyboardInterrupt:
        raise
    except TimeoutError as e:
        passion_logger.warning("passion_engine_timeout", extra={"error_type": "TimeoutError", "stage": "attach_results", "safe_message": "Hard timeout or budget exceeded"})
        if strict_attach:
            raise
        return _with_passion_status(result, "timeout")
    except Exception as e:
        passion_logger.warning(
            "passion_engine_failed",
            extra={
                "error_type": type(e).__name__,
                "stage": "attach_results",
                "row_count": len(debits) if 'debits' in locals() and isinstance(debits, pd.DataFrame) else 0,
                "column_count": len(debits.columns) if 'debits' in locals() and isinstance(debits, pd.DataFrame) else 0,
            },
            exc_info=True,
        )
        if strict_attach:
            raise
        return _with_passion_status(result, "failure")
  ```

  Rollback: Remove added functions.

  STEP [10.3]
  File:           pipeline.py
  Action:         MODIFY
  Target:         run_pipeline
  Source file:    passion_plan_part6.md
  Source section: 18. pipeline.py — Integration Hook (run_pipeline)
  Block ID:       CB-P6-04
  Flags:          NONE

  Note: `INSIGHT_ENGINE_CRASH_TEST` is test-only. It must never be set in production, staging, or deployment workflows.

  Before:
  ```python
        # return result
        return result
    except Exception:
  ```

  Instruction: Apply the passion hook before the final return in `run_pipeline`. Ensure `PipelineResult` is assigned to `result` variable.

  After:
  ```python
    # Phase 7: Passion Engine (optional — errors are swallowed)
    # FIX 15: Extract debits from result inside _attach_passion_results, no raw debits passed
    result = _attach_passion_results(result)

    # FIX 19: Support end-to-end testing of crash dumps with populated passion fields
    import os as _os
    if _os.environ.get("INSIGHT_ENGINE_CRASH_TEST", "false").lower() == "true":
        raise ValueError("Simulated post-passion crash")

    return result
  ```

  Rollback: Revert `run_pipeline` end.

  STEP [10.4]
  File:           pipeline.py
  Action:         MODIFY
  Target:         run_inference
  Source file:    passion_plan_part6.md
  Source section: 18. pipeline.py — Integration Hook (run_inference)
  Block ID:       CB-P6-05
  Flags:          NONE

  Before:
  ```python
        return PipelineResult(
            debits=debits,
            credits=credits,
            personal_debits=personal_debits,
            personal_credits=personal_credits,
            personal_summary=personal_summary
        )
  ```

  Instruction: Replace the exact literal code block above. If the exact Before block is not found exactly once, STOP. Do not infer the edit location. Assign the result to a variable and apply the passion hook.

  After:
  ```python
    result = PipelineResult(
        debits=debits,
        credits=credits,
        personal_debits=personal_debits,
        personal_credits=personal_credits,
        personal_summary=personal_summary,
    )

    # Phase 7: Passion Engine (optional — errors are swallowed)
    # FIX 15: Extract debits from result inside _attach_passion_results, no raw debits passed
    result = _attach_passion_results(result)
    return result
  ```
  Important: If the live run_inference PipelineResult call contains additional keyword arguments, list every one explicitly. No placeholders are allowed.

  Rollback: Revert `run_inference` end.

  STEP [10.5]
  File:           pipeline.py
  Action:         MODIFY
  Target:         run_pipeline crash handler
  Source file:    passion_plan_part6.md
  Source section: 18. pipeline.py — Integration Hook (crash handler)
  Block ID:       CB-P6-06
  Flags:          NONE

  Before:
  ```python
        if config.ENABLE_CRASH_DUMPS:
            try:
                os.makedirs(config.CRASH_DUMP_DIR, exist_ok=True)
                if debits is not None and isinstance(debits, pd.DataFrame) and not debits.empty:
                    debits.head(1000).to_csv(os.path.join(config.CRASH_DUMP_DIR, f"{run_id}_debits.csv"), index=False)
                if credits is not None and isinstance(credits, pd.DataFrame) and not credits.empty:
                    credits.head(1000).to_csv(os.path.join(config.CRASH_DUMP_DIR, f"{run_id}_credits.csv"), index=False)
                logger.info(
                    "Crash state snapshots written.",
                    extra={"event_type": "crash_dump_success", "stage": "crash_handler"}
                )
            except Exception:
                logger.warning(
                    "Failed to write state dump to CSV during crash handling sequence.",
                    extra={"event_type": "crash_dump_failed", "stage": "crash_handler"},
                    exc_info=True
                )
  ```

  Instruction: Replace the exact literal code block above. If the exact Before block is not found exactly once, STOP. Do not infer the edit location. Update the crash handler to use the new `_write_crash_dumps` helper and the `_resolve_passion_crash_fields` helper.

  After:
  ```python
        if config.ENABLE_CRASH_DUMPS:
            try:
                _passion_debits, _passion_insights, _passion_signals = _resolve_passion_crash_fields(
                    result=locals().get("result"),
                    locals_snapshot=dict(locals()),
                )

                _write_crash_dumps(
                    debits=debits,
                    credits=credits,
                    crash_dump_dir=config.CRASH_DUMP_DIR,
                    passion_debits=_passion_debits,
                    passion_insights=_passion_insights,
                    passion_signals=_passion_signals,
                    run_id=run_id,
                )
                logger.info(
                    "Crash state snapshots written.",
                    extra={"event_type": "crash_dump_success", "stage": "crash_handler"}
                )
            except Exception:
                logger.warning(
                    "Failed to write state dump to CSV during crash handling sequence.",
                    extra={"event_type": "crash_dump_failed", "stage": "crash_handler"},
                    exc_info=True
                )
  ```

  Rollback: Revert crash handler logic.

POST-EXECUTION VALIDATION
[ ] `pipeline.py` contains `_attach_passion_results` and `_write_crash_dumps`.
[ ] `run_pipeline` calls `_attach_passion_results` and `_write_crash_dumps`.
[ ] grep -n "# ... existing kwargs ..." pipeline.py returns no matches.
[ ] `run_inference` assigns PipelineResult(...) to result before returning.
[ ] `run_inference` calls `_attach_passion_results(result)`.
[ ] python3 -m py_compile pipeline.py succeeds.
[ ] python3 -c "import pipeline; assert hasattr(pipeline, '_attach_passion_results'); assert hasattr(pipeline, '_write_crash_dumps'); assert hasattr(pipeline, '_resolve_passion_crash_fields')"

GO / NO-GO
All checks pass → proceed to CHECKPOINT [11]
