# Application Design Plan: ReportTrace

## Purpose

This plan sets how the components of ReportTrace, their methods, the services that orchestrate them, and their dependencies are designed. Detailed business rules, for example how passages are split or how quotes are matched, are left to Functional Design for each unit in Construction.

**Inputs:** `requirements.md`, `stories.md`, `personas.md`, `execution-plan.md`, and the fixed rules of `tech-env.md`.

**Outputs** (in `aidlc-docs/inception/application-design/`):

- `components.md`
- `component-methods.md`
- `services.md`
- `component-dependency.md`
- `application-design.md`, which consolidates the four documents above

---

## 1. Planning Progress

- [X] Analyse the context: requirements, stories, and the fixed rules of `tech-env.md` (Sections 2 and 3)
- [X] Draft the component list (Section 3)
- [X] Write the questions on the open design decisions (Section 4)
- [x] Collect the team's answers to all questions in Section 4 (Q1 to Q9: A)
- [x] Analyse the answers for ambiguities and contradictions, with follow-up questions if needed (no contradictions or ambiguities; no follow-up questions needed)
- [x] Generate the design artifacts (Section 5)
- [x] Get the team's explicit approval of the application design (approved 2026-11-20T08:06:56Z)

---

## 2. Design Decisions Already Fixed by `tech-env.md`

These rules are not asked again. The design follows them.

| Rule                                                                                                                                                                                              | Source in `tech-env.md`             |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| Python 3.12, `src/` layout, one package `reporttrace`                                                                                                                                         | Language and Package Manager          |
| Layers go one way: `ui` to `api` to `agent` to `tools` to `storage`. The web page calls the backend over HTTP only                                                                      | Architecture and Patterns             |
| One agent from `create_agent`, with four tools (`search_report`, `read_section`, `find_financial_data`, `calculate`), a system prompt, and middleware. No custom graph, no second agent | Architecture and Patterns             |
| Retrieval only through tools. Tools are read-only. Only the ingestion script writes to the library                                                                                                | Architecture and Patterns             |
| The selection and the data stores reach the tools as runtime context (`AgentContext`, read through `ToolRuntime`)                                                                             | Architecture and Patterns             |
| Each tool is a plain function plus a thin `@tool` wrapper                                                                                                                                       | Architecture and Patterns             |
| A `wrap_tool_call` middleware turns tool errors into tool messages. `ToolCallLimitMiddleware` limits tool calls                                                                               | Architecture and Patterns, LLM Access |
| Steps are streamed with `agent.stream(..., stream_mode="updates")`                                                                                                                              | Architecture and Patterns             |
| The answer check runs after the agent and before the response, as ordinary code                                                                                                                   | Architecture and Patterns             |
| Conversation memory uses `InMemorySaver` with one `thread_id` per conversation                                                                                                                | Architecture and Patterns             |
| Only `agent/models.py` creates the chat model and the embedding model                                                                                                                           | LLM Access                            |
| Endpoints are thin `def` functions that validate input, call one function from a lower layer, and return a Pydantic model. Errors are raised as `ApiError` and returned in one shape          | Example Code Patterns                 |
| SQLite through `sqlite3` with parameterised queries; one Chroma collection `passages`                                                                                                         | Frameworks and Libraries              |
| `uv run reporttrace` starts the backend on `127.0.0.1:8000` and the page on `127.0.0.1:8501`                                                                                                | Cloud Provider and Deployment Model   |

---

## 3. Draft Component List

The questions in Section 4 settle the open boundaries in this list. The final list is written in `components.md`.

