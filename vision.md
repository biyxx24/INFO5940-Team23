# Vision: ReportTrace

## Executive Summary

ReportTrace is a locally run web app that lets research assistants and junior analysts ask questions about the annual reports (Form 10-K) of U.S.-listed companies. The user selects a company and one or more fiscal years, then asks a chatbot any question in plain English. An LLM agent searches the selected reports, looks up the reported figures, and answers with the source of every statement. When several years are selected, the answer shows what changed between them. The goal is to turn a slow and error-prone reading task into an answer that arrives in about a minute and that a reviewer can check.

## Problem Statement

Research assistants and junior analysts are often asked one specific question about a company, such as "Why did gross margin fall this year?" or "How has the description of supply chain risk changed?". The answer is in the annual report, but a 10-K runs over 100 pages and is organised by topic, not by question: the figure is in the financial statements, the explanation is in Management's Discussion and Analysis, and the detail is in a note. Comparing years means repeating the search in each year's report.

Done by hand, this is slow and goes wrong in four typical ways:

1. **Missed evidence.** The report uses different words from the reader, for example "concentration of credit risk" for "customer dependence", so keyword search does not find the passage.
2. **Mixed-up years.** Consecutive reports repeat most of their text, so last year's explanation is quoted as this year's.
3. **Wrong figures.** Numbers are copied and calculated by hand.
4. **Unsupported conclusions.** The reader writes down a cause that the report never states, and a reviewer cannot check it without redoing the reading.

## Target Users

| User                                           | Who they are                                                                                               | What they need                                                                                          |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| Research assistant or junior analyst (primary) | Works on an investment research team, is new to reading 10-Ks, and answers questions from a senior analyst | A fast answer to a specific question, with figures and quotes that can go straight into a research note |
| Reviewer (secondary)                           | A senior analyst who reads the assistant's notes                                                           | To check any statement against its source in one click                                                  |
| Finance student                                | Is learning to read annual reports                                                                         | To find where a topic is discussed and how it changed over the years                                    |

## Success Metrics

All metrics are measured on the evaluation set (feature F11) unless stated otherwise.

| Metric          | Target                                                                                        | How it is measured                                                     |
| --------------- | --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Quote accuracy  | 100% of quotes shown to the user match the cited passage word for word                        | Automatic check in the evaluation script                               |
| Figure accuracy | At least 95% of figures in answers equal the filed value or the correct calculation           | Each test question lists its expected figures                          |
| Retrieval       | For at least 85% of questions, at least one retrieved passage comes from the expected section | Each test question lists its expected section                          |
| Honest gaps     | At least 90% of unanswerable questions get "Not found in the reports" and no invented answer  | The set contains at least 6 unanswerable questions                     |
| Answer quality  | At least 80% of answers are rated correct and complete                                        | Two team members rate each answer as correct, partly correct, or wrong |
| Speed           | Median time to answer is at most 30 seconds for one year and 60 seconds for a comparison      | Timings in the run log (F10)                                           |

## Full Scope Vision

In the long run, ReportTrace could be the first place an analyst goes with a question about any company filing. The MVP below is the first step.

| Phase              | Focus                                                                                                               |
| ------------------ | ------------------------------------------------------------------------------------------------------------------- |
| MVP (this project) | One company at a time, Form 10-K only, pre-built library, one local user                                            |
| Phase 2            | Comparing different companies, quarterly reports (10-Q) and 8-K filings, adding a company from the web page, charts |
| Phase 3            | Uploaded PDFs, reports from other countries and in other languages, user accounts, hosted deployment                |

## MVP Scope

### MVP objective

Show that an LLM agent can answer open questions about a company's annual reports, for one year or across several years, with answers in which every quote and every figure can be traced to its source.

### Features In Scope (MVP)

