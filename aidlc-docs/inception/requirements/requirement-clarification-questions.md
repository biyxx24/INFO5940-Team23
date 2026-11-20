# Requirements Clarification Questions: ReportTrace

All 26 answers in `requirement-verification-questions.md` are valid letter choices. Five points need clarification before `requirements.md` can be written: two where an answer conflicts with `vision.md`, and three where an answer leaves a rule open.

**How to answer:** write the letter of your choice after each `[Answer]:` tag. If no option fits, choose X and describe what you want after the tag.

---

## Contradiction 1: Section labels (Q3) and Journey 1

You answered Q3 with A: passages are labelled with the standard 10-K items only. Journey 1 in `vision.md` says that the agent "opens a note that the first passage refers to". With item labels only, `read_section` cannot open one note. It can open only the whole of Item 8, which holds the financial statements and every note and can run to tens of pages. Sending that much text to the model also works against Q19 B (keep API usage low).

### Clarification Question 1

How does the agent read one note, and what does `read_section` return for a long item?

A) Keep Q3 A. The agent reaches a note with another `search_report` call. `read_section` returns a long item in parts of a limited size, and the agent asks for the next part when it needs it. Journey 1 is read as "searches again for the note"

B) Keep Q3 A for the labels. In addition, `read_section` can open the passages that come directly before and after a passage the agent has already found, which is how it reads the rest of a note

C) Change Q3 to B: each note to the financial statements in Item 8 is its own section

X) Other (please describe after [Answer]: tag below)

[Answer]: c

---

## Ambiguity 1: Follow-up questions (Q10) and the answer check (Q12, F8)

You answered Q10 with B: the agent receives the earlier turns of the conversation. F8 requires every quote to appear in "a retrieved passage" and every number to come from "a tool result", and Q12 A requires a citation to an item "that a tool returned for this question". It is not clear whether evidence retrieved in an earlier turn still counts in a follow-up. Example: after "How did revenue change?", the user asks "And what about its largest segment?", and the second answer repeats a revenue figure that `find_financial_data` returned in the first turn.

### Clarification Question 2

In a follow-up, may the answer use passages and figures that were retrieved in an earlier turn of the same conversation?

A) Yes. Evidence from earlier turns of the same conversation counts, as long as the selection has not changed. The answer check uses every passage and tool result of the conversation

B) No. Only evidence retrieved for the current question counts. The earlier turns help the agent understand the follow-up, but it must retrieve again whatever it cites

X) Other (please describe after [Answer]: tag below)

[Answer]:a

---

## Ambiguity 2: Second attempt (Q13) and the limit of 10 tool calls

You answered Q13 with B: after a failed check the agent writes the answer again, once. F3 in `vision.md` and the LLM Access section of `tech-env.md` limit the agent to 10 tool calls per question. It is not clear whether the second attempt may call tools, and if it may, whether those calls count towards the same limit.

### Clarification Question 3

What may the agent do in its second attempt?

A) It may call tools, but both attempts together stay within the one limit of 10 tool calls per question

B) It may not call tools. It writes the answer again from the evidence it has already retrieved

C) It gets a new limit of 10 tool calls, so one question can use up to 20 (this changes F3 in `vision.md` and the limit in `tech-env.md`, which the team would update)

X) Other (please describe after [Answer]: tag below)

[Answer]: a
---

## Contradiction 2: Showing removed text (Q14) and the success metrics

You answered Q14 with B: the user sees the removed text itself, marked "could not be verified in the reports". Two targets in `vision.md` point the other way. Quote accuracy requires that "100% of quotes shown to the user match the cited passage word for word", and a quote that failed the check does not match. Honest gaps requires "no invented answer" for an unanswerable question, and removed text may be exactly that.

### Clarification Question 4

How do the removed text and the metrics fit together?

A) Keep Q14 B. Removed text appears only inside the "Not found in the reports" part, marked "could not be verified in the reports", with no quotation marks and no citation. The metrics count only the checked parts of the answer (Answer, Figures, What the company says)

B) Keep Q14 B for removed sentences and figures, but never show a removed quote. For a removed quote the user sees only a neutral line about what is missing

C) Change Q14 to A: removed text is never shown and is kept only in the run log

X) Other (please describe after [Answer]: tag below)

[Answer]:c

---

## Ambiguity 3: Extensions switched off (Q24, Q25) and `tech-env.md`

You answered Q24 with B and Q25 with C, so the AI-DLC Security Baseline and Property-Based Testing rule sets are switched off. `tech-env.md` has its own Security section (for example binding to 127.0.0.1, the API key as `SecretStr`, input validation, read-only SQLite) and requires `hypothesis` property-based tests for the `calculate` tool and the answer check. It is not clear whether these parts of `tech-env.md` still apply.

### Clarification Question 5

Do the Security section and the `hypothesis` tests in `tech-env.md` remain requirements?

A) Yes. Switching the extensions off means only that the AI-DLC extension rule files are not loaded and no extension compliance check is run. Everything in `tech-env.md` remains a requirement

B) No. Some of these parts of `tech-env.md` are dropped (please say which after the tag)

X) Other (please describe after [Answer]: tag below)

[Answer]:a
