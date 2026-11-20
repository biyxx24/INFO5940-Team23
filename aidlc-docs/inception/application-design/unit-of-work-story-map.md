# Unit of Work Story Map: ReportTrace

## 1. How to Read This Map

The map follows answer Q3 B of the unit of work plan:

- **Implementing unit.** Each acceptance criterion (AC) of each story in `stories.md` is assigned to the one unit that implements it. That unit's tests must cover it.
- **Also uses.** This column names the units whose work the criterion needs but does not test. One example is the data that U1 stores and U4 shows.
- **Primary unit.** Each story also has one primary unit: the unit that implements most of the story's criteria. When two units implement the same number, the later unit in the construction order is the primary unit.
- **Whole-story check.** The owner of the primary unit is responsible for checking the whole story against all its criteria. Every story has at least one criterion that the user sees on the page (U4), so the whole-story check is an end-to-end test. It is added in Build and Test (Q5) and reviewed by the primary unit's owner.

Unit owners:

| Unit | Owner |
| --- | --- |
| U0 | all four members |
| U1 | Zheng Kang |
| U2 | Chengcheng Chen |
| U3 | Xihe Xu |
| U4 | Jiayi Luo |

## 2. Summary

| Story | Primary unit | U1 | U2 | U3 | U4 | Total ACs |
| --- | --- | --- | --- | --- | --- | --- |
| US-2.1 Choose the company and years | U4 | | 2 | | 6 | 8 |
| US-2.2 Change the selection during a conversation | U4 | | | 2 | 3 | 5 |
| US-3.1 Ask a question and get a structured answer | U4 | | | 4 | 5 | 9 |
| US-3.2 Ask a follow-up question | U3 | | | 4 | 1 | 5 |
| US-4.1 See how the answer was found | U3 | | | 3 | 1 | 4 |
| US-4.2 Watch the steps while the chatbot works | U4 | | | | 3 | 3 |
| US-5.1 Compare the selected years side by side | U4 | | 2 | 1 | 3 | 6 |
| US-6.1 Use reported figures with their source | U2 | | 6 | | | 6 |
| US-6.2 See how a calculated value was worked out | U2 | | 4 | | | 4 |
| US-7.1 Check a statement against its passage | U4 | 1 | 1 | | 4 | 6 |
| US-7.2 Check a figure against its filing | U4 | | | | 4 | 4 |
| US-8.1 See only statements that the reports support | U2 | | 4 | 3 | | 7 |
| US-8.2 Know what the reports do not answer | U3 | | 2 | 2 | 1 | 5 |
| US-9.1 Be told when a question is outside the reports | U3 | | 1 | 4 | | 5 |
| **Total** | | **1** | **22** | **23** | **31** | **77** |

U0 implements no acceptance criterion directly. It provides the foundation that every criterion needs.

## 3. Acceptance Criteria by Story

### US-2.1 Choose the company and years (primary unit: U4)

| AC | Criterion (short) | Implementing unit | Also uses |
| --- | --- | --- | --- |
| AC1 | The panel lists every company; exactly one is picked | U4 | U1 (companies) |
| AC2 | Each fiscal year shows its own label and end date | U4 | U1 (labels and end dates, FR-1.7) |
| AC3 | One year gives a single-year answer; two or three give a comparison | U4 | U3 |
| AC4 | Any two years can be ticked, including years not next to each other | U4 | |
| AC5 | No question can be sent while no year is ticked | U4 | |
| AC6 | No answer uses a passage from another company or an unselected report | U2 | U1 (filtered search) |
| AC7 | A question cannot make the chatbot use another company or year | U2 | |
| AC8 | The selection is always visible next to the chat | U4 | |

### US-2.2 Change the selection during a conversation (primary unit: U4)

