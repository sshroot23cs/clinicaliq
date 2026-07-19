# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

ClinicalIQ is an AI patient guidance assistant for "Apollo Health Clinic" (a fictional clinic), built as a Launchpad
project for the Agentic AI Engineering course. The course releases starter code one **session folder** at a time
(`s01/`, `s02/`, ... up to `s17`), each adding a new capability on top of the last via TODOs that get filled in.
Currently only `s01/` (Session 1: basic conversational agent) exists.

Two reference docs drive all session work — read them before implementing a new user story:
- `clinicaliq-prd.md` — full product spec, user stories (US-00, US-01, ...), acceptance criteria, and test inputs
- `ai-glossary.md` — terminology used in the PRD, introduced in the order it first appears in the course

## Commands

```bash
# One-time setup (from repo root)
pip install -r requirements.txt
copy .env.example .env          # Windows; fill in GROQ_API_KEY at minimum

# Run the current session's agent (each session is its own package)
cd s01
python -m clinicaliq.agent
```

There is no test suite, lint config, or build step yet, despite `pytest`/`pytest-mock`/`pytest-asyncio` being in
`requirements.txt` (they're there in advance for later sessions).

## Architecture

Each session folder (`s01/clinicaliq/`, and future `sNN/clinicaliq/`) is a self-contained LangGraph app with the same
file layout and the same responsibility split — later sessions extend this pattern, they don't replace it:

- `__init__.py` — runs `load_dotenv()` on package import, before any other module reads `os.environ`. This is why
  API keys are readable in `config.py`/`tools.py` without each of them loading `.env` themselves.
- `config.py` — pure constants: model name/temperature/max_tokens and `SYSTEM_PROMPT`. No API calls, no logic. This
  is the file to edit when tuning agent behavior/tone/rules — not `nodes.py`.
- `state.py` — the `TypedDict` schema (`ClinicalIQState`) that flows through the graph. Nodes read the full state and
  return a partial dict of only the keys they changed; LangGraph merges it in.
- `tools.py` — LLM client construction (`ChatGroq`) and, from later sessions on, `@tool` functions for querying the
  SQLite database. Fails fast at import time if `GROQ_API_KEY` is missing.
- `nodes.py` — the node functions registered on the graph (e.g. `respond`). Each node wraps its LLM/tool call in
  try/except and returns a safe fallback response on failure rather than letting exceptions propagate — patient-facing
  code must never crash mid-conversation.
- `agent.py` — builds the `StateGraph` (`build_graph()`), exposes the compiled `graph` at module level (required by
  `langgraph.json` for LangGraph Studio: `"clinicaliq": "./clinicaliq/agent.py:graph"`), and runs the terminal REPL loop.

Session 1's graph is intentionally trivial: `START -> respond -> END`. Later sessions add nodes/edges (routing,
tool-calling, RAG retrieval, memory checkpointing) onto this same structure — check the PRD user story before
assuming a new node belongs elsewhere.

### Non-negotiable compliance boundary

ClinicalIQ must never give a medical diagnosis, recommend medication, or assess symptoms. Emergencies get redirected
to "call 112 or go to the nearest ER"; anything symptom/diagnosis/medication-related gets redirected to "please speak
with our nurse." Department navigation ("which doctor for a knee problem?" -> Orthopaedics) is in scope; diagnosing
the problem is not. This rule lives in `SYSTEM_PROMPT` (`config.py`) and is enforced entirely through prompting in
Session 1 — do not weaken or remove it when editing the prompt.

### Data layer (introduced in later sessions, not yet present)

Per the PRD (US-00), the project will grow two data sources that `data/seed.py` and `data/ingest.py` populate:
- **SQLite** (`data/clinic_data.db`) — doctors, services, health_packages, price_history; queried via tool calls.
  Prices/fees live *only* here, never in markdown docs.
- **ChromaDB** (`data/vectorstore/`) — RAG over the markdown docs already present in `data/documents/`
  (`appointment_guide.md`, `departments_overview.md`, `test_preparation.md`, `privacy_policy.md`, `faq.md`).

On Windows, ChromaDB persistence paths must use forward slashes or `pathlib.Path`, not backslashes.

## Working style for this repo

Each session's `CLAUDE_CODE_PROMPTS.md` (e.g. `s01/CLAUDE_CODE_PROMPTS.md`) contains the intended prompts a course
participant would type to fill in that session's TODOs, plus the reasoning behind each one. When asked to complete a
TODO, prefer making the smallest change that satisfies it and matches the surrounding file's existing style — these
files are teaching artifacts, and unrelated edits work against the lesson.
