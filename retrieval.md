# Retrieval

The `retrieval` package builds and queries a searchable policy knowledge base.
It converts policy PDFs into text, splits text into identifiable chunks,
embeds chunks with a SentenceTransformer, stores vectors in persistent
ChromaDB, filters results by product, and provides citation validation for
grounded answers.

```text
data/policy_documents/*.pdf
data/policy_documents_manifest.json
                |
                v
document_loader -> chunking -> vector_store
                                      |
                                      v
                         database/policy_store
                                      |
                         question + product filter
                                      |
                                      v
                           filtered_search
                                      |
                                      v
                         assistant/tools.py
```

This package is the policy-data side of the assistant. It is separate from
the deterministic rules in [`core/`](../core/) and from statistical default
prediction in [`risk_models/`](../risk_models/).

## Files

### [`__init__.py`](./__init__.py)

Marks the directory as a Python package. It has no additional runtime logic.

### [`document_loader.py`](./document_loader.py)

`load_policy_documents` scans a directory for sorted PDF files using
`Path.glob("*.pdf")`. For each PDF it:

1. uses `pypdf.PdfReader`;
2. extracts text from every page;
3. skips pages with no extracted text;
4. joins page text;
5. returns a record containing the PDF stem as `document_id` and full `text`.

The document ID is therefore derived from names such as `POL-ELG-002.pdf`:

```python
{"document_id": "POL-ELG-002", "text": "..."}
```

The loader does not use the manifest; metadata enrichment happens later.

### [`chunking.py`](./chunking.py)

This module creates manageable, traceable passages.

`looks_like_heading` applies lightweight heuristics to identify headings:
non-empty, 3-60 characters, not sentence punctuation, not document-reference
or effective-date lines, starts with an uppercase character, and has at most
eight words.

`split_into_sections` walks non-empty lines, starts a new section at each
heading, and joins body lines under that heading. It returns
`(heading, body)` pairs.

`split_without_breaking_words` splits a body at the configured character
limit, only between words. This keeps chunks bounded without damaging a word.

`chunk_documents` processes every document and creates records like:

```python
{
    "chunk_id": "POL-ELG-002_003",
    "document_id": "POL-ELG-002",
    "heading": "Eligibility",
    "text": "..."
}
```

The chunk index resets for each document. Chunk IDs are later used as
retrieval identifiers and citations.

### [`vector_store.py`](./vector_store.py)

This module owns embedding and ChromaDB persistence.

`load_embedder` loads the `all-MiniLM-L6-v2` SentenceTransformer.

`open_policy_store` opens a persistent Chroma client and gets or creates the
`policy_documents` collection using cosine distance.

`describe_chunks` reads the JSON manifest, maps document IDs to metadata, and
adds product type, category, effective date, document ID, and heading to each
chunk. Defaults are used when a document is absent from the manifest.

`build_policy_store` performs the full indexing process:

1. load PDFs;
2. chunk their text;
3. enrich chunks with manifest metadata;
4. load the embedder;
5. delete the existing collection if it exists;
6. create a fresh cosine collection;
7. encode all texts with normalized embeddings;
8. add IDs, embeddings, documents, and metadata;
9. return the collection count.

The normal persistent store is:

```text
database/policy_store/
```

`search_policy` embeds one question, performs cosine vector search, and
returns result dictionaries containing chunk ID, text, heading, document ID,
and distance.

The rebuild operation intentionally replaces the collection, so it should be
run when policy source documents or the manifest change.

### [`filtered_search.py`](./filtered_search.py)

This module adds product-aware filtering before semantic retrieval.

`product_in_question` recognizes:

```text
home/housing/mortgage/property -> home
vehicle/car/auto/two-wheeler/bike -> vehicle
gold/ornaments/jewellery/purity -> gold
personal -> personal
otherwise -> all
```

`policy_filter` creates the Chroma metadata filter. Universal rules use
`{"product_type": "all"}`. Product-specific questions use an `$in` filter
allowing the product and universal rules.

`filtered_search` embeds the question, queries Chroma with the metadata
filter, and returns the same identifiable passage shape as `search_policy`.
It is the function used by the assistant's policy-search tool.

### [`grounded_answer.py`](./grounded_answer.py)

This module enforces citation discipline for generated policy answers.

`build_prompt` creates a prompt containing only retrieved passages, each
prefixed by its chunk ID, and instructs the answer writer to cite every fact.

`cited_chunk_ids` extracts bracketed IDs, ignores plain numbers and ordinary
lowercase words, and removes duplicates.

`check_citations` requires at least one citation and verifies every cited ID
belongs to the retrieved chunk set. It raises `UngroundedAnswer` for missing
or foreign citations.

The assistant currently places the best retrieved chunk ID in its draft
directly. The validation functions are available for a stricter generated
answer path.

## Indexing flow

```text
PDF files
    |
    v
pypdf text extraction
    |
    v
section detection
    |
    v
word-safe chunking
    |
    v
manifest metadata
    |
    v
SentenceTransformer embeddings
    |
    v
ChromaDB persistent collection
```

Run the index build from the repository root:

```powershell
python -m loanserve.retrieval.vector_store
```

The command reads:

```text
data/policy_documents
data/policy_documents_manifest.json
```

and writes the persistent vector data under:

```text
database/policy_store
```

## Query flow

```text
Customer question
       |
       v
assistant.tools.search_policy_documents
       |
       +--> validate question and result count
       +--> open_policy_store
       +--> load_embedder
       +--> filtered_search
                    |
                    +--> detect product
                    +--> build metadata filter
                    +--> embed question
                    +--> Chroma cosine search
       |
       v
Passages with chunk IDs, metadata, and distances
       |
       v
assistant.agents.draft_response
```

`search_policy_documents` limits `how_many` to 1-20 and rejects questions
shorter than five characters. Retrieval errors become `ToolError`, which the
assistant records in `tool_results`.

## Connections to other folders

- [`assistant/tools.py`](../assistant/tools.py) is the online caller. It opens
  the store and embedder, then invokes `filtered_search`.
- [`assistant/agents.py`](../assistant/agents.py) turns the best passage into a
  draft and includes its chunk ID.
- [`assistant/workflow.py`](../assistant/workflow.py) persists the overall
  assistant run, not the policy vectors.
- [`core/exceptions.py`](../core/exceptions.py) provides `LoanServeError`,
  inherited by `UngroundedAnswer`.
- [`config/constants.py`](../../config/constants.py) supplies policy store
  path and chunk-size settings.
- [`data/policy_documents_manifest.json`](../../data/policy_documents_manifest.json)
  supplies product/category/effective-date metadata.
- [`data/policy_documents/`](../../data/policy_documents/) supplies the source
  PDFs.

## Operational considerations

- The vector store must be built before policy questions can be answered.
- The same embedding model must be used for indexing and querying.
- Rebuilding deletes and recreates the collection.
- Chunk IDs are the traceability link from a response back to source policy
  text.
- Retrieval relevance is not the same as authorization or eligibility. The
  assistant should still apply its workflow and the core domain rules.
