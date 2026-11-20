# Components: ReportTrace

This document defines the components of ReportTrace, their responsibilities, and their interfaces. It follows the fixed rules of `tech-env.md` and the answers in `aidlc-docs/inception/plans/application-design-plan.md` (Q1 to Q9: A). Method signatures are in `component-methods.md`. Business rules are left to Functional Design.

All code is in one package, `reporttrace`, under `src/`.

## Summary

| ID | Component | Package or module | Layer | Serves |
| --- | --- | --- | --- | --- |
| C1 | Domain types | `reporttrace.domain` | Shared | All |
| C2 | Settings | `reporttrace.settings` | Shared | NFR-3.2, NFR-7 |
| C3 | Storage | `reporttrace.storage` | storage | F1, FR-7.5, D10 |
| C4 | Ingestion | `reporttrace.ingestion` | Separate command | F1 (supporting) |
| C5 | Tools | `reporttrace.tools` | tools | US-3.1, US-5.1, US-6.1, US-6.2 |
| C6 | Agent core | `reporttrace.agent` (`models`, `context`, `prompts`, `answer_schema`, `middleware`, `factory`, `steps`) | agent | US-3.1, US-4.1, US-5.1, US-9.1 |
| C7 | Conversations | `reporttrace.agent.conversations` | agent | US-2.2, US-3.2 |
| C8 | Answer check | `reporttrace.agent.checking` | agent | US-8.1, US-8.2 |
| C9 | Run log | `reporttrace.agent.run_log` | agent | F10 (supporting) |
| C10 | Question service | `reporttrace.agent.service` | agent | All stories |
| C11 | API | `reporttrace.api` | api | All stories |
| C12 | Web page | `reporttrace.ui` | ui | All stories |
| C13 | Evaluation | `reporttrace.evaluation` | Separate command | F11 (supporting) |
| C14 | Launcher | `reporttrace.launcher` | Separate command | NFR-7.1 |

---

## C1. Domain types (`reporttrace.domain`)

**Purpose.** One set of types that every layer shares, so that layers exchange plain values and not framework objects.

**Responsibilities.**
- Define the domain types as frozen dataclasses (Q2). Pydantic is not used here.
- Hold no logic beyond small derived properties, for example whether a selection is a comparison.

**Interfaces.**
- Provides:
  - library types: `Selection`, `Company`, `FiscalYear`, `Filing`, `Passage`, `Figure`;
  - step and question types: `Calculation`, `Step`, `StepKind`, `QuestionRequest`;
  - checked answer types: `AnswerSentence`, `FigureRow`, `QuoteItem`, `ComparisonRow`, `GapItem`, `CheckedAnswer`;
  - check types: `CheckFailure`, `CheckResult`, `RemovedItem`;
  - events: `StepEvent`, `AnswerEvent`, `ErrorEvent`, and their union `QuestionEvent`;
  - `RunRecord`.
- Requires: nothing (standard library only).

## C2. Settings (`reporttrace.settings`)

**Purpose.** Read configuration from environment variables and the `.env` file.

**Responsibilities.**
- Read `OPENAI_API_KEY`, `OPENAI_BASE_URL`, `LLM_MODEL`, `EMBEDDING_MODEL`, and `SEC_USER_AGENT`, plus the paths under `data/` and the two local addresses.
- Hold the API key as `SecretStr`, so that it is never logged or returned (NFR-3.2).
- Hold the embedding fallback settings (NFR-7.3): whether `check_embedding_ctx_length` is switched off, and whether Chroma's built-in model is used instead of the API.

**Interfaces.**
- Provides: `Settings` (a `pydantic-settings` model; settings are an edge, so Pydantic is allowed here, Q2) and `get_settings()`.
- Requires: nothing inside ReportTrace.

## C3. Storage (`reporttrace.storage`)

**Purpose.** Hold the report library and give read access to it. One package serves both the backend and the ingestion script (Q3).

