# Services: ReportTrace

A service is a component that coordinates other components to complete one whole task. ReportTrace has four services. Each one runs inside a single local process; there are no remote services apart from the model API and the SEC.

| Service | Component | Started by | Task |
| --- | --- | --- | --- |
| S1 Question service | C10 `agent.service.QuestionService` | `POST /api/questions` (C11) and the evaluation runner (C13) | Answer one question, from the selection to the checked answer |
| S2 Ingestion service | C4 `ingestion.pipeline.IngestionService` | `uv run reporttrace-ingest` | Build the report library |
| S3 Evaluation service | C13 `evaluation.runner.EvaluationRunner` and `evaluation.cli` | `uv run reporttrace-eval` | Run the evaluation set and report the metrics |
| S4 Launcher | C14 `launcher.main` | `uv run reporttrace` | Start and stop the backend and the page |

The API (C11) is not a service in this sense. Each endpoint validates its input, calls one function of a lower layer, and returns the result.

---

## S1. Question service

**Responsibilities.** Resolve the conversation, run the agent, stream the steps, check the answer, run at most one second attempt, keep the conversation's memory clean, write the run log, and return the checked answer or an error.

**Collaborators.**
- C7 `ConversationRegistry`;
- C6 `build_agent`, `steps_from_update`, and `AgentContext`;
- C5 tools, through the agent;
- C3 readers;
- C8 `AnswerChecker`;
- C9 `RunLogWriter`.

**Lifetime.** The API builds one `QuestionService` when the backend starts. The service holds the one `InMemorySaver` and the one `ConversationRegistry`, so conversations live as long as the backend process (FR-3.4).

### Orchestration of `ask(request)`

1. **Resolve the conversation.** `registry.resolve(request.conversation_id, request.selection)` returns the conversation. A new conversation, with a new thread ID and an empty evidence record, starts if the ID is missing or unknown, or if the selection differs (Q8, FR-2.5).
2. **Build the context.** `AgentContext(selection, index, passages, figures, evidence)`, with the conversation's evidence record (Q5).
3. **First attempt.**
   - Build the agent with `tool_call_limit = 10` and run `agent.stream({"messages": [question]}, config={"configurable": {"thread_id": ...}}, context=context, stream_mode="updates")`.
   - For each update: count the tool calls, turn the update into steps with `steps_from_update`, collect the token usage, and **yield a `StepEvent` for each step at once** (Q7).
   - The tool wrappers add every returned passage, figure, and calculation to the evidence record as they run (Q5).
4. **Read the structured answer.** When the run ends, read `AgentAnswer` from the agent's structured response (Q4). If the tool-call limit was reached, record `limit_reached` and add a `LIMIT_REACHED` step (FR-3.5).
5. **Check.** `checker.check(answer, evidence)`.
6. **Second attempt, only if the check failed** (FR-8.6):
   - remaining = 10 minus the tool calls of the first attempt;
   - build the agent with `tool_call_limit = remaining`, which is 0 if the limit was reached, so that the answer is rewritten without new steps (D5);
   - send `build_retry_message(failures)` in the same thread and stream as in step 3, yielding a `SECOND_ATTEMPT` step first;
   - read the new `AgentAnswer` and check it again.
7. **Build the checked answer.** `checker.build_checked_answer(...)` drops what failed, adds the neutral gap lines, and returns the removed items for the run log (FR-8.7, D4).
8. **Clean the memory.** If a second attempt ran, remove the failed first draft and the retry message from the thread (Q6). In both cases, replace the agent's final structured answer in the thread with the checked answer. A follow-up then sees only what the user saw, and the tool results stay for FR-8.5.
9. **Write the run log.** Build a `RunRecord` with every field in FR-10.1 and pass it to `run_log.write`.
10. **Finish.** Yield one `AnswerEvent` with the conversation ID, the selection, the checked answer, all steps, and `limit_reached`.

### Failure handling

