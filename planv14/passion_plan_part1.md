# Passion Detection Engine — Master Plan (Part 1 of 8)

## 0. Exact Module Build & Implementation Order

Because the passion sidecar has tight circular coupling risks, the migration must explicitly enforce the exact order of implementation, build, and verification. The required implementation order is:

0. **`config.py`** (Update `config.py` with the new structured `TIP_CORPUS` schema to prevent import-time crash blockers in `contracts.py`)
1. **`schema.py`** (add inferred subcategory and confidence constants)
2. **`PipelineResult` migration** (audit and convert all codebase constructor calls to keyword-only)
3. **`pipeline_result.py`** (define `PassionResult` immutable container with lazy imports)
4. **`passion_models.py`** (define `PassionSignal` dataclass with schema fields)
5. **`candidate.py`** (define `Candidate` container)
6. **`passion_utils.py`** (implement `StepBudgetGuard` timing and soft/hard warnings)
7. **`config_passion.py`** (set thresholds, aliases, MappingProxyType config)
8. **`marketplace_subcategory.py`** (implement subcategory classifications)
9. **`passion_detector.py`** (candidate extraction & anomaly suppression checks)
10. **`passion_insight_generator.py`** (lifestyle opportunities & template rendering)
11. **`passion_pipeline.py`** (orchestrate sub-stages)
12. **`pipeline.py` integration hook** (wire `_attach_passion_results` and `_write_crash_dumps`)
13. **`tests/`** (full 174-test regression verification suite)

---

## 1. `schema.py` Update

Add only the two missing constants to the existing `Col` class. Insert after the `INSIGHT_SCORE` line (around line 95):

```python
    # ── Passion Engine: Subcategory Inference ─────────────────────────────
    INFERRED_SUBCATEGORY   = "inferred_subcategory"
    SUBCATEGORY_CONFIDENCE = "subcategory_confidence"
```

**Existing constants already present (no changes needed):**
- `INSIGHT_SCORE = "insight_score"` (line 95)
- `IS_RECURRING = "is_recurring"` (line 86)
- `RECURRING_FREQUENCY = "recurring_frequency"` (line 87)
- `IS_ANOMALY = "is_anomaly"` (line 83)
- `IS_KNOWN_PERSON = "is_known_person"` (line 61)
- `PERCENT_DEVIATION = "percent_deviation"` (line 80)

> **CRITICAL**: All Col values are **lowercase** strings (e.g. `"insight_score"`, NOT `"INSIGHT_SCORE"`). All test DataFrames and new modules must use Col constants or lowercase string keys.

> **STRUCTURAL MANDATE** (Fix 20): `Col.INFERRED_SUBCATEGORY` and `Col.SUBCATEGORY_CONFIDENCE` are structurally present in all deployments after migration. `INSIGHT_ENGINE_PASSION_ENABLED` controls runtime execution only. Partial deploys without these schema constants are unsupported.



## 2. `contracts.py`

> **A2 MANDATE**: `TIP_CORPUS` schema is structurally changing — direct production imports of `TIP_CORPUS` from `config` are **forbidden** after migration. Only `contracts.py` itself may import from `config`. All consumers (including `insight_generator.py`, `bootstrap.py`) must import `TIP_CORPUS` from `contracts`, never from `config` directly. Add a test asserting `insight_generator.py` does not import `TIP_CORPUS` from `config` (see Part 8).
>
> **B4**: `Col` is **NOT** re-exported from `contracts.py`. Remove it from `__all__` and remove the `from schema import Col` import. All callers must import `Col` directly from `schema`.

