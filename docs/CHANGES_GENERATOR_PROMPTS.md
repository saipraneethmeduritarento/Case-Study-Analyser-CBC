# Change Log: Generator Prompt Changes

19 Oct 2026 · Sai Praneeth Meduri · Branch `generator-prompts-changes`

## Why

After Step 1, the Gemini analyser showed where generated cases still lose points:

| Criterion | Average composite points lost per report | Cause |
|---|---|---|
| Citation Quality | 3.9 | No in-text citations |
| Factual Correctness | 3.6 | Facts and figures not linked to a source |
| Language Quality | 2.9 | US spellings, passive voice, broken sentences |
| Theories & Frameworks | 2.5 | GPT-4o teaching notes name no theory |
| Logical Coherence | 2.1 | Leftover editor notes ("Correct to state…", "Paraphrase as…") |
| Captivating Hook | 1.2 | Caselets open with a dry factual paragraph |
| Impact Visibility | 1.1 | Results given without before and after figures |

## What changed

All changes are in `case_generator.py`. Prompt changes **add** instructions. No existing instruction was removed, except one field description in the fact check (item 6).

### Prompt changes

| # | Plan item | Prompt | Before | Now |
|---|---|---|---|---|
| 1 | 2.1 | Main writing rules (`MASTER_SYSTEM_PROMPT`). These are added to every Generator call | British English and past tense only | Also: an (Author, Year) citation after every fact, figure, quote and attributed view, taken from the source it came from; active voice; sentences under 30 words; -ise spellings; no "may/can" for past facts; complete sentences; no notes to the author; only `###` sub-headings, inline quotes, no bold or italic |
| 2 | 2.2 | Lesson-Drawing Hook | Opens with a factual scene, no outcome | Also: stays before the project began; quotes only as complete sentences |
| 3 | 2.4 | Lesson-Drawing Chronological Implementation | Phases with outcomes | Also: ends with a "### Outcomes and Impact" sub-section giving before and after figures with citations, results not achieved, and "not measured" where there is no figure |
| 4 | 2.4 | Decision-Forcing Epilogue and Caselet "Response or Outcome" | Describe what happened | Also: give each result with before and after figures and a citation |
| 5 | 2.6 | Style check | Checks spelling, tense, adjectives, neutrality, structure | Also flags leftover editor notes, sentences missing a subject or verb (4.6), and facts without a citation (4.7). "suggestion" must now be the finished replacement text, never advice |
| 6 | 2.7 | Fact and sensitivity check | "suggestion" was "what the author should do (cite source / verify / remove / generalise)". The "Apply selected fixes" button pasted that advice into the case | "suggestion" must be the rewritten text that carries out the action (adds the citation, generalises, removes the claim, or leaves the text unchanged for "verify"); advice goes in "explanation" |
| 7 | 2.8 | References check | Lists sources used in the draft | Also: lists exactly the sources cited in the text, with matching author and year |
| 8 | 2.9 | Teaching note | "If a theoretical framework applies, describe it" | Must name at least one framework, tie it to a Learning Objective, explain it in Teaching Strategy and apply it in an answer. Also: Case Summary of about 250–300 words, a "Pre-reading" line, and a similar case listed in Additional Teaching Resources |
| 9 | 2.10 | Caselet "Context" section | Opens with setting and protagonist | Opens with two or three sentences of a concrete moment or tension from before the initiative, without revealing the outcome |

### Code changes

| # | Plan item | Before | Now | Where |
|---|---|---|---|---|
| 10 | 2.12 | The quality checks read only part of the draft: style check and KCM mapping the first 18,000 characters; fact check, references check and teaching note the first 14,000 | All five read the **full** draft | `run_compliance_passes`, `generate_teaching_note`, KCM mapping |
| 11 | 2.15 | The word count included every word | (Author, Year) citations are left out, so the word target and the 2,000-word Caselet limit cover the narrative only | `_CITATION_RE` and the word count in `run_compliance_passes` |

Item 10 matters for items 5–7: without it, the new checks never reach Lessons Learnt or the later sections of a long case.

## What did not change

- The Analyser, the rubric and the scoring
- Section names and order for every case type (the new "Outcomes and Impact" is a sub-heading inside Chronological Implementation)
- The JSON shape each Generator step returns
- Export formats (DOCX, PDF, TXT)
- Not done from the plan: 2.3 (Introduction) and 2.5 (Lessons), because both criteria already score full marks; and 2.11, 2.13, 2.14, 2.16 (code fixes left for later)

## What to expect

- **Citations in the text**, such as "(Chaturvedi, 2026)", and a reference list that matches them.
- **Fewer leftover editor notes**, because "Apply selected fixes" now pastes rewritten text instead of advice.
- **More findings at first.** The style check now flags every fact without a citation, so drafts written before this change will show many findings.
- **Longer AI calls.** Quality checks send the full draft, and the drafted sections get a little longer.
- **Teaching notes always name a framework.** The AI may invent a title for the similar case or a reading, so spot-check Additional Teaching Resources.
- **"Applied N fixes" may be slightly higher**, because a "verify" fix returns the original text unchanged.

