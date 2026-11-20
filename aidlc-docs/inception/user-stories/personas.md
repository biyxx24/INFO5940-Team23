# Personas: ReportTrace

These personas come from the target users in `vision.md` and Section 4 of `requirements.md`. Each has a short profile: role, goals, frustrations, experience with 10-K reports, and how the persona uses ReportTrace (story plan Q11).

| ID | Persona | Type | Uses ReportTrace | Stories |
| --- | --- | --- | --- | --- |
| P1 | Research assistant or junior analyst | Primary | Yes | Owns all 14 stories in `stories.md` |
| P2 | Reviewer (senior analyst) | Secondary | No, works from the research note | None of its own; benefits from 6 stories (story plan Q10, CQ2) |
| P3 | Finance student | Secondary | Yes | None of its own; uses 10 of P1's stories (story plan Q9) |

The project team is not a persona (story plan Q8). The team's features, the report library (F1), the run log (F10), and the evaluation (F11), are supporting requirements in Section 6 of `stories.md`.

---

## P1: Research Assistant or Junior Analyst (primary)

**Role.** Works on an investment research team and answers specific questions from a senior analyst, such as "Why did gross margin fall this year?" or "How has the description of supply chain risk changed?". Writes the answers into research notes that a senior analyst reviews.

**Goals.**
- An answer to one specific question in about a minute, instead of hours of reading.
- Figures and quotes that can go straight into a research note.
- A comparison across years without opening each year's report.
- A source for every statement that the reviewer can check.

**Frustrations.**
- The report uses different words from the question, for example "concentration of credit risk" for "customer dependence", so keyword search misses the passage.
- Consecutive reports repeat most of their text, so last year's explanation is easily quoted as this year's.
- Copying and calculating figures by hand leads to wrong numbers.
- A cause that the report never states ends up in the note, and the reviewer cannot check it without redoing the reading.

**Experience with 10-K reports.** New to them. Knows the main financial statements, but not where a topic is discussed in a report of over 100 pages, or how its items and notes are organised.

**How this persona uses ReportTrace.** Picks a company and one or more fiscal years, asks questions in plain English, and asks follow-up questions. Reads the chatbot's steps, opens evidence cards to confirm quotes and figures, and copies the answer with its citations (company, fiscal year, section, and sec.gov link) into a research note.

**Stories.** Owns every story: US-2.1, US-2.2, US-3.1, US-3.2, US-4.1, US-4.2, US-5.1, US-6.1, US-6.2, US-7.1, US-7.2, US-8.1, US-8.2, US-9.1.

---

## P2: Reviewer (secondary)

**Role.** A senior analyst who reads the research assistant's notes and decides whether they can be used in the team's research.

**Goals.**
- To check any statement in a note against its source in one click.
- To trust that every figure is the reported value or a correct calculation from it.
- To see clearly what the reports do not say, instead of a guess.

**Frustrations.**
- Notes with conclusions that the report never states.
- Checking one statement means redoing the reading.
- Figures whose source or calculation is unknown.

**Experience with 10-K reports.** Expert. Knows the structure of a 10-K and reads filings on sec.gov.

**How this persona uses ReportTrace.** Does not use ReportTrace. ReportTrace runs on one laptop for one user, and conversations are not saved (FR-3.4). The reviewer works from the research note: each statement in the note carries the company, fiscal year, section, and sec.gov link that the research assistant copied from the evidence card. The reviewer's one click is the sec.gov link (story plan Q10).

**Stories.** None of its own (story plan CQ2). Its need is written into the acceptance criteria of US-7.1 (AC4 and AC5) and US-7.2 (AC3). It benefits from US-6.1, US-6.2, US-7.1, US-7.2, US-8.1, and US-8.2, because these decide what reaches the note.

---

## P3: Finance Student (secondary)

**Role.** A student who is learning to read annual reports.

**Goals.**
- To find where a topic is discussed in a report.
- To see how the description of a topic changed over the years.
- To learn how a 10-K is organised into items and notes.

**Frustrations.**
- The length of a 10-K, and the fact that it is organised by topic, not by question.
- Not knowing which section to look in.
- Repeated text between years, which hides what actually changed.

**Experience with 10-K reports.** Beginner.

**How this persona uses ReportTrace.** Uses ReportTrace in the same way as the research assistant. Picks a company and years, and asks "where" and "how did it change" questions. Reads the steps to see where the chatbot looked, and opens evidence cards to see the section and the passages around a quote.

**Stories.** None of its own (story plan Q9). Uses US-2.1, US-2.2, US-3.1, US-3.2, US-4.1, US-4.2, US-5.1, US-7.1, US-8.2, and US-9.1.
