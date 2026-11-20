# Requirements: ReportTrace

## 1. Intent Analysis

| Item | Value |
| --- | --- |
| User request | "Using AI-DLC, I want to start a new project. Please read vision.md and tech-env.md, then begin the AI-DLC workflow." |
| Request type | New Project (greenfield) |
| Scope estimate | System-wide: ingestion script, storage, four agent tools, one agent, answer check, FastAPI backend, Streamlit page, run log, evaluation script |
| Complexity estimate | Complex |
| Requirements depth | Comprehensive (four stakeholder groups, measurable success metrics, eleven features that need traceability) |
| Sources | `vision.md`, `tech-env.md`, `requirement-verification-questions.md` (26 answers), `requirement-clarification-questions.md` (5 answers) |

## 2. Summary

ReportTrace is a local web app for asking questions about the Form 10-K annual reports of ten U.S.-listed companies. The user selects one company and one to three fiscal years. One LangChain agent with four read-only tools answers the question. Code then checks every quote, figure, and statement against what the tools returned. The checked answer is shown with its agent steps, its figures, its quotes, and a list of what the reports do not answer. Each citation opens an evidence card.

Key decisions made in this stage:

- Passages are labelled by 10-K item, and each note to the financial statements is its own section.
- Figures come from a fixed list of filed line items. A figure found only in the report text is shown as a labelled quote and may be used in calculations.
- Figures are rounded for reading. A rounded figure is correct when it equals the exact value rounded to the precision shown.
- Follow-up questions use the earlier turns of the conversation, and their evidence counts, until the selection changes.
- An answer that fails the check gets one second attempt within the same limit of 10 tool calls. Removed text is never shown to the user.
- The success metrics are goals. The final evaluation reports the real numbers.
- The three AI-DLC extensions are switched off. All rules in `tech-env.md` remain requirements, including its Security section and its `hypothesis` tests.

## 3. Decisions From the Clarifying Questions

| Question | Decision | Requirements |
| --- | --- | --- |
| Q1 | Exactly the ten proposed companies; a company is replaced only if its filings cannot be parsed | FR-1.2 |
| Q2 | The company's own fiscal year label, shown with the date on which the year ended | FR-1.7, FR-2.2 |
| Q3, changed by CQ1 | Standard 10-K items, each note to the financial statements as its own section, and "Other" | FR-1.6 |
| Q4 | A fixed list of common line items, mapped by the team | FR-1.8, FR-6.1 |
| Q5 | Earlier-year figures printed in a selected report may be used, shown with their own fiscal year and source report | FR-6.2 |
| Q6 | Use the value from the fiscal year's own report when that report is selected; otherwise from a selected report that contains it | FR-6.3 |
| Q7 | A figure found only in the text is a labelled quote, may be used in calculations, and the result is labelled | FR-6.5 |
| Q8 | Rounded for reading everywhere; correct when equal to the exact value rounded to the precision shown | FR-6.6, FR-8.3 |
| Q9 | Any two or all three available years, including years that are not next to each other | FR-2.3 |
| Q10 | Follow-up questions use the earlier turns; memory only | FR-3.4 |
| Q11 | On a change of selection the chat stays, with a note and selection labels; the agent does not use earlier turns | FR-2.5 |
| Q12, extended by CQ2 | Every paraphrased sentence must cite an item that a tool returned in the current conversation | FR-8.4, FR-8.5 |
| Q13, extended by CQ3 | One second attempt; it may call tools, but both attempts share the limit of 10 tool calls | FR-8.6, FR-3.3 |
| Q14, changed by CQ4 | Only a neutral line about what is missing is shown; removed text goes to the run log only | FR-8.7 |
| Q15 | The evidence card shows the cited passage, and the passages before and after it on request | FR-7.2, FR-7.3 |
| Q16 | At the tool-call limit, the best answer from the evidence found so far, with a note | FR-3.5 |
| Q17 | No overall time limit for one question | NFR-1.3 |
| Q18 | GPT-4o tool calling through the Cornell API is confirmed; embeddings are not yet tested | NFR-7.2, NFR-7.3 |
| Q19 | The quota is unknown; usage is kept low by design, and token usage is logged per question | NFR-2, FR-10.1 |
| Q20 | About 10 single-year text, 8 figure and calculation, 6 comparison, and 6 unanswerable questions | FR-11.1 |
| Q21 | All four members write a share; a different member checks each question against the filing | FR-11.3 |
| Q22 | Disagreeing raters discuss until they agree; the question is marked "disputed" | FR-11.5 |
| Q23 | Metric targets are goals, not acceptance criteria | Section 7 |
| Q24 to Q26 | Security Baseline, Property-Based Testing, and Resiliency Baseline extensions are switched off | Section 9 |
| CQ5 | Everything in `tech-env.md` remains a requirement | NFR-3, NFR-6 |

