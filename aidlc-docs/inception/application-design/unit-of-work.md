# Units of Work: ReportTrace

ReportTrace is split into five units of work. Construction builds them one at a time, in order. Every unit is a set of modules inside the one package `reporttrace`; there are no separately deployed services. The decisions behind this split are recorded in `aidlc-docs/inception/plans/unit-of-work-plan.md` (Q1 to Q8, FU1, FU2).

Related documents:
- `unit-of-work-dependency.md`: what each unit uses from earlier units, and the integration tests at each boundary;
- `unit-of-work-story-map.md`: every acceptance criterion assigned to the unit that implements it.

## 1. Summary

| Unit | Name | Owner | Components (see `components.md`) | Primary unit of | Supporting requirements |
| --- | --- | --- | --- | --- | --- |
| U0 | Project foundation | All four members | Project setup, C1 domain types, C2 settings, `agent/models.py` (part of C6), shared test fakes, the Cornell API check | none | NFRs shared by all units |
| U1 | Report library | Zheng Kang | C3 storage, C4 ingestion | none | F1 |
| U2 | Evidence | Chengcheng Chen | C5 tools, `agent/context.py`, `agent/answer_schema.py` (parts of C6), C8 answer check | US-6.1, US-6.2, US-8.1 | none |
| U3 | Agent and question service | Xihe Xu | The rest of C6, C7 conversations, C9 run log, C10 question service | US-3.2, US-4.1, US-8.2, US-9.1 | F10 |
| U4 | API, web page, launcher, and evaluation | Jiayi Luo | C11 API, C12 web page, C13 evaluation, C14 launcher | US-2.1, US-2.2, US-3.1, US-4.2, US-5.1, US-7.1, US-7.2 | F11 |

The primary unit of a story is defined in `unit-of-work-story-map.md` Section 1.

---

## 2. Unit Definitions

### U0. Project foundation

- **Owner:** all four members together (FU1).
- **Purpose:** everything that every later unit builds on, and an early check of the two Cornell API features that later units depend on.

**Contents.**
- Project setup:
  - `pyproject.toml`: Python `>=3.12,<3.13`, the dependencies, the three entry points, ruff with line length 100, mypy, pytest with pytest-cov;
  - `uv.lock`;
  - `.env.example` with the variable names and no values;
  - `.gitignore` covering `.env` and `data/`.
- C1 domain types (`domain.py`) and C2 settings (`settings.py`).
- `agent/models.py`, the only module that creates the chat model and the embedding model, with the fallbacks of NFR-7.3 (FU2).
- Shared test fakes in `tests/fakes/`:
  - `FakeToolCallingModel`, a subclass of `GenericFakeChatModel` whose `bind_tools` returns the model itself;
  - a helper for `DeterministicFakeEmbedding`;
  - a small sample 10-K web page and a sample financial data file;
  - `httpx.MockTransport` helpers for SEC downloads.
- **The Cornell API check, as the first task (Q6, FU2).** A small diagnostic, run by hand with `uv run`, that checks three things:
  1. embeddings work through the API, trying `check_embedding_ctx_length=False` and then Chroma's built-in model if needed;
  2. GPT-4o tool calling works;
  3. `create_agent` with `response_format` and one tool returns a structured response.

  The result is recorded in U0's code summary. If a check fails, the team decides on a change through the workflow before going on. The module name and the command are set in U0 Functional Design.

**Requirements.** NFR-1.2, NFR-3.2, NFR-3.8, NFR-4.2 (no tracing), NFR-6.1, NFR-6.3, NFR-6.6, NFR-6.9, NFR-6.10, NFR-7.2, NFR-7.3, NFR-7.5.

**Tests.** Unit tests for the domain types, the settings (the key stays a `SecretStr` and never appears in `repr`), and `models.py` (it builds the configured objects without any network call).

**Done when.**
- `uv sync` installs the project on Windows and macOS.
- `uv run ruff check .`, `uv run mypy src`, and `uv run pytest` pass.
- The Cornell API check has been run, and its result has been recorded and reviewed by the team.

