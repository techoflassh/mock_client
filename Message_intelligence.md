# Message Intelligence

The `message_intelligence` package turns customer messages into structured,
safe, operational information. It cleans text, classifies category and
urgency, extracts application references and financial facts, detects prompt
injection, summarizes threads, calls an optional language-model layer, and
produces batch reports.

It sits between raw customer communications in
[`data/`](../../data/) and the operational/API/assistant layers:

```text
data/customer_messages.csv
data/message_threads.csv
data/adversarial_messages.csv
                 |
                 v
message_intelligence
                 |
                 +--> processed_data/messages_cleaned.csv
                 +--> output/message_pipeline.csv
                 +--> output/message_entities.csv
                 +--> output/thread_summaries.csv
                 +--> output/injection_report.csv
                 +--> output/escalation_report.csv
                 |
                 +--> assistant triage and safety
                 +--> risk model and database lookups
```

The package performs two related jobs:

1. **Offline preparation and reporting**: process CSV corpora and write audit
   files.
2. **Online message handling**: validate a question, classify it, extract
   entities, and protect the assistant before it runs tools.

## Files and responsibilities

### [`__init__.py`](./__init__.py)

Marks this directory as a Python package. It does not currently expose
additional runtime logic.

### [`text_cleaning.py`](./text_cleaning.py)

This is the shared normalization layer for text models and thread summaries.

Regular expressions identify:

- URLs;
- application references such as `LA000001` and `LP000001`;
- ticket references such as `TKT-12345`, `REF#12345`, `SR-12345`, and
  `CASE-12345`;
- quoted email replies;
- signatures;
- greetings and sign-offs.

The masking functions replace sensitive or variable identifiers with the
configured placeholders:

```text
URL                  -> <url>
application ID       -> <application_ref>
ticket/reference ID  -> <ticket>
```

`strip_quoted_reply`, `strip_signature`, `strip_greeting`, and
`strip_sign_off` remove conversational boilerplate. `clean_message` applies
those operations in a deliberate order, collapses whitespace, strips the
result, and lowercases it. Non-string, null, and missing messages become an
empty string.

`clean_corpus` reads a message CSV with Pandas, applies `clean_message` to the
`message_text` column, adds `cleaned_text`, and writes a new CSV. The command
example reads `data/customer_messages.csv` and writes
`processed_data/messages_cleaned.csv`, also reporting character reduction.

The cleaning function is embedded directly into the classifier's
`TfidfVectorizer`, so online classification receives the same normalization
as offline cleaning.

### [`message_classifier.py`](./message_classifier.py)

This module trains two text classifiers from one message corpus:

```text
message_text -> clean_message -> TF-IDF unigrams/bigrams -> LogisticRegression
```

`build_message_classifier` creates an unfitted scikit-learn pipeline. The
vectorizer uses:

- unigrams and bigrams (`ngram_range=(1, 2)`);
- `min_df=3` to ignore extremely rare terms;
- sublinear term frequency;
- the shared cleaning function.

The classifier is balanced and uses the configured random seed.

`labelled_messages` reads a CSV and removes rows missing either `category` or
`urgency`. `train_for_label` creates a stratified holdout split, fits one
classifier for one label, and returns accuracy, macro-F1, and vocabulary size.
The module trains one model for `category` and one for `urgency`.

`classify_message` accepts a dictionary of trained models and a raw message,
then returns a dictionary such as:

```python
{"category": "payment_issue", "urgency": "medium"}
```

The command example saves both models together to
`artifacts/message_classifier.pkl`. The same artifact is loaded by the
assistant, injection audit, escalation, and batch pipeline.

### [`entity_extraction.py`](./entity_extraction.py)

This module extracts structured facts from free text.

`extract_references` finds `LA`/`LP` application IDs, uppercases them,
deduplicates them, and preserves their first-seen order.

`extract_amounts` recognizes values preceded by `Rs`, `Rs.`, `INR`, or `₹`,
removes commas, and returns floats.

`extract_dates` recognizes day-first and ISO date formats, validates them with
`datetime.strptime`, converts two-digit years to the 2000s, and returns
`YYYY-MM-DD` strings.

`link_references` checks each extracted application ID against
[`data_access.database_loader.ApplicationDatabase`](../data_access/database_loader.py).
It separates references into `found` and `not_found`.

`extract_entities_from_corpus` processes every row of a message CSV, links
references to `database/loanserve.db`, keeps only rows with at least one
entity, and writes:

```text
output/message_entities.csv
```

This is where message text is connected to stored application data.

### [`attention.py`](./attention.py)

This is a small NumPy implementation of scaled dot-product attention used by
thread summarization.

`stable_softmax` subtracts the maximum score before exponentiation to reduce
overflow risk. It also avoids division by zero for fully masked inputs.

`attention_scores` computes:

```text
(queries @ keys.T) / sqrt(key_dimension)
```

`scaled_dot_product_attention` optionally masks disallowed positions, applies
softmax, renormalizes allowed weights, and multiplies the weights by values.
It returns both attended vectors and attention weights.

This is not the LLM itself. It is a deterministic ranking mechanism for
identifying important sentences.

### [`thread_summary.py`](./thread_summary.py)

This module creates short summaries from conversation threads without calling
an external language model.

`split_into_sentences` cleans each message with `clean_message`, splits on
punctuation/newlines, removes empty pieces, and drops sentences consisting
only of placeholders.

