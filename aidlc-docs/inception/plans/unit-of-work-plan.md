# Unit of Work Plan: ReportTrace

## Purpose

This plan sets how ReportTrace is split into units of work for Construction. A unit of work is a group of components and stories that goes through Functional Design and Code Generation together. Construction then works through the units one at a time, completing each unit's design and code before the next one starts. In ReportTrace every unit is a set of modules inside the one package `reporttrace`: there are no separately deployed services.

**Inputs:** the application design in `aidlc-docs/inception/application-design/` (components C1 to C14, services S1 to S4), `stories.md` (14 stories and the supporting requirements F1, F10, F11), `requirements.md`, `execution-plan.md`, and the constraints in `vision.md` and `tech-env.md`.

**Outputs** (in `aidlc-docs/inception/application-design/`):

- `unit-of-work.md`: unit definitions, responsibilities, and the code organization strategy
- `unit-of-work-dependency.md`: the dependency matrix between units
- `unit-of-work-story-map.md`: stories mapped to units

---

## 1. Planning Progress (Part 1)

- [X] Analyse the application design, the stories, and the constraints (Section 2)
- [X] Write the decomposition options and the questions (Sections 3 and 4)
- [x] Collect the team's answers to all questions in Section 4 (Q1:B, Q2:B, Q3:B, Q4:A, Q5:A, Q6:A, Q7:A, Q8:A)
- [x] Analyse the answers for ambiguities and contradictions, with follow-up questions if needed (2 follow-up questions in Part E, answered FU1:B with owners, FU2:A; no ambiguities remain)
- [x] Get the team's explicit approval of this plan (approved 2026-11-20T17:58:07Z)

### Approved decomposition (from the answers)

| Topic | Decision | Source |
| --- | --- | --- |
| Units | U0 Project foundation: project setup, C1 domain types, C2 settings, `agent/models.py`, shared test fakes and sample report, and the Cornell API check | Q2:B, FU2:A |
| | U1 Report library: C3 storage, C4 ingestion | Q1:B, FU2:A |
| | U2 Evidence: C5 tools, `agent/context.py`, `agent/answer_schema.py`, C8 answer check | Q1:B |
| | U3 Agent and question service: the rest of C6, and C7 conversations, C9 run log, C10 question service | Q1:B |
| | U4 API, web page, launcher, and evaluation: C11 to C14 | Q1:B |
| Story mapping | Each acceptance criterion is assigned to the unit that implements it; each story also has one primary unit where the whole story is checked | Q3:B |
| Construction order | U0, U1, U2, U3, U4, one after another in dependency order | Q4:A |
| Integration tests | Each unit adds integration tests for its boundary with the units already built; Build and Test runs all tests and adds the end-to-end tests | Q5:A |
| Cornell API check | The first task of U0: check by hand that embeddings (with the NFR-7.3 fallbacks) and `create_agent` with `response_format` and one tool work. If either fails, the team decides on a change through the workflow before going on | Q6:A, FU2:A |
| Owners | U0: all four members together; U1: Zheng Kang; U2: Chengcheng Chen; U3: Xihe Xu; U4: Jiayi Luo | Q7:A, FU1:B |
| Evaluation set | The team writes the evaluation questions in parallel with Construction from its first week; U4 builds only the evaluation script and its tests | Q8:A |

---

## 2. Context for the Decomposition

| Fact                                                                                                                       | Source                                   | Effect on the units                                                     |
| -------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- | ----------------------------------------------------------------------- |
| Four team members; each unit of work has one owner who reviews its code and tests                                          | `README.md`, collaboration plan        | About four units, or five with one owner for two                        |
| Construction runs from November 25 to December 16, 2026                                                                    | `vision.md`                            | Units must be small enough to finish one after another in three weeks   |
| Build order: library, tools, agent, answer check, chatbot, comparison view, evaluation; the live step display is cut first | `vision.md`, Risks                     | The unit order should follow it                                         |
| One package `reporttrace` in a `src/` layout; one-way layers `ui`, `api`, `agent`, `tools`, `storage`        | `tech-env.md`                          | The units are groups of modules, not separate projects                  |
| `agent/models.py` is the only place that creates models, and ingestion needs the embedding model                         | `component-dependency.md` Section 1    | `agent/models.py` belongs with the library unit, which needs it first |
| Tools read `agent/context.py` (`AgentContext`, `EvidenceRecord`)                                                     | `component-dependency.md` Section 1    | `agent/context.py` belongs with the tools                             |
| The answer check works on the agent's answer schema (`agent/answer_schema.py`)                                           | `component-methods.md` C8              | The schema belongs with the check, or in an earlier unit                |
| Structured output and embeddings through the Cornell API are not yet confirmed                                             | `application-design.md` Section 8; Q18 | An early check lowers the risk for every later unit                     |

