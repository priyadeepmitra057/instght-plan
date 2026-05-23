# Passion Detection Engine — Master Plan (Part 6 of 8)

**pipeline.py integration hook — FIX-01, FIX-33, FIX C9/C10.**

> **FIX-01**: `run_pipeline()` missing the passion hook. Both `run_pipeline` and
> `run_inference` must call `_attach_passion_results`. Extract the integration
> logic into a shared helper.

> **FIX-33**: `PASSION_ENGINE_ENABLED` kill switch — check at runtime so the
> engine can be disabled via environment variable without redeploying.

> **FIX C9/C10**: Construction of PipelineResult and lazy-import patching. Use lazy imports for Passion Engine components to avoid circular dependencies during initialization.

> **E1**: `passion_status` is written into `result.stats` via `dataclasses.replace`,
> NOT by in-place dict mutation. `result.stats["passion_status"] = ...` is forbidden:
> even though PipelineResult is frozen at field level, a mutable nested dict can be
> mutated silently — violating the defensive-copy contract this plan is built on.
> Always produce `new_stats = {**result.stats, "passion_status": status_value}` and
> return `dataclasses.replace(result, stats=new_stats, ...)`.

> **E2 KILL SWITCH ROLLOUT**: `INSIGHT_ENGINE_PASSION_ENABLED` defaults to `false`.
> Rollout sequence: (1) deploy with default false — passion engine is fully inert;
> (2) set `true` in staging, monitor `passion_engine_success` and
> `passion_engine_failed` log keys; (3) enable in production only after staging
> verification. Tests MUST cover both false (engine skipped) and true (engine runs)
> paths. See `test_attach_passion_results_disabled_does_not_call_process` and
> `test_attach_passion_results_real_pipeline_result_enabled` in Part 8.

> **E6 CONTEXT PROPAGATION**: `passion_logger = get_logger("passion.engine")` is
> valid as a logger name. However, the `pipeline_run_id_ctx` context variable set
> by `run_pipeline` / `run_inference` MUST still be active when passion logs are
> emitted. Do NOT create a parallel logging namespace or reset the context variable.
> `_attach_passion_results` is called within the same call stack as `run_pipeline`,
> so the context variable is automatically inherited. Do not wrap the passion call
> in a new thread without forwarding the context variable.

The integration hook is an optional sidecar over `result.debits`, with no ranker dependency.

## 18. `pipeline.py` — Integration Hook

> **FIX #1 — CRITICAL PLACEMENT ORDER**: `passion_logger` is referenced inside
> `_attach_passion_results` on the very first enabled-check path
> (`passion_logger.info("passion_engine_config", ...)`). If it is not defined at module scope
> **before** the function body, Python raises `NameError` on the first call when
> `INSIGHT_ENGINE_PASSION_ENABLED=false`. Apply **Step 1** imports first, then **Step 2** function.

### Step 1: Add to module top of `pipeline.py` (APPLY BEFORE the function definition)

```python
# FIX #1: passion_logger MUST be at module scope before _attach_passion_results.
# Do NOT shadow the existing pipeline-level logger — use a distinct name.
from logger_factory import get_logger
passion_logger = get_logger("passion.engine")
```

### Step 2: Add the following helper functions **above** `run_pipeline` and `run_inference`:


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

### Modifications to `run_pipeline`:

At the **end** of `run_pipeline`, explicitly ensure the final `PipelineResult` is assigned to a local variable named `result`. Do not leave a direct `return PipelineResult(...)`. Then add:

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

Where `result` is the PipelineResult available in `run_pipeline`'s scope.

### Modifications to `run_inference`:

At the **end** of `run_inference`, explicitly ensure the final `PipelineResult` is assigned to a local variable named `result`. Do not leave a direct `return PipelineResult(...)`. Then add:

```python
    # Phase 7: Passion Engine (optional — errors are swallowed)
    # FIX 15: Extract debits from result inside _attach_passion_results, no raw debits passed
    result = _attach_passion_results(result)
    return result
```

### Modifications to the crash handler in `run_pipeline`:

Inside `run_pipeline`, locate the crash handler under `except Exception:` and update the `ENABLE_CRASH_DUMPS` logic to call the helper `_write_crash_dumps` to cleanly output state:

```python
        if config.ENABLE_CRASH_DUMPS:
            try:
                _write_crash_dumps(
                    debits=debits,
                    credits=credits,
                    crash_dump_dir=config.CRASH_DUMP_DIR,
                    passion_debits=locals().get("passion_debits", None) if "passion_debits" in locals() else (getattr(result, "passion_debits", None) if "result" in locals() and result is not None else None),
                    passion_insights=locals().get("passion_insights", ()) if "passion_insights" in locals() else (getattr(result, "passion_insights", ()) if "result" in locals() and result is not None else ()),
                    passion_signals=locals().get("passion_signals", ()) if "passion_signals" in locals() else (getattr(result, "passion_signals", ()) if "result" in locals() and result is not None else ()),
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

### Required import additions at top of `pipeline.py`:

> **FIX #1 UPDATE**: The required imports (`passion_logger = get_logger("passion.engine")`) are
> now specified under **Step 1** at the top of this section. They must appear in the actual
> source file **before** the `_attach_passion_results` function definition — not as a
> final afterthought. The module-level placement ensures no `NameError` on any code path.

No other changes to existing `pipeline.py` functions outside the three hook sites above.
The integration hook is purely additive — no existing function signatures are modified.
