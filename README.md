# AI Agents and Workflows for Developers — Exam Project

This repository contains a single exam-style Jupyter notebook that demonstrates a **stateful multi-agent workflow** built with **LangChain** + **LangGraph**.

The project implements an educational **medical symptom assistant** that:
- Routes requests by intent (health vs. non-health)
- Extracts symptoms into strict JSON (with a required tool call)
- Performs lightweight background research via a Wikipedia retrieval tool
- Generates a short, console-friendly recommendation
- Pauses for **Human-in-the-Loop (HITL)** review (approve/revise) before finalizing

## Contents
- Notebook: `AI-Agents-and-Workflows-for-Developers-Exam-Svetoslav-Yavorov.ipynb`
- README: this file

## Requirements

### Runtime
- Google Colab (recommended). The notebook uses `google.colab.userdata` for secrets.

### API keys (Colab Secrets)
Add these in Colab → **Secrets**:
- `OPENAI_API_KEY` (required)
- `LANGSMITH_API_KEY` (optional, for tracing). The notebook enables LangSmith tracing by default.

### Models
The notebook configures three `ChatOpenAI` clients:
- `gpt-5-nano` (symptom extractor)
- `gpt-5-nano` (medical researcher)
- `gpt-5-mini` (report generator)

## Quickstart (Google Colab)
1. Open the notebook in Colab.
2. Add the secrets above.
3. Run the first code cell to install dependencies:
   - `langchain`, `langchain-openai`, `langchain-community`, `langgraph`, `wikipedia`
4. Run the remaining cells (or **Runtime → Run all**).

At the end, the notebook executes multiple test scenarios to demonstrate routing, tool usage, and the HITL review loop.

## What the notebook demonstrates (exam checklist)
- **Stateful LangGraph** using an explicit `MedicalState` contract (a `TypedDict`) passed between nodes.
- **Multiple agents with distinct roles** via separate system prompts:
  - Intent Router (health vs. non-health)
  - Symptom Extractor (structured JSON)
  - Medical Researcher (Wikipedia-backed JSON)
  - Report Generator (strict final format)
- **Tool-augmented agents** with explicit tool execution using `ToolNode` and conditional routing on pending tool calls.
- **Memory / checkpointing** via `InMemorySaver`.
- **Human-in-the-Loop interruption** via LangGraph `interrupt()` with approve/revise and a bounded revision loop.
- A single entry point function: `execute_workflow(user_request: str)` that runs, handles interruptions, and resumes.

## Architecture (high level)

### State
The workflow passes a single state object between nodes. Key fields include:
- `messages` (accumulated LangChain chat messages)
- `intent` (`health` | `non_health`)
- `extracted_data` (symptoms, duration, age, red flags)
- `possible_conditions` + `researcher_notes`
- `medical_report` (draft)
- `final_recommendation` (approved output)

### Tools (2 total)
- `symptom_severity_tool`: deterministic helper used by the Symptom Extractor to estimate severity.
- `wikipedia_medical_lookup`: safe Wikipedia retriever wrapper with query normalization and failure-safe responses.

### Graph nodes
The LangGraph `StateGraph` is assembled with explicit nodes and edges:
- `intent_guard` → routes to either:
  - `symptom_extractor` (health)
  - `non_medical_finalize` (non-health)
- `symptom_extractor` ↔ `symptom_tools` → `symptom_postprocess`
- `medical_researcher` ↔ `medical_researcher_tools` → `medical_researcher_postprocess`
- `report_generator` → `medical_expert_review` (HITL)
- Conditional loop: `medical_expert_review` → (`finalize` or back to `report_generator`)

The notebook also renders a visual graph image for quick inspection.

## Human-in-the-Loop (HITL)
Before producing the final output, the workflow interrupts for “Medical Expert Review”. The human can:
- Type `approve` / `ok` to accept the draft
- Type any other text to request revision; the text becomes revision guidance

The revision loop is capped (auto-approves after a small max revision count) to keep runs bounded.

For automated testing, the notebook supports a preset decision list (so tests can run without manual typing).

## Running the workflow

### Public entry point
Use `execute_workflow(user_request: str, print_whole_conversation: bool = False)`.

Notes:
- The function prints the final recommendation to the output.
- If HITL is enabled and no preset decisions are provided, it will pause and prompt for input.

### Example requests
- “Hello, how are you?” (non-health routing)
- “I have had a headache and fever for 2 days…”
- “I have chest pain and shortness of breath…” (tests red-flag handling)

## Tests / demo scenarios
The final section of the notebook runs multiple scenarios to showcase:
- Intent routing
- Structured extraction → tool call → parsing
- Wikipedia retrieval tool usage
- HITL approve and revise paths

## Troubleshooting
- **Missing secrets**: ensure `OPENAI_API_KEY` (and optionally `LANGSMITH_API_KEY`) exist in Colab Secrets.
- **Wikipedia tool returns empty**: the notebook intentionally fails safe with a short fallback string so the pipeline continues.
- **Running outside Colab**: the notebook uses `google.colab.userdata`; adapting for local Jupyter would require changing the secret-loading cell.

## Disclaimer
This project is **educational only** and does **not** provide medical advice, diagnosis, or treatment. If you have urgent symptoms, seek professional help.