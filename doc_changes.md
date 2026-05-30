# Build Fix Documentation

A record of every change made to get this project running, with the reasoning behind each decision.

---

## 1. Python Virtual Environment Setup

### What was done
```bash
python3 -m venv .venv
.venv/bin/pip install --upgrade setuptools wheel
```

### Why
A virtual environment isolates project dependencies from the system Python installation. Without it, package versions installed for this project can conflict with other projects or system tools.

### Problem encountered
Running `pip install -r requirements.txt` immediately failed with:
```
ModuleNotFoundError: No module named 'pkg_resources'
```

`pkg_resources` is part of `setuptools`. Python 3.13 ships with a minimal pip that does not bundle `setuptools` by default. Upgrading `setuptools` and `wheel` first fixes this before the main install.

### How to run the app
```bash
# Always use the venv's streamlit, not the system one
.venv/bin/streamlit run app.py
```

> **Note:** Even if `(.venv)` shows in your terminal prompt, typing `streamlit` may still resolve to the Anaconda/system install. Always prefix with `.venv/bin/` or activate with `source .venv/bin/activate`.

---

## 2. Full Dependency Installation — `requirements.txt` Fixes

Three package pins were incompatible with Python 3.13 and required updating.

### Fix A — `pandas==2.0.2` → `pandas>=2.2.0`
`pandas 2.0.2` has no pre-built wheel for Python 3.13. pip tries to compile from source and fails on `pkg_resources` inside pip's isolated build sandbox. `pandas 2.2.0+` ships native Python 3.13 wheels.

```diff
- pandas==2.0.2
+ pandas>=2.2.0
```

### Fix B — `en-core-web-sm 3.5.0` → `3.8.0`
`en_core_web_sm 3.5.0` requires `spacy>=3.5.0,<3.6.0`. spaCy 3.5.x has no Python 3.13 wheel, so pip tries a full C-extension source build (pulling `blis`, `thinc`, etc.) which cascades into hundreds of compilation steps and fails. Upgrading to `en_core_web_sm 3.8.0` requires spaCy 3.8.x which ships a native arm64/Python 3.13 wheel.

```diff
- en-core-web-sm @ https://github.com/explosion/spacy-models/releases/download/en_core_web_sm-3.5.0/en_core_web_sm-3.5.0.tar.gz
+ en-core-web-sm @ https://github.com/explosion/spacy-models/releases/download/en_core_web_sm-3.8.0/en_core_web_sm-3.8.0-py3-none-any.whl
```

### Fix C — `numpy>=2.0.0` added explicitly
`langchain 0.2.14` declared `numpy<2.0.0` in its metadata (written before numpy 2.0 existed). pip resolved to `numpy 1.26.4` which has no `cp313-arm64` wheel — it silently installed the `x86_64` wheel instead. On an Apple Silicon Mac this causes a `dlopen` architecture mismatch crash at runtime.

Pinning `numpy>=2.0.0` forces pip to pick `numpy-2.4.6-cp313-cp313-macosx_14_0_arm64.whl`. The numpy 2.x API is backward compatible for all usage in this project.

```diff
+ numpy>=2.0.0
  pandas>=2.2.0
```

### Learning point
When pip fails with `pkg_resources` or C-compilation errors, the root cause is almost always a pinned old package that predates Python 3.x wheel support. Check PyPI for `cpXYY` (e.g. `cp313`) tagged wheels for the target version. If none exist, bump the pin until you find one that ships a pre-built binary.

---

## 3. Code Fix — Deprecated / Wrong Groq Model Names

Each component intentionally uses a **different LLM model** to avoid evaluation bias. All original model names were either decommissioned or the wrong model type entirely.

