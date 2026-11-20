# Story Generation Plan: ReportTrace

## Purpose

This plan sets how the approved requirements (`aidlc-docs/inception/requirements/requirements.md`) are turned into personas and user stories. Part 1 decides the method through the questions in Section 3. Part 2 generates the stories by following the checklist in Section 4, step by step, once the team has approved this plan.

**Inputs:** `requirements.md` (FR-1.1 to FR-11.7, NFR-1 to NFR-8, derived rules D1 to D11), the user journeys and target users in `vision.md`, and `user-stories-assessment.md`.

**Outputs:**

- `aidlc-docs/inception/user-stories/personas.md`
- `aidlc-docs/inception/user-stories/stories.md`

---

## 1. Planning Progress (Part 1)

- [X] Assess whether user stories are needed (`user-stories-assessment.md`: Execute = Yes)
- [X] Create this plan with the breakdown options and the questions
- [x] Collect the team's answers to all questions in Section 3 (Q1:A, Q2:B, Q3:A, Q4:B, Q5:A, Q6:A, Q7:A, Q8:B, Q9:B, Q10:B, Q11:A, Q12:A)
- [x] Analyse the answers for ambiguities and contradictions, with follow-up questions if needed (1 contradiction and 1 ambiguity found and resolved in `story-clarification-questions.md`: CQ1:A, CQ2:A)
- [x] Get the team's explicit approval of this plan (approved 2026-11-20T07:32:35Z)

### Approved method (from the answers)

| Topic | Decision | Source |
| --- | --- | --- |
| Organisation | One epic for each feature that has stories: F2 to F9 (eight epics). Stories are written from the persona's point of view. A journey map shows which stories each user journey passes through | Q1:A, CQ1:A |
| Supporting requirements | F1 (report library), F10 (run log), and F11 (evaluation) have no epic and no stories. They are listed in a separate section of supporting requirements that the stories depend on | Q8:B, CQ1:A |
| Story size | One or two stories per feature, so about 8 to 16 stories in total | Q2:B |
| Story sentence | "As a [persona], I want [goal], so that [benefit]", with the priority from `vision.md` and the requirement IDs covered | Q3:A |
| Acceptance criteria | A checklist of short statements that can each be tested, written in the user's words; technical rules are referenced by requirement ID only | Q4:B, Q6:A |
| Failure and edge cases | Acceptance criteria inside the story they belong to | Q5:A |
| Non-functional requirements | Not stories. Listed by ID in a section of cross-cutting constraints; named in acceptance criteria only where the user can observe them | Q7:A |
| Personas | Three personas with short profiles (role, goals, frustrations, experience with 10-K reports, use of ReportTrace): the research assistant, who owns every story; the reviewer and the finance student, who have no stories of their own and are mapped to the research assistant's stories | Q9:B, Q10:B, Q11:A, CQ2:A |
| Reviewer's need | The reviewer works from the research note; the one click is the sec.gov link. This need becomes acceptance criteria of the evidence card story. `requirements.md` is unchanged | Q10:B, CQ2:A |
| Journey map | The three journeys in `vision.md` only: a question about one year, a comparison across years, and a question the reports cannot answer | Q12:A |

---

## 2. Story Breakdown Options

The stories can be organised in five ways. Question 1 chooses one.

| Approach           | How stories are grouped                                                                                           | Benefits for ReportTrace                                                             | Drawbacks for ReportTrace                                                                                                                                                  |
| ------------------ | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| User journey-based | By the steps of a user journey (ask one year, compare years, ask an unanswerable question)                        | Reads like real use; shows the order of the steps                                    | The library, the run log, and the evaluation fit no journey; one feature, such as evidence cards, is spread over several journeys, which weakens traceability to F1 to F11 |
| Feature-based      | One group per feature F1 to F11                                                                                   | Direct traceability to `requirements.md`; groups are likely to match units of work | Can read as a list of system functions rather than user goals                                                                                                              |
| Persona-based      | One group per persona                                                                                             | Shows the value for each type of user                                                | Most stories serve the primary user, so the groups are unbalanced; shared features are repeated                                                                            |
| Domain-based       | By business area, for example report library, asking and answering, checking and evidence, evaluation and logging | Close to the likely components                                                       | User value is less visible; group boundaries need their own decisions                                                                                                      |
| Epic-based         | Large epics, each split into smaller stories                                                                      | Keeps a large set of stories readable at two levels                                  | Needs a rule for what an epic is: a feature, a journey, or a domain                                                                                                        |

