# Passion Detection Engine — Master Plan (Part 2 of 8)

**pipeline_result.py — PassionResult. pipeline.py PipelineResult extension.**

## 8. PipelineResult Migration Appendix (PRE-REQUISITE)

Before modifying the existing `PipelineResult` dataclass in `pipeline.py` to add passion fields, you MUST perform a full migration of all existing constructor calls. Do not create or move `PipelineResult` into `pipeline_result.py` in this patch.

**Required Grep Commands:**
```bash
grep -rn "PipelineResult(" --include="*.py"
grep -rn "dataclasses.astuple" --include="*.py"
grep -rn "astuple(" --include="*.py"
grep -rn "asdict(" --include="*.py"
```

**Migration Steps:**
1. Identify all `PipelineResult(...)` instantiations across the codebase.
2. Convert any positional arguments to keyword arguments (`kw_only=True` will break positional arguments).
3. Identify any uses of `astuple(result)` or `asdict(result)` which might be disrupted by the addition of new fields and update them to rely on specific field access or filter out passion fields.
4. Only after all call sites are migrated should the 3 new passion fields be added to the dataclass.

> **A1 HARD REQUIREMENT**: You MUST grep every file listed below before touching the dataclass. Do not skip any file even if you believe it has no PipelineResult calls. Record every hit explicitly in the audit table.
> ```bash
> grep -rn "PipelineResult(" --include="*.py" .
> grep -rn "astuple(" --include="*.py" .
> grep -rn "asdict(" --include="*.py" .
> ```
> Mandatory audit scope: `pipeline.py`, `tutorial_real_data.py`, `train_and_save_models.py`, `model_benchmark.py`, `summary_utils.py`, `run_smoke.py`, `run_stress_legacy.py`, `run_stress_heavy.py`, and **all files under `tests/`**. Convert every positional constructor call to keyword args. Do not add `kw_only=True` to the dataclass until every call site is confirmed keyword-only. Keep `personal_summary` required if it is required today — do not change its default as part of this migration.

### PipelineResult Concrete Audit Table

**Constructor Audit (`grep -rn "PipelineResult(" --include="*.py"`)**
Audit ALL of the following files — mark "0 hits" explicitly if none found:
`pipeline.py`, `tutorial_real_data.py`, `train_and_save_models.py`, `model_benchmark.py`, `summary_utils.py`, `run_smoke.py`, `run_stress_legacy.py`, `run_stress_heavy.py`, and all files under `tests/`.

| File | Line | Status | Action | Done |
|------|------|--------|--------|------|
| `tests/test_passion_engine.py` | 65 | Convert to use only valid fields: `PipelineResult(debits=df, credits=pd.DataFrame())` | keyword-only | [x] |
| `tests/test_phase3.py` | 185 | ALREADY KEYWORD: `PipelineResult(debits=pd.DataFrame(...), credits=pd.DataFrame(...))` | no change | [x] |
| `tests/test_phase3.py` | 197 | ALREADY KEYWORD: `PipelineResult(debits=pd.DataFrame(...), credits=pd.DataFrame(...), global_mean=...)` | no change | [x] |
| `pipeline.py` | 333 | ALREADY KEYWORD: `PipelineResult(debits=debits, credits=credits, ...)` | no change | [x] |
| `pipeline.py` | 521 | ALREADY KEYWORD: `PipelineResult(debits=debits, credits=credits, ...)` | no change | [x] |
| `tutorial_real_data.py` | — | 0 hits | no change | [x] |
| `train_and_save_models.py` | — | 0 hits | no change | [x] |
| `model_benchmark.py` | — | 0 hits | no change | [x] |
| `summary_utils.py` | — | 0 hits | no change | [x] |
| `run_smoke.py` | — | 0 hits | no change | [x] |
| `run_stress_legacy.py` | — | 0 hits | no change | [x] |
| `run_stress_heavy.py` | — | 0 hits | no change | [x] |

**DataClass Serialization Audit**
- `grep -rn "astuple(" --include="*.py"`: 0 matches found.
- `grep -rn "asdict(" --include="*.py"`: 0 matches found.

**`personal_summary` Audit (`grep -rn "personal_summary" --include="*.py"`)**
| file | line | context | required edit | completed |
|---|---|---|---|---|
| `pipeline.py` | 100 | `personal_summary: dict = field(...)` | No change needed, core owns this | [x] |
| `pipeline.py` | 306 | Assignment from `detect_personal_patterns` | No change | [x] |
| `pipeline.py` | 345 | Passed to constructor kwarg | No change | [x] |

## 9. `pipeline.py` PipelineResult Extension

> **PHASE B MIGRATION**: Before changing `PipelineResult`, run these greps on the full codebase:
> `grep -rn "PipelineResult(" --include="*.py"`
> `grep -rn "dataclasses.astuple" --include="*.py"`
> `grep -rn "personal_summary" --include="*.py"`
> Convert every `PipelineResult` construction found to keyword-only arguments.
> Do NOT change `personal_summary` from required to optional unless every construction site has been audited and converted. If it currently has no default, keep it required until migration is complete.

In `pipeline.py`, extend the **existing** `PipelineResult` dataclass:
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

**Notes:**
- `PipelineResult` is frozen at the field-assignment level. The underlying DataFrames are defensive copies, not immutable views. Consumers must call `.copy()` before mutating result DataFrames to preserve state safety.
- **FIX-6 (kw_only=True verification)**: An audit of the codebase has confirmed that all instantiation sites of `PipelineResult` in `pipeline.py` and the test suites already supply keyword arguments. Therefore, adding `kw_only=True` is verified safe.

**Notes:**
- Do not add `personal_debits` if it already exists in core.
- Do not add `personal_summary` or `personal_insights` — core pipeline already owns these.

---

## 10. `pipeline_result.py`

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