| Component    | Package or module              | Main responsibility                                                                                                                              | Stories and requirements |
| ------------ | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------ |
| Domain types | `reporttrace.domain`         | Shared types: selection, company, filing, passage, figure, calculation, step, structured answer, checked answer                                  | All                      |
| Settings     | `reporttrace.settings`       | Reads `.env` with `pydantic-settings`; holds the API key as `SecretStr`                                                                    | NFR-3.2, NFR-7           |
| Ingestion    | `reporttrace.ingestion`      | EDGAR download, 10-K parsing into items and notes, passage splitting, line-item mapping, writing the library, the `reporttrace-ingest` command | F1                       |
| Storage      | `reporttrace.storage`        | SQLite stores for companies, filings, passages, and figures; the Chroma passage index; index metadata (D10)                                      | F1, FR-7.5               |
| Tools        | `reporttrace.tools`          | The four tools, their plain functions, tool errors, and result formatting                                                                        | F3, F5, F6               |
| Agent        | `reporttrace.agent`          | Model creation, agent context, system prompt, middleware, agent construction, conversations, and the question service                            | F3, F4, F8, F9, F10      |
| Answer check | Location decided by Question 1 | Checks quotes, numbers, and citations against the evidence; builds the gap list                                                                  | F8                       |
| Run log      | Inside the agent layer         | Writes one JSON-lines record per question                                                                                                        | F10                      |
| API          | `reporttrace.api`            | FastAPI app, routers, schemas, errors, dependencies                                                                                              | F2, F3, F4, F7           |
| Web page     | `reporttrace.ui`             | Streamlit page and its HTTP client for the backend                                                                                               | F2 to F9                 |
| Evaluation   | `reporttrace.evaluation`     | Question set, runner, metrics, results report, the `reporttrace-eval` command                                                                  | F11                      |
| Launcher     | `reporttrace.launcher`       | The `reporttrace` command: starts and stops the two local processes                                                                            | NFR-7.1                  |

---

## 4. Questions

**How to answer:** write the letter of your choice after each `[Answer]:` tag. If no option fits, choose X and describe what you want after the tag. These decisions shape the units of work and the code of every unit, so please answer every question before saying that you are done.

### Part A: Component boundaries

## Question 1

Where does the answer check (F8) live? In both options the question service in the agent layer calls it after the agent has finished, because endpoints must stay thin (Section 2).

A) In a module inside the `agent` package, next to the question service that calls it

B) In its own package, `reporttrace.checking`, which depends only on the domain types. It can be built and tested without any agent code

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 2

How are the domain types written (for example `Selection`, `Passage`, `Figure`, the structured answer)?

A) As frozen dataclasses inside the system. Pydantic models are used only at the edges: API schemas, settings, and the schema of the agent's structured answer

B) As Pydantic models everywhere, shared by every layer

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 3

How do the ingestion script and the backend share the storage code? The backend must open SQLite in read-only mode (NFR-3.7), and only the ingestion script may write (Section 2).

A) One `storage` package for both. The reading classes are used by the backend and the ingestion script; the writing code is in separate writer classes in the same package, which the backend never imports

B) The `storage` package only reads. All writing code, including the table definitions, lives in the `ingestion` package

X) Other (please describe after [Answer]: tag below)

[Answer]:A

### Part B: The agent and the answer check

## Question 4

How does the agent hand its final answer to the answer check? The check needs each statement with its citations and a short topic phrase (`requirements.md` Section 12, D4).

A) Structured output: the agent returns its final answer in a fixed schema (the four parts, each statement with its citations and topic phrase) through the `response_format` option of `create_agent`. The check works on the schema directly. This depends on structured output working through the Cornell API, so it is tested early, together with the embeddings (NFR-7.3)

B) Marked-up text: the agent writes the answer as text, with a fixed heading for each part and a citation marker after each statement, for example `[AAPL-FY2025-ITEM7-0012]`. Code parses the text into the schema before the check. This works with any model that writes text, but parsing can fail

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 5

How does the answer check know which passages and figures the tools returned in the current conversation (FR-8.5)?

A) An evidence record for each conversation, kept in the backend's memory. Each tool wrapper adds the passages and figures it returned to this record, which reaches the tools through the runtime context. The check reads the record. The tools still never write to the library

B) No separate record. The check rebuilds the evidence from the tool messages in the conversation's saved history: it reads the evidence IDs from them and loads the items from storage

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 6