A **hybrid** is possible, for example epics that follow the features F1 to F11, stories written from each persona's point of view, and a separate journey map that ties the stories to the user journeys.

---

## 3. Questions

**How to answer:** write the letter of your choice after each `[Answer]:` tag. If no option fits, choose X and describe what you want after the tag. These answers are kept as the record of how the stories were made, so please answer every question before saying that you are done.

### Part A: Breakdown and size

## Question 1

How should the stories be organised (see Section 2)?

A) Hybrid: one epic per feature F1 to F11, stories written from the persona's point of view, and a journey map that shows which stories each user journey passes through

B) User journey-based, with the library, the run log, and the evaluation as separate groups

C) Persona-based

D) Domain-based, with four groups: report library; asking and answering; checking and evidence; evaluation and logging

X) Other (please describe after [Answer]: tag below)

[Answer]:a

## Question 2

How large should one story be?

A) Small: one story for each user-visible behaviour that can be tested on its own (about 25 to 35 stories in total)

B) Medium: one or two stories per feature (about 12 to 20 stories), each with more acceptance criteria

C) Large: one story per feature (11 stories)

X) Other (please describe after [Answer]: tag below)

[Answer]:b

### Part B: Format and acceptance criteria

## Question 3

Which sentence format should each story use? Each story also carries its priority from `vision.md` (Must or Should) and the requirement IDs it covers.

A) The standard format: "As a [persona], I want [goal], so that [benefit]"

B) The job story format: "When [situation], I want [action], so that [outcome]"

X) Other (please describe after [Answer]: tag below)

[Answer]:a

## Question 4

How should acceptance criteria be written?

A) As Given / When / Then scenarios for every criterion

B) As a checklist of short statements that can each be tested

C) Given / When / Then for behaviour that the user sees, and a checklist for rules that code enforces (for example the answer check and the figure rules)

X) Other (please describe after [Answer]: tag below)

[Answer]:b

## Question 5

Where should failure and edge cases go? Examples: the agent reaches the limit of 10 tool calls (FR-3.5), a model call fails after its retries (NFR-5.2), the backend cannot be reached (NFR-5.3), the answer check removes a statement (FR-8.7), and a selected year has no evidence (FR-5.3).

A) As acceptance criteria inside the story that they belong to

B) As separate stories, one for each failure or edge case

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 6

May acceptance criteria name technical elements?

A) No. Stories and acceptance criteria use the user's words only. Technical rules stay in `requirements.md` and are referenced by requirement ID

B) Yes, where it makes a criterion testable (for example a tool name, an error code, or a field of the run log). The story sentence itself stays in the user's words

X) Other (please describe after [Answer]: tag below)

[Answer]:A

## Question 7

How should the non-functional requirements (NFR-1 to NFR-8, for example speed, security, testing, and portability) appear in `stories.md`?

A) Not as stories. `stories.md` lists them by ID in a section of cross-cutting constraints, and a story's acceptance criteria name an NFR only where the user can observe it (for example the speed targets, or the message when the backend cannot be reached)

B) As technical stories from the team's point of view, one for each NFR group

X) Other (please describe after [Answer]: tag below)

[Answer]:A

### Part C: Personas

## Question 8

`requirements.md` Section 4 lists the project team as the user of the report library (F1), the run log (F10), and the evaluation (F11). How should these three features be covered?

A) The project team is a persona, and F1, F10, and F11 get stories from its point of view

B) No team persona and no stories for F1, F10, and F11. `stories.md` lists them as supporting requirements that the user stories depend on

X) Other (please describe after [Answer]: tag below)

[Answer]: b

## Question 9

How should the finance student be treated?

A) As a separate persona with its own stories where its need differs from the primary user (finding where a topic is discussed, and seeing how it changed over the years); otherwise it is mapped to the primary user's stories

