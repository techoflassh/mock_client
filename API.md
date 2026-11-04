# API

The `api` package is the HTTP boundary of LoanServe. It validates incoming
requests, authenticates users, rate-limits and logs traffic, persists loan
applications, invokes core loan rules and calculations, and exposes the
assistant workflow.

```text
HTTP client
    |
    v
FastAPI application_factory
    |
    +--> middleware: rate limit, optional request log, CORS
    +--> auth router: signup/login/JWT
    +--> application router: CRUD, pricing, schedule
    +--> assistant router: question workflow
    |
    +--> schemas validation
    +--> repository persistence
    +--> core entities/calculations
    +--> assistant workflow
    |
    v
JSON response
```

## Files

### [`__init__.py`](./__init__.py)

Marks the directory as a Python package. It contains no additional behavior.

### [`schemas.py`](./schemas.py)

Defines the Pydantic request, response, and query models.

`LoanApplicationRequest` validates:

- identity/contact strings through `core.validators`;
- supported loan type through `Literal`;
- positive loan amount and income;
- tenure from 1 to 300 months;
- age from 18 to 100;
- credit score from 300 to 900.

Its model validator requires positive collateral for secured products listed in
`config.constants.SECURED_LOAN_TYPES`.

`LoanApplicationResponse` is the priced application response, including
interest rate, instalment, eligibility, and rejection reasons.
`RepaymentSchedule` contains the balance after each month.
`ApplicationFilters` and `ApplicationPage` define filtering and pagination.

`AccountRequest`, `AccountResponse`, and `TokenResponse` shape authentication
requests and results. `AssistantRequest` requires a question of at least five
characters and optionally accepts a thread ID. `AssistantResponse` returns the
answer, workflow steps, narration, refusal, and approval-wait status.

### [`orm_models.py`](./orm_models.py)

Defines the SQLAlchemy persistence model.

`Applicant` is keyed by `applicant_name` and has a one-to-many relationship
with `StoredApplication`.

`StoredApplication` stores application ID, applicant, product, amount, tenure,
income, age, credit score, collateral, PAN, mobile, and email.

`create_session_factory` creates an engine, supports in-memory SQLite with a
static pool for tests, creates tables, and returns a configured `sessionmaker`.

This database is the API application's application store. It is distinct from
the analytics SQLite loader in [`data_access/database_loader.py`](../data_access/database_loader.py),
although both use SQLite and may point to different database files.

### [`repository.py`](./repository.py)

Defines the `ApplicationRepository` abstract interface:

```text
save_application
find_application
list_applications
replace_application
delete_application
```

`InMemoryApplicationRepository` implements the interface with a dictionary. It
mints IDs such as `LA000001`, supports optional loan-type filtering, sorts
records, and slices pages.

`SqlApplicationRepository` implements the same interface with SQLAlchemy. It:

- derives the next numeric application ID;
- creates an applicant row when required;
- stores and returns only `STORED_COLUMNS`;
- supports lookup, pagination, replacement, and deletion;
- converts ORM rows to plain dictionaries with `as_record`.

The common interface lets the application factory use an in-memory repository
for simple/local operation and SQL persistence when `database_url` is passed.

### [`password_hashing.py`](./password_hashing.py)

`PasswordVault` wraps Argon2.

`hash_password` returns a salted Argon2 hash. `matches_stored_hash` verifies a
plain password against a stored hash and returns false for mismatch/invalid
hash failures. Passwords are never stored as plain text by the signup route.

### [`access_control.py`](./access_control.py)

`AccessControl` handles JWT and role authorization.

The constructor reads the JWT secret from `LOANSERVE_JWT_SECRET` or the
development constant and enables `open_mode` only when
`LOANSERVE_OPEN_MODE=1`.

`issue_token` creates claims for subject, role, and expiration.
`read_token` requires a `Bearer` header and decodes/verifies the JWT.
`authorise` is a FastAPI dependency; it verifies the token and restricts
methods in `ADMIN_ONLY_METHODS` (currently DELETE) to administrators.

`record_audit_entry` appends a timestamped user/action line to the configured
audit log.

### [`middleware.py`](./middleware.py)

`SlidingWindowRateLimiter` tracks request timestamps per client key. It removes
timestamps outside the window, rejects when the surviving count reaches the
maximum, and appends allowed requests.

`RequestLogger` is async callable middleware. It measures latency around the
downstream request and appends method, path, status, and milliseconds to a log
file.

### [`application_factory.py`](./application_factory.py)