| ID  | Feature                             | What it does                                                                                                                                                                                                                                                                                                                          | Priority                                                                | Why it is in the MVP                                                                                               |
| --- | ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| F1  | Report library                      | An ingestion script downloads the three most recent Form 10-K filings and the filed financial data of each company from the SEC's EDGAR service. It splits each report into passages labelled with company, fiscal year, and section, and stores them for search. The demo library holds at least 10 companies (at least 30 reports). | Must                                                                    | Every other feature depends on it. A pre-built library avoids upload and parsing problems while the app is in use. |
| F2  | Selection panel                     | The user picks one company from the library and ticks one fiscal year (single-year mode) or two or three years (comparison mode). The panel shows which years are available. The chatbot can use only the selected reports.                                                                                                           | Must                                                                    | Makes the scope of every answer explicit and prevents mixed-up years.                                              |
| F3  | Chatbot                             | Sits beside the selection panel. The user types any question in English. An LLM agent decides what to search, searches again when the evidence is not enough (at most 10 tool calls per question), and writes the answer.                                                                                                             | Must                                                                    | The core of the product.                                                                                           |
| F4  | Agent steps                         | The user sees each step of the agent, for example "Searching the FY2025 report for 'gross margin'". The steps stay visible with the answer.                                                                                                                                                                                           | Must (shown with the answer), Should (shown live while the agent works) | Lets the user see how the answer was found. Live display is the first thing to cut if time is short.               |
| F5  | Year comparison                     | In comparison mode, the agent searches each selected year separately. The answer shows the years side by side: what changed, what stayed the same, and what appears in only some years.                                                                                                                                               | Must                                                                    | The main thing a normal "chat with your PDF" tool cannot do.                                                       |
| F6  | Figures and calculations            | Figures come from the financial data that the company filed with the SEC. Changes, growth rates, and ratios are calculated by code, and the answer shows the formula.                                                                                                                                                                 | Must                                                                    | Prevents wrong figures.                                                                                            |
| F7  | Evidence cards                      | Every statement in an answer has a citation. Clicking it shows the quoted passage, the company, the fiscal year, the section, and a link to the filing on sec.gov.                                                                                                                                                                    | Must                                                                    | Lets a reviewer check a statement in one click.                                                                    |
| F8  | Answer check and gap list           | Before an answer is shown, code checks that every quote appears word for word in a retrieved passage and that every number comes from a tool result. Anything that fails is removed and listed under "Not found in the reports".                                                                                                      | Must                                                                    | Prevents unsupported conclusions.                                                                                  |
| F9  | Questions the reports cannot answer | For investment advice, forecasts, or anything outside the selected reports, the chatbot says that annual reports cannot answer the question and suggests what it can answer.                                                                                                                                                          | Must                                                                    | Keeps the product honest about its limits.                                                                         |
| F10 | Run log                             | The steps, tool calls, evidence, answer, and timing of each question are saved to a local log file.                                                                                                                                                                                                                                   | Must                                                                    | Needed for debugging and for the evaluation.                                                                       |
| F11 | Evaluation                          | A set of at least 30 test questions with expected figures and expected source sections, and a script that runs them and reports the success metrics.                                                                                                                                                                                  | Must                                                                    | Shows that the product works.                                                                                      |

### What every answer contains

1. **Answer.** Two to five sentences that answer the question directly.
2. **Figures.** Each figure used, with its label, fiscal year, value, and unit, and the formula for each calculated value.
3. **What the company says.** Quotes from the reports, each with a citation.
4. **Not found in the reports.** The parts of the question that the selected reports do not answer.

### Features Explicitly Out of Scope (MVP)

| Feature                                                                       | Why it is out                                                                                                | Later phase |
| ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ | ----------- |
| Comparing different companies in one question                                 | Needs matching definitions of figures across companies and doubles the retrieval and checking logic          | Phase 2     |
| Adding a company from the web page                                            | Ingestion takes minutes and can fail on unusual filings; a script run by the team is enough                  | Phase 2     |
| Filings other than Form 10-K (10-Q, 8-K, proxy statements, earnings calls)    | Each has a different structure; Construction has only three weeks                                            | Phase 2     |
| Charts                                                                        | Tables are enough to check figures                                                                           | Phase 2     |
| Uploading PDFs                                                                | Text and tables are unreliable to extract from PDFs; EDGAR already provides the reports in a standard format | Phase 3     |
| Companies that do not file a 10-K (foreign companies, Chinese annual reports) | No financial data in the same format; other languages                                                        | Phase 3     |
| Questions and answers in languages other than English                         | The reports are in English and quotes must be word for word                                                  | Phase 3     |
| Stock prices, news, and web search                                            | Not in the reports; would mix checked and unchecked sources                                                  | Phase 3     |
| User accounts and conversations saved between sessions                        | One local user is enough for the MVP                                                                         | Phase 3     |
| Cloud deployment and several users at once                                    | The course project only has to run on our laptops                                                            | Phase 3     |
| Investment advice, forecasts, price targets                                   | An annual report cannot support them                                                                         | Never       |