B) As a separate persona in `personas.md` with no stories of its own; it is mapped to the primary user's stories

C) Left out of the personas

X) Other (please describe after [Answer]: tag below)

[Answer]:b

## Question 10

The reviewer is a senior analyst who reads the research assistant's notes and checks a statement "in one click" (NFR-8.3). ReportTrace runs on one laptop for one user, and conversations are not saved (FR-3.4). How does the reviewer reach the evidence?

A) The reviewer looks at the answer in ReportTrace, on the same laptop and in the same session as the research assistant. The requirements stay as they are

B) The reviewer works from the research note. The research assistant copies the company, fiscal year, section, and sec.gov link from the evidence card into the note, and the reviewer's one click is the sec.gov link. The requirements stay as they are

C) A new feature: each answer can be copied or exported together with its citations (quote, company, fiscal year, section, sec.gov link). This changes `requirements.md` and is routed back through Requirements Analysis before the stories are generated

X) Other (please describe after [Answer]: tag below)

[Answer]:b

## Question 11

How detailed should each persona be?

A) A short profile: role, goals, frustrations, experience with 10-K reports, and how the persona uses ReportTrace

B) As A, plus a fictional name and a short scenario of a typical working day

X) Other (please describe after [Answer]: tag below)

[Answer]:a

### Part D: User journeys

## Question 12

`vision.md` has three user journeys: a question about one year, a comparison across years, and a question the reports cannot answer. Which journeys should the journey map in `stories.md` cover?

A) Only the three journeys in `vision.md`

B) The three journeys, plus three more: a reviewer checking an answer, a follow-up question, and a change of selection

C) As B, plus the project team building the library and running the evaluation

X) Other (please describe after [Answer]: tag below)

[Answer]:a

---

## 4. Generation Checklist (Part 2)

These steps run only after the team has approved this plan. Each step is marked [x] as soon as it is done. Where a step depends on an answer, the answer is named in brackets.

### Personas

- [x] Step 1: Draft the personas from `requirements.md` Section 4 and the target users in `vision.md` (Q8, Q9, Q10, CQ2)
- [x] Step 2: Write each persona profile at the chosen level of detail (Q11)
- [x] Step 3: Save `aidlc-docs/inception/user-stories/personas.md`

### Stories

- [x] Step 4: Set up the grouping of `stories.md` (Q1, CQ1): eight epics E2 to E9
- [x] Step 5: Write the stories for each group in the chosen sentence format, with priority and requirement IDs (Q2, Q3): 14 stories
- [x] Step 6: Write the acceptance criteria of each story, including the reviewer's need in the evidence card story (Q4, Q6, CQ2): reviewer's need in US-7.1 AC4 and AC5, and US-7.2 AC3
- [x] Step 7: Add the failure and edge cases (Q5)
- [x] Step 8: Add the non-functional requirements (Q7): `stories.md` Section 7
- [x] Step 9: Add the journey map, showing which stories each journey passes through (Q1, Q12): `stories.md` Section 4
- [x] Step 10: Map the personas to their stories, including the reviewer and the finance student (Q9, CQ2): `stories.md` Section 5
- [x] Step 11: Check coverage: every functional requirement FR-1.1 to FR-11.7 appears in at least one story, or is listed as a supporting requirement (Q8, CQ1): `stories.md` Section 8, no gaps
- [x] Step 12: Check each story against the INVEST criteria (Independent, Negotiable, Valuable, Estimable, Small, Testable) and split or merge stories that fail: `stories.md` Section 9, all pass; splits recorded
- [x] Step 13: Validate the content (markdown tables, any diagram) and save `aidlc-docs/inception/user-stories/stories.md`: no diagrams; tables and lists checked

### Mandatory artifacts

- [x] Generate `stories.md` with user stories that follow the INVEST criteria
- [x] Generate `personas.md` with user archetypes and their characteristics
- [x] Ensure the stories are Independent, Negotiable, Valuable, Estimable, Small, and Testable
- [x] Include acceptance criteria for each story
- [x] Map the personas to the relevant user stories
