# Change Log: Separate Model for the Analyser (Judge)


## Why

After Step 2, Gemini 3.8 Flash both wrote the benchmark cases and scored them, at 88–98%. AI models tend to score their own writing higher. To check these scores, the Analyser has to run on a different model from the Generator. Before this change, both used `GENAI_MODEL_NAME`.

## What changed

A new environment variable, `GENAI_MODEL_NAME_ANALYSER`, sets the model for the Case Study Analyser only. If it is not set, the Analyser uses `GENAI_MODEL_NAME`, as before.

### Code changes

| # | Before | Now | Where |
|---|---|---|---|
| 1 | No separate analyser model | `get_analyser_model_name()` reads `GENAI_MODEL_NAME_ANALYSER`, falling back to `GENAI_MODEL_NAME` | `gemini_client.py:74` |
| 2 | — | `analyser_model()` returns that model in Gemini mode, and `gpt-4o` in OpenAI mode | `utils.py:169` |
| 3 | Analyser calls used the default model | Every Analyser call passes `model=analyser_model()` | see the table below |
| 4 | The startup log showed one model | It also shows `analyser_model=…` | `gemini_client.py` `describe_provider()` |
| 5 | — | The new variable is documented in the README | `README.md:93` |

### Analyser calls now on the analyser model

| Analyser step | Line |
|---|---|
| Executive summary | `app.py:712` |
| Sector tags and keywords ("Generate Summary" button) | `app.py:742` |
| Scored and informational criteria | `app.py:870` |
| KCM competency mapping | `app.py:1016` |
| Sector tags and keywords ("Perform Full Assessment") | `app.py:1019` |
| Recommendations | `app.py:2587` |
| Writing Assistant | `utils.py:690` |

## What did not change

- The Generator, which still uses `GENAI_MODEL_NAME`
- CaseConnect (`utils.py:1802`), which still uses `GENAI_MODEL_NAME`
- The rubric, prompts, weights and grade bands
- The output-token cap (8,192) and thinking level (`low`), which are shared by both models
- Error handling: a failed or unreadable AI answer still scores 0

## How to use

```
GENAI_MODEL_NAME=gemini-3.8-flash          # Generator
GENAI_MODEL_NAME_ANALYSER=gemini-2.5-pro   # Analyser (judge)
```

## Results (run on 20 Oct 2026)

The reports are in `benchmark_docs/benchmark_analyser_judge_model/`. The three cases and teaching notes are **byte-identical** to those in `benchmark_docs/benchmark_generator_prompt_changes/`, so only the analyser model changed. All cases were written by Gemini 3.8 Flash.

### Scores

| Case type | 3.8 Flash (19 Oct) | 2.5 Pro | 3.1 Pro Preview, as reported | 3.1 Pro Preview, failed criteria left out |
|---|---|---|---|---|
| Caselet | 88.4% | 85.9% | 77.1% | 88.0% |
| Decision-Forcing | 87.9% | **93.2%** | 78.2% | 100.0% |
| Lesson-Drawing | 97.9% | 97.9% | 72.2% | 97.9% |

**Neither Pro model scores the cases lower than Flash.** 2.5 Pro gives the same score as Flash on 41 of 48 criteria. 3.1 Pro gives the same score on 32 of the 39 criteria it managed to score. These runs do not show Flash favouring its own writing. But all three models are Gemini, so this is not the cross-check against a different model family that the Step 2 change log asked for.

### Criteria where the models disagree

| Case | Criterion | Flash | 2.5 Pro | 3.1 Pro |
|---|---|---|---|---|
| Caselet | Sectional Connectivity | 2 | 1 | failed |
| Caselet | Citation Quality | 3 | 1 | 1 |
| Caselet | Language Quality | 2 | 4 | 4 |
| Caselet | Delivery & Implementation | 2 | 2 | 3 |
| Decision-Forcing | Captivating Hook | 0 | 0 | 1 |
| Decision-Forcing | Logical Coherence | 2 | 3 | failed |
| Decision-Forcing | Sectional Connectivity | 1 | 2 | 2 |
| Decision-Forcing | Chronology Maintained | 2 | 3 | 3 |
| Decision-Forcing | Citation Quality | 2 | 2 | 3 |
| Decision-Forcing | Language Quality | 4 | 3 | 4 |

- **Citation Quality (Caselet):** both Pro models gave 1 because the three sources are internal documents with no link or DOI. Flash gave 3 because each in-text citation matches the reference list. 3.1 Pro gave the Decision-Forcing case 3 for the same kind of sources, so it does not apply this rule consistently.
- **Language Quality (Caselet):** Flash took 2 points off for the duplicated paragraphs. The Pro models gave 4 and flagged the duplication in the Writing Assistant instead. Either way, the duplication costs the case points once.

### 3.1 Pro Preview: 19% of criteria failed

9 of the 48 criterion calls returned an answer that could not be read as JSON:

| Case | Criteria that failed |
|---|---|
| Caselet | Sectional Connectivity, Factual Correctness |
| Decision-Forcing | Key Theme, Logical Coherence, Protagonist & Stakeholder Perspectives |
| Lesson-Drawing | Key Theme, Delivery & Implementation, Best Practices & Lessons, Protagonist & Stakeholder Perspectives |

Each failed criterion shows "Failed to parse response from AI model" and scores **0** (`utils.py:603-609`). The 0 counts towards the composite score, which explains the whole drop to 72–78%. The 0 also counts as a low score for the recommendations. For example, the Lesson-Drawing report recommends "Introduce a central protagonist", for a case built around Collector Chaturvedi.

**Likely cause (not confirmed):** the answer is cut off at the 8,192-token output cap, because Pro models use more of the cap on thinking. `gemini_client.generate_text()` retries only when the answer is completely empty, not when it is cut short. The app does not log `finish_reason`, so this cannot be checked from the reports.

2.5 Pro and Flash had no failed criteria.

### Which analyser found the real defects

Every defect claimed in the reports was checked against the case text. Four are real, and all four are Generator issues:

| Defect in the generated case | Flash | 2.5 Pro | 3.1 Pro |
|---|---|---|---|
| Caselet: the paragraphs "Budgetary limits prevented full optical…" and "This operational shift disrupted…" appear twice | Found | Found | Found |
| Decision-Forcing: Option 2 text and "In this situation, what would you do?" are pasted into the Background section | Found; took points off Logical Coherence, Sectional Connectivity and Chronology | Missed; full marks on all three | Missed |
| Decision-Forcing: broken sentence "outside the record room (CBC, 2026). that allowed" in the Epilogue | Found | Found (Writing Assistant only) | Found (Writing Assistant only) |
| Lesson-Drawing: the heading "Introduction and Context Setting" appears twice | Found (Writing Assistant) | Missed | Missed |

**Flash found the most defects.** 2.5 Pro gave its highest score (93.2%) to the Decision-Forcing case, which has the pasted Option 2 text.

| Writing Assistant issues found | Flash | 2.5 Pro | 3.1 Pro |
|---|---|---|---|
| Caselet | 3 | 1 | 2 |
| Decision-Forcing | 10 | 6 | 4 |
| Lesson-Drawing | 10 | 6 | 5 |

### Quoted evidence is reliable for all three

92–99% of the passages each report quotes appear word for word in the case or teaching note. Most of the rest are close paraphrases or competency names, not invented text.