```python
"""
contracts.py -- Facade for external dependencies.
Wraps config.py objects into immutable MappingProxyType at import time.

FIX: Always normalize regardless of source shape. If config.TIP_CORPUS is
already a MappingProxyType with plain dict inner values, the bypass would
skip re-freezing, and _validate_tip_corpus raises confusingly.
"""

from types import MappingProxyType
# B4: Col is NOT re-exported. All callers must import Col directly from schema.
from config import INSIGHT_TEMPLATES as _IT, TIP_CORPUS as _TC

# C3: Freezer rules
def _freeze_insight_templates(raw: dict | MappingProxyType) -> MappingProxyType:
    if not isinstance(raw, (dict, MappingProxyType)):
        raise TypeError("INSIGHT_TEMPLATES must be dict or MappingProxyType")
    result = {}
    for k, v in raw.items():
        if not isinstance(v, (list, tuple)):
            raise TypeError(f"INSIGHT_TEMPLATES[{k}] must be a list or tuple of strings")
        result[k] = tuple(v)
    return MappingProxyType(result)

def _freeze_tip_corpus(raw: dict | MappingProxyType) -> MappingProxyType:
    # Fix 13: _freeze_tip_corpus now owns ALL business rule validation.
    # Error order: required keys → type checks → content rules → freeze.
    _ALLOWED_INSIGHTS: frozenset[str] = frozenset({
        "subscription", "spending_spike", "lifestyle_opportunity",
        "trend_warning", "budget_risk", "no_action", "any",
    })
    if not isinstance(raw, (dict, MappingProxyType)):
        raise TypeError(f"TIP_CORPUS must be dict/MappingProxyType, got {type(raw)}")
    result = {}
    for k, v in raw.items():
        if not isinstance(v, (dict, MappingProxyType)):
            raise TypeError(
                f"Config Migration Error: TIP_CORPUS[{k!r}] must be a dict containing "
                f"'text', 'categories', and 'insights'. Got {type(v)}. "
                "Please update config.TIP_CORPUS to the new schema."
            )
        # Fix 13: Required key check BEFORE empty/content checks.
        for req in ("text", "categories", "insights"):
            if req not in v:
                raise ValueError(
                    f"Config Migration Error: TIP_CORPUS[{k!r}] is missing required key {req!r}. "
                    "Ensure config.TIP_CORPUS matches the new schema."
                )
        # Fix 13: text must be str.
        if not isinstance(v["text"], str):
            raise TypeError(f"TIP_CORPUS[{k!r}]['text'] must be str, got {type(v['text'])}")
        # Fix 13: categories and insights must be list/tuple of str.
        for seq_key in ("categories", "insights"):
            seq = v[seq_key]
            if not isinstance(seq, (list, tuple)):
                raise TypeError(
                    f"TIP_CORPUS[{k!r}][{seq_key!r}] must be list or tuple, got {type(seq)}"
                )
            bad = [x for x in seq if not isinstance(x, str)]
            if bad:
                raise TypeError(
                    f"TIP_CORPUS[{k!r}][{seq_key!r}] elements must be str, got {[type(x).__name__ for x in bad]}"
                )
        is_generic = k.startswith("generic_")
        cats = tuple(c.strip().lower() for c in v["categories"])
        insights_tup = tuple(i.strip().lower() for i in v["insights"])
        # Fix 13: Validate lowercase-stripped values.
        for cat in cats:
            if not cat:
                raise ValueError(f"TIP_CORPUS[{k!r}] 'categories' contains blank string after strip")
        for ins in insights_tup:
            if not ins:
                raise ValueError(f"TIP_CORPUS[{k!r}] 'insights' contains blank string after strip")
            if ins not in _ALLOWED_INSIGHTS:
                raise ValueError(
                    f"TIP_CORPUS[{k!r}] insight {ins!r} not in allowed set {sorted(_ALLOWED_INSIGHTS)}"
                )
        # Fix 13: Non-generic tips cannot have empty categories or insights.
        # B5: Empty-tuple wildcard is ONLY permitted for 'generic_*' tips.
        if not is_generic:
            if not cats:
                raise ValueError(
                    f"TIP_CORPUS[{k!r}] has empty 'categories' but its tip_id does not start "
                    f"with 'generic_'. Only 'generic_*' tips may use empty-tuple wildcard behavior."
                )
            if not insights_tup:
                raise ValueError(
                    f"TIP_CORPUS[{k!r}] has empty 'insights' but its tip_id does not start "
                    f"with 'generic_'. Only 'generic_*' tips may use empty-tuple wildcard behavior."
                )
            # Fix 13: Non-generic tips cannot contain "any" wildcard.
            if "any" in cats:
                raise ValueError(
                    f"TIP_CORPUS[{k!r}] is a non-generic tip but contains wildcard 'any' in 'categories'."
                )
            if "any" in insights_tup:
                raise ValueError(
                    f"TIP_CORPUS[{k!r}] is a non-generic tip but contains wildcard 'any' in 'insights'."
                )
        frozen_inner: dict = {"text": v["text"], "categories": cats, "insights": insights_tup}
        # Preserve any extra scalar fields (future-proof).
        for kk, vv in v.items():
            if kk in frozen_inner:
                continue
            if isinstance(vv, (str, int, float, bool)):
                frozen_inner[kk] = vv
            else:
                raise TypeError(f"Invalid type in TIP_CORPUS[{k!r}][{kk!r}]: {type(vv)}")
        result[k] = MappingProxyType(frozen_inner)
    return MappingProxyType(result)

INSIGHT_TEMPLATES = _freeze_insight_templates(_IT)
TIP_CORPUS = _freeze_tip_corpus(_TC)

# C2: Define lookup_matching_tip_ids
def lookup_matching_tip_ids(category: str, insight_type: str) -> list[str]:
    cat_norm = str(category).strip().lower()
    type_norm = str(insight_type).strip().lower()
    matches = []
    for tip_id, tip_data in TIP_CORPUS.items():
        cats = tip_data.get("categories", ())
        types = tip_data.get("insights", ())

        # B5: Empty-tuple wildcard is ONLY honoured for "generic_*" tips.
        # Non-generic tips must have explicit category/insight lists (enforced
        # at startup by _freeze_tip_corpus) so the wildcard path here is
        # purely a runtime safety valve for generic tips that did pass validation.
        is_generic = tip_id.startswith("generic_")
        cat_match = (cat_norm in cats) or ("any" in cats) or (is_generic and len(cats) == 0)
        type_match = (type_norm in types) or ("any" in types) or (is_generic and len(types) == 0)

        if cat_match and type_match:
            matches.append(tip_id)
    return matches


# B4: Col removed from __all__. Import Col from schema directly.
__all__ = ["INSIGHT_TEMPLATES", "TIP_CORPUS", "lookup_matching_tip_ids"]
```

---

## 3. `bootstrap.py`