| File | Old Model | New Model | Reason |
|------|-----------|-----------|--------|
| `models/question_generator.py` (init) | `whisper-large-v3-turbo` | `llama-3.1-8b-instant` | Whisper is a speech-to-text audio model — cannot handle chat prompts, errors immediately |
| `models/question_generator.py` (generation) | `llama-3.1-70b-versatile` | `llama-3.3-70b-versatile` | Decommissioned by Groq; 3.3 is the direct successor |
| `models/answer_evaluator.py` | `llama-3.2-90b-text-preview` | `meta-llama/llama-4-scout-17b-16e-instruct` | `-text-preview` decommissioned; Llama 4 Scout is a newer generation, different family branch |
| `models/resource_recommender.py` | `llama-3.2-90b-text-preview` | `compound-beta-mini` | Same decommissioned model; Groq Compound is an architecturally distinct family |

All replacements were validated with live `.invoke('OK')` calls against the Groq API before being committed. See full list of probed models below.

**Confirmed working:** `llama-3.1-8b-instant`, `llama-3.3-70b-versatile`, `meta-llama/llama-4-scout-17b-16e-instruct`, `compound-beta-mini`

**Confirmed decommissioned:** `whisper-large-v3-turbo`, `llama-3.1-70b-versatile`, `llama-3.2-90b-text-preview`, `llama3-70b-8192`, `llama3-8b-8192`, `mixtral-8x7b-32768`, `gemma2-9b-it`, `llama-3.2-90b-vision-preview`, `deepseek-r1-distill-llama-70b`, `qwen-qwq-32b`, `mistral-saba-24b`, `llama-3.3-70b-specdec`

---

## 4. Code Fix — Broken Question Parsing Regex

**File:** `models/question_generator.py`

```python
# Before — [\.$$] only matches '.' and '$' (literal dollar sign)
if line and re.match(r'^\d{1,2}[\.$$]', line):
    question = re.split(r'^\d{1,2}[\.$$]\s*', line)[1].strip()

# After — matches '.', ')', '-' which are the three formats LLMs use
if line and re.match(r'^\d{1,2}[\.\)\-]', line):
    question = re.split(r'^\d{1,2}[\.\)\-]\s*', line)[1].strip()
```

Inside a regex character class `[...]`, `$` is a literal dollar sign, not an end-of-string anchor. So `[\.$$]` never matched `1) Question` or `1- Question` — common LLM output formats — causing the parser to silently return generic placeholder questions for the entire interview.

---

## 5. Code Fix — JSON Code Fence Stripping in Answer Evaluator

**File:** `models/answer_evaluator.py` — `_evaluate_answer_with_groq`

LLMs frequently wrap JSON in markdown code fences (` ```json ... ``` `). `json.loads()` fails on the backtick character, the `JSONDecodeError` handler silently returned all-zero default scores, and the feedback screen showed nothing useful.

```python
content = response.content.strip()
if "```" in content:
    content = re.sub(r'^```(?:json)?\s*', '', content)
    content = re.sub(r'\s*```\s*$', '', content).strip()
return content
```

`import re` was also added to the file's imports.

---

## 6. Code Fix — Evaluator Prompt Key Mismatch

**File:** `models/answer_evaluator.py` — `evaluate_answer` prompt and `_get_default_evaluation`

The prompt instructed the LLM to return `relevance_score`, `technical_accuracy`, `communication_clarity` etc., but `app.py` reads `relevance`, `clarity`, `skills_demonstration`, `alignment`. Every key lookup returned the default ("No feedback available.") because no keys ever matched.

The prompt was updated to request exactly the keys `app.py` uses, and `_get_default_evaluation` was updated to match:

```python
# Now returns keys that app.py actually reads
{
    "relevance": "score as percentage e.g. 75%",
    "clarity": "score as percentage e.g. 80%",
    "skills_demonstration": "score as percentage e.g. 70%",
    "alignment": "text description ...",
    "detailed_feedback": "...",
    "suggestions": "..."
}
```

---

## 7. Code Fix — Missing `recommend_resources` Method

**File:** `models/resource_recommender.py`

`app.py` calls `resource_recommender.recommend_resources(clarity, relevance, skills_demonstration)` at the summary screen, but the method did not exist — causing an `AttributeError` crash every time the interview completed.

Added the method which identifies weak areas from the three float scores (threshold: < 0.7), then calls Groq to suggest targeted learning resources.

---

## 8. Code Fix — JD Parser Receives File Path, Not Text

**File:** `utils/JD_parser.py`

`app.py` calls `jd_parser.parse_job_description(jd_temp_path)` with a file path, but the method passed the path string directly to spaCy's NLP pipeline as raw text. The result was garbage extraction from a string like `/tmp/tmpXXXXX.pdf`.

Added a `_read_file()` helper that reads PDF, DOCX, or plain-text files before NLP processing:

```python
def _read_file(self, file_path: str) -> str:
    path = Path(file_path)
    if path.suffix.lower() == ".pdf":
        # read with pdfplumber
    elif path.suffix.lower() == ".docx":
        # read with python-docx
    else:
        return path.read_text(encoding="utf-8", errors="ignore")