### U1. Report library

- **Owner:** Zheng Kang.
- **Purpose:** build the report library and give read access to it.

**Contents.** C3 storage and C4 ingestion. In the tests: `FakePassageIndex` and a helper that builds a small library in `tmp_path` from the sample report. Later units use both.

**Requirements.** F1 (FR-1.1 to FR-1.10), D7, D10 (recording the embedding model), NFR-2.1, NFR-3.7, NFR-4.3, NFR-7.4. Acceptance criteria: US-7.1 AC5 (each passage stores the filing link on sec.gov). Most of this unit is the supporting requirement F1, which every story depends on.

**Notes for Functional Design.**
- The section and note recognition rules (FR-1.6).
- The passage splitting rule and the evidence ID scheme (FR-1.5).
- The format of the line-item map (`application-design.md` Section 8, note 6).
- The figure ID pattern (note 5).
- The rule for storing values that were filed more than once, which `lookup_figures` in U2 needs for FR-6.3 and D1.

**Tests.**
- Unit tests for the EDGAR client (with `httpx.MockTransport`), the parser, the splitter, the figure mapper, and every store.
- An integration test that runs the whole ingestion on the sample report with fake embeddings and reads the result back (Q5).

**Done when.**
- The general criteria in Section 3 hold.
- The demo library of ten companies with three fiscal years each has been built with `uv run reporttrace-ingest`.
- The ingestion summary has been reviewed, and any company that cannot be parsed has been replaced and recorded (FR-1.2, FR-1.9).

### U2. Evidence

- **Owner:** Chengcheng Chen.
- **Purpose:** the four read-only tools, which retrieve and calculate evidence, and the answer check, which decides what of the agent's answer is supported by that evidence.

**Contents.**
- C5 tools: the plain functions, the `@tool` wrappers, `ToolError`, the formatting, and `ALL_TOOLS`.
- `agent/context.py`: `AgentContext` and `EvidenceRecord`.
- `agent/answer_schema.py`: `AgentAnswer`.
- C8 answer check: `agent/checking.py`.

**Requirements.** FR-5.1, FR-6.1 to FR-6.6, FR-8.1 to FR-8.4, FR-8.7 (gap lines), D1, D2, D4, NFR-2.2, NFR-3.4, NFR-3.6, NFR-6.4, NFR-6.8. Acceptance criteria: 22, listed in `unit-of-work-story-map.md`.

**Notes for Functional Design.**
- The fixed list of `calculate` operations.
- The output text format of each tool.
- The fields of `AgentAnswer`.
- The quote normalisation rule (spacing and quotation marks).
- The number matching and rounding rule.
- How the neutral gap line is built from a topic phrase.

**Tests.**
- Unit tests for every plain function and for the check.
- `hypothesis` property-based tests for `calculate` and the answer check (NFR-6.8).
- An integration test that runs the tools on the small library from U1 and checks an answer against the evidence that the tools returned (Q5).

**Done when.** The general criteria in Section 3 hold.

### U3. Agent and question service

- **Owner:** Xihe Xu.
- **Purpose:** the one agent and the service that answers a question from start to end.

**Contents.**
- The rest of C6: `prompts.py`, `middleware.py`, `factory.py`, `steps.py`.
- C7 `conversations.py`, C9 `run_log.py`, and C10 `service.py`.

**Requirements.** FR-2.5 (new conversation), FR-3.2 to FR-3.6, FR-4.1, FR-4.2, FR-8.5, FR-8.6, FR-8.8, FR-9.1, FR-9.2, F10, D5, D6, D11, NFR-1.3, NFR-2.3, NFR-2.5, NFR-3.5, NFR-4.1, NFR-5.2. Acceptance criteria: 23, listed in `unit-of-work-story-map.md`.

**Notes for Functional Design** (`application-design.md` Section 8, notes 2 and 3).
- How `ToolCallLimitMiddleware` counts the structured-output step, and which of its options lets the agent still answer at the limit.
- How the failed draft and the retry message are removed from the thread.
- The wording of the system prompt and of the plain-language step descriptions.
- The fields of `RunRecord`.