```python
import os
import string
import sys
from config_passion import validate_merchant_aliases, PASSION_INSIGHT_TEMPLATES
from contracts import INSIGHT_TEMPLATES, TIP_CORPUS
from types import MappingProxyType

# FIX H11: Use structured logger from logger_factory instead of plain logging.
from logger_factory import get_logger

logger = get_logger(__name__)

# FIX-37: Explicit public API.
__all__ = ["run_startup_checks", "validate_template_fields", "ALLOWED_FORMAT_SPECS"]

ALLOWED_FORMAT_SPECS = frozenset({
    "",
    ".0%",
    ".1%",
    ".0f",
    ".1f",
    ".2f",
    ",.0f",
    ",.2f",
    "d",
    ",d",
})

# FIX M6 + FIX-14: validate_template_fields now rejects attribute/index access,
# conversions (!r, !s, !a), nested format specs, and positional fields.
# FIX-14: string.Formatter().parse() yields (literal, field, format_spec, conversion).
# Previous code ignored conversion and format_spec — "{x!r}" or "{x:{y}}" passed silently.
def validate_template_fields(template: str, allowed: set[str]) -> None:
    try:
        parsed = list(string.Formatter().parse(template))
    except (ValueError, KeyError) as e:
        raise ValueError(f"Malformed template {template!r}: {e}") from e
    fields = set()
    for _, field, format_spec, conversion in parsed:
        if field is None:
            continue
        if conversion is not None:
            raise ValueError(f"Conversions (!r, !s, !a) are forbidden in template: {template!r}")
        if "{" in (format_spec or "") or "}" in (format_spec or ""):
            raise ValueError(f"Nested format specs are forbidden in template: {template!r}")
        if (format_spec or "") not in ALLOWED_FORMAT_SPECS:
            raise ValueError(f"Format spec {format_spec!r} not in allowed list: {ALLOWED_FORMAT_SPECS}")
        if field == "" or field.isdigit():
            raise ValueError(f"Positional format fields are forbidden in template: {template!r}")
        if any(c in field for c in (".", "[", "]")):
            raise ValueError(
                f"Field names must be simple identifiers, got {field!r} in {template!r}"
            )
        fields.add(field)
    if unknown := fields - allowed:
        raise ValueError(f"Unknown template fields: {sorted(unknown)}")


def _validate_python_version() -> None:
    if sys.version_info < (3, 11):
        raise RuntimeError(
            f"Insight Engine requires Python 3.11 or later. Running: {sys.version}"
        )


# FIX #17 + P1.6: _validate_schema_columns.
# P1.6: Use vars(Col) instead of dir(Col). dir() includes inherited object
# attrs like __init__, __repr__ etc. vars() returns only the class's own namespace.
# FIX #17: Reject None and empty-string values before the lowercase check.
# FIX H6: Check for duplicate Col values — two constants sharing the same
# string value would cause DataFrames to silently overwrite each other's columns.
# FIX-26: Accept optional _col_cls parameter for testability — tests can now call
# _validate_schema_columns(mock_col) without patching bootstrap.Col.
def _validate_schema_columns(_col_cls=None) -> None:
    from schema import Col as _default_col
    col_cls = _col_cls or _default_col

    required = {
        "AMOUNT", "CLEANED_REMARKS", "DATE", "PREDICTED_CATEGORY",
        "IS_ANOMALY", "IS_RECURRING", "INSIGHT_SCORE", "IS_KNOWN_PERSON",
        "RECURRING_FREQUENCY", "INFERRED_SUBCATEGORY", "SUBCATEGORY_CONFIDENCE",
    }
    # P1.6: vars(col_cls) returns only class's own __dict__, not inherited attrs.
    present = {
        a for a, v in vars(col_cls).items()
        if not a.startswith("_") and isinstance(v, str)
    }
    missing = required - present
    if missing:
        raise RuntimeError(f"schema.Col missing required constants: {sorted(missing)}")
    for attr in required:
        val = getattr(col_cls, attr, None)
        if val is None:
            raise RuntimeError(f"schema.Col.{attr} is missing")
        if not val:
            raise RuntimeError(f"schema.Col.{attr} value must be non-empty")
        if val != val.lower():
            raise RuntimeError(
                f"schema.Col.{attr} value must be lowercase, got {val!r}"
            )

    # FIX H6 + FIX L5: Detect duplicate string values across all Col constants.
    from collections import Counter
    all_values = [
        v for a, v in vars(col_cls).items()
        if not a.startswith("_") and isinstance(v, str)
    ]
    duplicates = [item for item, count in Counter(all_values).items() if count > 1]
    if duplicates:
        raise RuntimeError(f"schema.Col has duplicate string values: {duplicates}")


def _validate_insight_templates() -> None:
    if not isinstance(INSIGHT_TEMPLATES, MappingProxyType):
        raise TypeError(f"INSIGHT_TEMPLATES must be MappingProxyType, got {type(INSIGHT_TEMPLATES)}")
    insight_allowed_fields = {"merchant", "amount", "category", "merchant_count", "spend_share",
                              "date", "pct", "frequency"}
    for category, templates in INSIGHT_TEMPLATES.items():
        if not isinstance(templates, tuple):
            raise TypeError(f"INSIGHT_TEMPLATES['{category}'] must be tuple, got {type(templates)}")
        for t in templates:
            if not isinstance(t, str):
                raise TypeError(f"INSIGHT_TEMPLATES['{category}'] entry must be str, got {type(t)}")
            validate_template_fields(t, insight_allowed_fields)


def _validate_passion_templates() -> None:
    if not isinstance(PASSION_INSIGHT_TEMPLATES, MappingProxyType):
        raise TypeError(f"PASSION_INSIGHT_TEMPLATES must be MappingProxyType, got {type(PASSION_INSIGHT_TEMPLATES)}")
    if "lifestyle_opportunity" not in PASSION_INSIGHT_TEMPLATES:
        raise ValueError("PASSION_INSIGHT_TEMPLATES must contain key 'lifestyle_opportunity'")
    passion_allowed_fields = {"category", "merchant_count", "spend_share", "trend_direction", "total_spend"}
    for key, templates in PASSION_INSIGHT_TEMPLATES.items():
        if not isinstance(templates, tuple):
            raise TypeError(f"PASSION_INSIGHT_TEMPLATES['{key}'] must be tuple, got {type(templates)}")
        for t in templates:
            if not isinstance(t, str):
                raise TypeError(f"PASSION_INSIGHT_TEMPLATES['{key}'] entry must be str, got {type(t)}")
            validate_template_fields(t, passion_allowed_fields)





def _validate_tip_corpus() -> None:
    # Fix 13: All business rule validation is now owned by contracts._freeze_tip_corpus,
    # which runs at import time. This function only asserts the frozen MappingProxyType
    # shape that contracts guarantees, and delegates template rendering to _dry_render_templates.
    if not isinstance(TIP_CORPUS, MappingProxyType):
        raise TypeError(f"TIP_CORPUS must be MappingProxyType, got {type(TIP_CORPUS)}")
    for tip_id, tip_data in TIP_CORPUS.items():
        if not isinstance(tip_data, MappingProxyType):
            raise TypeError(
                f"TIP_CORPUS['{tip_id}'] inner value must be MappingProxyType, got {type(tip_data)} "
                "(contracts._freeze_tip_corpus should have enforced this at import time)"
            )
        # Defensive: assert required keys are present (contracts enforces this, belt-and-suspenders).
        for req in ("text", "categories", "insights"):
            if req not in tip_data:
                raise ValueError(
                    f"TIP_CORPUS['{tip_id}'] missing required key '{req}' "
                    "(contracts._freeze_tip_corpus invariant violated)"
                )


def _validate_secret() -> None:
    from log_utils import _get_secret
    _get_secret()


# FIX-28: Dry-render all templates at startup to catch format errors early.
# A template like "{merchant} spent {ammount}" (typo) would only crash at
# runtime when a real insight is rendered. Dry-rendering catches it here.
def _dry_render_templates() -> None:
    # Blocker 17: Add total_spend and trend_direction to dry-render sample values
    _SAMPLE_VALUES = {
        "merchant": "sample_merchant", "amount": 1.0, "category": "food",
        "merchant_count": 1, "spend_share": 0.1, "date": "2025-01-01",
        "pct": 10, "frequency": "monthly",
        "total_spend": 100.0, "trend_direction": "non_declining",
    }
    all_corpora = [
        ("INSIGHT_TEMPLATES", INSIGHT_TEMPLATES),
        ("PASSION_INSIGHT_TEMPLATES", PASSION_INSIGHT_TEMPLATES),
    ]
    for corpus_name, corpus in all_corpora:
        for key, templates in corpus.items():
            for t in templates:
                try:
                    t.format(**_SAMPLE_VALUES)
                except (KeyError, ValueError, TypeError) as e:
                    raise ValueError(
                        f"{corpus_name}['{key}'] template fails dry render: {t!r} -- {e}"
                    ) from e
    for tip_id, tip_data in TIP_CORPUS.items():
        t = tip_data.get("text", "")
        try:
            t.format(**_SAMPLE_VALUES)
        except (KeyError, ValueError, TypeError) as e:
            raise ValueError(
                f"TIP_CORPUS['{tip_id}']['text'] fails dry render: {t!r} -- {e}"
            ) from e

    # Fix #8: Context-specific dry-render per insight type.
    # The superset pass above catches unknown fields; these passes catch fields
    # that are valid in the superset but missing in a specific render context.
    # e.g. {trend_direction} passes the superset but fails subscription context.
    _SUBSCRIPTION_VALUES = {
        "merchant": "sample_merchant",
        "amount": 1.0,
        "category": "food",
        "pct": 10,
        "frequency": "monthly",
        "date": "2025-01-01",
    }
    _LIFESTYLE_VALUES = {
        "category": "food",
        "merchant_count": 1,
        "spend_share": 0.1,
        "trend_direction": "non_declining",
        "total_spend": 100.0,
    }
    for tip_id, tip_data in TIP_CORPUS.items():
        t = tip_data.get("text", "")
        insights = tip_data.get("insights", ())
        for insight_type in insights:
            ctx = _LIFESTYLE_VALUES if insight_type == "lifestyle_opportunity" else _SUBSCRIPTION_VALUES
            try:
                t.format(**ctx)
            except (KeyError, ValueError, TypeError) as e:
                raise ValueError(
                    f"TIP_CORPUS[{tip_id!r}]['text'] fails context render "
                    f"for insight_type={insight_type!r}: {t!r} -- {e}"
                ) from e


def run_startup_checks(env: str | None = None) -> None:
    _validate_python_version()
    env = (env or os.environ.get("ENV", "development")).strip().lower()
    secret = os.environ.get("INSIGHT_ENGINE_SECRET")
    if env in ("production", "prod", "staging") and not secret:
        raise RuntimeError("CRITICAL: INSIGHT_ENGINE_SECRET missing in production/staging.")
    _validate_secret()
    _validate_schema_columns()
    validate_merchant_aliases()
    _validate_insight_templates()
    _validate_passion_templates()
    _validate_tip_corpus()
    # FIX-28: Catch template format errors at startup, not at runtime.
    _dry_render_templates()

    # FIX H13: config_passion.validate_config() was called at module import time,
    # which crashes before logging is configured. Now deferred to bootstrap.
    from config_passion import validate_config as _validate_passion_config
    _validate_passion_config()

    logger.info("Insight Engine startup checks passed successfully.")
```