## 4. Users and Stakeholders

| User | Need | Main requirements |
| --- | --- | --- |
| Research assistant or junior analyst (primary) | A fast answer to a specific question, with figures and quotes ready for a research note | F2, F3, F5, F6 |
| Reviewer (senior analyst) | To check any statement against its source in one click | F7, F8 |
| Finance student | To find where a topic is discussed and how it changed over the years | F3, F5, F7 |
| Project team (four students) | To build the library, debug answers, and measure accuracy | F1, F10, F11 |

## 5. Terms

| Term | Meaning |
| --- | --- |
| Selection | One company and one to three of its fiscal years, chosen in the selection panel |
| Single-year mode, comparison mode | One selected year; two or three selected years |
| Passage | A piece of report text stored for search, labelled with company, fiscal year, section, and evidence ID |
| Section | A standard 10-K item, a note to the financial statements in Item 8, or "Other" |
| Conversation | The questions and answers since the last change of selection or the last clear. The agent keeps one conversation thread for it |
| Evidence | Passages and figures that a tool returned in the current conversation |
| Statement | One sentence of the Answer part, one row of the Figures table, or one quote |
| Answer check | Code that runs after the agent and removes every statement that the evidence does not support |

## 6. Functional Requirements

### F1. Report library

- **FR-1.1** The command `uv run reporttrace-ingest --tickers ... --years N` downloads, for each ticker, the N most recent Form 10-K filings and the company's filed financial data from the SEC's EDGAR service, using `httpx` only.
- **FR-1.2** The demo library holds exactly ten companies with three fiscal years each (30 reports): AAPL, MSFT, NVDA, AMZN, WMT, COST, KO, NKE, TSLA, PFE. A company is replaced only if its filings cannot be parsed, and the team records the replacement.
- **FR-1.3** Every SEC request sends the contact string from `SEC_USER_AGENT`, and the script stays within the SEC's fair-access limit of 10 requests per second.
- **FR-1.4** Downloaded files are kept in `data/raw/{ticker}/` and are not downloaded again on a later run.
- **FR-1.5** Each report is split into passages. Each passage is stored in `data/reporttrace.db` and embedded once into the Chroma collection `passages` in `data/chroma/`, with the metadata `ticker`, `fiscal_year`, `section`, and `evidence_id`. Evidence IDs match `^[A-Z]{1,5}-FY\d{4}-[A-Z0-9]{4,8}-\d{4}$`.
- **FR-1.6** Each passage has one section label:
  - a standard 10-K item, for example "Item 1A Risk Factors" or "Item 7 Management's Discussion and Analysis";
  - inside Item 8, each note to the financial statements as its own section, for example "Item 8, Note 12"; the text of Item 8 outside the notes keeps the label "Item 8";
  - "Other" for text that is not recognised. It stays searchable.
- **FR-1.7** Each filing stores the company's own fiscal year label and the date on which the fiscal year ended.
- **FR-1.8** The team maps a fixed list of common line items (for example revenue, cost of revenue, gross profit, operating income, net income, earnings per share, total assets, total liabilities, equity, operating cash flow, capital expenditure, research and development expense) to the company's filed data. For each value the library stores the line item, the filed label, the fiscal year, the value, the unit, and the filing that reported it.
- **FR-1.9** At the end of a run, the script prints and saves a summary for each company: the reports ingested, the sections recognised, the amount of text under "Other", and the listed line items that are missing. If one company fails, the script continues with the others and reports the failure.
- **FR-1.10** Only the ingestion script writes to the library. The library changes only when the script is run again.

### F2. Selection panel

