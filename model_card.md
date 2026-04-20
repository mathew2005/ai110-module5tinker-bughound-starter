# BugHound Mini Model Card (Reflection)

Fill this out after you run BugHound in **both** modes (Heuristic and Gemini).

---

## 1) What is this system?

**Name:** BugHound
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.

**Intended users:** Students learning agentic workflows and AI reliability concepts — specifically, how to wrap an LLM with validation, fallback, and guardrail layers so the system still behaves sensibly when the model misbehaves or is unavailable.

---

## 2) How does it work?

Describe the workflow in your own words (plan → analyze → act → test → reflect).
Include what is done by heuristics vs what is done by Gemini (if enabled).

BugHound is structured as a five-step agent loop. The same loop runs in both modes; what changes is who does the analysis and rewriting work.

1. **PLAN** — the agent records its intent to scan the code and propose a fix. Identical in both modes.
2. **ANALYZE** — detect issues.
   - *Heuristic mode*: three regex rules in `_heuristic_analyze`. `print(` → Low "Code Quality"; bare `except:` → High "Reliability"; `TODO` → Medium "Maintainability".
   - *Gemini mode*: sends the system + user prompts from `prompts/analyzer_*.txt`, expects back a JSON array of `{type, severity, msg}` objects. The response is parsed, then each issue is checked against a schema (valid severity, non-empty message). Malformed items are dropped and logged. If every item fails validation, the agent falls back to heuristics.
3. **ACT** — propose a fix.
   - *Heuristic mode*: string-level rewrites in `_heuristic_fix`. Swap `except:` → `except Exception as e:` with a placeholder comment; swap `print(` → `logging.info(` and prepend `import logging` if absent.
   - *Gemini mode*: sends code + issues to the model, strips markdown fences, checks the reply is non-empty, then runs a Python syntax guardrail (`compile()` — Part 4 change). If the reply doesn't parse, the original code is returned unchanged; if the reply is empty, falls back to heuristics.
4. **TEST** — `risk_assessor.assess_risk` scores the fix on a 0–100 scale starting at 100 and deducting for issue severity (−40/−20/−5), for shrinkage under 50% (−20), for **growth over 130%** (−15, Part 3 change), for lost `return` statements (−30), and for bare-`except` removal (−5). Scores map to levels: ≥75 low, ≥40 medium, otherwise high.
5. **REFLECT** — `should_autofix` is set to `True` only when the risk level is low; otherwise the UI recommends human review.

Fallback is layered: API exception → heuristics; unparseable JSON → heuristics; items that fail schema → dropped (or full fallback if all fail); fix that doesn't compile → original returned; empty fix → heuristic fix then risk pinned to 0.

---

## 3) Inputs and outputs

**Inputs:**

- **What kind of code snippets did you try?** The four snippets in `sample_code/`: `cleanish.py` (6 lines, well-formed logging), `print_spam.py` (7 lines, repeated `print`), `flaky_try_except.py` (9 lines, file I/O inside a bare `except:`), `mixed_issues.py` (9 lines, TODO + `print` + bare `except:` on a `x / y` divide).
- **Shape of the input:** all four are short standalone Python functions. Three contain a try/except block; three contain a `print`; one is already clean.

**Outputs:**

- **Types of issues detected (heuristic mode):**
  - `cleanish.py`: none.
  - `print_spam.py`: 1 Low `Code Quality`.
  - `flaky_try_except.py`: 1 High `Reliability`.
  - `mixed_issues.py`: 1 Low + 1 High + 1 Medium (three rules firing at once).
- **Kinds of fixes proposed (heuristic mode):**
  - `print_spam.py` → adds `import logging`, replaces every `print(` with `logging.info(`.
  - `flaky_try_except.py` → rewrites `except:` as `except Exception as e:` with a `# [BugHound]` placeholder comment.
  - `mixed_issues.py` → combined: logging import, print→logging, except rewrite. The TODO comment is not touched by the fixer — the heuristic has no rule for it.
