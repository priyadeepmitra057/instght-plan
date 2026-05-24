# CHECKPOINT 05: Supporting Modules (Banned Content, Passion Config)

Directly modified:   banned_content.py, config_passion.py, insight_generator.py
Indirectly affected: Insight generation
Code blocks used:    CB-P1-09, CB-P1-10, CB-P1-12, CB-P1-13
Risk:                LOW
Depends on:          CHECKPOINT 04

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
Implementing specialized filtering and configuration for the passion engine.

PRE-CONDITIONS
[ ] Checkpoint 04 passed.

STEPS

  STEP [5.1]
  File:           banned_content.py
  Action:         CREATE
  Source file:    passion_plan_part1.md
  Source section: 6. banned_content.py
  Block ID:       CB-P1-09
  Flags:          NONE

  Before:
  ```
  FILE DOES NOT EXIST
  ```

  Instruction: Create `banned_content.py` with verbatim content.

  After:
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

  Rollback: Delete banned_content.py.

  STEP [5.2]
  File:           config_passion.py
  Action:         CREATE
  Source file:    passion_plan_part1.md
  Source section: 7. config_passion.py
  Block ID:       CB-P1-10
  Flags:          NONE

  Before:
  ```
  FILE DOES NOT EXIST
  ```

  Instruction: Create `config_passion.py` with verbatim content.

  After:
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

  Rollback: Delete config_passion.py.

  STEP [5.3]
  File:           insight_generator.py
  Action:         MODIFY
  Source file:    passion_plan_part1.md
  Source section: 7b. insight_generator.py — TIP_CORPUS Import Migration
  Block ID:       CB-P1-12, CB-P1-13
  Flags:          [INTERFACE BREAK RISK]

  Before:
  ```python
from config import TIP_CORPUS
  ```

  Instruction: Replace the exact import above with:

  ```python
from contracts import TIP_CORPUS, lookup_matching_tip_ids
  ```

  Then replace the existing `_select_tip` function exactly.
  If the old `_select_tip` function is not found exactly once, STOP.

  After:
  ```python
def _select_tip(category: str, insight_type: str, rng: random.Random) -> str:
    """Select a random tip matching category and insight type."""
    try:
        tip_ids = lookup_matching_tip_ids(category, insight_type)
    except (KeyError, TypeError, IndexError, ValueError) as e:
        logger.warning(
            "tip_lookup_failed",
            extra={
                "category": category,
                "insight_type": insight_type,
                "error_type": type(e).__name__,
            },
        )
        return ""

    if not tip_ids:
        return ""

    tip_id = rng.choice(tip_ids)
    tip_data = TIP_CORPUS.get(tip_id, {})
    return tip_data.get("text", "") if tip_data else ""
  ```

  Rollback: Revert imports and access patterns in insight_generator.py.

POST-EXECUTION VALIDATION
[ ] File exists at: banned_content.py
[ ] File exists at: config_passion.py
[ ] insight_generator.py imports TIP_CORPUS and lookup_matching_tip_ids from contracts.
[ ] insight_generator.py does not import TIP_CORPUS from config.
[ ] insight_generator.py contains no placeholder text like "equivalent rendering logic".
[ ] python3 -m py_compile insight_generator.py succeeds.

GO / NO-GO
All checks pass → proceed to CHECKPOINT [06]