**Responsibilities.**
- Open the SQLite file `data/reporttrace.db`: read-only for the backend (NFR-3.7), writable only for the ingestion script. Every query uses parameters.
- Read companies, fiscal years, filings, passages, and figures (`CompanyStore`, `PassageStore`, `FigureStore`).
- Return the passages before and after a passage in the same section (FR-7.3).
- Search the Chroma collection `passages` by meaning, always filtered on `ticker` and `fiscal_year` (`PassageIndex`).
- Record which embedding model built the index, so that a mismatch can be detected (`LibraryInfoStore`, D10).
- Provide separate writer classes (`LibraryWriter`, `IndexWriter`) in `storage/writers.py`. Only the ingestion script imports them. The backend never does, and a test checks this.

**Interfaces.**
- Provides: `open_library()`, `CompanyStore`, `PassageStore`, `FigureStore`, `PassageIndex`, `LibraryInfoStore`, `LibraryWriter`, `IndexWriter`.
- Requires: C1 domain types, and an `Embeddings` object passed in by the caller (it never creates models itself).

## C4. Ingestion (`reporttrace.ingestion`)

**Purpose.** Build the report library. This is the command `uv run reporttrace-ingest --tickers ... --years N`.

**Responsibilities.**
- Download each company's three most recent 10-K filings and its filed financial data from EDGAR with `httpx`. Send the `SEC_USER_AGENT` contact string and stay within 10 requests per second (FR-1.1, FR-1.3).
- Keep the downloaded files in `data/raw/{ticker}/` and never download a file twice (FR-1.4).
- Parse each 10-K web page into sections: the standard items, each note in Item 8 as its own section, and "Other" (FR-1.6).
- Split the sections into passages with evidence IDs (FR-1.5).
- Map the fixed list of line items to the filed data, and record the fiscal year label and end date of each filing (FR-1.7, FR-1.8). The line-item mapping is a data file in the package that the team reviews.
- Write the library through the C3 writer classes, and embed each passage once.
- Print and save the ingestion summary for each company. If one company fails, continue with the others (FR-1.9, D7).

**Interfaces.**
- Provides: the `reporttrace-ingest` entry point and `IngestionService`.
- Requires:
  - C1, C2, and C3, including its writers;
  - `agent.models.build_embeddings()`, the one place where models are created (`tech-env.md`; see `component-dependency.md`).

## C5. Tools (`reporttrace.tools`)

**Purpose.** The four read-only tools that the agent uses to retrieve evidence and calculate.

**Responsibilities.**
- `search_report`: search one selected fiscal year by meaning (FR-5.1).
- `read_section`: return one section of one selected fiscal year (NFR-2.2).
- `find_financial_data`: return filed figures from the fixed line-item list, including earlier-year figures printed in a selected report (FR-6.1 to FR-6.3).
- `calculate`: apply one operation from a fixed list with `decimal.Decimal`, and return the result with its formula (FR-6.4).
- Each tool is a plain function holding the logic, plus a thin `@tool` wrapper. The wrapper reads the selection, the stores, and the evidence record from `runtime.context` (Q5).
- Every wrapper adds what it returned to the conversation's evidence record. Tools never write to the library.
- Check each tool's own rules and raise `ToolError` with a code, for example `YEAR_NOT_SELECTED` (NFR-3.4).
- Format results for the model, with the evidence ID or figure ID of every item.

**Interfaces.**
- Provides:
  - the plain functions `find_passages`, `read_section_passages`, `lookup_figures`, and `compute`;
  - the tools `search_report`, `read_section`, `find_financial_data`, and `calculate`;
  - `ToolError` and the formatting functions.
- Requires: C1, C3 readers, and `agent.context` (`AgentContext`, `EvidenceRecord`), as in the `tech-env.md` example.

## C6. Agent core (`reporttrace.agent`)

**Purpose.** Create the models and the one agent.