- **FR-2.1** The sidebar lists the companies in the library. The user picks exactly one.
- **FR-2.2** The panel lists the fiscal years available for that company, each with its label and end date, for example "FY2025 (ended Jan 31, 2025)".
- **FR-2.3** The user ticks one year (single-year mode) or any two or all three available years (comparison mode), including years that are not next to each other.
- **FR-2.4** The selection is enforced by code. It is passed to the agent as runtime context. Tools read the company only from the context. The model can choose a fiscal year only from the selection and can never choose the company.
- **FR-2.5** When the user changes the company or the ticked years, the chat stays on the page with a visible note that the selection changed. Each earlier answer keeps a label showing the selection it was based on. A new conversation starts for the agent, which does not use the earlier turns.

### F3. Chatbot

- **FR-3.1** The chat sits beside the selection panel. The user types a question in English of 1 to 1,000 characters.
- **FR-3.2** One agent, created with LangChain's `create_agent`, answers every question. It has the chat model, the four tools `search_report`, `read_section`, `find_financial_data`, and `calculate`, a system prompt, and middleware.
- **FR-3.3** The agent decides what to search and searches again when the evidence is not enough. It makes at most 10 tool calls per question, counting the second attempt (FR-8.6).
- **FR-3.4** A follow-up question uses the earlier turns of the same conversation. The turns are kept in memory only, one thread per conversation. They are lost when the app restarts or the user clears the conversation. The page has a control to clear the conversation.
- **FR-3.5** If the agent reaches the limit of 10 tool calls before it has finished, it writes the best answer it can from the evidence found so far. Unanswered parts go under "Not found in the reports", and the answer carries a note that the search limit was reached.
- **FR-3.6** A tool error is returned to the model as a tool message so that the model can correct itself. The user never sees a crash.
- **FR-3.7** Every answer has four parts, in this order:
  1. **Answer**: two to five sentences that answer the question directly;
  2. **Figures**: each figure used, with its label, fiscal year, value, and unit, and the formula of each calculated value;
  3. **What the company says**: quotes from the reports, each with a citation;
  4. **Not found in the reports**: the parts of the question that the selected reports do not answer.

### F4. Agent steps

- **FR-4.1** Each agent step is shown in plain words, for example "Searching the FY2025 report for 'gross margin'".
- **FR-4.2** (Must) The steps stay visible with the answer.
- **FR-4.3** (Should) The steps are shown live while the agent works. They are streamed with `agent.stream(..., stream_mode="updates")`. This is the first feature to cut if time is short.

### F5. Year comparison

- **FR-5.1** In comparison mode the agent searches each selected year separately. One `search_report` call covers one fiscal year.
- **FR-5.2** The answer shows the selected years side by side: what changed, what stayed the same, and what appears in only some years. Each citation carries its fiscal year.
- **FR-5.3** If no evidence is found for a selected year, the answer says so under "Not found in the reports".

### F6. Figures and calculations

- **FR-6.1** Figures come from the filed financial data through `find_financial_data`, which covers the fixed list in FR-1.8.
- **FR-6.2** A figure of an earlier year that is printed in a selected report may be used, for example the FY2024 figure in the FY2025 report when only FY2025 is ticked. It is shown with its own fiscal year and the report it came from.
- **FR-6.3** If a figure was filed more than once, the value from the fiscal year's own report is used whenever that report is selected. Otherwise the value comes from the earliest filed selected report that contains it.
- **FR-6.4** Changes, growth rates, and ratios are calculated by the `calculate` tool. It supports a fixed list of operations, uses `decimal.Decimal`, and never evaluates text as code. The answer shows the formula and its inputs.
- **FR-6.5** A figure that exists only in the report text is shown as a quote labelled "quoted from the report", with a citation, and must appear word for word in a retrieved passage. It may be used in a calculation. The result then carries the label "based on a quoted figure".
- **FR-6.6** Figures are rounded for reading everywhere, for example "$391.0 billion". A rounded figure is correct when it equals the filed or calculated value rounded to the precision shown. A quoted figure is shown exactly as written in the report.

### F7. Evidence cards

- **FR-7.1** Every statement in an answer has a citation.
- **FR-7.2** Clicking a passage citation opens an evidence card that shows the cited passage with the quoted words highlighted, the company, the fiscal year, the section, and a link to the filing on sec.gov.
- **FR-7.3** The card has a control that shows, on request, the passage before and the passage after the cited passage in the same section.
- **FR-7.4** Clicking a figure citation opens a card with the filed label, the fiscal year, the value, the unit, the period, and a link to the filing that reported it.
- **FR-7.5** The backend returns a passage, and its neighbouring passages, by evidence ID. An ID that does not match the pattern is rejected. An unknown ID returns the error `EVIDENCE_NOT_FOUND`.

