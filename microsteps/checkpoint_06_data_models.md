# CHECKPOINT 06: Data Models (PassionSignal, PassionResult)

Directly modified:   passion_models.py, pipeline_result.py, pipeline.py
Indirectly affected: Pipeline execution
Code blocks used:    CB-P3-01, CB-P2-02, CB-P2-01
Risk:                MEDIUM
Depends on:          CHECKPOINT 05

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
Implementing the core data structures for carrying passion analysis results and extending the main `PipelineResult`.

PRE-CONDITIONS
[ ] Checkpoint 05 passed.

STEPS

  STEP [6.1]
  File:           passion_models.py
  Action:         CREATE
  Source file:    passion_plan_part3.md
  Source section: 11. passion_models.py
  Block ID:       CB-P3-01
  Flags:          NONE

  Before:
  ```
  FILE DOES NOT EXIST
  ```

  Instruction: Create `passion_models.py` with verbatim content.

  After:
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

  Rollback: Delete passion_models.py.

  STEP [6.2]
  File:           pipeline_result.py
  Action:         CREATE
  Source file:    passion_plan_part2.md
  Source section: 10. pipeline_result.py
  Block ID:       CB-P2-02
  Flags:          NONE

  Before:
  ```
  FILE DOES NOT EXIST
  ```

  Instruction: Create `pipeline_result.py` with verbatim content.

  After:
  ```python
from __future__ import annotations
from dataclasses import dataclass
from typing import Any, TYPE_CHECKING
import pandas as pd

if TYPE_CHECKING:
    from candidate import Candidate
    from passion_models import PassionSignal

# B1: PassionSignal is NOT imported at module top-level.
# pipeline_result.py is imported by core pipeline paths. A top-level sidecar
# import would make the optional Passion Engine mandatory for the entire core.
# TYPE_CHECKING guard above satisfies type checkers; runtime uses lazy import
# inside __post_init__ only.

__all__ = ["PassionResult"]


# M2: PassionResult frozen=True, kw_only=True, NO slots=True.
# Remove personal_insights and personal_summary (core owns these).
@dataclass(frozen=True, kw_only=True)
class PassionResult:
    """
    Immutable container for passion pipeline output.

    DataFrames are defensive copies.
    Call .copy(deep=True) before mutating.
    """
    debits: pd.DataFrame
    candidates: tuple[Candidate, ...]
    insights: tuple[str, ...]
    passion_signals: tuple[PassionSignal, ...]

    def __post_init__(self) -> None:
        from candidate import Candidate as _Candidate

        # B1: Lazy import — imported here (not at module level) so that importing
        # pipeline_result.py does not force passion_models to load on the core path.
        from passion_models import PassionSignal as _PassionSignal

        if not isinstance(self.debits, pd.DataFrame):
            raise TypeError(f"debits must be pd.DataFrame, got {type(self.debits)}")
        object.__setattr__(self, "debits", self.debits.copy(deep=True))

        field_element_types = {
            'candidates': _Candidate,
            'insights': str,
            'passion_signals': _PassionSignal,
        }

        for field_name, elem_type in field_element_types.items():
            val = getattr(self, field_name)
            # M3: Reject str, dict, set, frozenset
            if isinstance(val, (str, dict, set, frozenset)):
                raise TypeError(f"'{field_name}' must be tuple or list, not {type(val).__name__}")
            if not hasattr(val, '__iter__'):
                raise TypeError(f"'{field_name}' must be iterable, got {type(val)}")

            # M3: Convert list to tuple
            if not isinstance(val, tuple):
                val = tuple(val)
                object.__setattr__(self, field_name, val)

            if val and not all(isinstance(x, elem_type) for x in val):
                bad = [type(x).__name__ for x in val if not isinstance(x, elem_type)][:3]
                raise TypeError(
                    f"'{field_name}' elements must be {elem_type.__name__}, got {bad}"
                )
  ```

  Rollback: Delete pipeline_result.py.

  STEP [6.3]
  File:           pipeline.py
  Action:         MODIFY
  Target:         PipelineResult dataclass
  Source file:    passion_plan_part2.md
  Source section: 9. pipeline.py PipelineResult Extension
  Block ID:       CB-P2-01
  Flags:          [INTERFACE BREAK RISK]

  Before:
  ```python
@dataclass(frozen=True)
class PipelineResult:
    debits: pd.DataFrame
    credits: pd.DataFrame
    # ... existing fields ...
    personal_debits: pd.DataFrame = field(default_factory=pd.DataFrame)
    personal_credits: pd.DataFrame = field(default_factory=pd.DataFrame)
    personal_summary: dict = field(default_factory=dict)

    def __post_init__(self):
        # ... existing validation ...
  ```

  Instruction: Update `PipelineResult` verbatim. Add `kw_only=True` to the decorator. Add the 4 new fields (`stats`, `passion_debits`, `passion_insights`, `passion_signals`) at the end of the class fields. Update `__post_init__` to include defensive copy and type validation logic for both existing and new fields.

  After:
  ```python
import pandas as pd
from dataclasses import dataclass, field
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from passion_models import PassionSignal

# M1: Keep frozen=True.
# With all call sites audited and converted, enforce kw_only=True.
# Do not add slots=True.
@dataclass(frozen=True, kw_only=True)
class PipelineResult:
    # ... other existing fields unchanged ...

    # B6: Do not change personal_summary default unless audited.
    # Keep personal_debits unchanged.

    # B7: Add 4 new fields with defaults at the end only after B5 audit.
    stats: dict = field(default_factory=dict)
    passion_debits: pd.DataFrame = field(default_factory=pd.DataFrame)
    passion_insights: tuple = field(default=())
    passion_signals: tuple = field(default=())

    def __post_init__(self):
        # Merge changes into existing __post_init__ if one exists.
        # OPEN THE LIVE EXISTING PipelineResult.__post_init__ and PRESERVE EVERY EXISTING VALIDATION EXACTLY unless a test proves it must change.
        # APPEND only the new passion-field logic after existing validation.
        # Do not replace the method with this template version.

        # Existing __post_init__ logic goes here...
        # ...

        # New defensive copy logic:
        import pandas as pd
        if hasattr(self, "debits") and isinstance(self.debits, pd.DataFrame):
            object.__setattr__(self, "debits", self.debits.copy(deep=True))
        if hasattr(self, "credits") and isinstance(self.credits, pd.DataFrame):
            object.__setattr__(self, "credits", self.credits.copy(deep=True))
        if hasattr(self, "personal_debits") and isinstance(self.personal_debits, pd.DataFrame):
            object.__setattr__(self, "personal_debits", self.personal_debits.copy(deep=True))
        if hasattr(self, "passion_debits") and isinstance(self.passion_debits, pd.DataFrame):
            object.__setattr__(self, "passion_debits", self.passion_debits.copy(deep=True))
        if hasattr(self, "stats") and isinstance(self.stats, dict):
            # FIX-5: Validate that all stats dictionary keys and values are scalars to prevent deep nested mutation.
            import numpy as np
            _allowed_scalar_types = (str, int, float, bool, bytes, type(None), np.generic)
            for k, v in self.stats.items():
                if not isinstance(k, _allowed_scalar_types):
                    raise TypeError(f"stats key must be scalar, got {type(k).__name__}")
                if not isinstance(v, _allowed_scalar_types):
                    raise TypeError(f"stats value for key '{k}' must be scalar, got {type(v).__name__}")
            object.__setattr__(self, "stats", dict(self.stats))

        # P2-1: Passion field normalization and type validation
        object.__setattr__(self, "passion_insights",
            tuple(self.passion_insights) if self.passion_insights is not None else ()
        )
        object.__setattr__(self, "passion_signals",
            tuple(self.passion_signals) if self.passion_signals is not None else ()
        )

        # B1: Do NOT import passion_models unconditionally here.
        # An unconditional 'from passion_models import PassionSignal' makes the
        # optional Passion sidecar structurally mandatory for every core pipeline
        # import. Use duck-typing instead: check for required PassionSignal
        # attributes only when passion_signals is non-empty.
        for s in self.passion_signals:
            if not (
                hasattr(s, 'category') and
                hasattr(s, 'spend_share') and
                hasattr(s, 'is_suppressed') and
                hasattr(s, 'merchant_list')
            ):
                raise TypeError(
                    f"passion_signals elements must be PassionSignal-like "
                    f"(have category, spend_share, is_suppressed, merchant_list), "
                    f"got {type(s)}"
                )
        for t in self.passion_insights:
            if not isinstance(t, str):
                raise TypeError(f"passion_insights must contain str, got {type(t)}")
  ```

  Rollback: Restore original `PipelineResult` class.

POST-EXECUTION VALIDATION
[ ] `passion_models.py` exists.
[ ] `pipeline_result.py` exists.
[ ] `PipelineResult` in `pipeline.py` has `kw_only=True`.
[ ] `python3 -m py_compile passion_models.py pipeline_result.py pipeline.py` succeeds.
[ ] `python3 -c "from pipeline_result import PassionResult; import pandas as pd; r = PassionResult(debits=pd.DataFrame(), candidates=(), insights=(), passion_signals=()); assert r.debits.empty"` succeeds.

GO / NO-GO
All checks pass → proceed to CHECKPOINT [07]