**Tests.**
- Unit tests for the middleware, the step descriptions, the conversation registry, the run log, and the service with `FakeToolCallingModel`.
- An integration test of the service with the real tools, the real check, and the small library. It covers:
  - a single-year question and a comparison;
  - a tool error;
  - the tool-call limit;
  - a second attempt;
  - a follow-up question;
  - a change of selection;
  - a model failure (Q5).

**Done when.** The general criteria in Section 3 hold.

### U4. API, web page, launcher, and evaluation

- **Owner:** Jiayi Luo.
- **Purpose:** everything the user and the team use directly: the backend, the page, the start command, and the evaluation command.

**Contents.** C11 API, C12 web page, C13 evaluation, and C14 launcher. This includes switching off Streamlit usage statistics (NFR-4.2).

**Requirements.** FR-2.1 to FR-2.4, FR-3.1, FR-3.7, FR-4.3, FR-5.2, FR-7.1 to FR-7.5, F11 (FR-11.4 to FR-11.7, the evaluation script), D3, D8, D9, D10 (health check), NFR-1.1 (measured), NFR-3.1, NFR-3.3, NFR-5.1, NFR-5.3, NFR-7.1, NFR-8. Acceptance criteria: 31, listed in `unit-of-work-story-map.md`.

**Notes for Functional Design** (`application-design.md` Section 8, note 4).
- The event schemas of the streaming endpoint.
- The page layout of the four-part answer and of the side-by-side comparison.
- The location and format of the evaluation set file.
- The rating sheet format.

**Tests.**
- Integration tests of every endpoint with FastAPI `TestClient`, the real service, the fake model, and the small library. These include the stream, the evidence and figure cards, and the error shape.
- A Streamlit `AppTest` smoke test: the page loads and the selection panel lists the companies.
- Unit tests of the evaluation metrics, and a test of the runner on a few sample questions with the fake model (Q5).

**Done when.**
- The general criteria in Section 3 hold.
- `uv run reporttrace` starts both processes on `127.0.0.1` and stops both on Ctrl+C.

---

## 3. General "Done When" Criteria for Every Unit

1. Functional Design and Code Generation of the unit are approved through the workflow.
2. Every acceptance criterion assigned to the unit in `unit-of-work-story-map.md` has a test that passes.
3. The integration tests at the unit's boundary with the units already built pass (Q5).
4. `uv run ruff check .`, `uv run mypy src`, and `uv run pytest` pass. Tests use no network (NFR-6.9).
5. Line coverage is at least 80% for the unit's code in `tools`, `agent`, `storage`, and `ingestion` (NFR-6.7).
6. The unit's owner has reviewed its code and tests. For U0, all four members review together.

---

## 4. Construction Order and Parallel Team Work

**Construction order** (Q4): U0, U1, U2, U3, U4, one after another, then Build and Test. Each unit's design and code are completed and approved before the next unit starts. If time runs short, US-4.2 (live steps, Should) is cut first. US-4.1 still holds without it.

**Work that runs in parallel with Construction:**
- **Evaluation set (Q8).** From the first week of Construction, all four members write the evaluation questions with their expected figures and sections, and a different member checks each one against the filing (FR-11.1 to FR-11.3). U4 builds only the evaluation script and its tests. The full evaluation run against the real model happens in Build and Test.
- **Reviews.** The owner of each unit reviews that unit's design, code, and tests as Construction reaches it.

**Rule for changes.** Once a unit is approved, its interfaces (the signatures in `component-methods.md` and the decisions in its Functional Design) change only through the workflow, because later units build on them.

---

## 5. Code Organization Strategy

This is a greenfield project with several units in one package. `tech-env.md` fixes a `src/` layout with the one package `reporttrace`, so each unit owns a set of modules, not a directory of its own. Tests are grouped by type, as in the `tech-env.md` examples. Application code is at the workspace root, never in `aidlc-docs/`.

