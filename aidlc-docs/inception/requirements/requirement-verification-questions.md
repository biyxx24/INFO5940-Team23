# Requirements Verification Questions: ReportTrace

These questions come from reading `vision.md` and `tech-env.md`. Both documents are detailed, so the questions cover only what they leave open:

- the eight open questions listed at the end of `vision.md`,
- other points where the two documents allow more than one reading, and
- the three AI-DLC extension opt-ins.

There are 26 questions in seven parts.

**How to answer:** write the letter of your choice after each `[Answer]:` tag. If no option fits, choose X and describe what you want after the tag. You may add a short note after any letter.

---

**Part A: Report library and figures (F1, F6)**

## Question 1

Which companies go into the demo library (F1)? `vision.md` proposes ten: AAPL, MSFT, NVDA, AMZN, WMT, COST, KO, NKE, TSLA, PFE.

A) Exactly the ten proposed companies. A company is replaced only if its filings cannot be parsed

B) The ten proposed companies plus spare companies that are ingested from the start, so that a failing company can be dropped without delay (please list the spares after the tag)

C) A different list (please list the tickers after the tag)

X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 2

How is a fiscal year labelled when the company's fiscal year does not end in December (F1, F2)? Example: Walmart calls its year that ended on January 31, 2025 "fiscal 2025", although eleven of its months fall in 2024.

A) The company's own label (as `vision.md` proposes), and the selection panel also shows the date on which the year ended, for example "FY2025 (ended Jan 31, 2025)"

B) The company's own label, with no end date shown

C) The calendar year in which most of the fiscal year falls

X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 3

How detailed are the section labels given to passages (F1)? The label decides what `read_section` can open, what an evidence card shows as its section, and what counts as the "expected section" in the retrieval metric. In Journey 1 the agent opens a single note.

A) The standard 10-K items only (for example Item 1 Business, Item 1A Risk Factors, Item 7 Management's Discussion and Analysis, Item 8 Financial Statements), plus "Other" for text that is not recognised

B) The standard 10-K items, and in addition each note to the financial statements in Item 8 as its own section (for example "Item 8, Note 12"), plus "Other"

X) Other (please describe after [Answer]: tag below)

[Answer]:  A

## Question 4

Which filed figures must `find_financial_data` be able to return (F6)? The SEC's financial data for one company holds several hundred tagged line items.

A) A fixed list of common line items that the team chooses and maps by hand (for example revenue, cost of revenue, gross profit, operating income, net income, earnings per share, total assets, total liabilities, equity, operating cash flow, capital expenditure, research and development expense)

B) Every line item that the company filed for the fiscal year, found by searching the filed labels

C) The fixed list first; if the figure is not on it, a search over every filed line item, with the answer showing the filed label

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 5

A 10-K also reports figures of earlier years for comparison: the FY2025 report contains FY2024 figures, and FY2023 figures for some statements. If the user ticks only FY2025 and asks "Why did gross margin fall this year?", may the answer use the FY2024 figures that are printed in the FY2025 report (F2, F6)?

A) Yes. Any figure reported in a selected report may be used, including figures of earlier years, and each is shown with its own fiscal year and the report it came from

B) No. Only figures for the ticked fiscal years may be used. To compare with an earlier year the user must tick that year as well, and the answer says so

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 6

The same figure can be filed twice with different values: first in that year's own report, and again, sometimes restated, as a comparison figure in a later report. Which value does ReportTrace use (F6, figure accuracy metric)?

A) The value from the fiscal year's own report whenever that report is selected; otherwise the value from the selected report that contains it

B) The most recently filed value among the selected reports, marked "restated" when it differs from the value in an earlier selected report

C) Both values are shown when they differ

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 7

When a figure exists only in the report text and not in the filed financial data (for example some segment figures), what does the answer do (F6, F8)?

A) Show it as a quote with the label "quoted from the report" and a citation. It must appear word for word in a retrieved passage, and it is not used in calculations

B) As in A, but the figure may also be used in calculations, and the result carries the label "based on a quoted figure"

C) List it under "Not found in the reports"

X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Question 8

How are figures shown, and when does a rounded figure count as correct (F6, F8, figure accuracy metric)? Example: filed revenue of 391,035,000,000 US dollars.

A) Always the full filed value, in the Figures table and in the answer sentences

