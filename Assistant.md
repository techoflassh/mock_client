# Assistant

The `assistant` package implements the customer-facing question-answering
workflow. It uses a LangGraph state machine to validate a question, classify
it, choose a safe tool, execute that tool, draft a response, pause for human
approval when required, critique the draft, and return a final answer.

```text
API POST /api/v1/assistant/ask
              |
              v
workflow.run_assistant
              |
              v
LangGraph StateGraph
              |
  triage -> plan -> tools -> draft -> approval -> critic -> respond
              |
              +--> core calculations
              +--> database application lookup
              +--> risk-model probability
              +--> retrieval policy search
              |
              v
API response + SQLite checkpoint
```

The assistant is an orchestration layer. It does not own the loan formula,
database schema, risk model, or policy embeddings; it calls the corresponding
packages through controlled tools.

## Files

### [`__init__.py`](./__init__.py)

Marks the directory as a Python package. There is no additional runtime logic.

### [`state.py`](./state.py)

Defines the `AssistantState` `TypedDict`, the shared data contract passed
between graph nodes. Important fields are:

- `question`: original customer question;
- `messages`: LangChain narration messages;
- `steps`: append-only workflow step names;
- `category` and `urgency`: message classification;
- `plan`: selected tool calls;
- `tool_results`: tool outputs;
- `draft`: proposed response;
- `citations`: citation identifiers;
- `confidence`: critic/response confidence;
- `needs_approval`, `approved`: human-review state;
- `refused`: refusal reason;
- `answer`: final customer-facing response.

`Annotated[list[AnyMessage], add_messages]` tells LangGraph how to merge
messages. `Annotated[list[str], operator.add]` makes steps append as nodes
execute.

`new_assistant_state` creates a clean initial state for each question. It
prevents prior request data from leaking into a new run.

### [`tools.py`](./tools.py)

Contains the only operations the planner can invoke.

`ToolError` is the package-level failure type for invalid arguments, missing
records, or downstream failures.

#### `calculate_installment`

Validates a dictionary and requires principal, annual rate, and tenure.
It converts values to numbers, requires positive principal/tenure and a
non-negative rate, then delegates the formula to
[`core.loan_calculations.calculate_monthly_installment`](../core/loan_calculations.py).
It returns a rounded monthly instalment.

#### `lookup_application`

Normalizes an application ID to uppercase, opens
[`data_access.database_loader.ApplicationDatabase`](../data_access/database_loader.py),
and returns the stored record or raises `ToolError`.

#### `predict_default_risk`

Requires a dictionary under `application`, verifies every configured modeling
column exists, loads the champion pipeline from the configured artifact path,
creates a one-row DataFrame, and returns the model's default probability.

#### `search_policy_documents`

Validates a question and result count, opens the Chroma policy collection,
loads the SentenceTransformer embedder, and calls
[`retrieval.filtered_search`](../retrieval/filtered_search.py). It returns
the retrieved passages with identifiers and metadata.

### [`agents.py`](./agents.py)

This module contains deterministic planning, drafting, and response helpers.
The prompt constants describe the intended JSON contracts, but the current
implementation uses Python rules rather than an external LLM call.

`plan_tools` checks the question for installment terms, known loan terms,
application references, and otherwise treats the request as a policy question.
It returns one planned tool:

```text
installment/EMI -> calculate_installment
LA/LP reference -> lookup_application
otherwise       -> search_policy_documents
```

The installment branch currently recognizes a narrow set of example terms and
uses the corresponding parsed/default figures. It is therefore a routing
helper, not a general natural-language numerical parser.

`draft_response` examines the first tool result. It formats an EMI result,
application status, or best policy passage. Questions containing `waive`,
`override`, or `exception` set `needs_approval`.

`write_response` returns the draft and a confidence of `0.95`, or a pinned
fallback answer with zero confidence when the draft is empty.

### [`nodes.py`](./nodes.py)

Defines the functions attached to the graph.

#### `triage`

Runs injection detection first. A marker immediately sets `refused` and
stops the graph. Very short questions are also refused. Otherwise it loads
the message classifier and assigns category and urgency; classifier failures
fall back to `loan_query` and `low`.