| Situation | Behaviour | Requirement |
| --- | --- | --- |
| A tool raises `ToolError` or another error | The tool-error middleware turns it into a tool message. The agent continues, and a `TOOL_ERROR` step is shown | FR-3.6 |
| The limit of 10 tool calls is reached | The agent writes its answer from the evidence so far, the answer carries a note, and the second attempt gets no new tool calls | FR-3.5, D5 |
| A model call fails after its 60-second timeout and 2 retries | Yield one `ErrorEvent` with the steps so far. No unchecked answer is sent. The run log records the error | NFR-5.2, D6 |
| The structured answer is missing or invalid | Treated like a failed check of the whole answer: one second attempt; if that fails too, an `ErrorEvent` | FR-8.6, NFR-5.2 |
| The selection is not valid | Not reached: the API rejects it before calling the service | NFR-3.3 |

### `clear(conversation_id)`

Forget the conversation in the registry and delete its thread from the `InMemorySaver` (FR-3.4).

---

## S2. Ingestion service

**Responsibilities.** Download, parse, split, map, write, embed, and report, for each company.

**Collaborators.** C4 `EdgarClient`, `TenKParser`, `PassageSplitter`, and `FigureMapper`; C3 `LibraryWriter` and `IndexWriter`; C6 `build_embeddings`.

### Orchestration of `ingest(tickers, years)`

1. Open the library writable, and create the schema if it does not exist. Record the embedding model in use (D10).
2. For each ticker, in order:
   1. `get_cik`, then `list_annual_reports(cik, years)`.
   2. For each filing: `download_report` (from the cache if present), `parse`, then `split`.
   3. `download_company_facts`, then `map` to get the figures and the missing line items.
   4. Write the company, the filings with their labels and end dates, the passages, and the figures. Then embed the passages of each filing once through `IndexWriter.replace_filing`.
   5. Add a `CompanyReport`.
   6. If any step fails, record the error in that company's report and continue with the next ticker (D7).
3. Print the `IngestionReport` and save it under `data/` (FR-1.9).

Every SEC request goes through `EdgarClient`, which sends `SEC_USER_AGENT` and stays within 10 requests per second (FR-1.3).

---

## S3. Evaluation service

**Responsibilities.** Run the evaluation set in-process (Q9), compute the metrics, and write the results.

**Collaborators.** C13 modules, C10 `QuestionService`, C3 readers, and C6 model builders.

### Orchestration

1. **Run mode** (`reporttrace-eval run`):
   1. Build the stores, the models, and one `QuestionService`, as the API does.
   2. Load the question set.
   3. For each question, call `service.ask()` with a new conversation and collect the events into an `EvalResult`, including the timing and the sections of the retrieved passages.
   4. Write the results file and a rating sheet for the two raters.
2. **Report mode** (`reporttrace-eval report`):
   1. Read the results and the raters' ratings.
   2. Mark the questions on which the raters disagreed as "disputed" (FR-11.5).
   3. Compute the six metrics (D8, D9) and write the results report. The report gives the real value of every metric and a place to explain each missed target (FR-11.7).

---

## S4. Launcher

1. Read the addresses from the settings.
2. Start the backend: `uvicorn` with `reporttrace.api.app:create_app` as a factory, bound to `127.0.0.1:8000`.
3. Start the page: `streamlit run` on `reporttrace/ui/app.py`, bound to `127.0.0.1:8501`, with usage statistics switched off (NFR-4.2).
4. Wait. On Ctrl+C, or if one process exits, stop the other and return.

Both processes are started with the same Python interpreter (`sys.executable`), so the launcher works on Windows and macOS without shell scripts (NFR-7.1).

---

## Conversation Lifecycle

| Event | Page (C12) | Backend (C7, S1) |
| --- | --- | --- |
| First question | Sends no conversation ID | Starts a conversation and returns its ID in every event |
| Follow-up question | Sends the ID from the last answer | Uses the same thread and evidence record (FR-3.4, FR-8.5) |
| Selection changed | Shows the note and labels earlier answers; sends the new selection with the old ID | The selection differs, so a new conversation starts with a new ID (FR-2.5) |
| Clear | Calls `DELETE /api/conversations/{id}`, drops the ID, and empties the chat | Forgets the conversation and its thread |
| Backend restarts | Still holds the old ID and sends it with the next question | The ID is unknown, so a new conversation starts. All earlier conversations are gone |
