# Passion Detection Engine — Master Plan (Part 3 of 8)

**passion_models.py, passion_utils.py, candidate.py, marketplace_subcategory.py.**

## 11. `passion_models.py`

```python
from __future__ import annotations
import math
from dataclasses import dataclass
from typing import Final
import numpy as np

__all__ = ["PassionSignal"]

# Fix 11: Module-level epsilon for float comparison tolerance.
_EPS = 1e-9

# Blocker 2 & 13: Both anomaly and distress map to "suppressed".
_VALID_TRENDS: frozenset[str] = frozenset({
    "non_declining", "declining", "insufficient_history", "suppressed"
})

# H1: frozen=True, kw_only=True, NO slots=True
@dataclass(frozen=True, kw_only=True)
class PassionSignal:
    category: str
    merchant_list: tuple[str, ...]
    total_spend: float
    merchant_count: int
    spend_share: float
    trend_direction: str
    is_suppressed: bool = False
    suppression_reason: str = ""
    latest_ts: int = 0
    original_index: int = 0
    active_months: int = 0

    def __post_init__(self) -> None:
        if not isinstance(self.category, str) or not self.category.strip():
            raise ValueError("category must be a non-empty string")

        if isinstance(self.merchant_list, str):
            raise TypeError("merchant_list must be a tuple/list of str, not a bare string")

        if not isinstance(self.merchant_list, (tuple, list)):
            raise TypeError("merchant_list must be a tuple/list of str")

        object.__setattr__(self, "merchant_list", tuple(self.merchant_list))

        for m in self.merchant_list:
            if not isinstance(m, str):
                raise TypeError(f"merchant_list elements must be str, got {type(m)}")
            if not m.strip():
                raise ValueError("merchant_list elements must be non-empty str")
        if not isinstance(self.total_spend, (int, float, np.integer, np.floating)):
            raise TypeError(f"total_spend must be numeric, got {type(self.total_spend)}")
        if math.isnan(float(self.total_spend)) or math.isinf(float(self.total_spend)):
            raise ValueError("total_spend must be finite")
        # H2: raise ValueError without clamping for total_spend < 0
        if float(self.total_spend) < 0:
            raise ValueError("total_spend must be non-negative")
        if not isinstance(self.merchant_count, (int, np.integer)):
            raise TypeError(f"merchant_count must be int")
        # H2: raise ValueError without clamping for merchant_count != len(merchant_list)
        if int(self.merchant_count) != len(self.merchant_list):
            raise ValueError(f"merchant_count ({self.merchant_count}) != len(merchant_list) ({len(self.merchant_list)})")
        if not isinstance(self.spend_share, (int, float, np.integer, np.floating)):
            raise TypeError("spend_share must be numeric")

        # H2: raise ValueError without clamping for spend_share not in [0, 1]
        if not (-_EPS <= float(self.spend_share) <= 1.0 + _EPS):
            raise ValueError(f"spend_share must be in [0, 1], got {self.spend_share}")
        # NOTE: Do NOT clamp to clean value, raise ValueError if it's truly out of range.
        
        # H2: trend_direction not in allowed set
        if self.trend_direction not in _VALID_TRENDS:
            raise ValueError(f"trend_direction must be one of {sorted(_VALID_TRENDS)}, got {self.trend_direction!r}")
        if not isinstance(self.is_suppressed, bool):
            raise TypeError("is_suppressed must be bool")
        if not isinstance(self.suppression_reason, str):
            raise TypeError("suppression_reason must be str")
        
        # H2: is_suppressed True with empty suppression_reason
        if self.is_suppressed and not self.suppression_reason:
            raise ValueError("suppressed signals must have a non-empty suppression_reason")
        if not self.is_suppressed and self.suppression_reason:
            raise ValueError("non-suppressed signals must have empty suppression_reason")
            
        if not isinstance(self.latest_ts, (int, np.integer)) or self.latest_ts < 0:
            raise ValueError("latest_ts must be a non-negative integer")
        if not isinstance(self.original_index, (int, np.integer)) or self.original_index < 0:
            raise ValueError("original_index must be a non-negative integer")

        # P0-2: active_months validation
        if not isinstance(self.active_months, (int, np.integer)) or self.active_months < 0:
            raise ValueError("active_months must be a non-negative integer")

        # Fix 11: Normalize scalar fields to native Python types after all validation.
        # Prevents numpy scalars from leaking into logs, JSON, and downstream consumers.
        object.__setattr__(self, "total_spend", float(self.total_spend))
        object.__setattr__(self, "merchant_count", int(self.merchant_count))
        object.__setattr__(self, "spend_share", float(self.spend_share))
        object.__setattr__(self, "latest_ts", int(self.latest_ts))
        object.__setattr__(self, "original_index", int(self.original_index))
        object.__setattr__(self, "active_months", int(self.active_months))
        object.__setattr__(self, "is_suppressed", bool(self.is_suppressed))
        object.__setattr__(self, "category", self.category.strip().lower())
        object.__setattr__(self, "suppression_reason", str(self.suppression_reason).strip())
```