**Categories evaluated without a question:**

- **Deployment differences between units (Technical Considerations):** none. Every unit runs on the same laptop in the same two local processes, or as a separate command.
- **Directory structure (Code Organization):** fixed by `tech-env.md`. Code goes in `src/reporttrace/{package}/`; tests go in `tests/unit/{package}/`, `tests/integration/`, and `tests/fakes/`, as in the `tech-env.md` examples. `unit-of-work.md` records which modules and test folders belong to each unit.

---

## 3. Decomposition Options

Question 1 chooses one option. Component IDs refer to `components.md`.

| Option                        | Units                                                                                                                              | Benefits                                                                                                                            | Drawbacks                                                                                                                                                                       |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A. Five units by layer        | U1 Report library; U2 Tools; U3 Agent and answer check; U4 API, web page, and launcher; U5 Evaluation                              | Follows the layers and the build order exactly; each unit has a clear boundary and is tested against the unit below                 | Five units for four owners, so one member owns two; U3 is the largest unit                                                                                                      |
| B. Four units, one per member | U1 Report library; U2 Evidence (tools and answer check); U3 Agent and question service; U4 API, web page, launcher, and evaluation | One unit per owner. U2 keeps together the two parts that work on the same evidence record and the same rules for quotes and figures | U2 must also hold the agent's answer schema, which the check needs. U4 mixes the interface with the evaluation                                                                  |
| C. Vertical slices            | U1 Report library; U2 Single-year answer end to end; U3 Comparison and evidence cards end to end; U4 Evaluation                    | Something visible works after U2                                                                                                    | Every slice changes code in several layers, and later slices change code of earlier ones. This conflicts with completing each unit's design and code before the next one starts |

---

## 4. Questions

**How to answer:** write the letter of your choice after each `[Answer]:` tag. If no option fits, choose X and describe what you want after the tag. These decisions fix the units that Construction works through and who reviews each one, so please answer every question and **save the file** before saying that you are done.

### Part A: Story grouping and business domain

## Question 1

How should the system be split into units (see Section 3)?

A) Option A, five units by layer:

- U1 Report library: C1 to C4, and `agent/models.py`;
- U2 Tools: C5, and `agent/context.py`;
- U3 Agent and answer check: the rest of C6, and C7 to C10;
- U4 API, web page, and launcher: C11, C12, C14;
- U5 Evaluation: C13.

B) Option B, four units, one per team member:

- U1 Report library: C1 to C4, and `agent/models.py`;
- U2 Evidence: C5, `agent/context.py`, `agent/answer_schema.py`, and C8;
- U3 Agent and question service: the rest of C6, and C7, C9, C10;
- U4 API, web page, launcher, and evaluation: C11 to C14.

C) Option C, vertical slices

X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Question 2

Where do the shared foundations go? They are needed by every unit: the project setup (`pyproject.toml`, `uv.lock`, ruff and mypy settings, `.env.example`, `.gitignore`), the domain types (C1), the settings (C2), and the shared test fakes and sample report in `tests/fakes/`.

A) In U1, built first inside the library unit

B) In a separate small unit, U0 "Project foundation", designed, built, and approved before U1

X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Question 3

How should stories be mapped to units? Most stories need several units; for example, US-3.1 needs the tools, the agent, the API, and the page.

A) Each story has one primary unit, where all its acceptance criteria are tested together, and a list of contributing units

B) Each acceptance criterion of each story is assigned to the unit that implements it, so that each unit knows exactly which criteria its tests must cover. Each story also has one primary unit, where the whole story is checked

X) Other (please describe after [Answer]: tag below)

[Answer]: B

### Part B: Dependencies and integration

## Question 4

In what order are the units built in Construction?

A) In dependency order, one after another, as `vision.md` proposes: library, tools, agent and answer check, API and page, evaluation (in the unit numbering of Question 1)

B) In dependency order, except that the API and page unit is built before the agent unit, against a fake question service, so that the team can see and test the interface sooner

X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 5

How are units tested together?

A) Each unit adds integration tests for its boundary with the units already built. For example, the agent unit tests the agent with the real tools on the sample library and a fake model. Build and Test then runs all tests and adds the end-to-end tests

B) Units have unit tests only. All integration tests are written in Build and Test

X) Other (please describe after [Answer]: tag below)

[Answer]: A

### Part C: Technical considerations

## Question 6

When are the two unconfirmed Cornell API features checked (embeddings, and structured output with tool calling, `application-design.md` Section 8)?

A) As the first task of the first unit, before anything is built on them: a small check, run by hand, that embeddings work (with the fallbacks of NFR-7.3) and that `create_agent` with `response_format` and one tool works. If either fails, the team decides on a change through the workflow before going on