## Known risks

- OpenAI answers are capped at 2,000 tokens (`utils.py:550`), and the app does not check whether an answer was cut short. Longer sections, such as Chronological Implementation with its new sub-section, are more likely to be cut off. Gemini (8,192-token cap) is less affected.
- The word count shown under each section in the app (`app.py:2014`) still counts citations. Only the compliance word count, which decides the word-limit findings, leaves them out.

## Results (re-run on 19 Oct 2026)

The new cases and reports are in `benchmark_docs/benchmark_generator_prompt_changes/`. This run covers only Gemini 3.8 Flash as both generator and analyser.

### Scores

| Case type | Before both fixes | After Step 1 | After Step 2 |
|---|---|---|---|
| Caselet | 65.7% | 91.1% | **88.4%** |
| Decision-Forcing | 47.3% | 86.8% | **87.9%** |
| Lesson-Drawing | 45.0% | 91.4% | **97.9%** |

### What improved

| Criterion | After Step 1 | After Step 2 |
|---|---|---|
| Factual Correctness | 3 | 4 in all three cases |
| Citation Quality | 1–2 | 2–3 |
| Language Quality (Lesson-Drawing, Decision-Forcing) | 2 | 4 |
| Captivating Hook (Caselet) | 0 | 1 |

- Every case now has in-text citations: 37 (Caselet), 133 (Decision-Forcing) and 166 (Lesson-Drawing). Before, there were none.
- No leftover editor notes, `####` headings, `>` block quotes, bold text or US spellings were found in any of the three cases.
- The Lesson-Drawing case has the new "Outcomes and Impact" sub-section.

### What got worse

| Case | Criterion | After Step 1 → After Step 2 | Cause |
|---|---|---|---|
| Caselet | Logical Coherence | 3 → 1 | Two paragraphs appear twice, word for word ("Budgetary limits prevented full optical…" and "This operational shift disrupted…") |
| Caselet | Language Quality | 4 → 2 | Same duplication |
| Caselet | Delivery & Implementation | 3 → 2 | Some challenges are not matched with a solution |
| Decision-Forcing | Sectional Connectivity | 2 → 1 | "In this situation, what would you do?" appears in the middle of the Background section, as well as at the end of the Dilemma section where it belongs |
| Decision-Forcing | Captivating Hook | 1 → 0 | The Introduction opens with administrative detail |
| Decision-Forcing | Chronology Maintained | 3 → 2 | The chosen solution is described in the Narrative, before the decision point |

**Likely cause of the duplicated text (not yet confirmed):** the fact and style checks now return full rewritten sentences. If a rewrite is longer than the text it replaces, "Apply selected fixes" may paste in a second copy. To confirm, compare a draft before and after clicking "Apply selected fixes".

### Too many citations

- There is one citation every 26–32 words, which is more than a reader needs.
- Many cite weak sources, such as "(Dantewada PPT, n.d.)", or the case's own author, "(CBC, 2026)". In the Lesson-Drawing case, "(CBC, 2026)" appears 133 times.
- In the Decision-Forcing case, "Dantewada PPT" does not match the name used in its reference list.
- **Proposed fix:** soften the citation rule. Cite each source once per paragraph, not after every sentence; never cite the case itself or its author; use the author name exactly as it appears in the reference list.

## Are the high scores real?

The same model, Gemini 3.8 Flash, wrote these cases and scored them. Research shows that AI models tend to score their own writing higher ([Panickssery et al., 2024](https://arxiv.org/abs/2404.13076)). Scores of 88–98% need checking with a different analyser before they are reported.

### Which model to use as the analyser

| Model | Recommendation | Why |
|---|---|---|
| Gemini 2.5 Pro | **Do not use** | Vertex AI retires it on 20 October 2026 ([Google Cloud model lifecycle](https://cloud.google.com/vertex-ai/generative-ai/docs/learn/model-versions)) |
| GPT-4o | **Use as the cross-check analyser** | A different model family, so it does not share Gemini's writing preferences. Run it with `USE_GEMINI` unset, on the Step 1 code |

### How to read the results

- If GPT-4o and Gemini 3.8 Flash give similar scores, the high scores are probably real.
- If only Gemini 3.8 Flash scores 90% or more, it is favouring its own writing. Use the GPT-4o scores for decisions.
- Have one person score one case by hand against the rubric, as a reference point.

## Next

1. Score the three Step 2 cases with the GPT-4o analyser (on the Step 1 code).
2. Confirm whether "Apply selected fixes" causes the duplicated paragraphs.
3. Soften the citation rule (see "Too many citations").
4. Generate the three case types with GPT-4o as the generator, which this run did not cover.