```text
INFO5940-Team23/
+-- pyproject.toml                        U0
+-- uv.lock                               U0
+-- .env.example                          U0
+-- .gitignore                            U0
+-- src/
|   +-- reporttrace/
|       +-- __init__.py                   U0
|       +-- domain.py                     U0
|       +-- settings.py                   U0
|       +-- (Cornell API check module)    U0, name set in U0 Functional Design
|       +-- launcher.py                   U4
|       +-- storage/                      U1
|       |   +-- __init__.py
|       |   +-- database.py
|       |   +-- companies.py
|       |   +-- passages.py
|       |   +-- figures.py
|       |   +-- vectors.py
|       |   +-- library_info.py
|       |   +-- writers.py
|       +-- ingestion/                    U1
|       |   +-- __init__.py
|       |   +-- edgar.py
|       |   +-- parser.py
|       |   +-- splitter.py
|       |   +-- figures.py
|       |   +-- line_items.json
|       |   +-- pipeline.py
|       |   +-- cli.py
|       +-- tools/                        U2
|       |   +-- __init__.py
|       |   +-- errors.py
|       |   +-- formatting.py
|       |   +-- search_report.py
|       |   +-- read_section.py
|       |   +-- find_financial_data.py
|       |   +-- calculate.py
|       +-- agent/
|       |   +-- __init__.py               U0
|       |   +-- models.py                 U0
|       |   +-- context.py                U2
|       |   +-- answer_schema.py          U2
|       |   +-- checking.py               U2
|       |   +-- prompts.py                U3
|       |   +-- middleware.py             U3
|       |   +-- factory.py                U3
|       |   +-- steps.py                  U3
|       |   +-- conversations.py          U3
|       |   +-- run_log.py                U3
|       |   +-- service.py                U3
|       +-- api/                          U4
|       |   +-- __init__.py
|       |   +-- app.py
|       |   +-- errors.py
|       |   +-- dependencies.py
|       |   +-- schemas.py
|       |   +-- routers/
|       |       +-- __init__.py
|       |       +-- health.py
|       |       +-- library.py
|       |       +-- questions.py
|       |       +-- evidence.py
|       +-- ui/                           U4
|       |   +-- __init__.py
|       |   +-- app.py
|       |   +-- client.py
|       |   +-- views.py
|       +-- evaluation/                   U4
|           +-- __init__.py
|           +-- question_set.py
|           +-- runner.py
|           +-- metrics.py
|           +-- report.py
|           +-- cli.py
+-- tests/
    +-- conftest.py                       U0
    +-- fakes/
    |   +-- fake_model.py                 U0
    |   +-- embeddings.py                 U0
    |   +-- http.py                       U0
    |   +-- sample_report/                U0
    |   +-- fake_index.py                 U1
    |   +-- library.py                    U1
    +-- unit/
    |   +-- test_domain.py                U0
    |   +-- test_settings.py              U0
    |   +-- storage/                      U1
    |   +-- ingestion/                    U1
    |   +-- tools/                        U2
    |   +-- agent/
    |   |   +-- test_models.py            U0
    |   |   +-- test_context.py           U2
    |   |   +-- test_answer_schema.py     U2
    |   |   +-- test_checking.py          U2
    |   |   +-- test_middleware.py        U3
    |   |   +-- test_steps.py             U3
    |   |   +-- test_conversations.py     U3
    |   |   +-- test_run_log.py           U3
    |   |   +-- test_service.py           U3
    |   +-- api/                          U4
    |   +-- evaluation/                   U4
    |   +-- test_launcher.py              U4
    +-- integration/
    |   +-- test_ingestion.py             U1
    |   +-- test_tools_on_library.py      U2
    |   +-- test_question_service.py      U3
    |   +-- test_api.py                   U4
    |   +-- test_evaluation_runner.py     U4
    +-- ui/
        +-- test_page_smoke.py            U4
```

Not in the tree:
- `data/`, which Git ignores and each member builds with the ingestion script;
- the evaluation set file, which the team writes in parallel and whose location is set in U4 Functional Design.

Exact file names inside a unit may change in its Functional Design. The owner unit of each module may not change without a workflow change.