| AC | Criterion (short) | Implementing unit | Also uses |
| --- | --- | --- | --- |
| AC1 | Earlier questions and answers stay on the page | U4 | |
| AC2 | A visible note marks where the selection changed | U4 | |
| AC3 | Each earlier answer shows the selection it was based on | U4 | U3 (selection in the answer event) |
| AC4 | The first question after the change starts a new conversation | U3 | U4 |
| AC5 | Changing back to an earlier selection also starts a new conversation | U3 | |

### US-3.1 Ask a question and get a structured answer (primary unit: U4)

| AC | Criterion (short) | Implementing unit | Also uses |
| --- | --- | --- | --- |
| AC1 | The chat sits beside the panel; a question of 1 to 1,000 characters is accepted, others are refused with a reason | U4 | |
| AC2 | Every answer has the four parts in order | U4 | U2 (answer schema) |
| AC3 | The chatbot searches again when needed, with at most 10 steps | U3 | U2 (tools) |
| AC4 | At the limit of 10 steps, an answer is still shown with a note | U3 | U4 (display) |
| AC5 | A problem in one step does not end the answer and never shows a crash | U3 | U2 (`ToolError`) |
| AC6 | If the AI service fails after its retries, an error with the steps so far is shown and no unchecked answer | U3 | U4 (display) |
| AC7 | If the answering part is not running, the page shows a clear message | U4 | |
| AC8 | Median time for one year is at most 30 seconds, as measured by the evaluation | U4 (evaluation) | U3 (timings) |
| AC9 | The page, questions, and answers are in English | U4 | U3 (system prompt) |

### US-3.2 Ask a follow-up question (primary unit: U3)

| AC | Criterion (short) | Implementing unit | Also uses |
| --- | --- | --- | --- |
| AC1 | A follow-up is answered with the earlier turns in mind | U3 | |
| AC2 | A follow-up may use and cite evidence found for an earlier question | U3 | U2 (evidence record, check) |
| AC3 | The page has a control to clear the conversation | U4 | U3 (`clear`) |
| AC4 | After a restart, earlier conversations are gone | U3 | |
| AC5 | After a change of selection, the earlier turns are not used | U3 | |

### US-4.1 See how the answer was found (primary unit: U3)

| AC | Criterion (short) | Implementing unit | Also uses |
| --- | --- | --- | --- |
| AC1 | Each step is described in plain words with its year and what was looked for | U3 | |
| AC2 | Opening a section, a figure look-up, and a calculation are described the same way | U3 | |
| AC3 | The steps stay visible with their answer after later questions | U4 | |
| AC4 | Steps of a second try appear in the same list | U3 | |

### US-4.2 Watch the steps while the chatbot works (primary unit: U4; priority Should)

| AC | Criterion (short) | Implementing unit | Also uses |
| --- | --- | --- | --- |
| AC1 | Each step appears while the chatbot is still working | U4 | U3 (events yielded as they happen) |
| AC2 | The steps stay with the answer in order | U4 | |
| AC3 | If this story is cut, US-4.1 still holds | U4 | |

### US-5.1 Compare the selected years side by side (primary unit: U4)

| AC | Criterion (short) | Implementing unit | Also uses |
| --- | --- | --- | --- |
| AC1 | Each selected year is searched separately, and the steps show it | U3 | U2 (one year per search) |
| AC2 | The answer shows the years side by side: changed, same, only in some years | U4 | U2 (comparison rows in the schema) |
| AC3 | Every citation in a comparison shows its fiscal year | U4 | U2 |
| AC4 | A year with no evidence is listed under "Not found", not filled with another year's text | U2 | U3 (system prompt) |
| AC5 | A comparison of two years that are not next to each other uses those two years only | U2 | |
| AC6 | Median time for a comparison is at most 60 seconds, as measured by the evaluation | U4 (evaluation) | U3 (timings) |

### US-6.1 Use reported figures with their source (primary unit: U2)