### F8. Answer check and gap list

- **FR-8.1** The answer check runs after the agent and before the answer is sent. It is ordinary code and makes no model call.
- **FR-8.2** Every quote must appear word for word in a passage of the evidence. Differences in spacing and in the style of quotation marks are ignored.
- **FR-8.3** Every number must come from the evidence: a result of `find_financial_data` or `calculate`, or a passage for a quoted figure. Rounded numbers are matched by the rule in FR-6.6.
- **FR-8.4** Every sentence of the Answer part that contains no quote and no number must cite at least one passage or figure of the evidence. Code checks that the cited item exists. A sentence with no valid citation is removed.
- **FR-8.5** Evidence is everything a tool returned in the current conversation. Evidence from earlier turns counts until the selection changes or the conversation is cleared.
- **FR-8.6** If the check removes anything, the agent gets one second attempt. It is told what failed and writes the answer again. It may call tools, but both attempts together stay within the limit of 10 tool calls. If the limit has already been reached, the agent writes the answer again without tools. Whatever fails the second check is removed.
- **FR-8.7** For each removed statement, the user sees only a neutral line under "Not found in the reports" that says what is missing, for example "No supported statement on the cause of the margin change". The removed text itself is never shown to the user. It is kept in the run log.
- **FR-8.8** The run log records how many statements were removed from each answer, so that the team can measure how often correct statements are removed.

### F9. Questions the reports cannot answer

- **FR-9.1** For investment advice, forecasts, price targets, or anything outside the selected reports (stock prices, news, other companies), the chatbot says that annual reports cannot answer the question and suggests related questions that they can answer.
- **FR-9.2** For a question about a company that is not selected, the chatbot says that it can answer only about the selected company and suggests changing the selection.
- **FR-9.3** A question for which no evidence is found gets "Not found in the reports" and no invented answer.

### F10. Run log

- **FR-10.1** Each question adds JSON lines to a log file in `data/logs/`, with: the question, the selection, the conversation ID, every step, every tool call with its arguments and the evidence IDs it returned, each draft answer, the result of each answer check including the removed text, the final answer, the timing of each step and of the whole question, the token usage of each model call and of the whole question, whether the tool-call limit was reached, whether the second attempt was used, and any error.
- **FR-10.2** The log contains no keys and is ignored by Git.

### F11. Evaluation

- **FR-11.1** The evaluation set has at least 30 questions: about 10 single-year text questions, 8 figure and calculation questions, 6 year-comparison questions, and 6 unanswerable questions. Companies are chosen freely.
- **FR-11.2** Each question lists its company, its fiscal years, its expected figures, and its expected source sections. An unanswerable question is marked as such.
- **FR-11.3** All four team members write a share of the questions, and a different member checks each question against the filing.
- **FR-11.4** `uv run reporttrace-eval` runs every question against the real model and reports the metrics in Section 7. It is run by hand and is not part of `uv run pytest`.
- **FR-11.5** Quote accuracy, figure accuracy, retrieval, honest gaps, and speed are computed by the script. For answer quality, two team members rate each answer as correct, partly correct, or wrong. When they disagree, they discuss until they agree, and the question is marked "disputed" in the results.
- **FR-11.6** An expected section is given as an item (for example "Item 7") or as a note (for example "Item 8, Note 12"). A passage from a note also matches an expected section that names its item.
- **FR-11.7** The results report gives the real value of every metric and explains every missed target.

## 7. Success Metrics

These targets come from `vision.md`. They are goals, not acceptance criteria (Q23): the final evaluation run reports the real numbers, and a missed target is explained.

