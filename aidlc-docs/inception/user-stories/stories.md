# User Stories: ReportTrace

## 1. Overview

This document turns the approved requirements (`aidlc-docs/inception/requirements/requirements.md`) into user stories. It follows the approved method in `aidlc-docs/inception/plans/story-generation-plan.md`:

- **Eight epics**, one for each feature that has stories (F2 to F9). The report library (F1), the run log (F10), and the evaluation (F11) have no stories. They are supporting requirements (Section 6).
- **14 stories**, written from the research assistant's point of view. The reviewer and the finance student are mapped to these stories (Section 5). The persona profiles are in `personas.md`.
- Each story has:
  - the sentence "As a ..., I want ..., so that ...";
  - its priority from `vision.md`;
  - the requirement IDs it covers;
  - acceptance criteria as a numbered checklist in the user's words.
- **Failure and edge cases** are acceptance criteria of the story they belong to.
- **Technical rules** are referenced by requirement ID, for example FR-8.2, and are defined in `requirements.md`.
- **Non-functional requirements** are cross-cutting constraints (Section 7).

**Words used in the stories**

| Word | Meaning |
| --- | --- |
| The chatbot | Everything in ReportTrace that answers a question |
| Step | One search of a selected report, one opening of a section, one look-up of a reported figure, or one calculation. The chatbot takes at most 10 steps for one question |
| Citation | The link from a statement in an answer to its source |
| Conversation | The questions and answers since the last change of selection or the last clear (`requirements.md` Section 5) |
| ACn | Acceptance criterion number n of a story, for example US-6.1 AC2 |

## 2. Epics

| Epic | Feature | Stories | Priority |
| --- | --- | --- | --- |
| E2 | F2 Selection panel | US-2.1, US-2.2 | Must |
| E3 | F3 Chatbot | US-3.1, US-3.2 | Must |
| E4 | F4 Agent steps | US-4.1, US-4.2 | Must (US-4.1), Should (US-4.2) |
| E5 | F5 Year comparison | US-5.1 | Must |
| E6 | F6 Figures and calculations | US-6.1, US-6.2 | Must |
| E7 | F7 Evidence cards | US-7.1, US-7.2 | Must |
| E8 | F8 Answer check and gap list | US-8.1, US-8.2 | Must |
| E9 | F9 Questions the reports cannot answer | US-9.1 | Must |

---

## 3. Stories

### Epic E2: Selection panel (F2)

#### US-2.1 Choose the company and years

- **Persona:** Research assistant
- **Priority:** Must
- **Requirements:** FR-2.1, FR-2.2, FR-2.3, FR-2.4, NFR-3.3, NFR-3.4, NFR-8.2

**Story:** As a research assistant, I want to pick one company and tick one or more of its fiscal years, so that every answer is limited to the reports I chose.

**Acceptance criteria:**

1. The panel lists every company in the library, and exactly one company can be picked at a time.
2. For the picked company, the panel lists each available fiscal year with the company's own label and the date on which the year ended, for example "FY2025 (ended Jan 31, 2025)".
3. Ticking one year gives a single-year answer. Ticking two or all three years gives a comparison.
4. Any two years can be ticked, including two years that are not next to each other, for example FY2023 and FY2025.
5. A question cannot be sent while no year is ticked, and the page says why.
6. No answer uses a passage from another company or from a report that is not selected. Figures of an earlier year that are printed in a selected report are allowed (US-6.1 AC3).
7. A question cannot make the chatbot use another company or a year that is not ticked, even if the question asks for it (FR-2.4).
8. The current selection is always visible next to the chat.

#### US-2.2 Change the selection during a conversation

- **Persona:** Research assistant
- **Priority:** Must
- **Requirements:** FR-2.5, FR-8.5, NFR-8.2

**Story:** As a research assistant, I want to change the company or the years without losing the answers I have already read, so that I can move on and still know which reports each answer was based on.

**Acceptance criteria:**