B) Rounded for reading everywhere (for example "$391.0 billion"). A rounded figure is correct when it equals the filed or calculated value rounded to the precision shown

C) The Figures table shows the full filed value, and the answer sentences may use rounded figures under the rule in B

X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

**Part B: Selection and conversation (F2, F3)**

## Question 9

Which combinations of years can be ticked in comparison mode (F2, F5)?

A) Any two or all three of the available years, including two years that are not next to each other (for example FY2023 and FY2025)

B) Only years that are next to each other: two consecutive years, or all three

X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 10

Does a follow-up question use the earlier questions and answers of the same conversation (F3)? Example: after "How did revenue change?", the user asks "And what about its largest segment?".

A) No. Every question is answered on its own. Earlier turns stay visible on the page but are not sent to the agent

B) Yes. The agent receives the earlier turns of the conversation. They are kept in memory only and are lost when the app restarts or the conversation is cleared

X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Question 11

What happens to the conversation when the user changes the company or the ticked years (F2, F3)?

A) A new conversation starts and the chat is cleared

B) The chat stays on the page with a visible note that the selection changed. Each earlier answer keeps a label showing the selection it was based on, and the agent does not use the earlier turns

C) The chat stays on the page with a visible note, and the agent may still use the earlier turns (only possible if Question 10 is answered B)

X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

**Part C: Answers, checking, and evidence (F7, F8)**

## Question 12

F7 says that every statement in an answer has a citation, and F8 checks quotes and numbers by code. What must be true of a sentence in the "Answer" part that is written in the model's own words and contains no quote and no number?

A) It must cite at least one passage or figure that a tool returned for this question. Code checks that the cited item exists, and removes a sentence that has no valid citation

B) As in A, and in addition the cited item must itself be shown in the answer, as a quote under "What the company says" or as a row in the Figures table

C) No check. Only quotes and numbers are checked by code

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 13

When the answer check removes a quote, a figure, or a sentence, does the agent get one more attempt before the answer is shown (F8)? A second attempt costs time and API quota.

A) No. The parts that passed are shown at once, and the gaps are listed under "Not found in the reports"

B) Yes, once. The agent is told what failed and writes the answer again. Whatever fails the second check is removed as in A

X) Other (please describe after [Answer]: tag below)

[Answer]:B

## Question 14

F8 says that anything that fails the check is "removed and listed under Not found in the reports". What does the user see for a removed item?

A) Only what is missing, in neutral words (for example "No supported statement on the cause of the margin change"). The removed text itself is not shown and is kept only in the run log

B) The removed text itself, clearly marked as "could not be verified in the reports"

X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Question 15

What does an evidence card show (F7)?

A) The cited passage only, with the quoted words highlighted

B) The cited passage together with the passage before it and the passage after it from the same section, with the quoted words highlighted

C) The cited passage with the quoted words highlighted, and a control that shows the passages before and after it on request

X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

**Part D: Limits and failures (F3, speed metric)**

## Question 16

What happens when the agent reaches the limit of 10 tool calls before it has finished (F3)?

A) The agent writes the best answer it can from the evidence found so far. Unanswered parts go under "Not found in the reports", and the answer carries a note that the search limit was reached

B) No answer is shown. The user sees a message that the question needs too many steps and is asked to make it narrower

X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 17

The speed targets are medians (30 seconds for one year, 60 seconds for a comparison). Is there also a hard time limit for one question?

A) No overall limit. The limits in `tech-env.md` are enough (10 tool calls; 60 seconds and 2 retries for each model call)

B) Yes. After a fixed time the run stops, and the user sees an error message together with the steps completed so far (please give the number of seconds after the tag, for example 120)

C) Yes, with a fixed time as in B, but the evidence found so far is turned into a partial answer as in Question 16 option A (please give the number of seconds after the tag)

X) Other (please describe after [Answer]: tag below)

[Answer]:A

---

**Part E: LLM access**

## Question 18

What has the team already confirmed about the Cornell API? The answer decides whether the fallbacks in `tech-env.md` (embeddings computed on the laptop, a local Ollama server) are requirements or only precautions.

A) Confirmed: GPT-4o with tool calling and `text-embedding-3-small` both work through the Cornell API

B) Confirmed: GPT-4o with tool calling works. Embeddings are not yet tested