B) In the unit that first needs each feature: embeddings in the library unit, structured output in the agent unit

X) Other (please describe after [Answer]: tag below)

[Answer]: A

### Part D: Team alignment

## Question 7

Should the unit documents name an owner for each unit? The team members are Zheng Kang, Chengcheng Chen, Xihe Xu, and Jiayi Luo.

A) Yes. Please write the owner of each unit after the tag, for example "U1: name, U2: name, ..."

B) No. The team assigns owners outside the AI-DLC documents, for example in its GitHub Issues

X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 8

When does the team write the evaluation questions with their expected figures and sections? This is team work, not code (FR-11.1 to FR-11.3).

A) In parallel with Construction, from its first week, so that the set is ready when the evaluation is built. The evaluation unit then builds only the script and its tests

B) After the product works, as part of the evaluation unit

X) Other (please describe after [Answer]: tag below)

[Answer]: A

### Part E: Follow-up questions

All 8 answers are valid letter choices. Two points need clarification before this plan can be approved.

## Follow-up Question 1 (missing detail in Question 7)

You answered Question 7 with A (name an owner for each unit), but no names were written after the tag. Also, Question 1 B ("four units, one per team member") together with Question 2 B (a separate unit U0) gives five units for four members:

- U0 Project foundation;
- U1 Report library;
- U2 Evidence;
- U3 Agent and question service;
- U4 API, web page, launcher, and evaluation.

Who owns each unit?

A) One owner per unit, with one member owning two units. Please write all five owners after the tag, for example "U0: name, U1: name, U2: name, U3: name, U4: name"

B) U0 is owned by all four members together, because it is small and every unit builds on it. Please write the owners of U1 to U4 after the tag, for example "U1: name, U2: name, U3: name, U4: name"

X) Other (please describe after [Answer]: tag below)

[Answer]: b U1: ZhengKang, U2: Chengcheng Chen, U3: Xihe Xu, U4: Jiayi Luo

## Follow-up Question 2 (ambiguity between Question 2 and Question 6)

You answered Question 2 with B: the foundations (project setup, domain types, settings, test fakes) form a separate unit U0, built first. You answered Question 6 with A: the Cornell API check is "the first task of the first unit". With Question 4 A, the first unit is now U0. The check needs `agent/models.py`, which Question 1 B placed in U1.

Where do `agent/models.py` and the Cornell API check go?

A) In U0. U0 then holds the project setup, C1, C2, `agent/models.py`, the test fakes, and the Cornell API check as its first task. U1 holds C3 storage and C4 ingestion

B) In U1. U0 holds only the foundations listed in Question 2, which make no API call. The check is the first task of U1, the first unit whose code calls the API

X) Other (please describe after [Answer]: tag below)

[Answer]: a

---

## 5. Generation Checklist (Part 2)

These steps run only after the team has approved this plan. Each step is marked [x] as soon as it is done. Where a step depends on an answer, the answer is named in brackets.

- [x] Step 1: Define the units with their components, modules, and test folders (Q1, Q2)
- [x] Step 2: Write the responsibilities, the stories, and the "done when" criteria of each unit (Q3, Q5)
- [x] Step 3: Write the code organization strategy: the directory tree of `src/reporttrace/` and `tests/`, with the owner unit of each module
- [x] Step 4: Record the construction order, the early Cornell API check, the owners, and the timing of the evaluation set (Q4, Q6, Q7, Q8)
- [x] Step 5: Save `unit-of-work.md`
- [x] Step 6: Write the dependency matrix between units, with what each unit uses from each earlier unit and the integration tests at each boundary (Q4, Q5), and save `unit-of-work-dependency.md`
- [x] Step 7: Map every story, and every supporting requirement (F1, F10, F11), to its units (Q3), and save `unit-of-work-story-map.md`
- [x] Step 8: Validate (all pass; acceptance criteria counts checked by script against `stories.md`: 77 criteria, U1 1, U2 22, U3 23, U4 31):
  - every component and module belongs to exactly one unit;
  - every story and supporting requirement is mapped;
  - no unit depends on a later unit in the construction order;
  - Mermaid and ASCII content is valid.

### Mandatory artifacts

- [x] Generate `aidlc-docs/inception/application-design/unit-of-work.md` with unit definitions and responsibilities
- [x] Generate `aidlc-docs/inception/application-design/unit-of-work-dependency.md` with dependency matrix
- [x] Generate `aidlc-docs/inception/application-design/unit-of-work-story-map.md` mapping stories to units
- [x] Greenfield only: Document code organization strategy in `unit-of-work.md`
- [x] Validate unit boundaries and dependencies
- [x] Ensure all stories are assigned to units
