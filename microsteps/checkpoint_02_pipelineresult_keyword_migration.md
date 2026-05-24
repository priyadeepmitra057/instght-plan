# CHECKPOINT 02: PipelineResult Keyword Migration
Directly modified:   pipeline.py, tests/test_phase3.py
Indirectly affected: Pipeline execution
Code blocks used:    CB-P2-00
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
  Instruction:    Run AST validation to ensure `PipelineResult` calls use keyword arguments.

  Details:
  Run the script to ensure there are no positional arguments in `PipelineResult` constructor calls. If the script fails, manually refactor the codebase to use keyword-only arguments and run it again.

  After:
  ```bash
python3 - <<'PY'
import ast
from pathlib import Path

bad = []
for path in Path(".").rglob("*.py"):
    if ".venv" in path.parts or "venv" in path.parts:
        continue
    tree = ast.parse(path.read_text(encoding="utf-8"))
    for node in ast.walk(tree):
        if isinstance(node, ast.Call):
            fn = node.func
            name = fn.id if isinstance(fn, ast.Name) else getattr(fn, "attr", "")
            if name == "PipelineResult" and node.args:
                bad.append((str(path), node.lineno, len(node.args)))

if bad:
    raise SystemExit(f"PipelineResult positional calls found: {bad}")
print("PipelineResult keyword-only call validation passed")
PY
  ```

POST-EXECUTION VALIDATION
[ ] AST validation script passes.
[ ] `pytest tests/test_phase3.py` passes.

GO / NO-GO
All checks pass → proceed to CHECKPOINT [03]