| Metric | Target | How it is measured |
| --- | --- | --- |
| Quote accuracy | 100% of quotes shown to the user match the cited passage word for word | Automatic check in the evaluation script |
| Figure accuracy | At least 95% of figures in answers equal the filed value or the correct calculation, using the rounding rule in FR-6.6 | Expected figures of each test question |
| Retrieval | For at least 85% of questions, at least one retrieved passage comes from the expected section (FR-11.6) | Expected sections of each test question |
| Honest gaps | At least 90% of unanswerable questions get "Not found in the reports", or the response in FR-9.1, and no invented answer | The 6 or more unanswerable questions |
| Answer quality | At least 80% of answers are rated correct and complete | Two raters per answer (FR-11.5) |
| Speed | Median time to answer is at most 30 seconds for one year and 60 seconds for a comparison | Timings in the run log (FR-10.1) |

## 8. Non-Functional Requirements

### NFR-1. Performance

- **NFR-1.1** The speed targets in Section 7 apply.
- **NFR-1.2** Each model call has a 60-second timeout and 2 retries, with `temperature=0`.
- **NFR-1.3** There is no overall time limit for one question. The limit of 10 tool calls and the per-call limits are enough.

### NFR-2. API usage

The Cornell API quota is not known (Q19), so usage is kept low by design:

- **NFR-2.1** Each report is embedded only once.
- **NFR-2.2** `read_section` returns one section. A note is a section of its own (FR-1.6), so the agent does not need to load the whole of Item 8.
- **NFR-2.3** Both attempts of one question share the limit of 10 tool calls.
- **NFR-2.4** Automated tests never call the model. The full evaluation is run only a few times.
- **NFR-2.5** The token usage of every question is logged (FR-10.1).

### NFR-3. Security

The whole Security section of `tech-env.md` applies (CQ5):

- **NFR-3.1** There is no login. Both processes bind to `127.0.0.1` only, and CORS is not enabled.
- **NFR-3.2** The API key and the SEC contact string are stored in a `.env` file that Git ignores. `.env.example` is committed with the variable names and no values. The key is held as a Pydantic `SecretStr`. It is never logged, never returned by an endpoint, and never sent to the web page.
- **NFR-3.3** Every request is validated with Pydantic: the question has 1 to 1,000 characters, the ticker is in the library, `fiscal_years` has 1 to 3 values that exist for that company, and an evidence ID matches its pattern.
- **NFR-3.4** Tool arguments written by the model are untrusted. Each tool checks its own rules, for example that the year is one of the selected years.
- **NFR-3.5** Report text and user questions are data, not instructions. Passages reach the model only as tool results, and the system prompt tells the model to ignore instructions inside them.
- **NFR-3.6** `calculate` never evaluates text as code. `eval()`, `exec()`, and `compile()` are not used anywhere.
- **NFR-3.7** SQL queries use parameters only. The backend opens SQLite in read-only mode.
- **NFR-3.8** `uv.lock` is committed so that every laptop installs the same versions.

### NFR-4. Privacy and data leaving the laptop

- **NFR-4.1** Only the user's question, passages from public reports, and public financial figures are sent to the model API. The product collects no personal data.
- **NFR-4.2** LangSmith tracing is never switched on. Usage reporting is switched off in Streamlit (`gatherUsageStats = false`) and, if the installed version has it, in Chroma.
- **NFR-4.3** Apart from the model API, the only outside service is the SEC, and only while the library is being built.

### NFR-5. Reliability and error handling

- **NFR-5.1** Every endpoint returns errors in one shape: `{"error": {"code": "...", "message": "..."}}`.
- **NFR-5.2** If a model call still fails after its retries, the user sees a clear error message together with the steps completed so far. No unchecked answer is shown, and the run log records the error.
- **NFR-5.3** If the backend cannot be reached, the web page shows a clear message instead of a crash.

### NFR-6. Maintainability and testability

All rules in `tech-env.md` apply (CQ5):