---

## 12. `passion_utils.py`

```python
"""
passion_utils.py — Shared utility functions for the Passion Detection Engine.

NOTE: assert_columns_exist raises ValueError (not KeyError) for missing columns.
This matches schema.require_columns(). All callers should catch ValueError.
"""
import re as _re
import math
import pandas as pd
import numpy as np
from decimal import Decimal
from typing import Any

# FIX H11: Use structured logger from logger_factory instead of plain logging.
from logger_factory import get_logger

__all__ = [
    "to_bool_strict", "coerce_bool_column", "sanitize_mask",
    "safe_last_nonnull", "validate_template_values", "_safe_isna",
    "assert_columns_exist", "safe_numeric",
]

logger = get_logger(__name__)


def assert_columns_exist(df: pd.DataFrame, columns: list[str], context: str) -> None:
    """
    Validates that a DataFrame contains all required columns.

    Raises:
        ValueError: If any required columns are missing.
    """
    if missing := [c for c in columns if c not in df.columns]:
        raise ValueError(f"Missing columns in {context}: {missing}")


# FIX H7: _safe_isna array truth-value crash.
# pd.isna on an array returns an array. bool() on a multi-element array raises ValueError.
def _safe_isna(val: Any) -> bool:
    try:
        result = pd.isna(val)
        if isinstance(result, (np.ndarray, pd.Series)):
            return bool(np.all(result))
        return bool(result)
    except (TypeError, ValueError):
        return False


# FIX M10 + C2: Refactored to_bool_strict with explicit np.bool_ guard.
# C2: In NumPy 2.x, np.bool_ is NO LONGER a subclass of Python bool.
# The match/case bool(): pattern therefore does not catch np.bool_ values,
# causing them to fall through to the np.generic() case, which works but is
# less explicit. Add an isinstance guard BEFORE match/case for safety across
# all NumPy versions.
def to_bool_strict(val):
    if _safe_isna(val):
        return False
    # C2: Explicit guard — must precede match/case because np.bool_ is no longer
    # a subclass of bool in NumPy 2.x.
    if isinstance(val, (bool, np.bool_)):
        return bool(val)
    match val:
        case int() if val in (0, 1):
            return bool(val)
        case float() if val in (0.0, 1.0):
            return bool(int(val))
        case np.generic():
            num = val.item()
            if num in (0, 1):
                return bool(num)
            raise ValueError(f"Cannot coerce numpy scalar {num!r} to bool")
        case _:
            raise TypeError(f"Cannot coerce {type(val).__name__} to bool")


def coerce_bool_column(col: pd.Series) -> pd.Series:
    if col.dtype == bool:
        return col
    if isinstance(col.dtype, pd.BooleanDtype):
        return col.fillna(False).astype(bool)
    if pd.api.types.is_numeric_dtype(col):
        unique_vals = set(col.dropna().unique()) - {0, 1, 0.0, 1.0}
        if unique_vals:
            raise ValueError(
                f"Numeric column contains non-boolean values: {sorted(str(v) for v in unique_vals)}"
            )
        return col.fillna(0).eq(1).astype(bool)
    if pd.api.types.is_string_dtype(col):
        raise TypeError("String column cannot be coerced to bool")
    if col.dtype == object:
        if col.dropna().apply(lambda v: isinstance(v, str)).any():
            raise TypeError("Object column contains unconvertible string values")
        try:
            return col.map(to_bool_strict).astype(bool)
        except TypeError as e:
            raise TypeError(f"Object column contains unconvertible values: {e}") from e
    return col.map(to_bool_strict).astype(bool)


# FIX L1: sanitize_mask now logs when rows are silently added/dropped
def sanitize_mask(
    mask: Any, target_index: pd.Index, context: str
) -> pd.Series:
    if not target_index.is_unique:
        raise ValueError(f"Non-unique DataFrame index in {context}")
    if isinstance(mask, pd.Series):
        if hasattr(mask, 'cat'):
            mask = mask.astype(mask.cat.categories.dtype)
        if mask.dtype == "string" or (mask.dtype == object and mask.dropna().apply(lambda v: isinstance(v, str)).any()):
            raise TypeError(f"Mask contains string values in {context}")
        if not mask.index.is_unique:
            raise ValueError(f"Mask has non-unique index in {context}")
        if len(mask) != len(target_index):
            raise ValueError(
                f"Mask length mismatch in {context}: "
                f"mask={len(mask)}, target={len(target_index)}"
            )

        dropped = mask.index.difference(target_index)
        added = target_index.difference(mask.index)
        if len(dropped) or len(added):
            logger.debug(
                "sanitize_mask index mismatch in %s: dropped=%d, added=%d (filled False)",
                context, len(dropped), len(added),
            )

        return coerce_bool_column(mask.reindex(target_index, fill_value=False))
    elif isinstance(mask, (list, np.ndarray)):
        if len(mask) != len(target_index):
            raise ValueError(
                f"Mask length mismatch in {context}: "
                f"mask={len(mask)}, target={len(target_index)}"
            )
        return coerce_bool_column(pd.Series(mask, index=target_index, dtype=object))
    else:
        raise TypeError(f"Unsupported mask type in {context}: {type(mask)}")


def safe_last_nonnull(seq: list) -> str | None:
    for val in reversed(seq):
        if _safe_isna(val):
            continue
        try:
            return str(val)
        except (TypeError, ValueError):
            continue
    return None


_ALLOWED_TEMPLATE_SCALAR_TYPES = (str, int, float, bool, Decimal, np.integer, np.floating)

def validate_template_values(values: dict[str, Any]) -> None:
    for k, v in values.items():
        if not isinstance(v, _ALLOWED_TEMPLATE_SCALAR_TYPES):
            raise TypeError(
                f"Template value '{k}' must be a scalar (str/int/float/bool/Decimal), "
                f"got {type(v).__name__}."
            )


_CURRENCY_RE = _re.compile(r'[$\€\£\¥\u20A0-\u20CF\u20B9]')
_INR_RE = _re.compile(r'^(rs\.?\s*|inr\s+)', _re.IGNORECASE)

# FIX H5: Allow custom default and raise_on_invalid. 
# Previously returned 0.0 unconditionally, masking parse errors as zero-spend.
def safe_numeric(val: Any, default: float = 0.0, *, raise_on_invalid: bool = False) -> float:
    """Convert a value to float, stripping currency symbols, INR/Rs prefixes, and commas."""
    if _safe_isna(val):
        return default
    try:
        if isinstance(val, str):
            val = _INR_RE.sub("", val)
            val = _CURRENCY_RE.sub("", val).replace(",", "").strip()
        f = float(val)
    except (ValueError, TypeError):
        if raise_on_invalid:
            raise ValueError(f"Cannot parse numeric value: {val!r}")
        return default
    return default if (math.isinf(f) or math.isnan(f)) else f
```