| AC | Criterion (short) | Implementing unit | Also uses |
| --- | --- | --- | --- |
| AC1 | Each figure shows its fiscal year, value, unit, and label as reported | U2 | U1 (filed labels), U4 (display) |
| AC2 | Rounded figures equal the reported value rounded to the precision shown | U2 | U4 (display) |
| AC3 | An earlier-year figure printed in a selected report may be used, with its own year and source report | U2 | U1 (source report stored) |
| AC4 | A figure reported more than once takes its value by the rule in FR-6.3 and D1 | U2 | U1 |
| AC5 | A figure found only in the text is a labelled quote, exactly as written | U2 | U4 (display) |
| AC6 | A figure that is neither reported nor written is not guessed and is listed under "Not found" | U2 | |

### US-6.2 See how a calculated value was worked out (primary unit: U2)

| AC | Criterion (short) | Implementing unit | Also uses |
| --- | --- | --- | --- |
| AC1 | Each calculated value shows its formula and its input figures with their years | U2 | U4 (display) |
| AC2 | The result is exact and rounded only when shown | U2 | |
| AC3 | A calculation with a quoted figure is labelled "based on a quoted figure" | U2 | |
| AC4 | A missing input figure is listed under "Not found", and no value is invented | U2 | |

### US-7.1 Check a statement against its passage (primary unit: U4)