1. After a change of company or years, the earlier questions and answers stay on the page.
2. A visible note in the chat marks the point where the selection changed.
3. Each earlier answer shows the company and the years it was based on.
4. The first question after the change starts a new conversation. It does not use the earlier questions, the earlier answers, or the evidence found for them.
5. Changing back to an earlier selection also starts a new conversation.

---

### Epic E3: Chatbot (F3)

#### US-3.1 Ask a question and get a structured answer

- **Persona:** Research assistant
- **Priority:** Must
- **Requirements:** FR-3.1, FR-3.2, FR-3.3, FR-3.5, FR-3.6, FR-3.7, NFR-1.1, NFR-3.3, NFR-5.2, NFR-5.3, NFR-8.1, D6

**Story:** As a research assistant, I want to ask any question about the selected reports in plain English and get a direct answer in a fixed layout, so that I can put it into my research note quickly.

**Acceptance criteria:**

1. The chat sits beside the selection panel. A question of 1 to 1,000 characters can be sent. An empty question, or a longer one, is refused with a message that says why.
2. Every answer has four parts, in this order:
   - **Answer:** two to five sentences that answer the question directly;
   - **Figures**;
   - **What the company says**;
   - **Not found in the reports**.
3. When the first results are not enough, the chatbot searches again. It takes at most 10 steps for one question (FR-3.3).
4. If the chatbot reaches the limit of 10 steps before it has finished, it still shows an answer built from what it found so far. The unanswered parts are listed under "Not found in the reports", and a note says that the limit was reached (FR-3.5).
5. A problem inside one step does not end the answer. The chatbot is told about the problem and continues, and the user never sees a crash (FR-3.6).
6. If the AI service that writes the answers still does not respond after its retries, the user sees a clear error message with the steps completed so far. No unchecked answer is shown (NFR-5.2, D6).
7. If the part of ReportTrace that answers questions is not running, the page shows a clear message instead of crashing (NFR-5.3).
8. The median time to an answer about one year is at most 30 seconds, as measured by the evaluation (NFR-1.1).
9. The page, the questions, and the answers are in English (NFR-8.1).

#### US-3.2 Ask a follow-up question

- **Persona:** Research assistant
- **Priority:** Must
- **Requirements:** FR-3.4, FR-8.5

**Story:** As a research assistant, I want to ask a follow-up question that refers to an earlier answer, so that I do not have to repeat what I already asked.

**Acceptance criteria:**

1. The chatbot answers a follow-up question with the earlier questions and answers of the same conversation in mind. Example: after "How did revenue change?", the follow-up "And what about its largest segment?" is understood.
2. A follow-up answer may use and cite a passage or figure that was found for an earlier question of the same conversation.
3. The page has a control to clear the conversation. After clearing, the next question is answered on its own.
4. After ReportTrace restarts, earlier conversations are gone.
5. After a change of selection, the next question does not use the earlier turns (US-2.2 AC4).

---

### Epic E4: Agent steps (F4)

#### US-4.1 See how the answer was found

- **Persona:** Research assistant
- **Priority:** Must
- **Requirements:** FR-4.1, FR-4.2

**Story:** As a research assistant, I want to see each step the chatbot took to find the answer, so that I can judge how the answer was found.

**Acceptance criteria:**

1. Each step is described in plain words that name the fiscal year and what was looked for, for example "Searching the FY2025 report for 'gross margin'".
2. Opening a section, looking up a reported figure, and calculating a value are described in the same way, for example "Opening Item 8, Note 12 of the FY2025 report".
3. The steps stay visible with their answer, also after later questions are asked.
4. Steps taken in a second try (US-8.1 AC5) appear in the same list.

#### US-4.2 Watch the steps while the chatbot works

- **Persona:** Research assistant
- **Priority:** Should (the first feature to cut if time is short)
- **Requirements:** FR-4.3

**Story:** As a research assistant, I want to see each step as soon as the chatbot takes it, so that I know the chatbot is working and can follow it while I wait.