`create_application` builds an isolated FastAPI application.

It:

1. registers a domain-error handler for `LoanServeError`;
2. enables configured CORS;
3. creates access control, password, and account state;
4. chooses SQL or in-memory application repository;
5. stores dependencies on `app.state`;
6. installs rate limiting and optional logging middleware;
7. registers `/health`;
8. mounts authentication, application, and optional assistant routers.

The assistant import is optional and guarded so an unavailable assistant
dependency does not prevent the core API from starting.

The health endpoint queries the repository and returns status plus application
count.

### [`routers/user_accounts.py`](./routers/user_accounts.py)

Mounts under `/auth`.

`POST /signup` rejects duplicate usernames, hashes the password, and stores the
account in the app's in-memory `user_accounts` dictionary.

`POST /login` verifies credentials, issues a JWT, schedules an audit entry,
and returns a bearer token.

Authentication accounts are currently process-local; application records can
be SQL-backed, but user accounts are not persisted by `orm_models.py`.

### [`routers/loan_applications.py`](./routers/loan_applications.py)

Mounts under `/applications`.

`price_application` converts a stored dictionary into a core loan object with
`create_loan_application`, then calculates interest rate, monthly instalment,
eligibility, and rejection reasons. It returns a validated
`LoanApplicationResponse`.

Endpoints:

- `POST /applications`: validate, persist, and price a new application;
- `GET /applications`: filter/page applications and price every result;
- `GET /applications/{id}`: retrieve and price one application;
- `PUT /applications/{id}`: replace and re-price an application;
- `DELETE /applications/{id}`: delete an application;
- `GET /applications/{id}/schedule`: build balances with
  `core.loan_calculations.build_outstanding_balances`.

### [`routers/assistant.py`](./routers/assistant.py)

Mounts under `/assistant`.

`POST /assistant/ask` creates a thread ID when one is not supplied, calls
[`assistant.workflow.run_assistant`](../assistant/workflow.py), collects
LangGraph narration messages, and maps the final state to
`AssistantResponse`.

## Request flows

### Creating a loan application

```text
POST /api/v1/applications
        |
        v
LoanApplicationRequest
        |
        +--> PAN/mobile/email core validators
        +--> numeric and product validation
        |
        v
repository.save_application
        |
        v
create_loan_application
        |
        +--> core loan calculations
        +--> age/DTI/amount/LTV/credit checks
        |
        v
LoanApplicationResponse
```

### Reading a schedule

```text
GET /api/v1/applications/{id}/schedule
        |
        v
repository.find_application
        |
        v
core.entities.create_loan_application
        |
        v
core.loan_calculations.build_outstanding_balances
        |
        v
monthly balances response
```

### Asking the assistant

```text
POST /api/v1/assistant/ask
        |
        v
AssistantRequest
        |
        v
assistant.workflow.run_assistant
        |
        +--> message intelligence safety/classification
        +--> core calculation
        +--> database lookup
        +--> risk model
        +--> retrieval policy search
        |
        v
AssistantResponse
```

## Connections to other folders

- [`core/`](../core/) provides validators, domain entities, calculations, and
  deliberate exceptions.
- [`data_access/`](../data_access/) provides CSV/analytics storage; the API
  has its own repository abstraction for request-time application persistence.
- [`assistant/`](../assistant/) supplies the LangGraph question workflow.
- [`message_intelligence/`](../message_intelligence/) protects and classifies
  assistant input.
- [`risk_models/`](../risk_models/) supplies the champion model when the
  assistant's risk tool is used.
- [`retrieval/`](../retrieval/) supplies policy passages for assistant
  questions.
- [`config/constants.py`](../../config/constants.py) supplies CORS, rate,
  authentication, audit, and artifact settings.

## Running the API

Run from the repository root:

```powershell
python -m loanserve.api.application_factory
```

For a real ASGI server, point Uvicorn or another ASGI server at the factory
or application object according to the deployment configuration. The factory
is intentionally parameterized so tests can pass an in-memory or temporary
database, a custom rate limit, and a request-log path.

## Security and operational boundaries

- Configure a non-development JWT secret in deployment.
- Keep `LOANSERVE_OPEN_MODE` disabled outside controlled local/test use.
- Authentication accounts are currently in memory and disappear on restart.
- The rate limiter is process-local and does not coordinate across workers.
- `DELETE` is administrator-only when authorization is enabled.
- Domain errors are converted to HTTP 400 responses by the application-level
  handler.