- **What the risk report showed (MockClient walkthrough):**
  - `cleanish.py`: score 100, low, auto-fix yes.
  - `print_spam.py`: score 45, medium, auto-fix no (the mock's stub fixer shrinks the code >50%, triggering both the "much shorter" and "return removed" penalties).
  - `flaky_try_except.py`: score 5, high.
  - `mixed_issues.py`: score 0, high.
  - Real heuristic runs of the same snippets scored higher because the heuristic fixer grows rather than shrinks the code.

---

## 4) Reliability and safety rules

List at least **two** reliability rules currently used in `assess_risk`. For each:

### Rule A — Severity-based score deduction

- **What it checks:** for every detected issue, subtracts 40 (High), 20 (Medium), or 5 (Low) from the score. Severities outside that set contribute zero.
- **Why it matters:** it tells the reflection step how much human attention the fix deserves. A single High issue knocks the score to 60, already below the auto-apply threshold, which is correct — high-severity issues often involve error handling or correctness semantics.
- **False positive:** a model labels a cosmetic issue as "High" (e.g., "This function name is vague — High"). The score drops 40 points, blocking an auto-apply that would have been safe.
- **False negative:** an actually dangerous issue is silently marked with a non-canonical severity like "Critical" or "Warning". Because the rule's branch only matches `low/medium/high`, the score is not reduced at all. This is the gap that Part 2's schema validation covers — non-canonical severities now cause the whole issue (or batch) to be rejected rather than being counted at zero weight.

### Rule B — Python syntax guardrail (Part 4 addition)

- **What it checks:** after the LLM fix is cleaned, `compile(code, "<string>", "exec")` must succeed. If `SyntaxError` is raised, the fix is discarded and the original code is returned unchanged.
- **Why it matters:** the fixer prompt says "return only Python code" but the agent can't force that. Without this rule, a truncated or mis-fenced reply from Gemini would be rendered to the user as a "proposed fix" and scored against the risk assessor's length/return heuristics — potentially auto-applied despite not parsing at all.
- **False positive:** the model returns Python 3.12 match statements and the local interpreter is Python 3.9 — the code is structurally fine but `compile()` still raises. The fix is rejected even though it would run in the target environment.
- **False negative:** the fix parses but is semantically wrong — e.g., inverts a boolean, changes the return branch. `compile()` says "valid Python" and the guardrail passes, leaving the semantic check to the structural rules in `assess_risk` (lost return, shrinkage) which are much coarser.

---

## 5) Observed failure modes

Provide at least **two** examples:

### 1) Missed issue — resource leak not caught

**Snippet:** `sample_code/flaky_try_except.py`
```python
def load_text_file(path):
    try:
        f = open(path, "r")
        data = f.read()
        f.close()
    except:
        return None
    return data
```
**What BugHound reported:** one High reliability issue (bare `except:`).
**What it missed:** if `f.read()` raises, `f.close()` is never reached — the file handle leaks. The idiomatic fix is `with open(...) as f:`. Neither the heuristic (which only pattern-matches three things) nor the LLM prompt (which lists four example patterns and otherwise trusts the model's generality) flagged this. A clean fix of the bare-`except` alone leaves the leak intact.

### 2) Risky / wrong behavior — silent LLM failure masked as parse failure

**Snippet:** any of the samples, run in Gemini mode during this assignment.
**What happened:** every Gemini call returned an empty string, and the agent's trace read *"LLM output was not parseable JSON. Falling back to heuristics."* That log message implies a mis-formatted reply from the model, but the real cause lives one layer deeper — `GeminiClient.complete` has a broad `except Exception: return ""` that swallows genuine API errors (in this repo, the call sends `role: "system"`, which Gemini's chat API does not accept, so it raises and the wrapper converts the raise into `""`). The system behaved correctly overall (fell back gracefully) but the trace was misleading about *why* it fell back, making the Gemini path effectively invisible and hiding a real bug behind the safety net.

---

## 6) Heuristic vs Gemini comparison

Compare behavior across the two modes:

- **What Gemini was expected to detect that heuristics missed:** the resource leak in `flaky_try_except.py`; the divide-by-zero path in `mixed_issues.py` that returns `0` on failure (a silent error swallow); variable naming, missing docstrings, and other style issues outside the three heuristic rules.
- **What heuristics caught consistently:** exactly three patterns — `print(`, bare `except:`, `TODO` — across every input, every run, deterministically. Any code outside those three patterns produces zero issues.
- **How the proposed fixes differed (expected):** heuristic fixes are mechanical single-line substitutions; Gemini fixes would reshape the code more broadly — adding specific exception types, `with` blocks, docstrings, or type hints. This is why Part 3's new "noticeably longer" rule exists: a 40%+ expansion is a reasonable signal that the model ignored the prompt's "minimal change" constraint.
- **Did the risk scorer agree with my intuition?** Mostly yes. High severity drove the score below the auto-apply gate in every non-trivial case. The one mismatch in my intuition vs. the scorer: for `cleanish.py` with zero issues, the scorer stays at 100 and sets `should_autofix=True`, which technically means "auto-apply the unchanged code" — correct behavior, but the UI phrases it as if a fix is being applied when nothing happened.
- **Caveat on the comparison:** during this session I was unable to exercise the Gemini path directly. All of my Gemini-mode runs returned empty (see Section 5, Failure Mode 2) and fell back to heuristics. Observations about Gemini behavior above are inferred from the prompts and the agent's parsing/validation code, not from live model output.

---

## 7) Human-in-the-loop decision

Describe one scenario where BugHound should **refuse** to auto-fix and require human review.

**Scenario:** the proposed fix adds a new `import` statement that was not present in the original code.

- **Trigger:** set-difference between the top-level import names of `fixed_code` and `original_code`. If the fix introduces `import X` or `from X import Y` for any `X` the original didn't have, mark `should_autofix=False` regardless of the numeric score.
- **Where to implement it:** inside `risk_assessor.assess_risk`. This is a structural check, not a workflow-control decision, so it belongs next to the existing "much shorter" and "return removed" rules rather than in the agent or UI. Implementing it in the UI would be wrong because the CLI / tests also rely on `should_autofix`.
- **Why this rule:** new imports mean new dependencies. Even a "safe" import (`import logging`) changes the module's observable surface — it can shadow names, pull in transitive initialization, or simply not exist in the deployment environment. The fixer prompt says *"make the smallest changes needed"*, and adding a dependency is almost never the smallest change. Rejecting auto-apply here respects the spirit of that constraint.
- **Message to the user:** *"Proposed fix introduces new imports that were not in the original (e.g. `logging`). Auto-apply blocked — please verify the new dependencies are available and intended before accepting."*

---

## 8) Improvement idea

Propose one improvement that would make BugHound more reliable *without* making it dramatically more complex.

**Idea: surface silent LLM failures instead of masking them as parse errors.**

Motivation: the most visible reliability problem in this assignment wasn't a wrong answer — it was the absence of one. Every Gemini call in my session returned `""`, the agent logged *"not parseable JSON"*, and fell back to heuristics. The fallback worked, but the user (me) had no way to know whether Gemini was genuinely returning malformed JSON or whether the API was never actually succeeding. That ambiguity makes it hard to distinguish model bugs from agent bugs.

Change: in `llm_client.GeminiClient.complete`, replace the blanket `except Exception: return ""` with a narrower handler that lets specific categories of exceptions propagate. Re-raise on authentication errors, invalid-argument errors, and rate-limit errors; keep the `return ""` path only for content-blocked / safety-filter outcomes. The agent's existing `try/except` in `analyze()` and `propose_fix()` (lines 73–77, 109–113) already catches exceptions and logs them as `"API Error: ..."`, so this change does not require any agent-side logic — it just routes more failure cases through the existing error path.

Cost: about 10 lines in `llm_client.py` plus a targeted test. No new dependencies, no changes to the prompts, no changes to the risk assessor, no changes to the UI. The net effect is that a user running Gemini mode will see `"API Error: <specific reason>"` in the trace when the API is actually failing, instead of the misleading *"not parseable JSON"* message — and can then decide whether to fix their key, adjust their prompt, or accept the heuristic fallback.
