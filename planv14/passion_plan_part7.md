# Passion Detection Engine — Master Plan (Part 7 of 8)

## 19. Infrastructure

### `.github/workflows/deploy.yml`
```yaml
- name: Validate required secrets
  run: |
    if [ -z "$INSIGHT_ENGINE_SECRET" ]; then
      echo "CRITICAL: INSIGHT_ENGINE_SECRET is not set. Aborting."
      exit 1
    fi
    clean_secret=$(printf '%s' "$INSIGHT_ENGINE_SECRET" | tr -d '\r\n')
    byte_len=$(printf '%s' "$clean_secret" | wc -c)
    if [ "$byte_len" -lt 32 ]; then
      echo "CRITICAL: INSIGHT_ENGINE_SECRET too short (min 32 bytes)."
      exit 1
    fi
  env:
    INSIGHT_ENGINE_SECRET: ${{ secrets.INSIGHT_ENGINE_SECRET }}
```

### `pyproject.toml`
```toml
[project]
requires-python = ">=3.11"

[tool.pytest.ini_options]
log_cli_level = "INFO"
filterwarnings = [
    # FIX 20: Do not globally error every UserWarning
    "error::RuntimeWarning",
    "default::DeprecationWarning",
    "default::FutureWarning",
    "default::PendingDeprecationWarning",
    "ignore::RuntimeWarning:numpy",
    "ignore::UserWarning:pandas",
]
```

### `requirements.txt`

```text
numpy>=2.0,<3
pandas>=3.0,<4
scikit-learn>=1.8,<2
lightgbm>=4.6,<5
scipy>=1.17,<2
```

> **FIX H3**: The existing codebase runs `pandas==3.0.1`.
> `pandas>=2.2,<3` rejected the current environment. Pinned forward
> not backward. Upper caps prevent silent breakage on major bumps.

### Fix 20 — Structural Optionality Clarification

> **STRUCTURAL MANDATE**: The passion engine is structurally included in all
> deployments after migration. Partial deploys (e.g. shipping `pipeline.py` changes
> without the companion schema constants) are **unsupported** and will break at
> startup validation.

| Concern | Behaviour |
|---|---|
| `Col.INFERRED_SUBCATEGORY` | Always present in `schema.Col` after migration; never guarded |
| `Col.SUBCATEGORY_CONFIDENCE` | Always present in `schema.Col` after migration; never guarded |
| `INSIGHT_ENGINE_PASSION_ENABLED` | Controls **runtime execution only** — the schema constants exist regardless |
| Partial deploy (schema missing) | `bootstrap._validate_schema_columns` raises `RuntimeError` at startup |


### `tests/conftest.py`

```python
import os
import threading
import pytest

# FIX M1: REMOVED module-level os.environ mutations.
# Previously: os.environ.setdefault("ENV", "test")
# If CI has ENV=production already set, setdefault is a no-op.
# _ensure_initialized then rejects SKIP_STARTUP_CHECKS, breaking all tests.
# Now uses a session-scoped autouse fixture that saves/restores original values.

@pytest.fixture(autouse=True, scope="session")
def _set_test_env():
    """Force test environment for entire session, restore originals on teardown."""
    # C1: Save and forcibly set BOTH env vars. Do not rely on CI or developer
    # shell state: if ENV is already 'production', SKIP_STARTUP_CHECKS would be
    # rejected by _ensure_initialized, breaking the entire test session.
    # Setting ENV=test here overrides any ambient value for the session.
    _keys = ("INSIGHT_ENGINE_SKIP_STARTUP_CHECKS", "ENV")
    original = {k: os.environ.get(k) for k in _keys}
    os.environ["INSIGHT_ENGINE_SKIP_STARTUP_CHECKS"] = "true"
    os.environ["ENV"] = "test"

    # C1: Reset passion_pipeline init state NOW, after env vars are set.
    # Stale _init_complete from a previous session would cause _ensure_initialized
    # to skip the env-var checks entirely and use whatever state was cached.
    import sys
    if "passion_pipeline" in sys.modules:
        import passion_pipeline as _pp
        _pp._init_complete.clear()
        _pp._init_failed.clear()

    yield
    for k, v in original.items():
        if v is None:
            os.environ.pop(k, None)
        else:
            os.environ[k] = v


# P1.8 + P2.8: Reset threading.Event for _init_complete between tests.
# FIX M2: Also reset _init_lock. If a test crashes while holding the lock,
# subsequent tests would deadlock without this reset.
# FIX-10: Also reset _init_failed event.
# FIX 5: Reset _init_in_progress to False to prevent thread state contamination.
@pytest.fixture(autouse=True)
def _reset_pipeline_initialized(monkeypatch):
    import sys
    if "passion_pipeline" in sys.modules:
        monkeypatch.setattr("passion_pipeline._init_complete", threading.Event())
        monkeypatch.setattr("passion_pipeline._init_lock", threading.Lock())
        monkeypatch.setattr("passion_pipeline._init_failed", threading.Event())
        monkeypatch.setattr("passion_pipeline._init_in_progress", False)
    yield


# FIX L5 + FIX-T1-01: _secret_cache reset between tests.
@pytest.fixture(autouse=True)
def _reset_dev_secret():
    from log_utils import _reset_secret_cache
    _reset_secret_cache()
    yield

@pytest.fixture
def real_startup_env(monkeypatch):
    monkeypatch.setenv("ENV", "test")
    monkeypatch.delenv("INSIGHT_ENGINE_SKIP_STARTUP_CHECKS", raising=False)
    from log_utils import _reset_secret_cache
    _reset_secret_cache()
    import passion_pipeline
    monkeypatch.setattr(passion_pipeline, "_init_complete", threading.Event())
    monkeypatch.setattr(passion_pipeline, "_init_failed", threading.Event())
    monkeypatch.setattr(passion_pipeline, "_init_lock", threading.Lock())
    monkeypatch.setattr(passion_pipeline, "_init_in_progress", False)
    yield
```