---

## 13. `candidate.py`

```python
from __future__ import annotations
import math
from dataclasses import dataclass
from typing import Any
import numpy as np

__all__ = ["Candidate"]

def _coerce_finite_float(val: Any, name: str) -> float:
    # 1. Unpack numpy generic scalar types
    if isinstance(val, np.generic):
        val = val.item()
    # 2. Reject boolean types explicitly (since bool inherits from int)
    if isinstance(val, bool):
        raise TypeError(f"{name} must be numeric, got bool")
    # 3. Check for standard numeric types
    if not isinstance(val, (int, float)):
        raise TypeError(f"{name} must be numeric, got {type(val).__name__}")
    # 4. Check for NaN/Inf
    try:
        coerced = float(val)
    except (TypeError, ValueError) as e:
        raise TypeError(f"{name} must be numeric, got {type(val).__name__}") from e
    if math.isnan(coerced) or math.isinf(coerced):
        raise ValueError(f"{name} must be finite")
    return coerced

@dataclass(frozen=True, slots=True)
class Candidate:
    score: float
    category: str
    insight_type: str
    merchant: str
    amount: float
    sort_key_ts: int
    
    # H4, H5: Preserve signal metadata and add normalized_score
    merchant_count: int = 1
    spend_share: float = 0.0
    total_spend: float = 0.0
    trend_direction: str = ""
    suppression_reason: str = ""
    normalized_score: float = 0.0
    original_index: int = 0

    def __post_init__(self) -> None:
        object.__setattr__(self, 'score', _coerce_finite_float(self.score, 'score'))
        object.__setattr__(self, 'amount', _coerce_finite_float(self.amount, 'amount'))
        object.__setattr__(self, 'normalized_score', _coerce_finite_float(self.normalized_score, 'normalized_score'))

        # Standard integer validation for sort_key_ts
        val_ts = self.sort_key_ts
        if isinstance(val_ts, np.generic):
            val_ts = val_ts.item()
        if isinstance(val_ts, bool) or not isinstance(val_ts, int):
            raise TypeError(f"sort_key_ts must be int, got {type(val_ts).__name__}")
        if val_ts < 0:
            raise ValueError("sort_key_ts must be a non-negative integer")
        object.__setattr__(self, 'sort_key_ts', val_ts)
        
        # Standard integer validation for original_index
        val_idx = self.original_index
        if isinstance(val_idx, np.generic):
            val_idx = val_idx.item()
        if isinstance(val_idx, bool) or not isinstance(val_idx, int) or val_idx < 0:
            raise ValueError("original_index must be a non-negative integer")
        object.__setattr__(self, 'original_index', val_idx)

        if not isinstance(self.category, str) or not self.category.strip():
            raise ValueError("category must be non-empty str")
        if not isinstance(self.merchant, str) or not self.merchant.strip():
            raise ValueError("merchant must be non-empty str")
        if not isinstance(self.insight_type, str) or not self.insight_type.strip():
            raise ValueError("insight_type must be non-empty str")

    # H6: Candidate sort order must be deterministic
    def __lt__(self, other: "Candidate") -> bool:
        if not isinstance(other, Candidate): return NotImplemented
        
        type_priority = {"subscription": 0, "spending_spike": 1, "lifestyle_opportunity": 2}
        p1 = type_priority.get(self.insight_type, 99)
        p2 = type_priority.get(other.insight_type, 99)
        
        SCORE_TOLERANCE = 1e-12
        if abs(self.normalized_score - other.normalized_score) > SCORE_TOLERANCE:
            return self.normalized_score > other.normalized_score
        if p1 != p2:
            return p1 < p2
        if self.category != other.category:
            return self.category < other.category
        if self.merchant != other.merchant:
            return self.merchant < other.merchant
        if self.sort_key_ts != other.sort_key_ts:
            return self.sort_key_ts > other.sort_key_ts
        return self.original_index < other.original_index

    # H4: Candidate.passion(signal) classmethod
    @classmethod
    def passion(
        cls,
        signal,
        sort_key_ts: int | None = None,
        normalized_score: float | None = None,
        original_index: int | None = None,
    ) -> "Candidate":
        from passion_models import PassionSignal

        if not isinstance(signal, PassionSignal):
            raise TypeError("Must provide a PassionSignal")

        if sort_key_ts is None:
            sort_key_ts = signal.latest_ts
        if normalized_score is None:
            normalized_score = signal.spend_share
        if original_index is None:
            original_index = signal.original_index

        return cls(
            score=0.0,
            category=signal.category,
            insight_type="lifestyle_opportunity",
            merchant=signal.merchant_list[0] if signal.merchant_list else "unknown",
            amount=signal.total_spend,
            sort_key_ts=sort_key_ts,
            merchant_count=signal.merchant_count,
            spend_share=signal.spend_share,
            total_spend=signal.total_spend,
            trend_direction=signal.trend_direction,
            suppression_reason=signal.suppression_reason,
            normalized_score=normalized_score,
            original_index=original_index,
        )

    @classmethod
    def subscription(cls, score: float, category: str, merchant: str,
                     amount: float, sort_key_ts: int, normalized_score: float = 0.0, original_index: int = 0) -> "Candidate":
        return cls(
            score=score,
            category=category,
            insight_type="subscription",
            merchant=merchant,
            amount=amount,
            sort_key_ts=sort_key_ts,
            normalized_score=normalized_score,
            original_index=original_index,
        )

    @classmethod
    def spending_spike(cls, score: float, category: str, merchant: str,
                       amount: float, sort_key_ts: int, normalized_score: float = 0.0, original_index: int = 0) -> "Candidate":
        return cls(
            score=score,
            category=category,
            insight_type="spending_spike",
            merchant=merchant,
            amount=amount,
            sort_key_ts=sort_key_ts,
            normalized_score=normalized_score,
            original_index=original_index,
        )
```

