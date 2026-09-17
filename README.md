# webzone-ai-workflows

AI automation work built during my internship at **Webzone** (May – September 2026).
Everything here runs on n8n, with local LLMs, a vector store and a relational backend.

Four systems, built in order: foundation automations → a document pipeline → a curriculum-locked
assistant → an exam-practice engine. The last two are features of a live education product.

> **Scope note.** This repository contains my workflows, code and engineering notes.
> It does not contain the product's production prompts, the ministry exam papers, or any
> credentials — see [What is deliberately not here](#what-is-deliberately-not-here).

---

## Stack

| Layer | Choice | Why |
|---|---|---|
| Orchestration | **n8n** (self-hosted, Docker) | Visual DAG, every step inspectable, no vendor runtime |
| Chat model | **Ollama** · `qwen3:8b` / `qwen3:14b` | Runs locally — student data and curriculum never leave the machine |
| Embeddings | `qwen3-embedding` | Same family as the chat model, strong on Arabic |
| Vector store | **Chroma** | Simple to run beside n8n, per-subject collections |
| Database | **NocoDB** | Relational, REST out of the box, readable by non-engineers |
| PDF extraction | **LlamaParse** | Best structural fidelity on Arabic RTL documents I tested |
| Forms / notifications | Tally, Gmail, Telegram | |
| Code | JavaScript (n8n Code nodes), Python | |

---

## The systems

### 1 · Exam-practice engine

A student sits a real Baccalauréat past paper in chat, one question at a time, answering in
Arabic. Answers are graded against the official ministry marking scheme, with partial credit.

Two workflows.

**Ingestion — 6 nodes.** Turns a paper into the question bank.

```
Manual Trigger → Google Drive (Download) → Extract From File
    → Parse Canonical (Code) → Switch (recordType) → NocoDB ×3
```

`Parse Canonical` is the whole engine: it reads a canonical markdown version of the paper,
classifies every `###` block, builds question ids, dispatches to a key parser per question type
(matching, MCQ single/multi, odd-one-out, true/false, correction, open reasoning), and emits
`question` / `answer` / `material` rows.

It ends with an audit that was impossible in the previous design: **every domain's questions must
sum to the mark its heading declares, and the paper must sum to its stated total** — otherwise the
import throws. A spec violation is never turned into a flagged row.

**Session — 19 nodes.** Runs the exam.

```
Chat Trigger → List Exams → Get Session → Router → Route
  ├ passthrough    → AI Agent                        (the tutor, system 2)
  ├ ask_selection  → Ask Selection ─────────────┐
  ├ start          → Get Questions → Start Session ─┤
  ├ serve_next ┐                                    │
  ├ finish     ┴─→ Serve Next ──────────────────────┤
  └ grade → Get Question → Get Answer → Grade        │
              → Needs LLM? → [Feedback LLM] → Finalize ┤
                                                       ↓
                                       Save Session → Get Next Question → Compose Reply
```

Design points worth reading the code for:

- **All state lives in `exam_sessions`.** Every chat message is a separate n8n execution — nothing
  survives in memory between turns, so the session row is re-read from the database each time.
- **`Get Answer` is the only node that touches the `answers` table, and it runs after submission.**
  At question-serving time the key was never fetched, so no prompt can extract it early.
- **Grading is deterministic.** Set scoring is `max(0, hits − extras) / |key|`, so answering every
  option scores nothing. The model is only called to *phrase* feedback on open-ended answers —
  roughly one turn in four — never to decide a mark.

### 2 · Curriculum-locked assistant

A RAG assistant over the official curriculum for three subjects (history, geography, civics).
In scope it answers and guides; out of scope it declines clearly rather than improvising.

- Per-subject Chroma collections, chunked so that enumerated lists are never split across a
  boundary (see [Engineering notes](#engineering-notes) — this caused a real fabrication).
- Versioned system prompts, one per subject after a shared prompt proved too loose.
- Tested with 45 questions across the three subjects — in-scope, out-of-scope, and adversarial
  prompt-injection attempts. Results, including the failures, are in `docs/test-logs/`.

### 3 · Document processing pipeline

Upload a PDF → LlamaParse extracts structured content → fields are mapped → rows land in NocoDB →
a summary goes out by email. Tested against three unrelated document types (invoice, CV, form)
on the same pipeline.

Includes a one-page decision framework for **cloud vs. local LLM** — cost, privacy, latency,
quality — which is the reasoning behind running Ollama locally everywhere in this repo.

### 4 · Foundation automations

| Workflow | Chain |
|---|---|
| Form → notification | Tally webhook → format → Gmail |
| Scheduled report | Cron (Mon 09:00) → NocoDB → format → email |
| Lead capture | Tally webhook → NocoDB row → Telegram alert |
| AI email assistant | Gmail trigger → extract → LLM → draft reply → NocoDB |

---

## Engineering notes

The parts of this internship worth more than the node count.

**The extraction model invented content that passed every check.**
One question shipped with three fluent, plausible Arabic answers that were nowhere in the exam
paper, while four real answers were dropped — stored with `needs_review: false`. LlamaParse had
reproduced the page faithfully; the loss was the model's, from complete input. It was found only
by reading the original PDF line by line against the output.

The response was not a better prompt. Each paper is now rewritten once into a canonical format and
read by a deterministic parser: same input, same output, every time. The extraction model was
removed from the pipeline entirely — 8 nodes down to 6 — and the marks audit above now makes a
silent loss arithmetically impossible.

**A chunk boundary caused a fabrication.**
Asked for six causes of globalisation, the assistant returned five plus one invented detail. The
chunk boundary had landed inside the six-item list, so the retrieved chunk only ever held five.
Nothing was wrong with the model or the retrieval — only with where the document had been cut.
Re-chunking fixed it, verified by re-running the same question.

**An infrastructure setting looked like a model failure.**
At Ollama's default context length of 4096, one item silently disappeared from a four-item answer
even though the source chunk contained all four. Raising it to 16384 recovered the item — same
question, same chunk, one setting changed.

**Arabic needs real normalisation before anything is compared.**
Tashkeel, alef unification (أ إ آ ٱ → ا), ة→ه, ى→ي, ؤ→و, ئ→ي, tatweel, and Arabic-Indic digits.
Without it a correct answer spelled differently marks wrong. Two related traps cost time: literal
Arabic character *ranges* in a regex corrupt the file when it round-trips through a paste (use
`\uXXXX` escapes), and any pattern matched against normalised text must itself be written in the
normalised form.

**Regression safety came last and should have come first.**
Every early fix was verified by an ad-hoc script that was then thrown away, so nothing stopped a
fix from breaking a paper that already worked — and twice it did. `tests/test_rubrics.js` now holds
**118 cases** built from real rubric strings, run before any parser change.

---

## Repository layout

```
workflows/          n8n workflow exports (.json), credentials stripped
  01-foundations/
  02-document-pipeline/
  03-curriculum-chat/
  04-exam-ingestion/
  05-exam-session/
code/               the Code-node sources, one file per node
  exam-ingestion/   parse-canonical.js
  exam-session/     router.js · start-session.js · grade.js · finalize.js · compose-reply.js
  chunking/         per-subject chunkers
tests/              test_rubrics.js — 118 cases
docs/
  test-logs/        Module 8 test logs, three subjects, 45 questions
  issues/           Module 9 issues log
  decisions/        cloud vs local LLM framework · canonical format spec
  diagrams/         workflow maps
```

Each `workflows/` folder has its own README with the node list, required credentials and the
tables it expects.

---

## Running any of this

1. `docker compose up` — n8n, Chroma, NocoDB.
2. `ollama pull qwen3:8b && ollama pull qwen3-embedding`.
   **Set the Ollama Chat Model node's Context Length to 16384** — the 4096 default silently
   truncates long retrieved chunks.
3. Import a workflow JSON, then attach your own credentials (none are included).
4. Create the tables listed in that workflow's README.

n8n Code nodes reference each other by exact node name via `$('Node Name')`. **Renaming a node
breaks the workflow silently** — no error on save, only at runtime.

---

## What is deliberately not here

- **Credentials, API keys, webhook URLs.** Workflow exports are scrubbed before commit.
- **The ministry exam papers and their canonical transcriptions.** Not mine to publish.
- **The live product's production system prompts.** Webzone's, not mine. The prompt *structure* is
  described in `docs/decisions/`, without the content.
- Anything identifying a real student.

---

## Modules

The internship ran as eleven modules across three phases.

| # | Module | Built | Where |
|---|---|---|---|
| 1 | Setup & first automation | n8n, Ollama, NocoDB, Docker running; Tally → Gmail | `workflows/01-foundations/` |
| 2 | Automation logic & scheduling | Scheduled report; lead-capture pipeline; triggers, expressions, error handling | `workflows/01-foundations/` |
| 3 | LLMs in the loop | AI email assistant; first RAG diagram; prompt library started | `workflows/01-foundations/` |
| 4 | Document intelligence | Document processor tested on 3 document types; cloud-vs-local decision framework | `workflows/02-document-pipeline/` |
| 5 | Conversational AI & RAG | Knowledge-base chatbot answering only from one document, with graceful refusal | `workflows/03-curriculum-chat/` |
| 6 | Business audit & consultation | AI opportunity audit of a real local business; opportunity map; 20-minute presentation | `docs/decisions/` |
| 7 | Product onboarding | Curriculum ingestion pipeline → Chroma; retrieval verified | `code/chunking/` |
| 8 | Curriculum-locked chat | Live for 3 subjects; 45 documented tests including injection attempts | `workflows/03-curriculum-chat/` |
| 9 | Exam prep mode | Ingestion + session workflows; 7 papers, 123 questions, deterministic grading | `workflows/04-` · `05-` |
| 10 | Beta onboarding | *(not run in my cycle)* | — |
| 11 | Iteration & pitch | Top issues fixed and documented; services pitch deck | `docs/issues/` |

---

## Author

**Mohamad Chehade** — Computer Engineer, Rafik Hariri University.
AI Automation Intern at Webzone, 2026.

Code in this repository is MIT licensed. Curriculum and examination content is not mine to
license and is not included.