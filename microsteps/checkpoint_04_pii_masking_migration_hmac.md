# CHECKPOINT 04: PII Masking Migration (HMAC)

Directly modified:   log_utils.py, hash_utils.py, recurring_detector.py, tests/test_logging_safety.py
Indirectly affected: All logs containing merchant names
Code blocks used:    CB-P1-04, CB-P1-05, CB-P1-06, CB-P1-07, CB-P1-08
Risk:                MEDIUM
Depends on:          CHECKPOINT 03

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
Replacing the insecure `stable_hash` with HMAC-based `log_safe_merchant` to properly protect PII in logs.

PRE-CONDITIONS
[ ] Checkpoint 03 passed.

STEPS

  STEP [4.1]
  File:           log_utils.py
  Action:         CREATE
  Source file:    passion_plan_part1.md
  Source section: 4. log_utils.py
  Block ID:       CB-P1-04
  Flags:          [SECURITY SENSITIVE]

  Before:
  ```
  FILE DOES NOT EXIST
  ```

  Instruction: Create `log_utils.py` with verbatim content.

  After:
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

  Rollback: Delete log_utils.py.

  STEP [4.2]
  File:           hash_utils.py
  Action:         MODIFY
  Source file:    passion_plan_part1.md
  Source section: 5. hash_utils.py Update
  Block ID:       CB-P1-05
  Flags:          NONE

  Before:
  ```python
import hashlib

def stable_hash(value: str) -> str:
    """Produces a short, stable hash for masking PII in logs."""
    return hashlib.sha256(value.encode("utf-8")).hexdigest()[:12]
  ```

  Instruction: Replace entire content of `hash_utils.py` with verbatim content.

  After:
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

  Rollback: Restore original content of hash_utils.py.

  STEP [4.3]
  File:           recurring_detector.py
  Action:         MODIFY
  Target:         Imports/Logging
  Source file:    passion_plan_part1.md
  Source section: 5. hash_utils.py Update
  Block ID:       CB-P1-06
  Flags:          NONE

  Before:
  ```python
from hash_utils import stable_hash
import logging
logger = logging.getLogger(__name__)
  ```

  Instruction: Update imports and logger as verbatim.

  After:
  ```python
from log_utils import log_safe_merchant
from logger_factory import get_logger
logger = get_logger(__name__)
  ```

  Rollback: Restore original imports and logger.

  STEP [4.4]
  File:           recurring_detector.py
  Action:         MODIFY
  Target:         stable_hash calls
  Source file:    passion_plan_part1.md
  Source section: 5. hash_utils.py Update
  Block ID:       CB-P1-07
  Flags:          NONE

  Instruction: Replace every occurrence of `stable_hash(identifier)` with `log_safe_merchant(identifier)`.

  After:
  ```python
log_safe_merchant(identifier)
  ```

  Rollback: Revert to stable_hash(identifier).

  STEP [4.5]
  File:           tests/test_logging_safety.py
  Action:         MODIFY
  Target:         Imports/Tests
  Source file:    passion_plan_part1.md
  Source section: 5. hash_utils.py Update
  Block ID:       CB-P1-08
  Flags:          NONE

  Before:
  ```python
from hash_utils import stable_hash
# ... in test_pii_redaction_coverage ...
    assert any(stable_hash("Netflix") in msg for msg in caplog.messages)
  ```

  Instruction: Update imports and the specific assertion verbatim.

  After:
  ```python
from log_utils import log_safe_merchant
# ... in test_pii_redaction_coverage ...
    assert any(log_safe_merchant("Netflix") in msg for msg in caplog.messages)
  ```

  Rollback: Revert imports and assertion.

POST-EXECUTION VALIDATION
[ ] File exists at: log_utils.py
[ ] Import resolves: from log_utils import log_safe_merchant
[ ] `pytest tests/test_logging_safety.py` passes.
[ ] `grep "stable_hash" recurring_detector.py` returns 0 results.

GO / NO-GO
All checks pass → proceed to CHECKPOINT [05]
