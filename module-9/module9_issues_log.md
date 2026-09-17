# Module 9 — Issues Log

**Feature:** Exam Prep Mode — Lebanese Baccalauréat, تربية وطنية
**Scope of this log:** every problem that cost real time between first ingestion and a clean import
**Papers covered:** 2018 (both sessions), 2019 (both), 2021 (both), 2024-1 — 7 papers, 123 questions

---

## The short version

The exam papers went in through LlamaParse and an LLM extraction step. LlamaParse was
faithful; the model was not. Most of this log is the story of finding that out, and of the
twenty-odd smaller defects found on the way.

The module ended with the extraction model removed from the pipeline entirely, replaced by a
canonical version of each paper and a deterministic parser. Pipeline went from 8 nodes to 6.

---

## 1 · Extraction — the model invented content

### 1.1 Fabricated answers, stored as verified — the worst one

**Symptom.** `civic_2019_2_d1_ex4_qxب` shipped with `needs_review: false` and three fluent,
plausible civics answers in Arabic.

**Root cause.** None of the three were in the exam paper. The paper's four real answers had been
dropped. The model had complete, correct input — LlamaParse had reproduced the page faithfully —
and produced confident output that no automated check could distinguish from a correct row.

**How it was found.** Reading the original PDF line by line against the output. Nothing else
would have caught it.

**Fix.** Not a patch. This is what justified removing the model from the extraction path.

**Why it matters for the review:** every other defect in this log announces itself as a flag or a
wrong number. This one looked like success.

### 1.2 Answers lost silently in the key parser

`parseLabelToLetter` had four defects at once: it required the pair to start a line, captured
exactly one letter, understood digit labels only, and did not know tatweel (`هـ` = ه + ـ).

Measured before → after on the real keys:

```
2018 خامسا      {"2":"ب","3":"أ"}   →  {"1":"ج","2":"ب","3":"أ"}
2018 matching   {"1":"د","2":"أ"}   →  {"1":"د","2":"أ","3":"ه","4":"ج"}
2019_1 ex0      {"1":"د"}           →  {"1":"د","2":["ب","د"],"3":["أ","ج"],"4":["ب","ج"]}
2019_2 ex1      null                →  {"I":["أ","ب"],"II":"ب","III":["ب","د"],"VI":["أ","د"]}
```

2018 was affected too, and never flagged — those questions simply graded against an incomplete key.

### 1.3 Merged sub-parts

A question with parts أ/ب/ج/د came back as one row. The splitter handled exactly two parts, so
ج and د were not lost but *glued into part ب* — ب inherited their answer elements and their
marks, and the `/2` marks fallback paid a four-part question double.

Second defect underneath it: the sibling check returned `true` whenever `question_number` was
null, and domain-1 rows carry no question number — so the split never ran in the one place the
merged questions live.

### 1.4 Marking instructions stored as answers

`نكتفي بسلبيتين : نصف علامة لكل سلبية` reads as a perfect `label: value` pair. 2021's `d2_q2ج`
shipped **unflagged** requiring the student to write «نصف علامة لكل سلبية». Its two siblings, whose
rubrics say the same thing, came out correctly — by accident, because their first piece had no colon.

The filter that fixes it had to be made narrow: a looser first attempt matched **الإعلام** and
emptied the answers of a paper about the media.

### 1.5 Rubric headings scoring as free marks

2021's `d3_q4` shipped with the question itself as acceptable answer element #1. `d3_q3` shipped
with a bare section label as element #2. Both scored for free; neither was flagged.

### 1.6 Elements cut in half by PDF line wraps

A wrapped line became two half-elements. Fixing the join exposed an ordering bug: run before the
grading-tail strip, it glued the grading line onto the last bullet and 2021's `d1_ex3` silently
lost its sixth condition.

### 1.7 The document was showing students the questions

Domain-2 source documents carried the numbered question list at the end. Students would have been
handed the questions inside the reading material.

---

## 2 · Marks and validation

- **A dead safety check.** `points_max_rubric_mismatch` compared a value against itself by
  construction — it could never fire, and had been dead since points correction was added.
- **A check firing on correct data.** `points_max_uncertain` flagged rows whose rubric prints its
  own decimals, because the repeated mark line was summed without deduplication — 2019-2 summed
  to 8 against a resolved 2.
- **Missing required counts.** `d3_q3`'s rubric says «علامة على صعيد التشريع وعلامة على الصعيد
  المطلبي» — two marks, one per half. The count wasn't read from the same clause the marks were,
  so a student answering one half could take both marks.
- **No whole-paper arithmetic.** Nothing checked that the questions of a domain summed to the
  mark the paper declares. Per-domain extraction made it impossible. The canonical parser now
  refuses the import if they disagree.

### Rubric misattribution — three measures before one worked

A rubric attached to the wrong question needs a similarity measure. Three were tried:

| Measure | Why it failed |
|---|---|
| Word-overlap ratio | Divides by candidate length → the longest rubric matches everything. Three false flags on 2019-1. |
| Dice similarity | Divides by both token sets → a short prompt against a long rubric is capped (11 vs 51 tokens ⇒ ceiling 0.35). |
| **Coverage of the prompt** | Scale-free: the prompt is a fixed probe. Headings score 1.00 and 0.83, real answers 0.00. |

---

## 3 · Arabic-specific problems

- **Normalisation set required for grading:** tashkeel, alef unification (أ إ آ ٱ → ا), ة→ه,
  ى→ي, ؤ→و, **ئ→ي**, tatweel, Arabic-Indic digits. Without it, a correct answer spelled
  differently marks wrong.
- **Literal Arabic ranges in regex corrupt the file.** `[٠-٩]`, `[ء-ي]` — this has bitten the
  project twice through paste round-trips. Everything is `\uXXXX` escapes now, and grep for
  literal ranges returns zero.
- **Patterns matched against normalised text must be written normalised.** «استثنائية» in its
  natural spelling matched nothing, because `norm()` maps ئ→ي. The file carries a comment warning
  about exactly this; it still caught us.
- **Tatweel.** `هـ` is two characters. Any answer group keyed هـ was dropped until it was handled.

---

## 4 · Grading logic

- **Matching keys come in two directions.** Some label with digits and answer with letters; 2019-2
  labels with Arabic letters and answers with digits (`أ- 3 (0,50) ب -1 (0,50)`). The value kind
  now gets read off the key itself rather than assumed.
- **و is a conjunction, not an option letter.** Option letters are أ ب ج د ه only.
- **Partial credit.** `max(0, hits − extras) / |key|` — guessing every option cannot score.
- **True/false questions need both halves.** Verdict and correction, half a mark each. Before this,
  a label prefix disarmed the verdict parser and both parts fell through to "acceptable elements",
  the one outcome that path exists to prevent: both halves become acceptable, so a student who
  writes the *error* back scores full marks.

---

## 5 · Platform problems (n8n)

| Problem | What it cost |
|---|---|
| Code nodes run in `@n8n/task-runner`, which does not transfer binary payloads — `binary.data.data` holds a 13-character reference, not base64 | An **Extract From File** node is mandatory between Drive and any Code node that needs file content. Diagnosed only after the parser reported a 9-byte file. |
| NocoDB returns `{ id, id_fields, fields }` | Every read needs `flat()`; upsert is decided by the internal `Id`, not the business key |
| Every chat message is a separate workflow execution | All session state has to live in `exam_sessions` — nothing survives in memory between a student's messages |
| Code nodes reference each other by exact node name | Renaming a node breaks the workflow silently |
| Ollama default context length 4096 | Silently dropped one item from a four-item answer list. Raising to 16384 recovered it — same question, same chunk, one setting changed. |

---

## 6 · What actually resolved it

Rather than keep repairing extraction output, each paper was rewritten once into a **canonical
markdown format** — one mark notation, per-question marks, `###` as question boundary, key
directly beneath its question, one element per line, documents fenced, matching states both
columns and distractors, each domain declaring its own total.

A deterministic parser reads that format. Same input, same output, every time, and a whole-paper
marks audit that stops the import if any domain's questions don't sum to its declared total.

| | Before | After |
|---|---|---|
| Pipeline | 8 nodes | 6 nodes |
| Extraction | LLM | deterministic parser |
| Rows needing review | 3–10 per paper | 0 |
| Regression safety | ad-hoc scripts, thrown away | `test_rubrics.js`, 118 cases |

---

## 7 · Still open

- **Multi-select grading does not exist.** Groups with two correct options, and matching exercises
  where a left item takes two letters. The keys parse; there is no grader and no renderer.
- **`detectMisattribution` false-positives on terse keys** whose rubric is a bare bullet list
  rather than a restatement of the question.
- **Five of the seven papers are not yet imported.** 2018-1 and 2018-2 are in and verified clean.
- **The exam session has not been run end to end since the parser and grader rewrites.**
- **2018-1 domain 3 q2 has no published answer key** and is omitted — the paper totals 18, not 20.

---

## Appendix — three issues to carry into the Module 11 log

Module 11 asks for three issues framed as student-reported. These three are real, and each maps
to a fix already in the code:

1. **"It marked my answer wrong and my answer was right."** — Arabic spelling variants
   (أ/إ/آ, ة/ه, ى/ي) compared literally. Fixed by normalising both sides before comparison.
2. **"I got part of the question right and scored zero."** — All-or-nothing scoring on
   multi-element answers. Fixed with per-element partial credit, capped so that listing
   everything scores nothing.
3. **"The reading passage already had the questions in it."** — Domain-2 source documents carried
   the numbered question list. Fixed by cutting the document at the first numbered line that
   carries a mark annotation.
