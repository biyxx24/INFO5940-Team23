# Story Planning Clarification Questions: ReportTrace

All 12 answers in `story-generation-plan.md` are valid letter choices. Two points need clarification before the plan can be approved: one contradiction between two answers, and one ambiguity that no question settled.

**How to answer:** write the letter of your choice after each `[Answer]:` tag. If no option fits, choose X and describe what you want after the tag.

---

## Contradiction 1: One epic per feature (Q1) and no stories for F1, F10, F11 (Q8)

You answered Q1 with A: "one epic per feature F1 to F11". You answered Q8 with B: "no stories for F1, F10, and F11", which are listed as supporting requirements instead. Together, the epics for the report library (F1), the run log (F10), and the evaluation (F11) would have no stories.

With "one or two stories per feature" (Q2 B) applied to the eight features that have stories, the expected total is about 8 to 16 stories.

### Clarification Question 1

How should `stories.md` handle F1, F10, and F11?

A) Epics only for the eight features that have stories (F2 to F9). F1, F10, and F11 appear only in a separate section of supporting requirements

B) One epic for every feature F1 to F11, so that the numbering matches `requirements.md`. The epics for F1, F10, and F11 hold no stories, only a pointer to their supporting requirements

X) Other (please describe after [Answer]: tag below)

[Answer]:a

---

## Ambiguity 1: Does the reviewer have stories of its own? (Q10)

You answered Q10 with B: the reviewer works from the research note and does not use ReportTrace. The research assistant copies the company, fiscal year, section, and sec.gov link from the evidence card into the note. Q9 settled whether the finance student has stories of its own (no, it is mapped to the primary user's stories), but no question settled this for the reviewer.

### Clarification Question 2

How should the reviewer appear in the stories?

A) As a separate persona with no stories of its own. The reviewer's need, that every statement in the note can be checked through what the evidence card shows, becomes acceptance criteria of the research assistant's evidence card story

B) As a separate persona with one story of its own: each citation in the note carries the company, fiscal year, section, and sec.gov link, so that the reviewer can check it from the note. The evidence card delivers this story

X) Other (please describe after [Answer]: tag below)

[Answer]:a