- **NFR-6.1** Python 3.12 and `uv` only. Configuration lives in `pyproject.toml`, and `uv.lock` is committed. The code uses a `src/` layout with one package, `reporttrace`.
- **NFR-6.2** Layers go one way: `ui` to `api` to `agent` to `tools` to `storage`. The web page only calls the backend over HTTP.
- **NFR-6.3** One module, `agent/models.py`, creates the chat model and the embedding model. All other code receives them as `BaseChatModel` and `Embeddings`.
- **NFR-6.4** Tools are read-only. Each tool has a plain function that holds the logic and a thin `@tool` wrapper.
- **NFR-6.5** No library from the "Do NOT Use" table in `tech-env.md` is used.
- **NFR-6.6** Every function has type hints. `ruff` (line length 100) and `mypy` pass.
- **NFR-6.7** `pytest` with `pytest-cov` reaches at least 80% line coverage for `tools`, `agent`, `storage`, and `ingestion`. The Streamlit page is excluded.
- **NFR-6.8** `hypothesis` property-based tests cover the `calculate` tool and the answer check. This is a `tech-env.md` requirement and holds although the AI-DLC Property-Based Testing extension is switched off.
- **NFR-6.9** Tests never use the network. They use `FakeToolCallingModel`, `DeterministicFakeEmbedding`, and `httpx.MockTransport`, and each test builds its own small library in `tmp_path`. There are unit tests, integration tests with FastAPI `TestClient`, and a smoke test with Streamlit `AppTest`.
- **NFR-6.10** `uv run ruff check .`, `uv run mypy src`, and `uv run pytest` pass before each commit.

### NFR-7. Portability and LLM access

- **NFR-7.1** The product runs on Windows 11 and macOS. It uses `pathlib` for paths and has no shell scripts. Every command is a Python entry point started with `uv run`. `uv run reporttrace` starts the backend on `127.0.0.1:8000` and the web page on `127.0.0.1:8501`, and Ctrl+C stops both.
- **NFR-7.2** The chat model is GPT-4o through the Cornell API, reached with `ChatOpenAI` and Chat Completions tool calling. This has been confirmed to work (Q18).
- **NFR-7.3** Embeddings use `text-embedding-3-small` through the Cornell API. This has not been tested yet (Q18), so the embedding fallbacks in `tech-env.md` are requirements, not just precautions:
  - embedding requests must be testable early in Construction;
  - `check_embedding_ctx_length=False` must be available as a setting;
  - Chroma's built-in model on the laptop must be selectable by configuration, in `agent/models.py` only.
- **NFR-7.4** The library records which embedding model built the index. The backend detects a mismatch with the configured embedding model and reports it, because an index must be searched with the model that built it.
- **NFR-7.5** Pointing the same code at a local Ollama server by changing environment variables remains a precaution. The MVP does not have to test it.

### NFR-8. Usability

- **NFR-8.1** The interface, the questions, and the answers are in English.
- **NFR-8.2** The selection is always visible next to the chat, and each answer shows the selection it was based on.
- **NFR-8.3** A reviewer can open the source of any statement with one click.

## 9. AI-DLC Extension Configuration

| Extension | Enabled | Effect |
| --- | --- | --- |
| Security Baseline | No | The extension rules are not loaded or enforced. The Security section of `tech-env.md` still applies (NFR-3) |
| Property-Based Testing | No | The extension rules are not loaded or enforced. The `hypothesis` tests of `tech-env.md` still apply (NFR-6.8) |
| Resiliency Baseline | No | The extension rules are not loaded or enforced |

## 10. Constraints

- Construction runs from November 25 to December 16, 2026, and the product must be finished by December 16, 2026. The team has four students.
- The product runs locally, for one user at a time. Nothing is deployed to the cloud.
- The model is reached through the Cornell API, within its course quota.
- The architecture rules, libraries, and prohibited list of `tech-env.md` are fixed.

## 11. Derived Rules for Review

These rules are not stated in the input documents or the answers. They were derived to make the requirements complete. Please confirm or change them when you review this document.

| ID | Rule | Requirement |
| --- | --- | --- |
| D1 | If a figure was filed more than once and the fiscal year's own report is not selected, the earliest filed selected report that contains the figure is used | FR-6.3 |
| D2 | A quoted figure is shown exactly as written in the report, as an exception to the rounding rule | FR-6.6 |
| D3 | A figure citation opens a card with the filed line item and a link to the filing, because a filed figure has no passage | FR-7.4 |
| D4 | The neutral line for a removed statement is built from a short topic phrase that the agent gives with each statement. The phrase describes the topic and makes no claim | FR-8.7 |
| D5 | If the tool-call limit is reached in the first attempt, the second attempt rewrites the answer without tools | FR-8.6 |
| D6 | A model call that fails after its retries produces an error message with the steps so far, and no unchecked answer | NFR-5.2 |
| D7 | If one company fails during ingestion, the script continues with the other companies and reports the failure | FR-1.9 |
| D8 | A passage from a note matches an expected section that names its item | FR-11.6 |
| D9 | For the honest-gaps metric, the response in FR-9.1 also counts as an honest answer | Section 7 |
| D10 | The index records its embedding model, and a mismatch with the configured model is detected | NFR-7.4 |
| D11 | A question about a company that is not selected is answered with the response in FR-9.2 | FR-9.2 |

