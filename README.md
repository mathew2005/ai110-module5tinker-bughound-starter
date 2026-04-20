## TF reflection

The conceptual pivot in this module is the gap between a prompt's stated contract and a model's actual behavior: students can read *"Return ONLY valid JSON"* in `prompts/analyzer_system.txt` and assume that's what they'll receive, but the real engineering work sits in the validation, parsing, and fallback code the agent wraps around the reply. Where students most often stall is in confusing silent fallback with correct behavior — when Gemini returns an empty string and the agent logs *"not parseable JSON, falling back to heuristics,"* the UI still shows issues and a diff, so it looks like everything worked even though the AI path never executed. AI assistance is most useful when students apply it to explain code they've already read (prompt-to-parser alignment, the arithmetic inside `risk_assessor`) and least useful when they ask it to *propose* improvements, because its instinct is to expand scope and pile on guardrails that feel thorough but cost more than they're worth — exactly the failure mode the fixer prompt warns against. For the reliability changes in Parts 3 and 4, I would ask students to predict the numeric risk score by hand before running the tool and then compare against the output, because the formula is simple enough to trace mentally yet opaque enough that students reflexively trust whatever the UI prints; that predict-then-compare loop is where the habit of interrogating AI output actually forms. The deeper goal, more important than any particular guardrail a student ends up adding, is treating every layer between the model and the user as a deliberate response to a specific failure mode rather than as boilerplate scaffolding.

# 🐶 BugHound

BugHound is a small, agent-style debugging tool. It analyzes a Python code snippet, proposes a fix, and runs basic reliability checks before deciding whether the fix is safe to apply automatically.

---

## What BugHound Does

Given a short Python snippet, BugHound:

1. **Analyzes** the code for potential issues  
   - Uses heuristics in offline mode  
   - Uses Gemini when API access is enabled  

2. **Proposes a fix**  
   - Either heuristic-based or LLM-generated  
   - Attempts minimal, behavior-preserving changes  

3. **Assesses risk**  
   - Scores the fix  
   - Flags high-risk changes  
   - Decides whether the fix should be auto-applied or reviewed by a human  

4. **Shows its work**  
   - Displays detected issues  
   - Shows a diff between original and fixed code  
   - Logs each agent step

---

## Setup

### 1. Create a virtual environment (recommended)

```bash
python -m venv .venv
source .venv/bin/activate   # macOS/Linux
# or
.venv\Scripts\activate      # Windows
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

---

## Running in Offline (Heuristic) Mode

No API key required.

```bash
streamlit run bughound_app.py
```

In the sidebar, select:

* **Model mode:** Heuristic only (no API)

This mode uses simple pattern-based rules and is useful for testing the workflow without network access.

---

## Running with Gemini

### 1. Set up your API key

Copy the example file:

```bash
cp .env.example .env
```

Edit `.env` and add your Gemini API key:

```text
GEMINI_API_KEY=your_real_key_here
```

### 2. Run the app

```bash
streamlit run bughound_app.py
```

In the sidebar, select:

* **Model mode:** Gemini (requires API key)
* Choose a Gemini model and temperature

BugHound will now use Gemini for analysis and fix generation, while still applying local reliability checks.

---

## Running Tests

Tests focus on **reliability logic** and **agent behavior**, not the UI.

```bash
pytest
```

You should see tests covering:

* Risk scoring and guardrails
* Heuristic fallbacks when LLM output is invalid
* End-to-end agent workflow shape