`score_sentences` converts sentences to TF-IDF vectors and uses attention to
calculate how much attention each sentence receives from the others. Empty or
stop-word-only input receives uniform scores.

`summarise_thread` keeps at most `max_sentences` highest-scoring sentences,
then restores their original conversation order.

`summarise_corpus` sorts `data/message_threads.csv` by thread and turn,
summarizes each thread, and writes:

```text
output/thread_summaries.csv
```

The output includes thread ID, message count, sentence count, and summary.

### [`injection_guard.py`](./injection_guard.py)

This is the prompt-injection safety boundary. `INJECTION_PATTERNS` detects
categories such as:

- instruction override;
- role reassignment;
- guardrail suppression;
- prompt extraction;
- delimiter smuggling;
- data exfiltration.

`SAFE_LOOKALIKES` documents benign phrases that contain overlapping words.
`find_injection_markers` applies the patterns and explicit exclusions, then
returns marker names. `is_safe_to_send` returns true only when no marker is
found.

`scan_corpus_for_injections` reads a message corpus and a saved classifier.
Safe messages are classified; blocked messages receive blank category and
urgency values. The audit is written to:

```text
output/injection_report.csv
```

The assistant calls `find_injection_markers` before planning or executing any
tool. The batch pipeline calls `is_safe_to_send` and rejects unsafe messages.

### [`llm_client.py`](./llm_client.py)

This module contains configuration and response-validation helpers for a
Gemini-style language-model integration. It does not make the network call in
the current implementation.

`load_settings` reads a local `.env` file only for keys not already in the
environment, then requires `GEMINI_API_KEY` and `GEMINI_MODEL`.

`cache_key` generates a SHA-256 key from the model name and exact prompt.

`read_answer` removes optional Markdown code fences, parses JSON, checks that
the result is an object, and verifies the configured required keys:

```text
summary, sentiment, action
```

`load_cache` reads the JSON cache and returns an empty dictionary when the
cache is absent or unreadable. `save_cache` creates the parent directory and
writes the cache. `PROMPT_TEMPLATE` and `REMINDER` define the expected
structured response format.

### [`escalation.py`](./escalation.py)

This module decides whether conversation threads need human attention.

`escalation_reasons` checks three independent signals:

- predicted urgency is `high`;
- sentiment is in `UNHAPPY_SENTIMENTS`;
- linked loan exposure is at least `LARGE_EXPOSURE_INR`.

`must_be_seen_by_a_person` escalates immediately for very large exposure, or
when at least two reasons exist.

`exposure_for` looks up an application in SQLite and returns its loan amount.

`generate_escalation_report` samples the first configured number of threads,
extracts references, finds exposure, classifies the first message's urgency,
uses keyword rules for a conservative angry/worried/neutral sentiment, applies
the escalation rules, and writes:

```text
output/escalation_report.csv
```

### [`batch_pipeline.py`](./batch_pipeline.py)

This is the end-to-end batch processor for individual messages.

`UnusableMessage` is a domain error for invalid, blank, oversized, corrupted,
or unsafe messages.

`check_message` checks type, whitespace, maximum configured length, replacement
characters, NUL characters, and prompt injection.

`process_message` then:

1. validates the raw message;
2. predicts category and urgency;
3. extracts and database-links application references;
4. extracts amounts and dates;
5. returns a normalized result dictionary.

`run_batch` reads a CSV, optionally selects its last `batch_size` rows, loads
the classifier and SQLite database, processes each row, and records either a
`processed` result or a `failed` reason. It writes:

```text
output/message_pipeline.csv
```

## Complete message flow

```text
Raw message
    |
    +--> check_message
    |       |
    |       +--> length/encoding checks
    |       +--> injection_guard
    |
    +--> text_cleaning
    |       |
    |       +--> classifier -> category, urgency
    |       +--> thread_summary -> important sentences
    |
    +--> entity_extraction
    |       |
    |       +--> application IDs -> SQLite lookup
    |       +--> amounts
    |       +--> dates
    |
    +--> escalation rules
    |
    v
Operational result or refusal
```

## Connections to other folders

- [`core/exceptions.py`](../core/exceptions.py) supplies the common
  `LoanServeError` base used by batch and LLM errors.
- [`data_access/database_loader.py`](../data_access/database_loader.py) links
  message application references to stored loan records.
- [`risk_models/baseline_model.py`](../risk_models/baseline_model.py) provides
  the shared `save_model`/`load_model` serialization helpers used for the text
  classifier artifact.
- [`assistant/nodes.py`](../assistant/nodes.py) uses the injection guard and
  message classifier during assistant triage.
- [`api/routers/assistant.py`](../api/routers/assistant.py) exposes the
  assistant workflow over HTTP.
- [`config/constants.py`](../../config/constants.py) owns placeholders,
  message-size limits, classifier paths, sentiment rules, and escalation
  thresholds.

## Typical commands

Run from the repository root:

```powershell
python -m loanserve.message_intelligence.text_cleaning
python -m loanserve.message_intelligence.message_classifier
python -m loanserve.message_intelligence.batch_pipeline
python -m loanserve.message_intelligence.entity_extraction
python -m loanserve.message_intelligence.thread_summary
python -m loanserve.message_intelligence.injection_guard
python -m loanserve.message_intelligence.escalation
```

The classifier artifact must exist before running modules that classify
messages. The SQLite database must contain the application table before
reference-linking or exposure reports can resolve application IDs.