C) Nothing is confirmed yet

X) Other (please describe after [Answer]: tag below)

[Answer]:B

## Question 19

What is the usage limit of the Cornell API that the product must stay within?

A) We know the limit (please describe it after the tag, for example a budget in dollars, tokens per day, or requests per minute)

B) We do not know it yet. The requirements should keep usage low by design and record the token usage of every question in the run log, so that the team can track it

X) Other (please describe after [Answer]: tag below)

[Answer]:B

---

**Part F: Evaluation (F11)**

## Question 20

How is the evaluation set of at least 30 questions divided (F11)? `vision.md` fixes only that at least 6 questions are unanswerable.

A) Fixed by question type only: about 10 single-year text questions, 8 figure and calculation questions, 6 year-comparison questions, and 6 unanswerable questions. Companies are chosen freely

B) Fixed by question type as in A and also by company: every demo company has at least 3 questions

C) The team decides the split during Construction. The requirements fix only the minimums in `vision.md`

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 21

Who writes the expected figures and expected sections for the evaluation set (F11)?

A) All four members: each writes a share of the questions, and a different member checks each one against the filing

B) One or two members own the whole set, and the other members review it

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 22

Two team members rate every answer as correct, partly correct, or wrong. What happens when they disagree (answer quality metric)?

A) A third member decides

B) The lower of the two ratings counts

C) The two raters discuss until they agree, and the question is marked as "disputed" in the results

X) Other (please describe after [Answer]: tag below)

[Answer]:C

## Question 23

How binding are the targets in the Success Metrics table of `vision.md`?

A) They are acceptance criteria: the product is not finished until the final evaluation run meets every target

B) They are goals: the final evaluation run reports the real numbers, and a missed target is explained in the report

C) Mixed: quote accuracy (100%) and honest gaps (90%) must be met; the other targets are goals

X) Other (please describe after [Answer]: tag below)

[Answer]:B

---

**Part G: AI-DLC extensions**

Each extension that is switched on adds rules that every later stage must follow. Context from the input documents: the product runs locally for one user with no login and no cloud services; `tech-env.md` has its own Security section and already requires `hypothesis` property-based tests for the `calculate` tool and the answer check.

## Question 24: Security Extensions

Should security extension rules be enforced for this project?

A) Yes — enforce all SECURITY rules as blocking constraints (recommended for production-grade applications)

B) No — skip all SECURITY rules (suitable for PoCs, prototypes, and experimental projects)

X) Other (please describe after [Answer]: tag below)

[Answer]:B

## Question 25: Property-Based Testing Extension

Should property-based testing (PBT) rules be enforced for this project?

A) Yes — enforce all PBT rules as blocking constraints (recommended for projects with business logic, data transformations, serialization, or stateful components)

B) Partial — enforce PBT rules only for pure functions and serialization round-trips (suitable for projects with limited algorithmic complexity)

C) No — skip all PBT rules (suitable for simple CRUD applications, UI-only projects, or thin integration layers with no significant business logic)

X) Other (please describe after [Answer]: tag below)

[Answer]:C

## Question 26: Resiliency Extensions

Should the resiliency baseline be applied to this project?

**What this extension is.** Enabling it applies a set of **directional, design-time best practices** for building resilient systems, derived from the **AWS Well-Architected Framework (Reliability Pillar)** and resilience-review guidance. It steers requirements, design, and code toward fault tolerance, high availability, observability, and recoverability — covering 15 practice areas across business goals, change management, observability, high availability, disaster recovery, and continuous improvement.

**What this extension is NOT.** Enabling it does **not** make your workload production-ready, nor does it certify or guarantee any availability, RTO, or RPO target. It is a **starting point** that scaffolds good resiliency decisions early — it is not a substitute for a formal **AWS Well-Architected Review** of the built system.

Treat the output as a well-grounded **first draft of your resiliency posture** to build on and validate — not a finished, production-certified result.

A) Yes — apply the resiliency baseline as directional best practices and design-time guidance (recommended for business-critical workloads, as an informed starting point that you can validate and harden before go-live)

B) No — skip the resiliency baseline (suitable for PoCs, prototypes, and experimental projects where rapid iteration matters more than reliability)

X) Other (please describe after [Answer]: tag below)

[Answer]:B