---

## 4. `log_utils.py`

```python
import os
import hmac
import hashlib
import threading
import numpy as np
import pandas as pd
from typing import Any

# FIX H11: Use structured logger from logger_factory instead of plain logging.
from logger_factory import get_logger

__all__ = ["log_safe_merchant", "log_safe_text", "verify_merchant_token"]
logger = get_logger(__name__)

# FIX L5: Renamed for clarity. _DEV_SECRET_FALLBACK is the constant.
_DEV_SECRET_FALLBACK = b"dev-only-not-for-production-use!"
_active_secret: bytes | None = None
_secret_lock = threading.Lock()

# E2: Provide _reset_secret_cache for tests
def _reset_secret_cache() -> None:
    global _active_secret
    with _secret_lock:
        _active_secret = None

def _get_secret() -> bytes:
    global _active_secret
    if _active_secret is not None:
        return _active_secret
    with _secret_lock:
        if _active_secret is not None:
            return _active_secret
        
        env_name = os.environ.get("ENV", "development").strip().lower()
        env_val = os.environ.get("INSIGHT_ENGINE_SECRET")
        if env_val:
            secret_stripped = env_val.strip()
            if not secret_stripped:
                raise RuntimeError("INSIGHT_ENGINE_SECRET is set but blank.")
            secret_bytes = secret_stripped.encode('utf-8')
            if len(secret_bytes) < 32:
                raise RuntimeError(f"INSIGHT_ENGINE_SECRET must be at least 32 bytes, got {len(secret_bytes)}.")
            _active_secret = secret_bytes
        else:
            # E1: Never fall back in prod/staging
            if env_name in ("production", "prod", "staging"):
                raise RuntimeError("CRITICAL: INSIGHT_ENGINE_SECRET missing in production/staging.")
            logger.warning(
                "insight_engine_dev_secret_active: HMAC tokens are "
                "deterministic and NOT secure. Set INSIGHT_ENGINE_SECRET "
                "before deploying to production."
            )
            _active_secret = _DEV_SECRET_FALLBACK
            
        return _active_secret

def _hmac_hex(value: str) -> str:
    return hmac.new(_get_secret(), value.encode('utf-8'), hashlib.sha256).hexdigest()[:32]

def _is_safe_scalar(val: Any) -> bool:
    return isinstance(val, (str, float, int, type(None), np.generic))

def log_safe_merchant(merchant: Any) -> str:
    if not _is_safe_scalar(merchant): return ""
    try:
        if pd.isna(merchant) or not str(merchant).strip(): return ""
    except (TypeError, ValueError): return ""
    return f"merchant:{_hmac_hex(str(merchant))}"

def log_safe_text(text: Any) -> str:
    if not _is_safe_scalar(text): return ""
    try:
        if pd.isna(text) or not str(text).strip(): return ""
    except (TypeError, ValueError): return ""
    return f"text:{_hmac_hex(str(text))}"


# P3.3: verify_merchant_token — constant-time HMAC comparison.
# Use whenever comparing HMAC tokens to prevent timing side-channels.
def verify_merchant_token(token: str, merchant: str) -> bool:
    """Constant-time token verification. Use whenever comparing HMAC tokens."""
    expected = log_safe_merchant(merchant)
    # FIX L8: Catch TypeError in compare_digest (e.g. if token is None or wrong type)
    try:
        return hmac.compare_digest(token, expected)
    except TypeError:
        return False
```

---

## 5. `hash_utils.py` Update

**FIX H7**: The new HMAC-based `log_utils.py` provides secure PII masking,
but `hash_utils.stable_hash()` still exists and produces reversible hashes.
Any caller using the old function still leaks PII. Add deprecation warning.

> **A3 CI MANDATE**: After migration, add a test that fails if any production module (outside `hash_utils.py` itself and its explicit deprecation tests) imports `stable_hash`. Use `ast.parse` or `grep` in a pytest session-scoped fixture. No production module may import `stable_hash` — permitted only in `hash_utils.py` (definition) and tests that explicitly validate the deprecation warning. See Part 8 for the test implementation.

Replace the entire body of `hash_utils.py` with:

```python
import warnings
import hashlib

# FIX-37: Explicit public API.
__all__ = ["stable_hash"]


def stable_hash(value: str) -> str:
    """
    DEPRECATED: This function produces reversible, non-keyed hashes.
    Use log_utils.log_safe_merchant() for HMAC-keyed PII masking instead.
    """
    warnings.warn(
        "stable_hash is deprecated and produces reversible hashes. "
        "Use log_utils.log_safe_merchant instead.",
        DeprecationWarning,
        stacklevel=2,
    )
    return hashlib.sha256(value.encode("utf-8")).hexdigest()[:12]
```

**FIX-21**: The `recurring_detector.py` and `tests/test_logging_safety.py` still import and call `stable_hash()`.
Migrate the callers:

In `recurring_detector.py`:
```python
# Remove:
from hash_utils import stable_hash
import logging
logger = logging.getLogger(__name__)

# Add:
from log_utils import log_safe_merchant
from logger_factory import get_logger
logger = get_logger(__name__)
```

Replace every occurrence of:
```python
stable_hash(identifier)
```
with:
```python
log_safe_merchant(identifier)
```

In `tests/test_logging_safety.py`:
```python
# Remove:
from hash_utils import stable_hash

# Add:
from log_utils import log_safe_merchant
```

And in `test_pii_redaction_coverage`:
```python
# Replace:
    assert any(stable_hash("Netflix") in msg for msg in caplog.messages)
# With:
    assert any(log_safe_merchant("Netflix") in msg for msg in caplog.messages)
```

---

## 6. `banned_content.py`

> **FIX-12**: NFKC normalization does NOT map Cyrillic lookalikes to Latin.
> Cyrillic ѕ (U+0455) maps to ѕ under NFKC, NOT to Latin "s". The original
> plan's claim that NFKC handles Cyrillic was incorrect. An explicit
> confusables translation table is required.
>
> **FIX-13**: `_contains_banned_content` was a private function in
> `passion_insight_generator.py`, imported by `passion_pipeline.py` — a
> contract violation. Move the public API to `banned_content.py`.

```python
import re
import unicodedata

# P2.7: Add common plurals/variants to BANNED_DISPLAY_WORDS.
# \b prevents "scams" matching "scam". Rather than changing the regex
# (which risks false positives), add explicit variants.
BANNED_DISPLAY_WORDS: frozenset[str] = frozenset({
    "fraud",
    "scam", "scams",
    "illegal",
    "banned",
    "weapon", "weapons",
    "drugs", "drug",
    "abuse",
    "porn", "pornography",
    "escort", "escorts", "escorting",
    "gambling", "gamble",
    "casino", "casinos",
})

# FIX-13: Pre-compiled banned-word pattern — longest-first ordering prevents
# shorter words from shadowing longer compounds.
# FIX-13: Uses (?<!\w)/(?!\w) instead of \b for word boundary matching.
# \b fails when keywords start/end with non-word characters.
_ordered_banned = sorted(BANNED_DISPLAY_WORDS, key=len, reverse=True)
BANNED_PATTERN = re.compile(
    r'(?<!\w)(?:' + '|'.join(map(re.escape, _ordered_banned)) + r')(?!\w)',
    re.IGNORECASE,
)

# FIX-12: Explicit confusables table for Cyrillic lookalikes.
# NFKC normalization does NOT convert these — they are distinct Unicode
# codepoints that happen to be visually identical to Latin characters.
_CONFUSABLES = str.maketrans({
    "\u0455": "s",  # Cyrillic small dze  → Latin s
    "\u0441": "c",  # Cyrillic small es   → Latin c
    "\u0430": "a",  # Cyrillic small a    → Latin a
    "\u0435": "e",  # Cyrillic small ie   → Latin e
    "\u043E": "o",  # Cyrillic small o    → Latin o
    "\u0440": "p",  # Cyrillic small er   → Latin p
    "\u0445": "x",  # Cyrillic small ha   → Latin x
    "\u0443": "y",  # Cyrillic small u    → Latin y
    "\u0405": "S",  # Cyrillic capital DZE
    "\u0421": "C",  # Cyrillic capital ES
    "\u0410": "A",  # Cyrillic capital A
    "\u0415": "E",  # Cyrillic capital IE
    "\u041E": "O",  # Cyrillic capital O
    "\u0420": "P",  # Cyrillic capital ER
    "\u0425": "X",  # Cyrillic capital HA
    "\u0423": "Y",  # Cyrillic capital U
})


# FIX 25: Exact supported threat model documentation.
# Coverage: Common Latin variants, explicit Cyrillic lookalikes (_CONFUSABLES), 
# and zero-width character stripping. 
# NOT covered: Greek/Armenian homoglyphs, advanced bidirectional overrides.
# Adding broad regex obfuscation causes false positives, so we only add terms 
# based on observed real-world misses.
def _normalize_display_text(text: str) -> str:
    """NFKC + Cyrillic confusable mapping for banned-content detection."""
    text = unicodedata.normalize("NFKC", text).translate(_CONFUSABLES).lower()
    # F2: Strip zero-width characters
    text = re.sub(r'[\u200B\u200C\u200D\uFEFF]', '', text)
    return text

# Blocker 15: Safe terms for compact substring matching
_COMPACT_SAFE_TERMS = frozenset({
    "fraud", "scam", "scams", "porn", "pornography", 
    "escort", "escorts", "escorting", "casino", "casinos",
})

# FIX-10: Narrowed separator to whitespace, punctuation, and zero-width chars to prevent
# matching across words (e.g. "spoke person" matching "porn").
_SEP = r"[\s._\-\u200B\u200C\u200D\uFEFF]*"

def _obfuscated_word_pattern(word: str) -> str:
    chars = [re.escape(c) for c in word]
    return r"(?<![a-z])" + _SEP.join(chars) + r"(?![a-z])"

_OBFUSCATED_SAFE_PATTERN = re.compile(
    "|".join(_obfuscated_word_pattern(term) for term in sorted(_COMPACT_SAFE_TERMS, key=len, reverse=True)),
    re.IGNORECASE,
)

# FIX-13: Public API — replaces the private _contains_banned_content that was
# previously defined in passion_insight_generator.py.
def contains_banned_content(text: str) -> bool:
    """Check if text contains any banned display words after normalization."""
    if not isinstance(text, str):
        text = str(text) if text is not None else ""
    normalized = _normalize_display_text(text)
    if BANNED_PATTERN.search(normalized):
        return True
    
    return bool(_OBFUSCATED_SAFE_PATTERN.search(normalized))


__all__ = ["BANNED_DISPLAY_WORDS", "BANNED_PATTERN", "contains_banned_content"]
```