**Acceptance criteria:**

1. Each step appears on the page while the chatbot is still working, before the answer is ready.
2. When the answer appears, the steps stay with it in the order in which they were taken, as in US-4.1.
3. If this story is cut, US-4.1 still holds: all steps are shown with the answer.

---

### Epic E5: Year comparison (F5)

#### US-5.1 Compare the selected years side by side

- **Persona:** Research assistant
- **Priority:** Must
- **Requirements:** FR-5.1, FR-5.2, FR-5.3, FR-2.3, NFR-1.1

**Story:** As a research assistant, I want to see how the selected years differ on my question, so that I see the change over the years without opening several reports.

**Acceptance criteria:**

1. With two or three years ticked, each year's report is searched separately, and the steps show a separate search for each year.
2. The answer shows the selected years side by side: what changed, what stayed the same, and what appears in only some of the years.
3. Every citation in a comparison shows its fiscal year.
4. If nothing is found for one of the selected years, the answer says so under "Not found in the reports" and does not fill the gap with another year's text.
5. A comparison of two years that are not next to each other, for example FY2023 and FY2025, compares those two years only.
6. The median time to a comparison answer is at most 60 seconds, as measured by the evaluation (NFR-1.1).

---

### Epic E6: Figures and calculations (F6)

#### US-6.1 Use reported figures with their source

- **Persona:** Research assistant
- **Priority:** Must
- **Requirements:** FR-6.1, FR-6.2, FR-6.3, FR-6.5, FR-6.6, D1, D2

**Story:** As a research assistant, I want every figure in an answer to come from what the company reported, with its label and year, so that I can use it without copying numbers by hand.

**Acceptance criteria:**

1. Each figure in the Figures part shows its fiscal year, its value, its unit, and its label as the company reported it. The label makes clear which line item it is, so that, for example, total revenue cannot be mistaken for product revenue.
2. Figures are rounded for reading, for example "$391.0 billion". A rounded figure equals the reported value rounded to the precision shown.
3. A figure of an earlier year that is printed in a selected report may be used. Example: the FY2024 figure in the FY2025 report when only FY2025 is ticked. The figure shows its own fiscal year and the report it came from.
4. When the same figure was reported in more than one selected report, the value from that year's own report is used. If that report is not selected, the value comes from the earliest selected report that contains it (FR-6.3, D1).
5. A figure that is not among the reported figures but is written in the report text is shown as a quote labelled "quoted from the report", exactly as written, with a citation (FR-6.5, D2).
6. A figure that is neither among the reported figures nor written in the selected reports is not guessed. It is listed under "Not found in the reports".

#### US-6.2 See how a calculated value was worked out

- **Persona:** Research assistant
- **Priority:** Must
- **Requirements:** FR-6.4, FR-6.5, FR-6.6

**Story:** As a research assistant, I want changes, growth rates, and ratios to be calculated for me with the formula shown, so that I can trust the result and repeat it myself.

**Acceptance criteria:**

1. Each calculated value shows its formula and the figures it uses, each with its fiscal year.
2. The result is exact and is rounded only when it is shown, by the rule in US-6.1 AC2 (FR-6.4).
3. A calculation may use a figure quoted from the report text. Its result then carries the label "based on a quoted figure" (FR-6.5).
4. If a figure needed for a calculation is missing, no value is invented, and the missing figure is listed under "Not found in the reports".

---

### Epic E7: Evidence cards (F7)

#### US-7.1 Check a statement against its passage

- **Persona:** Research assistant
- **Priority:** Must
- **Requirements:** FR-7.1, FR-7.2, FR-7.3, FR-7.5, NFR-8.3, and the reviewer's need (story plan Q10, CQ2)

**Story:** As a research assistant, I want to open the passage behind any citation in one click, so that I can confirm the statement and give my reviewer a source that they can check.

**Acceptance criteria:**

