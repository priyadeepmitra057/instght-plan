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
    "tip_food_spike_02": {
        "text": "Your food spending has doubled compared to last month.",
        "categories": ["food"],
        "insights": ["spending_spike"],
    },
    "tip_food_recurring_01": {
        "text": "You have a recurring food expense of ₹{amount} every {frequency}.",
        "categories": ["food"],
        "insights": ["recurring_expense"],
    },

    # ── Utilities ─────────────────────────────────────────────────────────────
    "tip_utils_spike_01": {
        "text": "Utility bills are {pct_change}% higher this month.",
        "categories": ["utilities"],
        "insights": ["spending_spike"],
    },
    "tip_utils_recurring_01": {
        "text": "Expected utility payment of ₹{amount} is due soon.",
        "categories": ["utilities"],
        "insights": ["recurring_expense", "budget_risk"],
    },
}

SPECIFIC_MERCHANT_ALIASES: dict[str, str] = {
    # Food & Delivery
    r"zomato":                              "Zomato",
    r"swiggy":                              "Swiggy",
    r"blinkit":                             "Blinkit",
    r"instamart":                           "Instamart",
    r"zepto":                               "Zepto",
    r"dunzo":                               "Dunzo",
    r"mcdonald'?s":                         "McDonalds",
    r"kfc":                                 "KFC",
    r"domino'?s":                           "Dominos",
    r"pizza hut":                           "PizzaHut",
    r"subway":                              "Subway",
    r"starbucks":                           "Starbucks",
    r"burger king":                         "BurgerKing",

    # Transportation & Travel
    r"uber":                                "Uber",
    r"ola":                                 "Ola",
    r"rapido":                              "Rapido",
    r"makemytrip":                          "MakeMyTrip",
    r"goibibo":                             "Goibibo",
    r"yatra":                               "Yatra",
    r"irctc":                               "IRCTC",
    r"redbus":                              "RedBus",
    r"cleartrip":                           "Cleartrip",
    r"indigo":                              "IndiGo",
    r"air india":                           "AirIndia",
    r"spicejet":                            "SpiceJet",

    # Shopping & Ecommerce
    r"amazon":                              "Amazon",
    r"flipkart":                            "Flipkart",
    r"myntra":                              "Myntra",
    r"nykaa":                               "Nykaa",
    r"meesho":                              "Meesho",
    r"snapdeal":                            "Snapdeal",
    r"ajio":                                "Ajio",
    r"tatacliq":                            "TataCliq",
    r"reliance smart":                      "RelianceSmart",
    r"dmart":                               "DMart",
    r"bigbasket":                           "BigBasket",

    # Entertainment
    r"netflix":                             "Netflix",
    r"amazon prime":                        "PrimeVideo",
    r"hotstar":                             "Hotstar",
    r"spotify":                             "Spotify",
    r"bookmyshow":                          "BookMyShow",
    r"pvr":                                 "PVR",
    r"inox":                                "INOX",

    # Utilities & Telecom
    r"jio":                                 "Jio",
    r"airtel":                              "Airtel",
    r"vi\b":                                "Vi",
    r"bsnl":                                "BSNL",
    r"bescom":                              "BESCOM",
    r"bwssb":                               "BWSSB",

    # Fintech & Payments (Often intermediaries)
    r"paytm":                               "Paytm",
    r"phonepe":                             "PhonePe",
    r"google pay":                          "GPay",
    r"gpay":                                "GPay",
    r"cred":                                "CRED",
    r"mobikwik":                            "MobiKwik",
    r"bharatpe":                            "BharatPe",
    r"razorpay":                            "Razorpay",
    r"payu":                                "PayU",
    r"billdesk":                            "BillDesk",
}
  ```

  Instruction: Replace the exact literal code block above. If the exact Before block is not found exactly once, STOP. Do not infer the edit location.
  Migrate the existing TIP_CORPUS in place.
  Do not replace the full corpus with the sample entries.
  For every existing tip_id, preserve the existing text and convert each value to the new schema:

  {
  "text": existing_text,
  "categories": tuple(existing_categories),
  "insights": tuple(existing_insights),
  }

  Rules:
  - Preserve every existing tip_id unless explicitly deprecated elsewhere.
  - Preserve every existing tip text.
  - Preserve every existing category and insight mapping.
  - Generic wildcard tips may use empty categories/insights only if tip_id starts with "generic_".
  - Non-generic tips must have non-empty categories and insights.
  - Do not introduce "any" wildcard for non-generic tips.
  - If any existing tip cannot be migrated mechanically, STOP and report the exact tip_id.

  For SPECIFIC_MERCHANT_ALIASES:
  Preserve the entire existing alias map.
  Only add or normalize these required entries if missing:

  "amazon": "amazon"
  "amzn": "amazon"
  "amazon prime": "amazon"
  "flipkart": "flipkart"
  "meesho": "meesho"
  "snapdeal": "snapdeal"

  Do not delete existing aliases.

  After:
  A patch-only instruction that explicitly preserves all existing entries and adds only the required entries, migrating the format as stated in the instruction.

  Rollback: Restore original `TIP_CORPUS` and `SPECIFIC_MERCHANT_ALIASES`.

POST-EXECUTION VALIDATION
[ ] `schema.py` contains `INFERRED_SUBCATEGORY` and `SUBCATEGORY_CONFIDENCE`.
[ ] `config.py` uses the new `TIP_CORPUS` structure.
[ ] Linter passes.

[ ] python3 -m py_compile schema.py config.py succeeds.
[ ] python3 -c "from schema import Col; from config import TIP_CORPUS, SPECIFIC_MERCHANT_ALIASES; assert Col.INFERRED_SUBCATEGORY == 'inferred_subcategory'; assert Col.SUBCATEGORY_CONFIDENCE == 'subcategory_confidence'; assert isinstance(TIP_CORPUS, dict); assert isinstance(SPECIFIC_MERCHANT_ALIASES, dict)"
[ ] python3 -c "from config import TIP_CORPUS; assert len(TIP_CORPUS) >= 5; assert all({'text','categories','insights'} <= set(v) for v in TIP_CORPUS.values())"
[ ] python3 -c "from config import SPECIFIC_MERCHANT_ALIASES; required={'amazon','amzn','amazon prime','flipkart','meesho','snapdeal'}; assert required <= set(SPECIFIC_MERCHANT_ALIASES)"
[ ] python3 -c "from config import SPECIFIC_MERCHANT_ALIASES; assert len(SPECIFIC_MERCHANT_ALIASES) >= 20"

GO / NO-GO
All checks pass → proceed to CHECKPOINT [02]