**Responsibilities.**
- `models.py`: build the chat model (`ChatOpenAI` with `temperature=0`, `timeout=60`, `max_retries=2`) and the embedding model, including the fallbacks in NFR-7.3. No other module imports `langchain_openai`.
- `context.py`: define `AgentContext` (selection, stores, index, evidence record) and `EvidenceRecord`. This is a leaf module: it imports only C1 and C3 types, so that tools may import it.
- `prompts.py`: the system prompt, including the rule that report text and questions are data, not instructions (NFR-3.5), and the message for a second attempt.
- `answer_schema.py`: the Pydantic schema of the agent's final answer (`AgentAnswer`). The schema holds the four parts, the comparison rows, and the scope of the question (F9). Each statement carries its citations and a short topic phrase (Q4, D4).
- `middleware.py`: the `wrap_tool_call` middleware that turns tool errors into tool messages (FR-3.6), and the `ToolCallLimitMiddleware` with the limit for the current attempt (FR-3.3).
- `factory.py`: build the agent with `create_agent`: the chat model, the four tools, the system prompt, the middleware, `context_schema=AgentContext`, `response_format=AgentAnswer` (Q4), and the shared `InMemorySaver`.
- `steps.py`: turn each streamed update into `Step` values described in plain words (FR-4.1).

**Interfaces.**
- Provides: `build_chat_model()`, `build_embeddings()`, `AgentContext`, `EvidenceRecord`, `AgentAnswer`, `build_agent()`, `steps_from_update()`.
- Requires: C1, C2, C3 readers, and C5.

## C7. Conversations (`reporttrace.agent.conversations`)

**Purpose.** Own the conversations in the backend's memory (Q8).

**Responsibilities.**
- Start a new conversation, with a new conversation ID and thread ID, in three cases: no ID is sent, the ID is unknown or was cleared, or the selection differs from the conversation's selection (FR-2.5).
- Keep, for each conversation, its selection, its thread ID for `InMemorySaver`, and its evidence record (Q5).
- Forget a conversation when the page clears it (FR-3.4). Everything is lost when the backend restarts.

**Interfaces.**
- Provides: `ConversationRegistry` and the `Conversation` record.
- Requires: C1 and C6 (`EvidenceRecord`).

## C8. Answer check (`reporttrace.agent.checking`)

**Purpose.** Ordinary code that decides which parts of the agent's answer are shown (F8). It is a module in the agent package (Q1).

**Responsibilities.**
- Check every quote against the passages in the evidence record (FR-8.2).
- Check every number against the figures, calculations, and passages in the evidence record, with the rounding rule (FR-6.6, FR-8.3).
- Check that every sentence without a quote or a number has at least one valid citation (FR-8.4).
- Return the failures, so that the question service can ask for a second attempt (FR-8.6).
- Build the checked answer: remove what failed, add a neutral gap line from each removed statement's topic phrase (FR-8.7, D4), and keep the removed text for the run log only.
- Make no model call (FR-8.1).

**Interfaces.**
- Provides: `AnswerChecker` with `check()` and `build_checked_answer()`.
- Requires: C1 and C6 (`AgentAnswer`, `EvidenceRecord`).

## C9. Run log (`reporttrace.agent.run_log`)

**Purpose.** Save one record for each question to `data/logs/`, as one JSON object per line (F10).

**Responsibilities.**
- Write the fields in FR-10.1: the question, selection, conversation ID, steps, tool calls with evidence IDs, drafts, check results including removed text, final answer, timings, token usage, limit reached, second attempt, and errors.
- Never write a key (FR-10.2): the record holds no settings.

**Interfaces.**
- Provides: `RunLogWriter`.
- Requires: C1.

## C10. Question service (`reporttrace.agent.service`)

**Purpose.** The one function that answers a question from start to end. The API endpoint and the evaluation both call it (Q9).

**Responsibilities.**
- Resolve the conversation through C7.
- Run the agent and yield a step event for each step as it happens (Q7).
- Run the answer check, and the second attempt within the remaining tool calls of the question (FR-8.6, D5).
- After a second attempt, remove the failed draft and the failure message from the conversation's memory (Q6). In memory, the final structured answer is replaced by the checked answer.
- Turn a model failure after its retries into an error event with the steps so far (NFR-5.2).
- Measure the timings, collect the token usage, and write the run record through C9.