### MVP user journeys

**Journey 1: a question about one year**

1. The research assistant opens ReportTrace, picks a company, and ticks FY2025.
2. She asks "How dependent is the company on its largest customers?".
3. She sees the agent's steps: it searches the FY2025 report, opens a note that the first passage refers to, and looks up the related figures.
4. She reads the answer: a direct answer, a table of figures, two quotes with citations, and one item under "Not found in the reports".
5. She clicks a citation, reads the passage on the evidence card, and opens the filing to confirm it.

**Outcome:** a checked answer that she can put into her note.

**Journey 2: a comparison across years**

1. She keeps the same company and ticks FY2023, FY2024, and FY2025.
2. She asks "How has the description of supply chain risk changed?".
3. The agent searches each year's report separately.
4. The answer shows the three years side by side: what was added, what was removed, and what stayed the same, each with a citation that carries its year.

**Outcome:** she sees the change over three years without opening three reports.

**Journey 3: a question the reports cannot answer**

1. She asks "Should I buy this stock?".
2. The chatbot says that annual reports cannot answer this and suggests related questions that they can answer, such as how revenue and margins changed.

**Outcome:** she is not misled by an invented answer.

### Constraints and assumptions

- **Deadline.** The product must be finished by December 16, 2026. Construction runs from November 25 to December 16. The team has four students.
- **Local only.** The product runs on the team's laptops (Windows and macOS), for one user at a time.
- **LLM access.** The LLM is reached through the API that Cornell provides to students, so usage must stay within the course quota.
- **Language.** Reports, questions, answers, and the interface are in English.
- **Assumption:** EDGAR database provides each demo company's 10-K as a web page and its financial data in a machine-readable form. **Risk if wrong:** we replace that company with another one.
- **Accepted limitation:** a figure that is not in the filed financial data, such as some segment figures, can only be quoted from the report text.
- **Accepted limitation:** the library changes only when the team runs the ingestion script again.

## Risks

| Risk                                                                                                | Likelihood | Impact | What we will do                                                                                                                 |
| --------------------------------------------------------------------------------------------------- | ---------- | ------ | ------------------------------------------------------------------------------------------------------------------------------- |
| The layout of 10-K web pages differs between companies, so some sections are not recognised         | High       | Medium | Test the parser on every demo company; keep unrecognised text searchable under "Other"; replace a company that cannot be parsed |
| The filed financial data lacks a figure that a user asks for                                        | Medium     | Medium | Quote the figure from the report text and label it as quoted, or list it as not found                                           |
| The agent picks the wrong financial line item, for example total revenue instead of product revenue | Medium     | High   | Show the label of every figure in the answer; include such cases in the evaluation set                                          |
| The answer check removes correct statements                                                         | Medium     | Medium | Ignore differences in spacing and quotation marks when matching quotes; measure how often statements are removed                |
| The course API quota or rate limit is too low                                                       | Low        | High   | Embed each report only once; use a fake model in automated tests; run the full evaluation only a few times                      |
| Three weeks are not enough for all features                                                         | Medium     | High   | Build in this order: library, tools, agent, answer check, chatbot, comparison view, evaluation. Live step display is cut first  |

## Open Questions

- Which companies go into the demo library? Our proposal is three technology companies, three retailers, two consumer goods companies, one carmaker, and one drug company (for example AAPL, MSFT, NVDA, AMZN, WMT, COST, KO, NKE, TSLA, PFE).
- Should a follow-up question use the earlier questions and answers in the same conversation, or is every question answered on its own?
- What happens to the conversation when the user changes the company or the selected years: does it start again, or continue with a note that the selection changed?
- How many years may be compared at once: any two or all three?
- How should fiscal years be labelled for a company whose fiscal year does not end in December? Our proposal is to use the company's own label.
- When a figure is only available in the report text, should the answer show it with a "quoted from the report" label, or list it as not found?
- Should the evidence card show only the quoted passage, or also the text before and after it?
- Who writes the expected answers for the evaluation set, and how do we settle a disagreement between the two raters?
