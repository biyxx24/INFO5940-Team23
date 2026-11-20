# Application Design: ReportTrace

This document brings the application design together in one place. The detailed documents in the same folder are the reference for each topic:

| Document | Content |
| --- | --- |
| `components.md` | The 14 components: purpose, responsibilities, interfaces |
| `component-methods.md` | Domain types and method signatures; endpoint table |
| `services.md` | The four services and their orchestration; conversation lifecycle |
| `component-dependency.md` | Dependency rules and matrix, communication patterns, diagrams |

Business rules (for example passage splitting, quote matching, value choice, and rounding) are designed in Functional Design for each unit in Construction.

---

## 1. Design Decisions

The fixed rules of `tech-env.md` are listed in Section 2 of `aidlc-docs/inception/plans/application-design-plan.md`. The team decided the open points as follows (Q1 to Q9, all A):

| Question | Decision |
| --- | --- |
| Q1 Answer check | A module inside the agent package (`agent/checking.py`), called by the question service |
| Q2 Domain types | Frozen dataclasses; Pydantic only at the edges (API schemas, settings, the agent's answer schema) |
| Q3 Storage | One `storage` package for the backend and the ingestion script. Writer classes live in `storage/writers.py`, and the backend never imports them |
| Q4 Final answer | Structured output: `create_agent(..., response_format=AgentAnswer)`. Each statement carries its citations and a topic phrase. Tested early through the Cornell API |
| Q5 Evidence | An in-memory evidence record for each conversation, passed in `AgentContext`. Every tool wrapper adds what it returned to it |
| Q6 Memory after a second attempt | The failed draft and the retry message are removed from the thread. The final answer in the thread is the checked answer. Tool results stay |
| Q7 Steps to the page | Streamed from the start: `POST /api/questions` returns one JSON object per line |
| Q8 Conversations | Owned by the backend. A new conversation starts when the ID is missing or unknown, or when the selection changes |
| Q9 Evaluation | Runs in-process through the same question service as the API |

## 2. Architecture Overview

```text
+-------------------------------------------------------------------+
|  Laptop (Windows 11 or macOS), started by: uv run reporttrace     |
|                                                                   |
|  +---------------------------+      +--------------------------+  |
|  | Process 1: Streamlit page |      | Process 2: FastAPI       |  |
|  | 127.0.0.1:8501            | HTTP | 127.0.0.1:8000           |  |
|  | C12 ui                    |----->| C11 api                  |  |
|  +---------------------------+      |  C10 question service    |  |
|                                     |  C7 conversations        |  |
|                                     |  C6 agent core, C8 check |  |
|                                     |  C9 run log, C5 tools    |  |
|                                     |  C3 storage (read-only)  |  |
|                                     +--------------------------+  |
|                                                |                  |
|  Separate commands:                            v                  |
|  reporttrace-ingest (C4, writes C3)   data/reporttrace.db         |
|  reporttrace-eval   (C13, in-process) data/chroma  data/logs      |
+-------------------------------------------------------------------+
         |                                       |
         v                                       v
   SEC EDGAR (ingestion only)          Model API (Cornell; chat and embeddings)
```

Layers go one way: `ui` to `api` to `agent` to `tools` to `storage`. Three narrow exceptions are allowed, each required by `tech-env.md`:
- tools import `agent/context.py`;
- ingestion imports `agent/models.py`;
- the page imports `api/schemas.py`.

## 3. Components

| ID | Component | Package or module | Responsibility in one line |
| --- | --- | --- | --- |
| C1 | Domain types | `domain` | Frozen dataclasses shared by every layer |
| C2 | Settings | `settings` | Environment and `.env`, key as `SecretStr`, embedding fallbacks |
| C3 | Storage | `storage` | Read-only stores for companies, filings, passages, and figures; Chroma search; writer classes for ingestion only |
| C4 | Ingestion | `ingestion` | Download, parse items and notes, split, map line items, write, embed, report |
| C5 | Tools | `tools` | `search_report`, `read_section`, `find_financial_data`, `calculate`; each one fills the evidence record |
| C6 | Agent core | `agent` | Model creation, context, prompt, answer schema, middleware, agent factory, step descriptions |
| C7 | Conversations | `agent.conversations` | Conversation IDs, thread IDs, selection, and evidence record for each conversation |
| C8 | Answer check | `agent.checking` | Check quotes, numbers, and citations; build the checked answer and the gap lines |
| C9 | Run log | `agent.run_log` | One JSON line per question in `data/logs/` |
| C10 | Question service | `agent.service` | Answers one question from start to end |
| C11 | API | `api` | Thin endpoints, validation, the streaming question endpoint, one error shape |
| C12 | Web page | `ui` | Selection panel, chat, steps, answers, evidence and figure cards |
| C13 | Evaluation | `evaluation` | Run the evaluation set in-process, compute the metrics, ratings, report |
| C14 | Launcher | `launcher` | Starts and stops the two processes |

## 4. Interfaces Between the Page and the Backend

| Endpoint | Purpose |
| --- | --- |
| `GET /api/health` | Library loaded; embedding model matches the index (D10) |
| `GET /api/companies` | Companies with fiscal years, labels, and end dates (US-2.1) |
| `POST /api/questions` | Ask a question; streams step events, then one answer or error event (US-3.1, US-4.1, US-4.2) |
| `DELETE /api/conversations/{conversation_id}` | Clear a conversation (US-3.2) |
| `GET /api/evidence/{evidence_id}` | A passage for an evidence card (US-7.1) |
| `GET /api/evidence/{evidence_id}/context` | The passages before and after (US-7.1 AC3) |
| `GET /api/figures/{figure_id}` | A figure for a figure card (US-7.2) |

Every error has the shape `{"error": {"code": "...", "message": "..."}}`.

## 5. Services

| Service | What it does |
| --- | --- |
| S1 Question service | Resolve the conversation, then stream the agent and yield steps. Check the answer, with at most one second attempt within the remaining tool calls. Clean the memory, write the run log, and yield the checked answer or an error |
| S2 Ingestion service | For each ticker: download (cached), parse, split, map, write, embed, and report. Continue after a failure |
| S3 Evaluation service | Run mode: answer every question in-process and write the results and a rating sheet. Report mode: add the ratings, mark disputed questions, and write the metrics report |
| S4 Launcher | Start uvicorn and Streamlit on `127.0.0.1`, and stop both on Ctrl+C |

The ordered steps of S1 are in `services.md`. The data flow diagrams are in `component-dependency.md` Section 4.

## 6. Traceability

### Stories to components

| Story | Main components |
| --- | --- |
| US-2.1 Choose the company and years | C12, C11 (`/api/companies`, request validation), C3 |
| US-2.2 Change the selection | C12 (note and labels), C7 (new conversation), C10 |
| US-3.1 Ask a question | C12, C11 (`/api/questions`), C10, C6, C5 |
| US-3.2 Follow-up question | C7, C10, C6 (`InMemorySaver`), C12 (clear control) |
| US-4.1 Steps with the answer | C6 (`steps.py`), C10, C12 |
| US-4.2 Live steps | C11 (streaming), C12 (reads the stream) |
| US-5.1 Year comparison | C5 (one year per search), C6 (`AgentAnswer.comparison`), C12 |
| US-6.1 Reported figures | C5 (`find_financial_data`), C3 (`FigureStore`), C4 (line-item map) |
| US-6.2 Calculations | C5 (`calculate`) |
| US-7.1 Passage cards | C12, C11 (`/api/evidence`), C3 (`PassageStore.neighbours`) |
| US-7.2 Figure cards | C12, C11 (`/api/figures`), C3 (`FigureStore`) |
| US-8.1 Checked statements | C8, C10 (second attempt), C5 (evidence record) |
| US-8.2 Gap list | C8 (gap lines), C9 (removed text) |
| US-9.1 Outside the reports | C6 (system prompt, `AgentAnswer.scope`), C8 |

### Supporting requirements to components

| Supporting requirement | Components |
| --- | --- |
| F1 Report library (FR-1.1 to FR-1.10) | C4, C3 |
| F10 Run log (FR-10.1, FR-10.2, FR-8.8) | C9, C10 |
| F11 Evaluation (FR-11.1 to FR-11.7) | C13, C10 |

### Cross-cutting constraints to components

| NFR | Where it is placed |
| --- | --- |
| NFR-1 Performance | C6 (`timeout=60`, `max_retries=2`, `temperature=0`); C9 (timings) |
| NFR-2 API usage | C3 and C4 (embed once); C5 (`read_section` returns one section); C10 (one shared limit for both attempts); C9 (token usage) |
| NFR-3 Security | C11 (validation, `127.0.0.1`, no CORS); C2 (`SecretStr`); C3 (read-only, parameterised queries, writers kept apart); C5 (`ToolError`, no `eval`); C6 (system prompt treats report text as data) |
| NFR-4 Privacy | C6 (only the question, passages, and figures go to the model); C12 and C14 (Streamlit usage statistics off); no tracing anywhere |
| NFR-5 Reliability | C11 (one error shape); C10 (`ErrorEvent`); C12 (backend unavailable message) |
| NFR-6 Maintainability and testability | Every component: plain functions behind tool wrappers; models passed in as `BaseChatModel` and `Embeddings`, so that tests use fakes |
| NFR-7 Portability and LLM access | C14 (`sys.executable`, no shell scripts); C6 `models.py` and C2 (embedding fallbacks); C3 (`LibraryInfoStore`, D10) |
| NFR-8 Usability | C12 |

## 7. Completeness and Consistency Check

| Check | Result |
| --- | --- |
| Every story US-2.1 to US-9.1 is served by at least one component | Pass (Section 6) |
| Every supporting requirement (F1, F10, F11) is served | Pass (Section 6) |
| Every dependency follows the one-way layer order, apart from the three exceptions in `component-dependency.md` Section 1 | Pass (dependency matrix) |
| One agent from `create_agent`, four tools, no custom graph, no sub-agents | Pass (C6) |
| Retrieval only through tools; tools never write to the library | Pass (C5, C3) |
| Only `agent/models.py` creates models | Pass (C6; C4 and C13 call its builders) |
| No library from the "Do NOT Use" table of `tech-env.md` | Pass: the design uses only LangChain 1.x, `langchain-openai`, `langchain-chroma`, FastAPI, Pydantic, uvicorn, Streamlit, `sqlite3`, `httpx`, `beautifulsoup4` with `lxml`, `pydantic-settings`, and `logging` |
| AI-DLC extensions | N/A: all three are disabled |

## 8. Notes for Functional Design

These points are implementation details, not open decisions. Functional Design of the unit concerned settles each one, and testing confirms it early in Construction.

1. **Structured output through the Cornell API (Q4).** Test `response_format=AgentAnswer` together with the embedding test (NFR-7.3), before the agent unit is built on it. If it does not work, the team decides on a change through the workflow. The marked-up text option of Q4 was the alternative.
2. **Tool-call limit.** Two points need confirming: whether `ToolCallLimitMiddleware` counts the structured-output step as a tool call, and which of its options lets the agent still write an answer once the limit is reached (FR-3.5). The limit for the question must stay at 10 real tool calls across both attempts.
3. **Cleaning the memory (Q6).** Confirm how the failed draft and the retry message are removed from the thread, and how the final answer is replaced with the checked answer in the thread.
4. **Streaming endpoint (Q7).** `POST /api/questions` returns a streaming response instead of one Pydantic model. This is the only exception to the endpoint pattern of `tech-env.md`, and each line is still a Pydantic event schema.
5. **Figure ID pattern.** Define the pattern of figure IDs, which the API validates as it does evidence IDs.
6. **Line-item map.** Define the format of the data file that maps the fixed line items to the filed data (FR-1.8).
