# CHECKPOINT 02: PipelineResult Keyword Migration
Directly modified:   pipeline.py, tests/test_phase3.py
Indirectly affected: Pipeline execution
Code blocks used:    NONE (Audit-based refactor)
Risk:                MEDIUM
Depends on:          CHECKPOINT 01

---
EXECUTOR DIRECTIVE
Follow each step exactly as written.
Do not infer. Do not improvise. Do not skip.
Confirm each step complete before moving to next.
---

CONTEXT
Before adding new fields to the `PipelineResult` dataclass and enforcing `kw_only=True`, all existing instantiation sites must be converted to use keyword arguments.

PRE-CONDITIONS
[ ] `grep -rn "PipelineResult(" --include="*.py" .` confirms call sites.

STEPS

  STEP [2.1]
  File:           pipeline.py
  Action:         MODIFY
  Target:         run_pipeline and run_inference
  Instruction:    Audit and ensure `PipelineResult` calls use keyword arguments.

  Details:
  Line 333: Already keyword?
  ```python
        return PipelineResult(
            debits=debits,
            credits=credits,
            # ...
  ```
  Line 521: Already keyword?
  ```python
        return PipelineResult(
            debits=debits,
            credits=credits,
            # ...
  ```
  (No action if already keyword)

  STEP [2.2]
  File:           tests/test_phase3.py
  Action:         MODIFY
  Target:         Test calls
  Instruction:    Audit and ensure `PipelineResult` calls use keyword arguments.

  Details:
  Line 185:
  ```python
    result = PipelineResult(
        debits=pd.DataFrame(...),
        credits=pd.DataFrame(...)
    )
  ```
  Line 197:
  ```python
    original = PipelineResult(
        debits=pd.DataFrame(...),
        credits=pd.DataFrame(...),
        global_mean=0.1
    )
  ```
  (No action if already keyword)

POST-EXECUTION VALIDATION
[ ] All `PipelineResult(...)` calls in the codebase use keyword arguments for all parameters.
[ ] `pytest tests/test_phase3.py` passes.

GO / NO-GO
All checks pass → proceed to CHECKPOINT [03]