---

## 7. `config_passion.py`

```python
from types import MappingProxyType

# B3: Silent fallback to empty dict is forbidden — missing SPECIFIC_MERCHANT_ALIASES
# would silently produce a broken alias map with no core aliases, hiding config errors.
try:
    from config import SPECIFIC_MERCHANT_ALIASES as _raw_core_aliases
except ImportError as e:
    raise RuntimeError(
        "config.SPECIFIC_MERCHANT_ALIASES is required. "
        "Ensure config.py exposes SPECIFIC_MERCHANT_ALIASES as a dict[str, str]."
    ) from e
if not isinstance(_raw_core_aliases, dict):
    raise TypeError(
        f"SPECIFIC_MERCHANT_ALIASES must be a dict mapping str->str, "
        f"got {type(_raw_core_aliases)}"
    )

# Normalize core aliases
_CORE_ALIASES = {}
for alias, canonical in _raw_core_aliases.items():
    if isinstance(alias, str) and isinstance(canonical, str):
        _CORE_ALIASES[alias.strip().lower()] = canonical.strip().lower()

# P3.2: Explicit type annotations on all config constants.
MAX_SPIKE_CANDIDATES: int = 5
PASSION_MIN_MONTHS: int = 3
# FIX-22: Keep 1 because subcategory enrichment is intentionally useful for 1-row input
# even though passion signals require PASSION_MERCHANT_COUNT_MIN (3) distinct merchants.
PASSION_MIN_DEBIT_ROWS: int = 1
PASSION_MERCHANT_COUNT_MIN: int = 3
PASSION_SPEND_SHARE_THRESHOLD: float = 0.25
PASSION_ANOMALY_SUPPRESSION_THRESHOLD: float = 0.30
DISTRESS_FEES_THRESHOLD: float = 0.15

MARKETPLACE_HIGH_AMOUNT_THRESHOLD: float = 1000.0
MARKETPLACE_HIGH_CONFIDENCE: float = 0.90
MARKETPLACE_LOW_CONFIDENCE: float = 0.85

PIPELINE_BUDGET_MS: float = 500.0
PIPELINE_TOP_N: int = 3

# FIX M4: Hard timeout is now configurable instead of baked into
# _StepBudgetGuard as a magic number. Allows CI/container environments
# to use a higher value without touching the class.
# PIPELINE_HARD_TIMEOUT_MS is cooperative only. It cannot interrupt blocked pandas, regex, or IO mid-call.
PIPELINE_HARD_TIMEOUT_MS: float = 2000.0

ELECTRONICS_ALLOWED_CATEGORIES: frozenset[str] = frozenset({"shopping"})

_PASSION_EXTRAS = {
    "amzn": "amazon",
    "amzn mktp": "amazon",
    "flpkrt": "flipkart",
    "uber bv": "uber",
}

# Normalize passion extras
_NORMALIZED_EXTRAS = {
    str(alias).strip().lower(): str(canonical).strip().lower()
    for alias, canonical in _PASSION_EXTRAS.items()
}

# B2: Compute conflicts at module load for later validation, but do NOT raise here.
# Raising at import time bypasses bootstrap logging and breaks module load order.
# validate_merchant_aliases() (called from bootstrap.run_startup_checks) raises instead.
_ALIAS_CONFLICTS = frozenset(
    alias for alias in set(_CORE_ALIASES) & set(_NORMALIZED_EXTRAS)
    if _CORE_ALIASES[alias] != _NORMALIZED_EXTRAS[alias]
)

PASSION_MERCHANT_ALIASES: MappingProxyType = MappingProxyType({
    **_NORMALIZED_EXTRAS,
    **_CORE_ALIASES,
})

# P1-5: Generalist canonicals — must match canonical values in PASSION_MERCHANT_ALIASES.
# All generalist detection goes through resolve_merchant_vectorized + .isin(GENERALIST_CANONICALS).
GENERALIST_CANONICALS: frozenset[str] = frozenset({"amazon", "flipkart", "meesho", "snapdeal"})

def validate_merchant_aliases(alias_map=None) -> None:
    # B2: Alias conflict detection deferred here from module-level so bootstrap
    # logging is available and errors are reported through structured channels.
    if alias_map is None and _ALIAS_CONFLICTS:
        raise ValueError(
            f"Alias conflict between core and passion extras: {sorted(_ALIAS_CONFLICTS)}"
        )
    aliases = alias_map if alias_map is not None else PASSION_MERCHANT_ALIASES
    bad_vals = [
        v for k, v in aliases.items()
        if not isinstance(v, str) or v != v.lower() or not v.strip()
    ]
    bad_keys = [
        k for k in aliases
        if not isinstance(k, str) or k != k.lower() or not k.strip()
    ]
    if bad_vals:
        raise ValueError(f"Non-lowercase, non-string, or empty values: {bad_vals}")
    if bad_keys:
        raise ValueError(f"Non-lowercase, non-string, or empty keys: {bad_keys}")

    # Validate GENERALIST_CANONICALS reachability
    # Fix #6: Check only alias_values — a canonical appearing only as an alias key
    # can still resolve away to a different value and fail resolved.isin(GENERALIST_CANONICALS).
    alias_values = set(aliases.values())
    missing = GENERALIST_CANONICALS - alias_values
    if missing:
        raise ValueError(f"GENERALIST_CANONICALS contains unreachable merchants: {missing}")



# Allowed fields: {category}, {merchant_count}, {spend_share}
PASSION_INSIGHT_TEMPLATES: MappingProxyType = MappingProxyType({
    "lifestyle_opportunity": (
        "You show strong lifestyle engagement in {category} with {merchant_count} merchants ({spend_share:.1%} of spend).",
    ),
})


# P1.10: Config numeric values not validated.
# Validates all numeric config constants. A misconfigured
# threshold (e.g. DISTRESS_FEES_THRESHOLD=2.0) would silently break
# the distress gate at runtime.
#
# FIX H13: This function is NO LONGER called at module import time.
# Module-level validate_config() crashed before logging was configured.
# It is now called from bootstrap.run_startup_checks() instead.
def validate_config() -> None:
    if MAX_SPIKE_CANDIDATES <= 0:
        raise ValueError("MAX_SPIKE_CANDIDATES must be positive")
    if PASSION_MIN_MONTHS <= 0:
        raise ValueError("PASSION_MIN_MONTHS must be positive")
    if PIPELINE_BUDGET_MS <= 0:
        raise ValueError("PIPELINE_BUDGET_MS must be positive")
    if PIPELINE_TOP_N <= 0:
        raise ValueError("PIPELINE_TOP_N must be positive")
    if PASSION_MERCHANT_COUNT_MIN <= 0:
        raise ValueError("PASSION_MERCHANT_COUNT_MIN must be positive")
    if not (0.0 <= PASSION_SPEND_SHARE_THRESHOLD <= 1.0):
        raise ValueError("PASSION_SPEND_SHARE_THRESHOLD must be in [0, 1]")
    if not (0.0 <= PASSION_ANOMALY_SUPPRESSION_THRESHOLD <= 1.0):
        raise ValueError("PASSION_ANOMALY_SUPPRESSION_THRESHOLD must be in [0, 1]")
    if not (0.0 <= DISTRESS_FEES_THRESHOLD <= 1.0):
        raise ValueError("DISTRESS_FEES_THRESHOLD must be in [0, 1]")
    # P1-6: Validate individual confidence bounds
    if not (0.0 <= MARKETPLACE_LOW_CONFIDENCE <= 1.0):
        raise ValueError("MARKETPLACE_LOW_CONFIDENCE must be in [0, 1]")
    if not (0.0 <= MARKETPLACE_HIGH_CONFIDENCE <= 1.0):
        raise ValueError("MARKETPLACE_HIGH_CONFIDENCE must be in [0, 1]")
    if MARKETPLACE_HIGH_CONFIDENCE <= MARKETPLACE_LOW_CONFIDENCE:
        raise ValueError("HIGH_CONFIDENCE must exceed LOW_CONFIDENCE")
    if MARKETPLACE_HIGH_AMOUNT_THRESHOLD <= 0:
        raise ValueError("MARKETPLACE_HIGH_AMOUNT_THRESHOLD must be positive")
    # FIX M4: Validate hard timeout exceeds budget.
    if PIPELINE_HARD_TIMEOUT_MS <= PIPELINE_BUDGET_MS:
        raise ValueError("PIPELINE_HARD_TIMEOUT_MS must exceed PIPELINE_BUDGET_MS")

    # B6: Do NOT swallow ImportError here. A broken config import means the
    # validation environment itself is broken, which is a hard failure.
    # Previously: except ImportError: pass — this silently skipped validation
    # of ELECTRONICS_ALLOWED_CATEGORIES when config was misconfigured.
    try:
        from config import CATEGORY_KEYWORDS, CATEGORY_PRIORITY
    except ImportError as e:
        raise RuntimeError(
            f"config import failed during validate_config: {e}. "
            "Ensure config.py exposes CATEGORY_KEYWORDS and CATEGORY_PRIORITY."
        ) from e
    cat_priority_keys = set()
    if isinstance(CATEGORY_PRIORITY, dict):
        cat_priority_keys = set(CATEGORY_PRIORITY.keys())
    elif isinstance(CATEGORY_PRIORITY, (list, tuple, set)):
        for item in CATEGORY_PRIORITY:
            if isinstance(item, (list, tuple)) and len(item) > 0 and isinstance(item[0], str):
                cat_priority_keys.add(item[0])
            elif isinstance(item, str):
                cat_priority_keys.add(item)
    valid_cats = set(CATEGORY_KEYWORDS.keys()) | cat_priority_keys
    if not valid_cats:
        raise RuntimeError("No valid categories could be derived from config")
    invalid = set(ELECTRONICS_ALLOWED_CATEGORIES) - valid_cats
    if invalid:
        raise ValueError(f"ELECTRONICS_ALLOWED_CATEGORIES contains invalid categories: {invalid}")


# FIX-37: Explicit public API.
# D5: PASSION_MIN_DEBIT_ROWS added — passion_pipeline.py now imports it at
# module top level; the dead try/except ImportError guard has been removed.
__all__ = [
    "validate_merchant_aliases", 
    "PASSION_INSIGHT_TEMPLATES", 
    "PASSION_MERCHANT_ALIASES",
    "GENERALIST_CANONICALS",
    "ELECTRONICS_ALLOWED_CATEGORIES",
    "MAX_SPIKE_CANDIDATES",
    "PIPELINE_BUDGET_MS",
    "PIPELINE_TOP_N",
    "PIPELINE_HARD_TIMEOUT_MS",
    "PASSION_MIN_MONTHS",
    "PASSION_MIN_DEBIT_ROWS",
    "PASSION_MERCHANT_COUNT_MIN",
    "PASSION_SPEND_SHARE_THRESHOLD",
    "PASSION_ANOMALY_SUPPRESSION_THRESHOLD",
    "DISTRESS_FEES_THRESHOLD",
    "MARKETPLACE_HIGH_AMOUNT_THRESHOLD",
    "MARKETPLACE_HIGH_CONFIDENCE",
    "MARKETPLACE_LOW_CONFIDENCE",
    "validate_config",
]
```