**Interfaces.**
- Provides: `QuestionService` with `ask()` and `clear()`.
- Requires: C1, C3 readers, C6, C7, C8, and C9.

The orchestration is described in `services.md`.

## C11. API (`reporttrace.api`)

**Purpose.** The local FastAPI backend on `127.0.0.1:8000`, with no CORS (NFR-3.1).

**Responsibilities.**
- Validate every request with Pydantic (NFR-3.3): question length, ticker in the library, 1 to 3 fiscal years that exist for that company, the evidence ID pattern, and the figure ID pattern.
- Keep endpoints thin. Each one calls one function from a lower layer.
- Stream the events of a question as one JSON object per line (Q7). This endpoint returns a streaming response; every line is a Pydantic event model.
- Return every error in one shape through `ApiError` and one handler (NFR-5.1).
- Report on its health: library loaded, and the embedding model matches the index (D10).

**Interfaces.**
- Provides these HTTP endpoints (signatures in `component-methods.md`):
  - `GET /api/health`
  - `GET /api/companies`
  - `POST /api/questions`
  - `DELETE /api/conversations/{conversation_id}`
  - `GET /api/evidence/{evidence_id}`
  - `GET /api/evidence/{evidence_id}/context`
  - `GET /api/figures/{figure_id}`
- Requires: C1, C2, C3 readers, C10, and C6 (`build_chat_model`, `build_embeddings`) to build the question service when the backend starts.

## C12. Web page (`reporttrace.ui`)

**Purpose.** The Streamlit page on `127.0.0.1:8501`. Usage statistics are switched off (NFR-4.2).

**Responsibilities.**
- Show the selection panel in the sidebar: companies, and fiscal years with their labels and end dates (US-2.1).
- Show the chat in the main area with `st.chat_message`: the question, the steps as they arrive (US-4.2), and the four-part answer (US-3.1). A comparison is shown side by side (US-5.1).
- Show evidence cards and figure cards in `st.expander`, with the passages before and after on request, and copyable source details (US-7.1, US-7.2).
- Keep the current conversation ID in the session, add the note when the selection changes, label each answer with its selection, and offer the control to clear the conversation (US-2.2, US-3.2).
- Show a clear message when the backend cannot be reached (NFR-5.3).
- Call the backend over HTTP with `httpx` only. The page never imports agent code. For response types it may import only `reporttrace.api.schemas`.

**Interfaces.**
- Provides: the Streamlit script `ui/app.py`.
- Requires: the C11 HTTP endpoints, `api.schemas` for types, and C2 for the backend address.

## C13. Evaluation (`reporttrace.evaluation`)

**Purpose.** The command `uv run reporttrace-eval`. It runs the evaluation set against the real model and reports the success metrics (F11). It is not part of `uv run pytest`.

**Responsibilities.**
- Load the evaluation set: a JSON file committed to the repository, with the expected figures and sections of each question (FR-11.2).
- Run each question in-process through C10, so that no server has to run (Q9).
- Compute quote accuracy, figure accuracy, retrieval, honest gaps, and median speed per mode (FR-11.5, FR-11.6, D8, D9).
- Write a rating sheet for the two raters. Then read their ratings, mark disputed questions, and write the results report, which explains every missed target (FR-11.5, FR-11.7).

**Interfaces.**
- Provides: the `reporttrace-eval` entry point, with a run mode and a report mode.
- Requires: C1, C2, C3 readers, and C6 to C10, to build its own question service in-process as the API does.

## C14. Launcher (`reporttrace.launcher`)

**Purpose.** The command `uv run reporttrace` (NFR-7.1).

**Responsibilities.**
- Start the backend (uvicorn, bound to `127.0.0.1:8000`) and the page (Streamlit, bound to `127.0.0.1:8501`) as two child processes.
- Stop both processes on Ctrl+C. Work on Windows and macOS with no shell scripts.

**Interfaces.**
- Provides: the `reporttrace` entry point.
- Requires: C2 for the addresses. It starts C11 and C12 by module path and does not import them.