1. Every statement in an answer has a citation.
2. Clicking a passage citation opens an evidence card. The card shows:
   - the cited passage, with the quoted words highlighted;
   - the company and the fiscal year;
   - the section, for example "Item 1A Risk Factors" or "Item 8, Note 12";
   - a link to the filing on sec.gov.
3. On request, the card also shows the passage just before and the passage just after the cited passage, from the same section.
4. The company, fiscal year, section, and sec.gov link are shown as plain text that can be selected and copied into a research note (reviewer's need).
5. The sec.gov link opens the filing that contains the passage, so that a reviewer who reads only the note can open the source in one click (reviewer's need).
6. If a citation no longer points to a stored passage, the card shows a clear "evidence not found" message instead of crashing (FR-7.5).

#### US-7.2 Check a figure against its filing

- **Persona:** Research assistant
- **Priority:** Must
- **Requirements:** FR-7.4, NFR-8.3, D3, and the reviewer's need (story plan Q10, CQ2)

**Story:** As a research assistant, I want to open the source of any figure in one click, so that I can confirm the reported value and pass its source on to my reviewer.

**Acceptance criteria:**

1. Clicking a figure citation opens a card that shows:
   - the label as the company reported it;
   - the fiscal year, the value, and the unit;
   - the period that the figure covers;
   - a link to the filing that reported it.
2. For a figure of an earlier year printed in a later report (US-6.1 AC3), the card names the report that the value came from.
3. The details and the link are shown as plain text that can be copied into a research note, so that a reviewer can open the filing from the note in one click (reviewer's need).
4. A citation of a figure quoted from the report text opens a passage card, as in US-7.1.

---

### Epic E8: Answer check and gap list (F8)

#### US-8.1 See only statements that the reports support

- **Persona:** Research assistant
- **Priority:** Must
- **Requirements:** FR-8.1, FR-8.2, FR-8.3, FR-8.4, FR-8.5, FR-8.6, FR-3.3, D5

**Story:** As a research assistant, I want every quote, figure, and statement to be checked against the reports before I see the answer, so that I never pass on something that the reports do not say.

**Acceptance criteria:**

1. Every quote that is shown matches its cited passage word for word. Differences only in spacing or in the style of quotation marks do not count as a mismatch (FR-8.2).
2. Every figure that is shown comes from a reported figure, a calculation, or a passage found in the current conversation. It matches its source under the rounding rule in US-6.1 AC2 (FR-8.3).
3. Every sentence of the Answer part without a quote or a figure cites at least one passage or figure found in the current conversation (FR-8.4, FR-8.5).
4. A quote, figure, or sentence that fails any of these checks is never shown.
5. If anything fails, the chatbot gets one more try before the answer is shown. The second try may take more steps, but both tries together stay within the limit of 10 steps for the question (FR-8.6).
6. If the limit of 10 steps was already reached, the second try rewrites the answer from what was already found, without new steps (D5).
7. Whatever fails the check a second time is removed, and the answer is shown with the parts that passed.

#### US-8.2 Know what the reports do not answer

- **Persona:** Research assistant
- **Priority:** Must
- **Requirements:** FR-8.7, FR-9.3, FR-3.7, FR-5.3, D4

**Story:** As a research assistant, I want a clear list of what the selected reports do not answer, so that I know where my note has gaps and do not fill them with guesses.

**Acceptance criteria:**

1. "Not found in the reports" lists every part of the question that the selected reports do not answer.
2. For each statement that the check removed, the list shows a neutral line that says what is missing, for example "No supported statement on the cause of the margin change" (FR-8.7, D4).
3. The removed text itself is never shown to the user. It is kept only in the team's log (FR-8.7, FR-10.1).
4. When nothing is missing, the part is still shown and says so (FR-3.7).
5. If no evidence is found for the question at all, the answer says "Not found in the reports" and gives no invented answer (FR-9.3).

---

### Epic E9: Questions the reports cannot answer (F9)

#### US-9.1 Be told when a question is outside the reports

- **Persona:** Research assistant
- **Priority:** Must
- **Requirements:** FR-9.1, FR-9.2, FR-9.3, D9, D11

**Story:** As a research assistant, I want to be told plainly when annual reports cannot answer my question, and what I could ask instead, so that I am not misled by an invented answer.

**Acceptance criteria:**

1. For investment advice, forecasts, or price targets, for example "Should I buy this stock?", the chatbot says that annual reports cannot answer the question.
2. The chatbot says the same for stock prices, news, and any other topic outside the selected reports.
3. The response suggests related questions that the reports can answer, for example how revenue and margins changed.
4. For a question about a company that is not selected, the chatbot says that it can answer only about the selected company and suggests changing the selection (FR-9.2, D11).
5. The response contains no figure or quote that does not come from the selected reports.

---

## 4. Journey Map

The map covers the three user journeys in `vision.md` (story plan Q12).

### Journey 1: A question about one year

| Step | What happens | Stories |
| --- | --- | --- |
| 1 | The research assistant opens ReportTrace, picks a company, and ticks FY2025 | US-2.1 |
| 2 | The research assistant asks "How dependent is the company on its largest customers?" | US-3.1 |
| 3 | The steps show a search of the FY2025 report, the opening of a note that the first passage refers to, and a look-up of the related figures | US-4.1, US-4.2 (live display, Should) |
| 4 | The answer shows a direct answer, a table of figures, two quotes with citations, and one item under "Not found in the reports" | US-3.1, US-6.1, US-8.1, US-8.2 |
| 5 | The research assistant clicks a citation, reads the passage on the evidence card, and opens the filing to confirm it | US-7.1 |

**Outcome:** a checked answer that can go into the research note. The reviewer checks it from the note through the sec.gov link (US-7.1 AC4 and AC5).

### Journey 2: A comparison across years

| Step | What happens | Stories |
| --- | --- | --- |
| 1 | The research assistant keeps the same company and ticks FY2023, FY2024, and FY2025 | US-2.1, US-2.2 |
| 2 | The research assistant asks "How has the description of supply chain risk changed?" | US-3.1 |
| 3 | The chatbot searches each year's report separately | US-5.1, US-4.1 |
| 4 | The answer shows the three years side by side: what was added, what was removed, and what stayed the same, each with a citation that carries its year | US-5.1, US-7.1, US-8.1 |

**Outcome:** the research assistant sees the change over three years without opening three reports.

### Journey 3: A question the reports cannot answer

| Step | What happens | Stories |
| --- | --- | --- |
| 1 | The research assistant asks "Should I buy this stock?" | US-3.1 |
| 2 | The chatbot says that annual reports cannot answer this, and suggests related questions that they can answer, such as how revenue and margins changed | US-9.1 |

**Outcome:** the research assistant is not misled by an invented answer.

**Stories that no journey passes through:** US-3.2 (follow-up questions), US-6.2 (calculations), and US-7.2 (figure cards). They come from requirements that the three journeys in `vision.md` do not show.

---

## 5. Persona Mapping

- **Owner:** the persona whose story it is.
- **Uses:** uses the same feature in ReportTrace.
- **Benefits:** does not use ReportTrace, but relies on the result in the research note.

| Story | P1 Research assistant | P2 Reviewer | P3 Finance student |
| --- | --- | --- | --- |
| US-2.1 Choose the company and years | Owner | | Uses |
| US-2.2 Change the selection during a conversation | Owner | | Uses |
| US-3.1 Ask a question and get a structured answer | Owner | | Uses |
| US-3.2 Ask a follow-up question | Owner | | Uses |
| US-4.1 See how the answer was found | Owner | | Uses (sees where a topic is discussed) |
| US-4.2 Watch the steps while the chatbot works | Owner | | Uses |
| US-5.1 Compare the selected years side by side | Owner | | Uses (sees how a topic changed) |
| US-6.1 Use reported figures with their source | Owner | Benefits (figures in the note carry their label, year, and source) | |
| US-6.2 See how a calculated value was worked out | Owner | Benefits (formulas can be repeated) | |
| US-7.1 Check a statement against its passage | Owner | Benefits (need in AC4 and AC5) | Uses (sections show where a topic is discussed) |
| US-7.2 Check a figure against its filing | Owner | Benefits (need in AC3) | |
| US-8.1 See only statements that the reports support | Owner | Benefits (only supported statements reach the note) | |
| US-8.2 Know what the reports do not answer | Owner | Benefits (gaps are stated, not guessed) | Uses |
| US-9.1 Be told when a question is outside the reports | Owner | | Uses |

---

## 6. Supporting Requirements

These features have no stories (story plan Q8, CQ1). The stories depend on them.

| Feature | Requirements | What the stories depend on |
| --- | --- | --- |
| F1 Report library | FR-1.1 to FR-1.10, D7 | Every story needs the library of ten companies with three fiscal years each. Three parts of the library appear directly in the stories: the fiscal year labels and end dates (FR-1.7) in US-2.1; the section labels, including one section per note (FR-1.6), in US-4.1 and US-7.1; and the fixed list of reported figures (FR-1.8), which limits what US-6.1 can return |
| F10 Run log | FR-10.1, FR-10.2, FR-8.8 | Keeps the removed text of US-8.2, the timings behind the speed criteria of US-3.1 and US-5.1, the token usage of every question, and the number of statements removed from each answer (FR-8.8) |
| F11 Evaluation | FR-11.1 to FR-11.7, D8, D9 | Measures the success metrics that the stories serve. Each metric and its stories are listed below the table |

The metrics measured by F11, with the stories they serve:

| Metric | Stories |
| --- | --- |
| Quote accuracy | US-7.1, US-8.1 |
| Figure accuracy | US-6.1, US-6.2, US-8.1 |
| Retrieval | US-3.1, US-5.1 |
| Honest gaps | US-8.2, US-9.1 |
| Answer quality | All stories |
| Speed | US-3.1, US-5.1 |

The two raters in the evaluation can use the acceptance criteria and the personas as a shared standard.

---

## 7. Cross-Cutting Constraints

The non-functional requirements are not stories (story plan Q7). They apply to every story and are defined in `requirements.md` Section 8. A story names an NFR only where the user can observe it.

| NFR | Summary | Where the user can observe it |
| --- | --- | --- |
| NFR-1 Performance | Speed targets; a 60-second timeout and 2 retries for each model call; no overall time limit | US-3.1 AC8, US-5.1 AC6 |
| NFR-2 API usage | Usage of the Cornell API is kept low by design | Not observed by the personas |
| NFR-3 Security | Local addresses only, secrets kept out of code and logs, input validation, untrusted model input, report text treated as data, no dynamic code, safe database queries, a committed lock file | US-2.1 AC5 and US-3.1 AC1 (input limits, NFR-3.3); US-2.1 AC7 (the selection cannot be overridden, NFR-3.4) |
| NFR-4 Privacy | Only the question, public passages, and public figures leave the laptop | Not observed by the personas |
| NFR-5 Reliability | One error format; clear messages when the AI service or the answering part of ReportTrace fails | US-3.1 AC6 and AC7 |
| NFR-6 Maintainability and testability | Fixed tools, layers, coverage, and tests | Not observed by the personas |
| NFR-7 Portability and LLM access | Windows and macOS, one start command, the model and embedding settings and their fallbacks | Not observed by the personas |
| NFR-8 Usability | English; the selection is always visible; one click to the source | US-3.1 AC9; US-2.1 AC8 and US-2.2 AC3; US-7.1 and US-7.2 |

---

## 8. Requirements Coverage

Every functional requirement from FR-1.1 to FR-11.7 is covered by at least one story or by a supporting requirement.

| Requirements | Covered by |
| --- | --- |
| FR-1.1 to FR-1.10 | Supporting requirement F1 |
| FR-2.1, FR-2.2, FR-2.4 | US-2.1 |
| FR-2.3 | US-2.1, US-5.1 |
| FR-2.5 | US-2.2 |
| FR-3.1, FR-3.2, FR-3.6 | US-3.1 |
| FR-3.3 | US-3.1, US-8.1 |
| FR-3.4 | US-3.2 |
| FR-3.5 | US-3.1 |
| FR-3.7 | US-3.1, US-8.2 |
| FR-4.1, FR-4.2 | US-4.1 |
| FR-4.3 | US-4.2 |
| FR-5.1, FR-5.2 | US-5.1 |
| FR-5.3 | US-5.1, US-8.2 |
| FR-6.1, FR-6.2, FR-6.3 | US-6.1 |
| FR-6.4 | US-6.2 |
| FR-6.5, FR-6.6 | US-6.1, US-6.2 |
| FR-7.1, FR-7.2, FR-7.3, FR-7.5 | US-7.1 |
| FR-7.4 | US-7.2 |
| FR-8.1, FR-8.2, FR-8.3, FR-8.4, FR-8.6 | US-8.1 |
| FR-8.5 | US-8.1, US-3.2, US-2.2 |
| FR-8.7 | US-8.2 |
| FR-8.8 | Supporting requirement F10 |
| FR-9.1, FR-9.2 | US-9.1 |
| FR-9.3 | US-9.1, US-8.2 |
| FR-10.1, FR-10.2 | Supporting requirement F10 |
| FR-11.1 to FR-11.7 | Supporting requirement F11 |

The derived rules in `requirements.md` Section 11 are covered as follows.

| Derived rule | Covered by |
| --- | --- |
| D1, D2 | US-6.1 |
| D3 | US-7.2 |
| D4 | US-8.2 |
| D5 | US-8.1 |
| D6 | US-3.1 |
| D7 | Supporting requirement F1 |
| D8 | Supporting requirement F11 |
| D9 | US-9.1, supporting requirement F11 |
| D10 | Cross-cutting constraint NFR-7 |
| D11 | US-9.1 |

---

## 9. INVEST Check

Each story was checked against the six INVEST criteria:

- **Independent:** a story can be built and tested once the stories it depends on exist. All dependencies point to stories listed earlier, and none is circular.
- **Negotiable:** a story states what the user sees, not how it is built. Application Design decides the how.
- **Valuable:** each story gives the persona a result it can see.
- **Estimable:** the scope of each story is bounded by the requirement IDs it lists.
- **Small:** one or two stories per feature (story plan Q2).
- **Testable:** every acceptance criterion is a statement that is either true or false for a given answer.

| Story | Depends on | Result | Note |
| --- | --- | --- | --- |
| US-2.1 | F1 | Pass | |
| US-2.2 | US-2.1, US-3.1 | Pass | |
| US-3.1 | US-2.1, F1 | Pass | The largest story. Kept whole because the four-part answer, the step limit, and the error cases belong to one user action |
| US-3.2 | US-3.1 | Pass | |
| US-4.1 | US-3.1 | Pass | |
| US-4.2 | US-4.1 | Pass | Split from US-4.1 because it has a different priority (Should) and is the first feature to cut |
| US-5.1 | US-2.1, US-3.1 | Pass | |
| US-6.1 | US-3.1, F1 | Pass | Split from US-6.2 so that reported figures can be built and tested before calculations |
| US-6.2 | US-6.1 | Pass | |
| US-7.1 | US-3.1 | Pass | |
| US-7.2 | US-6.1 | Pass | Separate from US-7.1 because a figure card shows different content from a passage card (D3) |
| US-8.1 | US-3.1 | Pass | |
| US-8.2 | US-8.1 | Pass | Split from US-8.1 so that the check and the gap list can be tested separately |
| US-9.1 | US-3.1 | Pass | |