---

## 20. `tests/test_passion_engine.py` (first half)

> **P1-16 FIX**: The existing `test_passion_engine.py` file is completely replaced by this new suite.
> Old references like `_execute_pipeline_phases`, `_run_ranker_with_timeout`, and `_circuit_breaker` are removed.
> **FIX C3b**: All `TestReadOnlyDataFrame` and `TestReadOnlyAccessor` test
> classes are DELETED — `_ReadOnlyDataFrame` and `_ReadOnlyAccessor` no longer
> exist. `_needs_deepcopy` tests are also deleted.

```python
import pytest
import random
import threading
import time
import pandas as pd
import numpy as np
from decimal import Decimal
from unittest.mock import patch

from passion_pipeline import (
    process_pipeline, _StepBudgetGuard,
    _normalize_ts, safe_assign_new_columns,
)
from passion_utils import (
    to_bool_strict, coerce_bool_column, sanitize_mask,
    safe_last_nonnull, validate_template_values, _safe_isna,
)
from pipeline_result import PassionResult
from log_utils import log_safe_merchant, log_safe_text, verify_merchant_token
from passion_detector import (
    _is_non_declining, detect_passions, _check_anomaly_suppression, _check_distress_gate,
)
from config_passion import validate_merchant_aliases
from passion_insight_generator import generate_passion_insights
from marketplace_subcategory import resolve_merchant_vectorized, enrich_subcategories

# FIX-13: Import from banned_content.py (public module), not passion_insight_generator.
from banned_content import contains_banned_content as _contains_banned_content


def make_pipeline_result(**kwargs) -> "PipelineResult":
    from pipeline import PipelineResult
    import pandas as pd
    import dataclasses
    defaults = {
        "debits": pd.DataFrame(),
        "credits": pd.DataFrame(),
        "insights": [],
        "cat_pipeline": None,
        "spend_pipeline": None,
        "ranker_pipeline": None,
        "global_mean": 0.0,
        "global_std": 0.0,
        "raw_global_mean": 0.0,
        "raw_global_std": 0.0,
        "stats_version": "v1_raw",
        "personal_summary": {},
        "transfer_patterns": [],
        "exclusion_stats": {},
        "kp_config_hash": "",
        "personal_debits": pd.DataFrame(),
        "personal_credits": pd.DataFrame(),
        "stats": {},
        "passion_debits": pd.DataFrame(),
        "passion_insights": (),
        "passion_signals": (),
    }
    valid_fields = {f.name for f in dataclasses.fields(PipelineResult)}
    merged = {**defaults, **kwargs}
    filtered = {k: v for k, v in merged.items() if k in valid_fields}
    return PipelineResult(**filtered)


# ── Bool Coercion ─────────────────────────────────────────────────────────────

class TestBoolCoercion:
    def test_basic_values(self):
        assert to_bool_strict(True) is True
        assert to_bool_strict(False) is False
        assert to_bool_strict(1) is True
        assert to_bool_strict(0) is False
        assert to_bool_strict(np.bool_(True)) is True

    def test_none_and_na(self):
        assert to_bool_strict(None) is False
        assert to_bool_strict(pd.NA) is False
        assert to_bool_strict(np.nan) is False

    def test_string_raises(self):
        with pytest.raises(TypeError):
            to_bool_strict("True")

    # C2: np.bool_ explicit tests — np.bool_ is NOT a subclass of Python bool
    # in NumPy 2.x. The isinstance guard in to_bool_strict must catch these.
    def test_np_bool_true(self):
        assert to_bool_strict(np.bool_(True)) is True

    def test_np_bool_false(self):
        assert to_bool_strict(np.bool_(False)) is False

    # C2: np.int64 coercion — must pass through the int() match/case arm
    # after the isinstance guard (since np.int64 is NOT bool/np.bool_).
    def test_np_int64_one(self):
        assert to_bool_strict(np.int64(1)) is True

    def test_np_int64_zero(self):
        assert to_bool_strict(np.int64(0)) is False

    # C2: np.float64 coercion
    def test_np_float64_one(self):
        assert to_bool_strict(np.float64(1.0)) is True

    def test_np_float64_zero(self):
        assert to_bool_strict(np.float64(0.0)) is False

    # C2: np.nan → _safe_isna returns True → False (not a TypeError)
    def test_np_nan_returns_false(self):
        assert to_bool_strict(np.nan) is False

    # C2: pd.NA → _safe_isna returns True → False (not a TypeError)
    def test_pd_na_returns_false(self):
        assert to_bool_strict(pd.NA) is False

    def test_coerce_bool_column_mixed(self):
        s = pd.Series([1, 0, np.nan, True, False])
        res = coerce_bool_column(s)
        assert res.tolist() == [True, False, False, True, False]

    def test_coerce_bool_column_boolean_dtype(self):
        s = pd.array([True, False, pd.NA], dtype=pd.BooleanDtype())
        result = coerce_bool_column(pd.Series(s))
        assert result.dtype == bool
        assert result.tolist() == [True, False, False]

    def test_coerce_bool_column_int64_nullable(self):
        s = pd.array([1, 0, pd.NA], dtype="Int64")
        result = coerce_bool_column(pd.Series(s))
        assert result.dtype == bool
        assert result.tolist() == [True, False, False]

    def test_coerce_bool_column_object_mixed_without_strings(self):
        s = pd.Series([True, 1, 0, None, pd.NA], dtype=object)
        res = coerce_bool_column(s)
        assert res.tolist() == [True, True, False, False, False]

    def test_coerce_bool_column_object_string_raises(self):
        s = pd.Series([True, "False", 1, 0, None, pd.NA], dtype=object)
        with pytest.raises(TypeError, match="string"):
            coerce_bool_column(s)


# ── sanitize_mask ─────────────────────────────────────────────────────────────

class TestSanitizeMask:
    def test_length_mismatch_raises(self):
        with pytest.raises(ValueError):
            sanitize_mask([True, False], pd.Index([1, 2, 3]), "test")


    def test_valid_list_mask(self):
        idx = pd.Index([1, 2, 3])
        res = sanitize_mask([1, 0, np.nan], idx, "test")
        assert res.tolist() == [True, False, False]

    def test_nonunique_target_index_raises(self):
        with pytest.raises(ValueError, match="Non-unique DataFrame index"):
            sanitize_mask([True, False, True], pd.Index([1, 1, 2]), "test")



    def test_categorical_bool_mask(self):
        idx = pd.Index([0, 1, 2])
        cat_mask = pd.Series(
            pd.Categorical([True, False, True]),
            index=idx,
        )
        result = sanitize_mask(cat_mask, idx, "test")
        assert result.dtype == bool
        assert result.tolist() == [True, False, True]


# ── Timestamp Normalisation ───────────────────────────────────────────────────

class TestNormalizeTs:
    def test_int_handled_as_seconds(self):
        assert _normalize_ts(1673740800) == 1673740800

    def test_nat_returns_nan(self):
        assert np.isnan(_normalize_ts(pd.NaT))

    def test_invalid_returns_nan(self):
        assert np.isnan(_normalize_ts("not-a-date"))

    # P0-1 FIX: MemoryError from pd.Timestamp must propagate — not be swallowed by the
    # except (ValueError, TypeError, OverflowError, ...) clause.
    def test_normalize_ts_memory_error_propagates_from_timestamp(self, monkeypatch):
        def boom(*args, **kwargs):
            raise MemoryError("timestamp oom")
        monkeypatch.setattr("passion_pipeline.pd.Timestamp", boom)
        with pytest.raises(MemoryError, match="timestamp oom"):
            _normalize_ts("2023-01-01")

    def test_bool_returns_nan(self):
        assert np.isnan(_normalize_ts(True))
        assert np.isnan(_normalize_ts(False))

    def test_inf_returns_nan(self):
        assert np.isnan(_normalize_ts(float('inf')))

    def test_negative_millisecond_truncation(self):
        result = _normalize_ts(-1001)
        assert result == -1001

    def test_negative_millisecond_large(self):
        result = _normalize_ts(-100_000_000_001)
        assert result == -100_000_000_001 // 1000

    # P1-3 FIX: Exact _normalize_ts boundary tests
    def test_normalize_ts_exact_boundaries(self):
        # Below ms boundary: treated as seconds (no truncation)
        assert _normalize_ts(99_999_999_999) == 99_999_999_999
        # At ms boundary: truncated to seconds
        assert _normalize_ts(100_000_000_000) == 100_000_000_000 // 1000
        
        # Below us boundary: still truncated as ms
        assert _normalize_ts(99_999_999_999_999) == 99_999_999_999_999 // 1000
        # At us boundary: truncated as us
        assert _normalize_ts(100_000_000_000_000) == 100_000_000_000_000 // 1_000_000
        
        # Below ns boundary: still truncated as us
        assert _normalize_ts(99_999_999_999_999_999) == 99_999_999_999_999_999 // 1_000_000
        # At ns boundary: truncated as ns
        assert _normalize_ts(100_000_000_000_000_000) == 100_000_000_000_000_000 // 1_000_000_000

    def test_milliseconds(self):
        result = _normalize_ts(1672531200000)
        assert result == 1672531200

    def test_nanosecond_timestamp(self):
        ns = 1672531200_000_000_000
        result = _normalize_ts(ns)
        assert result == 1672531200

    def test_yyyymmdd_optin_false_returns_nan(self):
        # FIX 9: compact YYYYMMDD with allow_yyyymmdd=False returns np.nan, not the raw int.
        # _looks_like_compact_yyyymmdd(20230101) is True (10M <= 20.2M <= 100M).
        # The early-return path produces np.nan when opt-in is False.
        result = _normalize_ts(20230101, allow_yyyymmdd=False)
        assert np.isnan(result)

    def test_yyyymmdd_optin_true_converts(self):
        # C5 + C3: Use UTC-aware expected value. _normalize_ts now localizes naive
        # Timestamps to UTC before calling .timestamp(), so the expected value
        # must also be computed with UTC to avoid local-timezone divergence.
        result = _normalize_ts(20230101, allow_yyyymmdd=True)
        # 2023-01-01T00:00:00 UTC = 1672531200 (constant, timezone-independent)
        expected = int(pd.Timestamp("2023-01-01", tz="UTC").timestamp())
        assert result == expected

    def test_invalid_yyyymmdd_returns_nan(self):
        # C5: Renamed from test_invalid_yyyymmdd_falls_through for clarity.
        # Invalid month 13: pd.Timestamp('20231301') raises OutOfBounds/ValueError.
        # The except branch returns np.nan — NOT the raw integer.
        result = _normalize_ts(20231301, allow_yyyymmdd=True)
        assert np.isnan(result)

    # C3: Timezone-invariance test — a naive datetime string must always produce
    # the same Unix epoch second regardless of the host machine's local timezone.
    # Prior to the C3 fix, pd.Timestamp("2023-01-01").timestamp() used local tz,
    # which differed by hours between UTC, IST, PST, etc.
    def test_naive_datetime_string_is_utc_anchored(self):
        result = _normalize_ts("2023-01-01")
        # 2023-01-01 00:00:00 UTC = 1672531200. Must be equal on all machines.
        assert result == 1672531200

    def test_np_int64_handled(self):
        val = np.int64(1673740800)
        assert _normalize_ts(val) == 1673740800

    def test_np_float64_handled(self):
        val = np.float64(1673740800.0)
        assert _normalize_ts(val) == 1673740800

    def test_datetime_time_returns_nan(self):
        import datetime
        assert np.isnan(_normalize_ts(datetime.time(12, 30)))


# ── StepBudgetGuard ───────────────────────────────────────────────────────────

class TestStepBudgetGuard:
    def test_within_budget(self):
        cb = _StepBudgetGuard(budget_ms=500)
        cb.check("early")

    def test_budget_exceeded(self):
        with patch("passion_pipeline.time.monotonic", side_effect=[0.0, 0.011]):
            cb = _StepBudgetGuard(budget_ms=10)
            # FIX-23: Expect 'Budget exceeded' message for soft budget limit violations
            with pytest.raises(TimeoutError, match="Budget exceeded"):
                cb.check("late")

    def test_hard_deadline_exceeded(self):
        with patch("passion_pipeline.time.monotonic", side_effect=[0.0, 0.011]):
            cb = _StepBudgetGuard(budget_ms=10000, hard_timeout_ms=10)
            # FIX-23: Expect 'Hard deadline exceeded' message for hard timeout limits
            with pytest.raises(TimeoutError, match="Hard deadline"):
                cb.check("late")

# ── Config Validation ─────────────────────────────────────────────────────────

class TestConfigPassion:
    def test_validate_aliases_uppercase_key_raises(self):
        # P0-4 FIX: Pass dictionary to validate_merchant_aliases instead of patching nonexistent constant
        with pytest.raises(ValueError, match="Non-lowercase"):
            validate_merchant_aliases({"UPPER": "lower"})


# ── log_utils ─────────────────────────────────────────────────────────────────

class TestLogUtils:
    def test_deterministic(self):
        assert log_safe_merchant("amazon") == log_safe_merchant("amazon")

    def test_prefix_and_length(self):
        token = log_safe_merchant("amazon")
        assert token.startswith("merchant:")
        assert len(token) == 41

    def test_collision_resistance(self):
        assert log_safe_merchant("a") != log_safe_merchant("b")

    def test_empty_returns_empty(self):
        assert log_safe_merchant("") == ""

    def test_nan_returns_empty(self):
        assert log_safe_merchant(np.nan) == ""

    def test_list_returns_empty(self):
        assert log_safe_merchant(["amazon"]) == ""

    def test_bytes_returns_empty(self):
        assert log_safe_merchant(b"amazon") == ""

    def test_verify_merchant_token(self):
        token = log_safe_merchant("amazon")
        assert verify_merchant_token(token, "amazon") is True
        assert verify_merchant_token(token, "flipkart") is False


# ── Banned Content ────────────────────────────────────────────────────────────

class TestBannedContent:
    def test_detects_in_sentence(self):
        assert _contains_banned_content("this is a scam") is True

    def test_clean_text_passes(self):
        assert _contains_banned_content("normal grocery purchase") is False

    def test_banned_in_middle_of_word_does_not_match(self):
        assert _contains_banned_content("escapade") is False

    def test_plural_scams_detected(self):
        assert _contains_banned_content("known scams alert") is True

    def test_cyrillic_homoglyph_blocked(self):
        cyrillic_scam = "\u0455\u0441\u0430m"
        assert _contains_banned_content(cyrillic_scam) is True

    def test_compact_does_not_match_inside_words(self):
        assert _contains_banned_content("scamper") is False
        assert _contains_banned_content("classical casinoid text") is False

    def test_obfuscated_safe_terms_detected(self):
        assert _contains_banned_content("s-c-a-m") is True
        assert _contains_banned_content("s c a m") is True

    # FIX 25: Explicit test proving Greek homoglyphs are NOT covered.
    def test_greek_homoglyph_not_covered_by_design(self):
        # Greek lowercase sigma 'σ' (U+03C3) instead of 's'
        greek_scam = "\u03C3cam"
        # Banned content detector will return False as Greek is not in _CONFUSABLES
        assert _contains_banned_content(greek_scam) is False

    # FIX-10 + Fix 4: Test narrowed separators and boundaries
    def test_narrowed_separators_boundary(self):
        assert _contains_banned_content("s-c-a-m") is True
        assert _contains_banned_content("f r a u d") is True
        assert _contains_banned_content("casino") is True
        assert _contains_banned_content("drugstore") is False
        assert _contains_banned_content("cafe-fraud") is True # Standalone "fraud" inside hyphenated text matches
        assert _contains_banned_content("cafe-fr-aud") is True # Obfuscated fractioned across word boundary
        assert _contains_banned_content("cafe-fr-auditor") is False
        assert _contains_banned_content("spoke person") is False # Does not false positive on "porn"


# ── Distress Gate ─────────────────────────────────────────────────────────────

class TestDistressGate:
    def test_coffee_offered_feedback_do_not_trigger(self):
        df = pd.DataFrame({
            "cleaned_remarks": ["coffee shop", "offered discount", "feedback form",
                                "coffee house", "services offered", "feedback received",
                                "latte art"],
            "amount": [100, 200, 150, 100, 300, 250, 100],
        })
        assert _check_distress_gate(df) is False

    def test_actual_fee_keywords_trigger(self):
        df = pd.DataFrame({
            "cleaned_remarks": ["late payment fee", "penalty charge", "fee applied",
                                "bounce charge", "interest overdue"],
            "amount": [100, 200, 150, 100, 300],
        })
        assert _check_distress_gate(df) is True

    # FIX-8 / P0-5 FIX: Test boundary conditions using ROW RATIO (fee_count/total_rows),
    # not amount share. _check_distress_gate uses strict >  DISTRESS_FEES_THRESHOLD (0.15).
    def test_distress_gate_boundaries(self):
        # 1. 2/14 = 14.28% — below 15% threshold → False
        df_below = pd.DataFrame({
            "cleaned_remarks": ["late payment fee", "penalty charge"] + ["normal shop"] * 12,
        })
        assert _check_distress_gate(df_below) is False

        # 2. 3/20 = exactly 15%; implementation uses strict >, so NOT > 0.15 → False
        df_exact = pd.DataFrame({
            "cleaned_remarks": [
                "late payment fee", "penalty charge", "interest overdue"
            ] + ["normal shop"] * 17,
        })
        assert _check_distress_gate(df_exact) is False

        # 3. 4/20 = 20% — above 15% threshold → True
        df_above = pd.DataFrame({
            "cleaned_remarks": [
                "late payment fee", "penalty charge", "interest overdue", "bounce charge"
            ] + ["normal shop"] * 16,
        })
        assert _check_distress_gate(df_above) is True


# ── Merchant Vectorizer ───────────────────────────────────────────────────────

class TestMerchantVectorizer:
    def test_basic_alias(self):
        s = pd.Series(["AMZN", "amzn"])
        assert resolve_merchant_vectorized(s).tolist() == ["amazon", "amazon"]

    def test_compound_alias_ordering(self):
        s = pd.Series(["amzn mktp us"])
        result = resolve_merchant_vectorized(s)
        assert "amazon" in result.iloc[0]


class TestMarketplaceSubcategory:
    def test_enrich_subcategories_accepts_currency_amount_strings(self):
        df = pd.DataFrame({
            "amount": ["₹1,500", "Rs 500", "INR 1600"],
            "cleaned_remarks": ["amazon", "amazon", "flipkart"],
            "predicted_category": ["shopping", "shopping", "shopping"],
        })
        from marketplace_subcategory import enrich_subcategories
        result = enrich_subcategories(df)
        assert result["inferred_subcategory"].iloc[0] == "electronics"
        assert result["inferred_subcategory"].iloc[1] == "general_purchase"
        assert result["inferred_subcategory"].iloc[2] == "electronics"



# ── _is_non_declining ─────────────────────────────────────────────────────────

class TestIsNonDeclining:
    def test_single_point_returns_false(self):
        df = pd.DataFrame({"date": ["2023-01-01"], "amount": [100]})
        assert _is_non_declining(df) is False

    def test_flat_trend_returns_true(self):
        df = pd.DataFrame({
            "date": ["2023-01-01", "2023-02-01", "2023-03-01", "2023-04-01"],
            "amount": [100, 100, 100, 100],
        })
        assert _is_non_declining(df) is True

    def test_strict_decline_returns_false(self):
        df = pd.DataFrame({
            "date": ["2023-01-01", "2023-02-01", "2023-03-01", "2023-04-01"],
            "amount": [100, 80, 50, 20],
        })
        assert _is_non_declining(df) is False

    def test_all_zero_returns_false(self):
        df = pd.DataFrame({
            "date": ["2023-01-01", "2023-02-01", "2023-03-01", "2023-04-01"],
            "amount": [0, 0, 0, 0],
        })
        assert _is_non_declining(df) is False

    def test_all_negative_returns_false(self):
        df = pd.DataFrame({
            "date": ["2023-01-01", "2023-02-01", "2023-03-01"],
            "amount": [-100, -50, -10],
        })
        assert _is_non_declining(df) is False

    def test_integer_date_column_declining(self):
        # C4: _is_non_declining must handle integer dtype internally with unit="s".
        # 1672531200 = 2023-01-01, 1675209600 = 2023-02-01, 1677628800 = 2023-03-01.
        # Without unit="s", pd.to_datetime interprets these as nanoseconds (year ~1970),
        # collapsing all three into the same month bucket and returning True incorrectly.
        df = pd.DataFrame({
            "date": [1672531200, 1675209600, 1677628800],
            "amount": [100, 80, 50],
        })
        assert _is_non_declining(df) is False

    # C4: Direct standalone call — non-declining with integer epoch seconds.
    # Verifies correct monthly bucketing independent of any pipeline context.
    def test_integer_epoch_seconds_non_declining(self):
        # 1672531200 = 2023-01-01, 1675209600 = 2023-02-01, 1677628800 = 2023-03-01.
        # Spend increases each month → non-declining.
        df = pd.DataFrame({
            "date": [1672531200, 1675209600, 1677628800],
            "amount": [100, 200, 300],
        })
        assert _is_non_declining(df) is True

    # C4: Cross-month boundary — two distinct integer epoch seconds in different months.
    # This test ensures bucketing is by calendar month, not by raw seconds.
    def test_integer_epoch_seconds_same_month_buckets_correctly(self):
        # Both 1672531200 (2023-01-01) and 1672617600 (2023-01-02) are January.
        # 1675209600 = 2023-02-01. Two months, spend goes 200→300 → non-declining.
        df = pd.DataFrame({
            "date": [1672531200, 1672617600, 1675209600, 1677628800],
            "amount": [100, 100, 200, 300],
        })
        assert _is_non_declining(df) is True

    # P3-3 FIX: Sparse months are absent observations, not zero-spend.
    # Two active months with a gap: only 2 data points, both 100 → non-declining.
    def test_gap_month_non_declining(self):
        df = pd.DataFrame({
            "date": ["2023-01-01", "2023-03-01"],
            "amount": [100, 100],
        })
        assert _is_non_declining(df) is True


# ── enrich_subcategories ──────────────────────────────────────────────────────

class TestEnrichSubcategories:
    def test_no_generalist(self):
        df = pd.DataFrame({"amount": [100.0], "cleaned_remarks": ["zomato"], "predicted_category": ["food"]})
        result = enrich_subcategories(df)
        assert result["inferred_subcategory"].isna().all()

    def test_schema_col_lowercase(self):
        from schema import Col
        assert Col.INFERRED_SUBCATEGORY == "inferred_subcategory"

    def test_high_amount_gets_electronics_subcategory(self):
        df = pd.DataFrame({
            "amount": [1500.0, 500.0],
            "cleaned_remarks": ["amazon", "amazon"],
            "predicted_category": ["shopping", "shopping"],
        })
        result = enrich_subcategories(df)
        assert result["inferred_subcategory"].iloc[0] == "electronics"
        assert result["inferred_subcategory"].iloc[1] == "general_purchase"


# ── anomaly suppression ──────────────────────────────────────────────────────

class TestAnomalySuppression:
    def test_anomaly_suppression_majority(self):
        df = pd.DataFrame({"is_anomaly": [True, True, False]})
        assert _check_anomaly_suppression(df) is True

    # FIX-8: Test boundary conditions for anomaly suppression ratio threshold (0.30)
    def test_anomaly_suppression_boundaries(self):
        import pandas as pd
        from passion_detector import _check_anomaly_suppression

        # 1. Below threshold (2/10 = 20.0% anomaly ratio)
        df_below = pd.DataFrame({"is_anomaly": [True, True, False, False, False, False, False, False, False, False]})
        assert _check_anomaly_suppression(df_below) is False

        # 2. Exactly threshold (3/10 = 30.0% anomaly ratio)
        df_exact = pd.DataFrame({"is_anomaly": [True, True, True, False, False, False, False, False, False, False]})
        assert _check_anomaly_suppression(df_exact) is False

        # 3. Above threshold (4/10 = 40.0% anomaly ratio)
        df_above = pd.DataFrame({"is_anomaly": [True, True, True, True, False, False, False, False, False, False]})
        assert _check_anomaly_suppression(df_above) is True
```