---

## 14. `marketplace_subcategory.py`

```python
import re
import pandas as pd
import numpy as np
from collections import defaultdict
from schema import Col
from config_passion import (
    PASSION_MERCHANT_ALIASES,
    GENERALIST_CANONICALS,
    MARKETPLACE_HIGH_AMOUNT_THRESHOLD,
    MARKETPLACE_HIGH_CONFIDENCE,
    MARKETPLACE_LOW_CONFIDENCE,
    ELECTRONICS_ALLOWED_CATEGORIES,
)
from passion_utils import assert_columns_exist, safe_numeric, coerce_bool_column
from logger_factory import get_logger

__all__ = ["resolve_merchant_vectorized", "enrich_subcategories"]

logger = get_logger(__name__)

# P3-2: Derive categories directly from config_passion.py to remove DEBIT_CATEGORIES assumption

from types import MappingProxyType

# Fix #9: Build canonical lookup from PASSION_MERCHANT_ALIASES.
# resolve_merchant_vectorized returns the canonical name DIRECTLY (not a rewritten phrase).
# Use (?<!\w)/(?!\w) lookaround boundaries — \b breaks on special-char adjacent inputs like |amzn|.
_ALIAS_TO_CANONICAL: dict[str, str] = {
    str(alias).strip().lower(): str(canonical).strip().lower()
    for alias, canonical in PASSION_MERCHANT_ALIASES.items()
}
_ALIAS_PATTERN = re.compile(
    r"(?<!\w)("
    + "|".join(re.escape(a) for a in sorted(_ALIAS_TO_CANONICAL, key=len, reverse=True))
    + r")(?!\w)",
    re.IGNORECASE,
) if _ALIAS_TO_CANONICAL else None


def resolve_merchant_vectorized(series: pd.Series) -> pd.Series:
    """Map merchant descriptions to canonical names.

    Fix #9: Returns the canonical name directly — not a rewritten phrase with the alias
    substituted in context. If no alias matches, returns the normalised lowercased input.
    NaN / empty inputs return empty string "".
    """
    if not isinstance(series, pd.Series):
        raise TypeError(
            f"resolve_merchant_vectorized expects pd.Series, got {type(series).__name__}"
        )

    def _resolve(value: object) -> str:
        if pd.isna(value):
            return ""
        text = re.sub(r"\s+", " ", str(value).lower()).strip()
        if not text:
            return ""
        # Exact match first (O(1), cheapest path)
        canonical = _ALIAS_TO_CANONICAL.get(text)
        if canonical is not None:
            return canonical
        # Longest-match regex path
        if _ALIAS_PATTERN:
            m = _ALIAS_PATTERN.search(text)
            if m:
                return _ALIAS_TO_CANONICAL[m.group(1).lower()]
        return text

    return series.map(_resolve)


def enrich_subcategories(df: pd.DataFrame) -> pd.DataFrame:
    assert_columns_exist(df, [Col.AMOUNT, Col.CLEANED_REMARKS, Col.PREDICTED_CATEGORY], "enrich_subcategories")
    result = df.copy()
    
    # G1: Output adds ONLY INFERRED_SUBCATEGORY and SUBCATEGORY_CONFIDENCE.
    # G4: Unknown classification gets 0.0.
    result[Col.INFERRED_SUBCATEGORY] = pd.Series(
        [pd.NA] * len(result),
        index=result.index,
        dtype="object",
    )
    result[Col.SUBCATEGORY_CONFIDENCE] = pd.Series(
        0.0,
        index=result.index,
        dtype="float64",
    )

    # G2: Skip known-person rows
    if Col.IS_KNOWN_PERSON in result.columns:
        kp_mask = coerce_bool_column(result[Col.IS_KNOWN_PERSON].fillna(False))
    else:
        kp_mask = pd.Series(False, index=result.index)

    # G3: Invalid or unparseable amount rows get NA/0.0
    amounts_raw = result[Col.AMOUNT].replace([np.inf, -np.inf], np.nan)
    numeric_amounts = amounts_raw.map(
        lambda v: safe_numeric(v, default=np.nan)
    ).astype(float)
    valid_amount_mask = numeric_amounts.notna()

    eligible_mask = ~kp_mask & valid_amount_mask
    
    # Process only eligible rows
    if not eligible_mask.any():
        return result

    # P3-2: Resolve merchants first via resolve_merchant_vectorized, then
    # check resolved canonical against GENERALIST_CANONICALS.
    resolved = resolve_merchant_vectorized(result.loc[eligible_mask, Col.CLEANED_REMARKS])
    is_generalist = resolved.isin(GENERALIST_CANONICALS)

    if not is_generalist.any():
        return result

    # P3-2: Category-aware enrichment. Check PREDICTED_CATEGORY against
    # allowed set before assigning electronics subcategory.
    eligible_amounts = numeric_amounts[eligible_mask]

    for idx in is_generalist[is_generalist].index:
        canonical = resolved[idx]
        if canonical not in GENERALIST_CANONICALS:
            continue
        # P3-2: Gate on PREDICTED_CATEGORY
        pred_cat = ""
        if Col.PREDICTED_CATEGORY in result.columns:
            raw_cat = result.at[idx, Col.PREDICTED_CATEGORY]
            pred_cat = str(raw_cat).strip().lower() if pd.notna(raw_cat) and str(raw_cat).strip() else ""
        
        amount = eligible_amounts.get(idx, 0.0)
        if pred_cat in ELECTRONICS_ALLOWED_CATEGORIES and amount >= MARKETPLACE_HIGH_AMOUNT_THRESHOLD:
            result.at[idx, Col.INFERRED_SUBCATEGORY] = "electronics"
            result.at[idx, Col.SUBCATEGORY_CONFIDENCE] = MARKETPLACE_HIGH_CONFIDENCE
        else:
            result.at[idx, Col.INFERRED_SUBCATEGORY] = "general_purchase"
            result.at[idx, Col.SUBCATEGORY_CONFIDENCE] = MARKETPLACE_LOW_CONFIDENCE

    # Blocker 5: Prevent invalid/known rows from receiving subcategory enrichment.
    ineligible_idx = result[~eligible_mask].index
    if not ineligible_idx.empty:
        result.loc[ineligible_idx, Col.INFERRED_SUBCATEGORY] = pd.NA
        result.loc[ineligible_idx, Col.SUBCATEGORY_CONFIDENCE] = 0.0

    return result
```

---

## Migration Guide & CHANGELOG

**HIGH-14 | Token Length Change**: 
The PII masking utility has been changed from the insecure `stable_hash` to the HMAC-based `log_safe_merchant`. 
*   **Previous Behavior**: `stable_hash` produced tokens of length **12** (or other short fixed lengths depending on implementation).
*   **New Behavior**: `log_safe_merchant` produces tokens starting with the prefix `"merchant:"` followed by a **32-character** hex string (total length **41** characters).
*   **Action Required**: Any downstream fixed-width log parsers, database schemas with short string limits for merchant tokens, or exact-match tests expecting the old 12-character format MUST be updated to accommodate the new 41-character format.