## 12. Notes for Later Stages

- **Tool-call limit across two attempts.** `tech-env.md` sets `ToolCallLimitMiddleware(run_limit=10)`, which counts one agent run. If the second attempt (FR-8.6) is a separate run, Application Design must still keep the total for the question at 10 tool calls.
- **Answer format.** The answer check (FR-8.2 to FR-8.4) needs the agent's answer in a structured form in which each statement has its citations and its topic phrase (D4). Application Design decides that form.
- **Note-level sections.** Note headings differ between companies (FR-1.6). Construction should test the parser on every demo company early, and keep note text that is not recognised under "Item 8" so that it stays searchable.
- **Embeddings.** Embedding requests through the Cornell API should be tested at the start of Construction (NFR-7.3), before the full library is built.

## 13. Risks

| Risk | Likelihood | Impact | What we will do |
| --- | --- | --- | --- |
| The layout of 10-K web pages differs between companies, so some items or notes are not recognised | High | Medium | Test the parser on every demo company; keep unrecognised text searchable; replace a company that cannot be parsed (FR-1.2, FR-1.9) |
| The Cornell API does not accept embedding requests | Medium | High | Test early; use `check_embedding_ctx_length=False`, then Chroma's built-in model (NFR-7.3) |
| The Cornell API quota or rate limit is too low | Low | High | Keep usage low by design and log token usage (NFR-2) |
| The filed data lacks a figure that a user asks for | Medium | Medium | Quote the figure from the text with its label, or list it as not found (FR-6.5, FR-8.7) |
| The agent picks the wrong line item, for example total revenue instead of product revenue | Medium | High | Show the label of every figure; include such cases in the evaluation set (FR-3.7, FR-11.1) |
| The answer check removes correct statements | Medium | Medium | Ignore spacing and quotation mark style; allow one second attempt; measure removals (FR-8.2, FR-8.6, FR-8.8) |
| Three weeks are not enough for all features | Medium | High | Build in this order: library, tools, agent, answer check, chatbot, comparison view, evaluation. Cut the live step display first (FR-4.3) |

## 14. Out of Scope for the MVP

As listed in `vision.md`: comparing different companies in one question, adding a company from the web page, filings other than Form 10-K, charts, uploaded PDFs, companies that do not file a 10-K, languages other than English, stock prices and news and web search, user accounts and saved conversations, cloud deployment and several users at once, and investment advice, forecasts, and price targets.

## 15. Traceability

| Feature | Priority | Requirements | Decisions | Metrics |
| --- | --- | --- | --- | --- |
| F1 Report library | Must | FR-1.1 to FR-1.10 | Q1, Q2, Q3, Q4, CQ1 | Retrieval |
| F2 Selection panel | Must | FR-2.1 to FR-2.5 | Q2, Q9, Q11 | |
| F3 Chatbot | Must | FR-3.1 to FR-3.7 | Q10, Q16, CQ3 | Answer quality, Speed |
| F4 Agent steps | Must (with answer), Should (live) | FR-4.1 to FR-4.3 | | |
| F5 Year comparison | Must | FR-5.1 to FR-5.3 | Q9 | Answer quality, Speed |
| F6 Figures and calculations | Must | FR-6.1 to FR-6.6 | Q4, Q5, Q6, Q7, Q8 | Figure accuracy |
| F7 Evidence cards | Must | FR-7.1 to FR-7.5 | Q15 | Quote accuracy |
| F8 Answer check and gap list | Must | FR-8.1 to FR-8.8 | Q12, Q13, Q14, CQ2, CQ3, CQ4 | Quote accuracy, Figure accuracy, Honest gaps |
| F9 Questions the reports cannot answer | Must | FR-9.1 to FR-9.3 | | Honest gaps |
| F10 Run log | Must | FR-10.1, FR-10.2 | Q19 | Speed |
| F11 Evaluation | Must | FR-11.1 to FR-11.7 | Q20, Q21, Q22, Q23 | All |
