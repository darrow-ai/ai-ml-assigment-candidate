# Defendant Extraction from Legal Complaints

**Time:** 3 hours. Start whenever suits you, work in one sitting if you can, and submit what you have when the 3 hours are up. If you go over, that is not disqualifying, but say so in the report, with roughly how long you spent and what you would have cut. We are far more interested in how you reason and iterate than in how much you ship. Two well-measured iterations with an honest error analysis beat five features with no numbers.

### Context

Darrow scans public court filings to surface legal exposure. One of the first structured signals we need from every complaint is: **who is being sued?** The answer sounds trivial and is not. Defendant names in complaints arrive as OCR'd text with inconsistent formatting, legal designators (Inc., LLC, L.P.), trade-name qualifiers (d/b/a, f/k/a), placeholder parties ("Does 1 through 50"), individuals sued alongside their companies, and parent or affiliate companies that are described but not actually sued.

Your task is to build a lean, working version of this extraction, evaluate it, and iterate on it.

### Data you receive

Two files in `data/`, both drawn from public federal-court complaints.

**`dev.jsonl`, 15 documents, labeled.** A human annotator wrote the defendant names for these. Use them however you like: to design your evaluation, to debug, to tune.

**`eval.jsonl`, 40 documents, unlabeled.** Run your final pipeline over these. Your `predictions.jsonl` for this file is what we score against our own held-out labels.

Each line is one document:

```json
{
  "doc_id": "c0412",
  "primary_text": "…",        // text from the complaint
  "supplemental_text": "…",   // more text from the same complaint; may be empty
  "labels": ["…", "…"]        // dev.jsonl only: the defendants, as our reviewers recorded them
}
```

Things to know about this data, because they are true in production too:

- The text is OCR output from an automated pipeline. Look at it before you trust it.
- The labels are how our reviewers recorded the names, not how the complaint writes them: lower case, legal form spelled out ("acme, limited liability company" for "ACME, LLC"). They are not perfectly clean. You decide how a predicted name and a labeled name should be compared.
- Fifteen labels is not many. Whether and how you get more signal before you trust a number is your call, and we will ask about it.

### What to build

**1. An extraction pipeline.** A command-line program that reads a JSONL file, runs an LLM-based extraction over every document using the OpenAI API, and writes `predictions.jsonl`. Python preferred. For each document, output at minimum:

```json
{
  "doc_id": "c0412",
  "defendants": [
    {
      "name_raw": "Wal-Mart Stores, Inc.",       // as it appears in the text
      "name_normalized": "wal-mart stores",       // your canonical form for matching
      "designator": "incorporated",               // normalized legal form, or null
      "is_organization": true
    }
  ]
}
```

A two-line example is in `examples/predictions.example.jsonl`. You may add fields. Two we care about in production, and treat as stretch goals: `us_state_of_registration` (from statements like "a Delaware corporation") and a `name_quality` flag for OCR-corrupted or placeholder names.

The contract we will run:

```bash
python run.py --input data/eval.jsonl --output predictions.jsonl
python evaluate.py --predictions predictions.jsonl --gold data/dev.jsonl
```

**2. An evaluation.** A script that scores predictions against labels and prints per-document and aggregate precision, recall and F1. You define the matching semantics and the aggregation. Write down why.

**3. An experiment report** (`REPORT.md`, one to two pages). This is the most important deliverable. For each iteration:

- what you changed and why,
- the metric before and after,
- what the error analysis showed: categorize the failures you saw, and separate model errors from label problems and from problems in the data itself,
- what you would do next if you had another day.

Also cover: how you decided what to measure and how, what you chose not to build and why, roughly what a run over the 55 documents cost in tokens or dollars, how long you spent, and how you used AI coding assistants, if you did.

### What we look for

- **Where you start.** Do you read the data and the labels before writing a prompt? Do you define what "correct" means before measuring it?
- **How you iterate.** Is each change motivated by a failure you observed and can point to? Do you re-measure after every change?
- **Domain judgment.** Complaints have structure and conventions. Noticing them, and encoding them in the prompt or in post-processing, matters more than prompt-wording tricks.
- **Engineering.** Concurrency, retries, schema-validated outputs, caching or resumability, and a clean separation between the LLM call, the deterministic post-processing, and the scoring. Tests for the deterministic parts.
- **Honesty.** Tell us what does not work. We will find it anyway, and we hire people who find it first.

### Practicalities

- **LLM access.** We provide an OpenAI API key with a $20 spending cap. Any OpenAI model is fine. A run over 55 short documents costs well under a dollar on a small model, so the cap is not a constraint you should feel. If you hit it, tell us; do not switch to a personal key.
- **No infrastructure.** No Docker, no orchestrator, no UI. A repo with a README that gets us from clone to `evaluate.py` output in under five minutes is the target.
- **AI coding assistants** are allowed and expected. Ownership of the result is yours: be ready to explain and defend every line.

### Submission

A private Git repository (GitHub or GitLab) shared with the account we give you, containing:

- the code, with a `README.md` covering setup and the two commands above,
- `predictions.jsonl` from your final run over `eval.jsonl`,
- `REPORT.md`.

We will schedule a 45-minute session to walk through your iterations together.

---