---

## 7a. `config.py` — TIP_CORPUS Schema Migration

> **FIX #7**: `config.py` must be updated to the new TIP_CORPUS schema.
> `contracts._freeze_tip_corpus` expects every entry to have `"text"`, `"categories"`, and `"insights"` keys.
> Non-generic tips must have non-empty categories and insights. Only `generic_*` tips may use empty tuples.

Update `config.py` to use the new schema. Replace the existing `TIP_CORPUS` dict:

```python
# config.py — Migrate TIP_CORPUS to new schema {text, categories, insights}
TIP_CORPUS = {
    # Non-generic tips: must have non-empty categories and insights.
    # No "any" wildcard allowed. All values lowercase stripped.
    "food_spike_tip": {
        "text": "Your {category} spending spiked by {pct}% at {merchant}.",
        "categories": ("food",),
        "insights": ("spending_spike",),
    },
    "subscription_tip": {
        "text": "{merchant} bills you {amount:.0f} every {frequency}.",
        "categories": ("entertainment", "utilities"),
        "insights": ("subscription",),
    },
    "lifestyle_tip": {
        "text": "Strong lifestyle signal in {category}: {merchant_count} merchants, {spend_share:.1%} of spend.",
        "categories": ("shopping", "travel", "fitness"),
        "insights": ("lifestyle_opportunity",),
    },
    # Generic tips: empty categories/insights = wildcard (matches any category/insight).
    # tip_id MUST start with 'generic_' to use empty-tuple wildcard behavior.
    "generic_budget": {
        "text": "Review this pattern before it becomes expensive.",
        "categories": (),
        "insights": (),
    },
}

# SPECIFIC_MERCHANT_ALIASES must include at minimum these entries
# so that GENERALIST_CANONICALS validation passes.
SPECIFIC_MERCHANT_ALIASES = {
    "amazon": "amazon",
    "amzn": "amazon",
    "amazon prime": "amazon",
    "flipkart": "flipkart",
    "meesho": "meesho",
    "snapdeal": "snapdeal",
}
```