| AC | Criterion (short) | Implementing unit | Also uses |
| --- | --- | --- | --- |
| AC1 | Every statement in an answer has a citation | U2 | |
| AC2 | The card shows the passage with the quoted words highlighted, the company, year, section, and sec.gov link | U4 | U1 (passages and sections) |
| AC3 | On request, the card shows the passages just before and just after | U4 | U1 (`neighbours`) |
| AC4 | The source details can be copied into a research note (reviewer's need) | U4 | |
| AC5 | The sec.gov link opens the filing that contains the passage (reviewer's need) | U1 | U4 (link on the card) |
| AC6 | A citation to a missing passage shows "evidence not found" instead of a crash | U4 | U1 |

### US-7.2 Check a figure against its filing (primary unit: U4)

| AC | Criterion (short) | Implementing unit | Also uses |
| --- | --- | --- | --- |
| AC1 | The figure card shows the label, year, value, unit, period, and filing link | U4 | U1 (figures) |
| AC2 | For an earlier-year figure, the card names the report the value came from | U4 | U1 (source report) |
| AC3 | The details and link can be copied into a research note (reviewer's need) | U4 | |
| AC4 | A citation of a quoted figure opens a passage card | U4 | |

### US-8.1 See only statements that the reports support (primary unit: U2)

| AC | Criterion (short) | Implementing unit | Also uses |
| --- | --- | --- | --- |
| AC1 | Every shown quote matches its passage word for word, ignoring spacing and quotation mark style | U2 | |
| AC2 | Every shown figure comes from the evidence and matches under the rounding rule | U2 | |
| AC3 | Every sentence without a quote or figure cites evidence from the current conversation | U2 | U3 (evidence of earlier turns) |
| AC4 | A quote, figure, or sentence that fails is never shown | U2 | U3 |
| AC5 | After a failure, one more try within the same limit of 10 steps | U3 | |
| AC6 | If the limit was already reached, the second try rewrites without new steps | U3 | |
| AC7 | What fails a second time is removed, and the rest is shown | U3 | U2 (`build_checked_answer`) |

### US-8.2 Know what the reports do not answer (primary unit: U3, tie with U2 goes to the later unit)

| AC | Criterion (short) | Implementing unit | Also uses |
| --- | --- | --- | --- |
| AC1 | "Not found" lists every part of the question that the reports do not answer | U3 | U2 (schema) |
| AC2 | Each removed statement gets a neutral line about what is missing | U2 | |
| AC3 | Removed text is never shown and is kept only in the team's log | U3 | U2 (removed items) |
| AC4 | When nothing is missing, the part is still shown and says so | U4 | |
| AC5 | With no evidence at all, the answer says "Not found" and invents nothing | U2 | U3 (system prompt) |

### US-9.1 Be told when a question is outside the reports (primary unit: U3)

| AC | Criterion (short) | Implementing unit | Also uses |
| --- | --- | --- | --- |
| AC1 | Investment advice, forecasts, or price targets get "annual reports cannot answer this" | U3 | U2 (`scope` in the schema) |
| AC2 | The same for stock prices, news, and other outside topics | U3 | |
| AC3 | The response suggests related questions the reports can answer | U3 | U4 (display) |
| AC4 | A question about another company asks the user to change the selection | U3 | |
| AC5 | The response contains no figure or quote that is not from the selected reports | U2 | |

## 4. Supporting Requirements

| Supporting requirement | Units | Notes |
| --- | --- | --- |
| F1 Report library (FR-1.1 to FR-1.10) | U1 | U1 also builds the demo library of ten companies |
| F10 Run log (FR-10.1, FR-10.2, FR-8.8) | U3 | |
| F11 Evaluation (FR-11.1 to FR-11.3: the question set) | All four members | Team work in parallel with Construction (Q8) |
| F11 Evaluation (FR-11.4 to FR-11.7: the script and the report) | U4 | The full run against the real model happens in Build and Test |

## 5. Derived Rules

| Derived rule | Unit |
| --- | --- |
| D1 Value choice when the year's own report is not selected | U2 (`lookup_figures`), with the stored values from U1 |
| D2 Quoted figures shown as written | U2 |
| D3 Figure cards | U4 |
| D4 Neutral gap line from the topic phrase | U2 |
| D5 Second try without new steps at the limit | U3 |
| D6 Error event after a model failure | U3 |
| D7 Ingestion continues after one company fails | U1 |
| D8 Note-level passages match an item-level expected section | U4 (evaluation metrics) |
| D9 The response to an outside question counts as an honest gap | U4 (evaluation metrics) |
| D10 Embedding model recorded with the index and checked | U1 (records it), U4 (health check) |
| D11 Question about a company that is not selected | U3 |

## 6. Non-Functional Requirements

| NFR | Units |
| --- | --- |
| NFR-1 Performance | U0 (timeouts and retries in `models.py`), U3 (timings), U4 (measured by the evaluation) |
| NFR-2 API usage | U1 (embed once), U2 (`read_section` returns one section), U3 (one limit for both tries, token usage in the log) |
| NFR-3 Security | U0 (`SecretStr`, `.env.example`, `uv.lock`), U1 (read-only database, parameterised queries, writers kept apart), U2 (tool rules, no `eval`), U3 (system prompt treats report text as data), U4 (validation, `127.0.0.1`, no CORS) |
| NFR-4 Privacy | U0 and U3 (no tracing), U4 (Streamlit usage statistics off) |
| NFR-5 Reliability | U3 (error event), U4 (error shape, message when the backend is down) |
| NFR-6 Maintainability and testability | Every unit (layers, type hints, ruff, mypy, coverage); U0 (fakes); U2 (`hypothesis` tests) |
| NFR-7 Portability and LLM access | U0 (models, fallbacks, Cornell API check), U1 (`LibraryInfoStore`), U4 (launcher) |
| NFR-8 Usability | U4 |

## 7. Coverage Check

| Check | Result |
| --- | --- |
| Every story US-2.1 to US-9.1 has a primary unit | Pass (14 of 14) |
| Every acceptance criterion has exactly one implementing unit | Pass (77 of 77; per-story totals match `stories.md`) |
| Every supporting requirement (F1, F10, F11) is assigned | Pass |
| Every derived rule D1 to D11 is assigned | Pass |
| No criterion is implemented by a unit that comes before a unit it needs | Pass. Where a criterion also uses a later unit, for example U2 criteria shown by U4, the implementing unit tests its own part, and the end-to-end check in Build and Test covers the rest |