```

---

## 9. Code Fix — Score Type Mismatch in Summary Metrics

**File:** `app.py` — `display_interview_summary`

`app.py` called `.rstrip('%')` on feedback scores to strip the `%` before converting to float. The LLM sometimes returns scores as integers (`75`) instead of strings (`"75%"`), and integers have no `.rstrip()` method — crashing the summary screen with `AttributeError: 'int' object has no attribute 'rstrip'`.

```python
# Before
float(f.get('relevance', '0').rstrip('%'))

# After — str() handles both int 75 and string "75%"
float(str(f.get('relevance', '0')).rstrip('%'))
```

---

## Summary of All File Changes

| File | Changes |
|------|---------|
| `requirements.txt` | `pandas` unpinned to `>=2.2.0`; spaCy model updated to 3.8.0; `numpy>=2.0.0` added |
| `models/question_generator.py` | Model names updated × 2; regex fix × 2 |
| `models/answer_evaluator.py` | Model name updated; `import re` added; JSON fence stripping; prompt keys aligned to `app.py`; default evaluation keys fixed |
| `models/resource_recommender.py` | Model name updated; `recommend_resources` method added |
| `utils/JD_parser.py` | `_read_file()` helper added; `parse_job_description` updated to read file before parsing |
| `app.py` | `str()` cast added to score metric calculation |

---

## Next Steps — Push to Origin and Open Pull Request

### Step 1 — Add upstream remote (one-time setup)
The `upstream` remote points to the original repository this project was forked from:
```bash
git remote add upstream git@github-account1:niti007/AI-Mock-Interview_.git
git remote -v
# origin    git@github-account1:NaveenBabuBommisetty/AI-Mock-Interview_.git (your fork)
# upstream  git@github-account1:niti007/AI-Mock-Interview_.git              (original repo)
```

### Step 2 — Commit all changes on the fix branch
```bash
git add app.py models/answer_evaluator.py models/question_generator.py \
        models/resource_recommender.py utils/JD_parser.py requirements.txt doc_changes.md

git commit -m "fix: replace deprecated Groq models, fix parsing bugs, and runtime crashes"
```

### Step 3 — Push the fix branch to your fork (origin)
```bash
git push -u origin fix/replace-deprecated-groq-models-and-parsing-bugs
```

### Step 4 — Open a Pull Request
Go to your fork on GitHub and open a PR from `fix/replace-deprecated-groq-models-and-parsing-bugs` targeting the `main` branch of the upstream repo (`niti007/AI-Mock-Interview_`).

**Suggested PR title:** `fix: replace deprecated Groq models and fix all runtime crashes`

**PR body should cover:**
- Replaced 4 deprecated/wrong Groq model names (live-tested against API)
- Fixed regex that silently dropped `1)` and `1-` formatted questions
- Fixed JSON code fence stripping so evaluator scores are never all-zero
- Aligned evaluator prompt keys to match what `app.py` reads
- Added missing `recommend_resources` method (summary screen crash fix)
- Fixed JD parser to read file content instead of passing path as text
- Fixed score type error (`int` has no `.rstrip`) in summary metrics
- Updated `requirements.txt` for Python 3.13 / Apple Silicon compatibility