#### `plan`

Does nothing for a refused state. Otherwise calls `agents.plan_tools` and
stores one tool plan.

#### `run_tools`

Dispatches each planned tool by name. `ToolError` is converted into a
`tool_error` result so the drafting node can explain the failure rather than
crashing the graph.

#### `draft`

Uses tool results to create a draft and records whether manager approval is
needed.

#### `approval`

If approval is needed and the request has not already been approved, it calls
LangGraph `interrupt`, creating a human-in-the-loop pause. Otherwise it marks
the state approved and continues.

#### `critic`

Refuses an empty draft using `LOW_CONFIDENCE_REFUSAL`; otherwise records
confidence `0.95`. It is the graph's final quality gate before responding.

#### `respond`

Returns the pinned refusal for refused states. For successful states it calls
`write_response`, preserves the generated figures, and writes the final answer
and confidence.

### [`graph.py`](./graph.py)

`build_assistant_graph` creates a `StateGraph(AssistantState)`, registers the
seven nodes, and connects them:

```text
START -> triage -> plan -> tools -> draft -> approval -> critic -> respond -> END
```

`still_going` routes to `END` whenever `state["refused"]` is present; otherwise
the graph continues. The graph can receive a checkpointer, which is how
approval pauses and conversation state are persisted.

### [`workflow.py`](./workflow.py)

`run_assistant` is the public entry point. It:

1. chooses the configured checkpoint path or environment override;
2. creates the parent directory;
3. opens a SQLite `SqliteSaver`;
4. builds the graph with that saver;
5. creates a fresh state;
6. invokes the graph with a `thread_id`;
7. detects whether execution is waiting on an interrupt;
8. ensures refused states have the configured refusal answer.

It returns:

```text
(final_state, waiting_for_approval)
```

The checkpoint database is normally
`database/assistant_threads.sqlite`.

## End-to-end request flow

```text
HTTP question
    |
    v
api.schemas.AssistantRequest
    |
    v
api.routers.assistant.ask_assistant
    |
    v
workflow.run_assistant
    |
    v
triage
    +--> injection_guard.find_injection_markers
    +--> message_classifier.classify_message
    |
    v
plan
    |
    v
tools
    +--> core.loan_calculations
    +--> data_access.database_loader
    +--> risk_models champion model
    +--> retrieval vector store
    |
    v
draft -> optional human approval -> critic -> respond
    |
    v
AssistantResponse
```

## Connections to other folders

- [`api/routers/assistant.py`](../api/routers/assistant.py) exposes
  `run_assistant` at `POST /api/v1/assistant/ask`.
- [`api/schemas.py`](../api/schemas.py) validates the request minimum length
  and shapes the final response.
- [`message_intelligence/injection_guard.py`](../message_intelligence/injection_guard.py)
  blocks prompt injection before tools run.
- [`message_intelligence/message_classifier.py`](../message_intelligence/message_classifier.py)
  supplies category and urgency during triage.
- [`core/loan_calculations.py`](../core/loan_calculations.py) performs EMI
  calculations.
- [`data_access/database_loader.py`](../data_access/database_loader.py)
  retrieves stored applications.
- [`risk_models/baseline_model.py`](../risk_models/baseline_model.py) loads
  the champion risk pipeline.
- [`retrieval/vector_store.py`](../retrieval/vector_store.py) and
  [`retrieval/filtered_search.py`](../retrieval/filtered_search.py) answer
  policy questions from indexed PDFs.
- [`config/constants.py`](../../config/constants.py) supplies artifact paths,
  checkpoint path, refusal text, and confidence thresholds.

## Running it

From the repository root:

```powershell
python -m loanserve.assistant.workflow
```

The API route is normally exercised through the application factory:

```text
POST /api/v1/assistant/ask
{
  "question": "What is the monthly instalment on a 15 lakh loan at 9.5 percent over 120 months",
  "thread_id": "optional-thread-id"
}
```

Required supporting artifacts depend on the question:

- classifier artifact for triage;
- champion model for risk questions;
- SQLite application database for lookup questions;
- Chroma policy store and embedder model for policy questions.
