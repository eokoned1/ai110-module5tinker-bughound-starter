# BugHound Mini Model Card (Reflection)

Fill this out after you run BugHound in **both** modes (Heuristic and Gemini).

---

## 1) What is this system?

**Name:** BugHound  
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.

**Intended users:** Students learning agentic workflows and AI reliability concepts.

---

## 2) How does it work?

Describe the workflow in your own words (plan → analyze → act → test → reflect).  
Include what is done by heuristics vs what is done by Gemini (if enabled).

BugHound starts with a lightweight plan step that logs intent, then analyzes the code for issues. In Heuristic mode, analysis and fix generation use deterministic rules (for example, spotting `print`, `TODO`, and bare `except`). In Gemini mode, the agent requests structured JSON issues first and then a full-code rewrite, but now falls back to heuristics when the model output is not parseable or returns empty issues while deterministic signals still exist. After proposing a fix, the test stage runs `assess_risk` to score possible behavior-change risk. The reflect stage uses that risk score and policy thresholds to decide whether to auto-apply or require human review.

---

## 3) Inputs and outputs

**Inputs:**

- What kind of code snippets did you try?
- What was the “shape” of the input (short scripts, functions, try/except blocks, etc.)?

Tested snippets from `sample_code/` included:
- `cleanish.py` (small function with logging and return)
- `mixed_issues.py` (print + TODO + bare except + division)
- `flaky_try_except.py` (file I/O with bare except)
- `print_spam.py` (multiple print statements)

Inputs were short function-based scripts and try/except-heavy patterns where reliability issues are common.

**Outputs:**

- What types of issues were detected?
- What kinds of fixes were proposed?
- What did the risk report show?

Detected issue categories included Code Quality, Reliability, and Maintainability. Proposed fixes mainly replaced `print` with `logging.info`, inserted `import logging`, and rewrote `except:` as `except Exception as e:` in heuristic mode. Risk outputs varied by file: `mixed_issues.py` was consistently high risk and blocked from auto-fix, while `cleanish.py` stayed low risk with auto-fix allowed. Under mock Gemini behavior, some fixes were lower quality, which increased risk and more often disabled auto-fix.

---

## 4) Reliability and safety rules

List at least **two** reliability rules currently used in `assess_risk`. For each:

- What does the rule check?
- Why might that check matter for safety or correctness?
- What is a false positive this rule could cause?
- What is a false negative this rule could miss?

Rule 1: Penalize high-severity findings heavily (`-40` each).
- What it checks: Whether detected issues are marked High severity.
- Why it matters: High-severity findings imply higher chance of behavior or correctness breakage.
- Possible false positive: A true but low-impact warning mislabeled as High can over-block auto-fix.
- Possible false negative: If the analyzer under-labels severity, the scorer may be too permissive.

Rule 2: Penalize changes that remove `return` behavior (`if "return" in original and not in fixed`).
- What it checks: Potential control-flow/value-output regressions.
- Why it matters: Missing returns can silently change function contracts.
- Possible false positive: Valid refactors that preserve behavior without explicit `return` can still be penalized.
- Possible false negative: Return count can stay the same while semantics still change dangerously.

Rule 3 (updated in this activity): Auto-fix only when level is low, score is at least 90, and findings are low-severity only.
- What it checks: Confidence threshold before autonomous action.
- Why it matters: Forces caution for borderline or ambiguous cases.
- Possible false positive: Some safe medium-risk edits now require manual review.
- Possible false negative: A subtly unsafe edit might still pass if heuristics miss key risk signals.

---

## 5) Observed failure modes

Provide at least **two** examples:

1. A time BugHound missed an issue it should have caught  
2. A time BugHound suggested a fix that felt risky, wrong, or unnecessary  

For each, include the snippet (or describe it) and what went wrong.

1. Missed issue scenario: If model analysis returns `[]` for code that clearly contains `print` or bare `except`, the original workflow could accept that empty list and skip useful detections.
	- What went wrong: The parser treated structurally valid but semantically weak output as trustworthy.
	- Guardrail added: When LLM returns no issues, run heuristic analysis and use it if deterministic signals exist.

2. Risky/unnecessary fix scenario: In mock Gemini mode, fixer output may be generic or low-context, causing broad edits that do not clearly preserve behavior.
	- What went wrong: Output format was technically consumable but confidence was not high enough for autonomous apply.
	- Guardrail impact: Tightened auto-fix threshold and severity gating reduced unsafe auto-apply decisions.

---

## 6) Heuristic vs Gemini comparison

Compare behavior across the two modes:

- What did Gemini detect that heuristics did not?
- What did heuristics catch consistently?
- How did the proposed fixes differ?
- Did the risk scorer agree with your intuition?

Heuristics were consistent and predictable on known patterns (`print`, `TODO`, `except:`), but narrow in coverage. Gemini mode is broader in principle, but output quality/format can vary and force fallback logic to maintain reliability. Proposed fixes in heuristic mode were mechanical and small, while model-driven paths can be more variable and sometimes over-broad. The risk scorer generally matched intuition when obvious control-flow or severity concerns were present, but needed stricter auto-fix policy for borderline low-risk scores.

---

## 7) Human-in-the-loop decision

Describe one scenario where BugHound should **refuse** to auto-fix and require human review.

- What trigger would you add?
- Where would you implement it (risk_assessor vs agent workflow vs UI)?
- What message should the tool show the user?

Scenario: The fix changes exception handling and return behavior in the same patch (for example, replacing `except:` while also altering function outputs).
- Trigger: Any medium/high-severity finding or risk score below 90 should block auto-fix.
- Implementation location: `reliability/risk_assessor.py` (policy gate) plus agent logs for clear explanation.
- User message: "BugHound generated a candidate fix, but confidence is not high enough for auto-apply. Please review this diff manually."

---

## 8) Improvement idea

Propose one improvement that would make BugHound more reliable *without* making it dramatically more complex.

Examples:

- A better output format and parsing strategy
- A new guardrail rule + test
- A more careful “minimal diff” policy
- Better detection of changes that alter behavior

Write your idea clearly and briefly.

Add a lightweight "diff-size" signal in risk scoring that penalizes edits when changed-line ratio exceeds a threshold (for example, 30%), and add a test asserting that large rewrites default to human review. This is low complexity, measurable, and directly targets over-editing failure modes.