After a second attempt (FR-8.6), what stays in the conversation's memory, which later follow-up questions can see?

A) The failed first draft and the list of what failed are removed from the conversation's memory. The tool results and the final checked answer stay. The failed draft is kept in the run log only

B) Everything stays, including the failed first draft and the list of what failed. Later questions may see them, but anything they produce is checked again

X) Other (please describe after [Answer]: tag below)

[Answer]:A

### Part C: Communication between the page and the backend

## Question 7

How do the agent's steps reach the web page (US-4.1 Must, US-4.2 Should)?

A) Streamed from the start: the question endpoint returns a stream of events, one JSON object per line (each step as it happens, then the final answer). The page reads the stream as it arrives. US-4.1 and US-4.2 use the same path

B) One response first: the question endpoint returns the final answer together with all its steps in one JSON response (US-4.1). Streaming for US-4.2 is added later, as a second endpoint, only if time allows

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 8

Who decides when a new conversation starts (FR-2.5, FR-3.4)?

A) The backend. The page sends the selection, and the conversation ID it received with the previous answer. The backend starts a new conversation, with a new ID, when no ID is sent, when the user has cleared the conversation, or when the selection differs from the conversation's selection

B) The page. It creates a new conversation ID when the selection changes or the user clears the conversation, and sends the ID with each question. The backend rejects an ID that arrives with a different selection

X) Other (please describe after [Answer]: tag below)

[Answer]:A

### Part D: Evaluation

## Question 9

How does the evaluation script (F11) run its questions?

A) In-process: it calls the same question service that the backend endpoint uses, so no server needs to run. The API and the page are not part of the evaluation

B) Over HTTP: it sends each question to the running backend, as the page does, so the API is part of every evaluation run. `uv run reporttrace` must be running during the evaluation

X) Other (please describe after [Answer]: tag below)

[Answer]:A

---

## 5. Generation Checklist

These steps run only after all answers are collected and analysed. Each step is marked [x] as soon as it is done.

- [x] Step 1: Finalise the component list from Section 3 with the answers (Q1, Q2, Q3): 14 components; the agent layer is split into agent core, conversations, answer check, run log, and question service
- [x] Step 2: Write `components.md`: the name, purpose, responsibilities, and interfaces of each component
- [x] Step 3: Write `component-methods.md`: the method signatures of each component, with their purpose and their input and output types (Q2, Q4, Q5). Business rules are left to Functional Design
- [x] Step 4: Write `services.md`: the question service, the ingestion service, and the evaluation service, with their orchestration (Q4, Q5, Q6, Q8, Q9)
- [x] Step 5: Write `component-dependency.md`: the dependency matrix, the communication patterns (Q7, Q8), and data flow diagrams for asking a question and for building the library, each with a text alternative
- [x] Step 6: Write `application-design.md`, which consolidates Steps 2 to 5
- [x] Step 7: Check completeness and consistency (`application-design.md` Section 7, all checks pass):
  - every story US-2.1 to US-9.1 and every supporting requirement (F1, F10, F11) is served by at least one component;
  - every dependency follows the one-way layer order in Section 2, apart from three narrow exceptions that `tech-env.md` requires;
  - no library from the "Do NOT Use" table of `tech-env.md` appears.
- [x] Step 8: Validate the content (Mermaid syntax, ASCII diagrams, tables) before saving each file: Mermaid node IDs alphanumeric, every edge uses declared nodes, a text alternative for every diagram; every line of the ASCII box is 69 characters, with no Unicode box characters and no tabs

### Mandatory artifacts

- [x] Generate `components.md` with component definitions and high-level responsibilities
- [x] Generate `component-methods.md` with method signatures (business rules detailed later in Functional Design)
- [x] Generate `services.md` with service definitions and orchestration patterns
- [x] Generate `component-dependency.md` with dependency relationships and communication patterns
- [x] Validate design completeness and consistency