**Rules enforced at startup by `contracts._freeze_tip_corpus` and `bootstrap._validate_tip_corpus`:**
- Non-generic tips MUST have non-empty `categories` and `insights`.
- Only `generic_*` tips may use `()` (empty tuple) for categories or insights.
- No non-generic tip may contain `"any"` in categories or insights.
- All category and insight strings must be lowercase and stripped.
- `"text"` is required for every tip.

---

## 7b. `insight_generator.py` — TIP_CORPUS Import Migration

> **A2 MANDATE FIX #7**: `insight_generator.py` must not import `TIP_CORPUS` from `config` directly.
> Only `contracts.py` may import from `config`. All consumers must use `from contracts import TIP_CORPUS`.

In `insight_generator.py`, replace:

```python
from config import TIP_CORPUS
```

With:

```python
from contracts import TIP_CORPUS, lookup_matching_tip_ids
from types import MappingProxyType
```

Then update every TIP_CORPUS value access from plain-string to new `{"text": ...}` schema. Old pattern:

```python
tip_text = TIP_CORPUS.get(tip_id, "")
```

New pattern:

```python
tip_data = TIP_CORPUS.get(tip_id, {})
tip_text = tip_data.get("text", "") if tip_data else ""
```

Or use `lookup_matching_tip_ids(category, insight_type)` to get matching tip IDs, then access `TIP_CORPUS[tip_id]["text"]` directly.

**Verification command (must return 0 matches after migration):**
```bash
grep -n "from config import.*TIP_CORPUS\|config\.TIP_CORPUS" insight_generator.py
```

---

## B5 — TIP_CORPUS Generic Prefix Tests

Add the following tests to `tests/test_passion_engine.py` (or a dedicated `TestTipCorpus` class):

```python
class TestTipCorpusGenericPrefix:
    """B5: Enforce generic_ prefix rule for empty-tuple wildcard tips."""

    def test_generic_tip_matches_all_categories(self):
        """A tip with id starting 'generic_' and empty categories matches any category."""
        from contracts import lookup_matching_tip_ids
        from unittest.mock import patch
        from types import MappingProxyType
        fake_corpus = MappingProxyType({
            "generic_budget_tip": MappingProxyType({
                "text": "Review your budget.",
                "categories": (),
                "insights": (),
            })
        })
        with patch("contracts.TIP_CORPUS", fake_corpus):
            result = lookup_matching_tip_ids("food", "spending_spike")
        assert "generic_budget_tip" in result

    def test_non_generic_empty_categories_raises_at_startup(self):
        """B5: A non-generic tip with empty categories must fail _freeze_tip_corpus."""
        from contracts import _freeze_tip_corpus
        bad = {
            "my_special_tip": {
                "text": "Some advice.",
                "categories": (),
                "insights": ("spending_spike",),
            }
        }
        with pytest.raises(ValueError, match="generic_"):
            _freeze_tip_corpus(bad)

    def test_non_generic_empty_insights_raises_at_startup(self):
        """B5: A non-generic tip with empty insights must fail _freeze_tip_corpus."""
        from contracts import _freeze_tip_corpus
        bad = {
            "my_category_tip": {
                "text": "Some advice.",
                "categories": ("food",),
                "insights": (),
            }
        }
        with pytest.raises(ValueError, match="generic_"):
            _freeze_tip_corpus(bad)

    def test_category_specific_tip_does_not_match_unrelated_category(self):
        """B5: A tip with explicit categories must not match an unrelated category."""
        from contracts import lookup_matching_tip_ids
        from unittest.mock import patch
        from types import MappingProxyType
        fake_corpus = MappingProxyType({
            "food_tip": MappingProxyType({
                "text": "Food spending insight.",
                "categories": ("food",),
                "insights": ("spending_spike",),
            })
        })
        with patch("contracts.TIP_CORPUS", fake_corpus):
            result = lookup_matching_tip_ids("shopping", "spending_spike")
        assert "food_tip" not in result
```

---

## Migration Guide & CHANGELOG

**HIGH-14 | Token Length Change**: 
The PII masking utility has been changed from the insecure `stable_hash` to the HMAC-based `log_safe_merchant`. 
*   **Previous Behavior**: `stable_hash` produced tokens of length **12** (or other short fixed lengths depending on implementation).
*   **New Behavior**: `log_safe_merchant` produces tokens starting with the prefix `"merchant:"` followed by a **32-character** hex string (total length **41** characters).
*   **Action Required**: Any downstream fixed-width log parsers, database schemas with short string limits for merchant tokens, or exact-match tests expecting the old 12-character format MUST be updated to accommodate the new 41-character format.

**CONFIG-14 | TIP_CORPUS Schema Migration**:
`TIP_CORPUS` in `config.py` must be explicitly updated to the new schema:
`{"text": str, "categories": list[str], "insights": list[str]}`.
*   **Behavior Check**: Generic tips may keep empty `categories` or empty `insights` (which matches any category/insight respectively). Do not reject empty categories globally, as existing generic tips depend on this behavior.
*   **Validation Requirement**: Add validation during startup that ensures `text`, `categories`, and `insights` keys are present for every tip, raising a clean `ValueError` if any are missing.

**SECURITY-15 | Finish stable_hash Migration Audit**:
To prove all `stable_hash` callers are gone in production, run:
```bash
grep -R "stable_hash" -n . --exclude-dir=venv --exclude-dir=.git
grep -R "from hash_utils import" -n . --exclude-dir=venv --exclude-dir=.git
```
Migrate all production PII log call sites to `log_safe_merchant` or `log_safe_text`. Do not allow `recurring_detector.py` to import `hash_utils`.

**SECURITY-16 | Codebase-wide Logging Audit for PII**:
Audit logs for PII exposure:
```bash
grep -R "logger\." -n . --exclude-dir=venv --exclude-dir=.git
grep -R "cleaned_remarks\|merchant\|remarks\|description" -n *.py tests
```
Any log containing raw merchant/remarks must use `log_safe_merchant`, `log_safe_text`, or aggregate counts.

