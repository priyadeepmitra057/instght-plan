# CHECKPOINT 01: Core Schema and Config
Directly modified:   schema.py, config.py
Indirectly affected: All modules importing Col
Code blocks used:    CB-P1-01, CB-P1-11
Risk:                HIGH
Depends on:          NONE

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
Updating the core schema to include passion-related columns and migrating `TIP_CORPUS` in `config.py` to the new structured schema required by the passion engine.

PRE-CONDITIONS
[ ] Codebase is in original state.

STEPS

  STEP [1.1]
  File:           schema.py
  Action:         MODIFY
  Target:         Col class
  Source file:    passion_plan_part1.md
  Source section: 1. schema.py Update
  Block ID:       CB-P1-01
  Flags:          NONE

  Before:
  ```python
    # ── ML Insight Engine (benchmark / training) ──────────────────────────
    CATEGORY_CONFIDENCE = "category_confidence"
    INSIGHT_TYPE = "insight_type"
    TIP_ID = "tip_id"
    INSIGHT_SCORE = "insight_score"
  ```

  Instruction: Insert the two new constants after the `INSIGHT_SCORE` line.

  After:
  ```python
    # ── Passion Engine: Subcategory Inference ─────────────────────────────
    INFERRED_SUBCATEGORY   = "inferred_subcategory"
    SUBCATEGORY_CONFIDENCE = "subcategory_confidence"
  ```

  Rollback: Remove the two added lines.

  STEP [1.2]
  File:           config.py
  Action:         MODIFY
  Target:         TIP_CORPUS and SPECIFIC_MERCHANT_ALIASES
  Source file:    passion_plan_part1.md
  Source section: 7a. config.py — TIP_CORPUS Schema Migration
  Block ID:       CB-P1-11
  Flags:          [INTERFACE BREAK RISK]

  Before:
  ```python
TIP_CORPUS: dict[str, dict] = {
    # ── Food ──────────────────────────────────────────────────────────────────
    "tip_food_spike_01": {
        "text": "A single ₹500 meal substitution per week could save ~₹2,000/month.",
        "categories": ["food"],
        "insights": ["spending_spike"],
    },
    # ... (many more)
}

SPECIFIC_MERCHANT_ALIASES: dict[str, str] = {
    # ...
    r"amazon":                              "Amazon",
    r"flipkart":                            "Flipkart",
    r"myntra":                              "Myntra",
    r"nykaa":                               "Nykaa",
    r"meesho":                              "Meesho",
    r"snapdeal":                            "Snapdeal",
    # ...
}
  ```

  Instruction: Replace the existing `TIP_CORPUS` dictionary and ensure `SPECIFIC_MERCHANT_ALIASES` includes the required entries. Note that the plan provides a sample/migrated `TIP_CORPUS`.

  After:
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

  Rollback: Restore original `TIP_CORPUS` and `SPECIFIC_MERCHANT_ALIASES`.

POST-EXECUTION VALIDATION
[ ] `schema.py` contains `INFERRED_SUBCATEGORY` and `SUBCATEGORY_CONFIDENCE`.
[ ] `config.py` uses the new `TIP_CORPUS` structure.
[ ] Linter passes.

GO / NO-GO
All checks pass → proceed to CHECKPOINT [02]
