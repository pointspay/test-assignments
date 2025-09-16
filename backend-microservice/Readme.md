# AI‑First Microservice Assignment — Payments gRPC

Design an AI‑first coding exercise that **explicitly encourages using AI tools (Copilot, ChatGPT, Cursor, Codeium, etc.)**. We want to see *how* you collaborate with AI to deliver a small, production‑shaped service quickly and thoughtfully.

---

## 🎯 Goal

Build a small **gRPC microservice** that processes a minimal **RequestPayment** workflow with strong developer ergonomics (CI, tests, style, Docker). Embrace AI for speed, but demonstrate judgment, architecture sense, and testing discipline.

---

## 🧪 What we evaluate

1. **Architecture & reasoning** — clear boundaries, idempotency, error model, trade‑offs documented.
2. **AI collaboration** — how you prompt, iterate, verify, and correct AI output (trace required).
3. **Code quality** — readability, structure, naming, comments where they add value.
4. **Reliability** — tests, coverage, handling of unhappy paths.
5. **DevEx** — one‑command local run, clean Docker image, fast CI, style checks.
6. **Security hygiene** — secrets handling, dependency scanning basics.

---

## 📝 Deliverables (all required)

* `README.md` — how to run, test, build, and call the service.
* `ARCHITECTURE.md` — brief design (data model, idempotency strategy, error model, chosen patterns).
* `AI_TRACE.md` — **sanitized** log of your prompts and AI responses you used (see format below).
* `COPILOT.md` — your guidance/instructions for AI on this repo (context, constraints, preferred style, test strategy, example prompts).
* Working **gRPC service** implementing the spec (below), with **Dockerfile** and **docker-compose.yml** (if you use any additional deps).
* **GitHub Actions** CI: lint, type‑check (if applicable), test (with coverage gate), build the Docker image.
* **Style checks**: formatter + linter (and type checker where relevant).
* **Tests**: unit tests covering success and failure paths; include idempotency tests.

> Language choices: **Python**, or **Java**.


---

## 💳 gRPC API Specification

Use Protocol Buffers v3.

```proto
syntax = "proto3";
package payments.v1;

import "google/protobuf/timestamp.proto";

service PaymentService {
  // Creates a payment request. Must be idempotent by idempotency_key.
  rpc RequestPayment(RequestPaymentRequest) returns (RequestPaymentResponse);

  // Retrieves a single payment by its payment_id.
  rpc GetPayment(GetPaymentRequest) returns (GetPaymentResponse);

  // Lightweight health check.
  rpc Health(HealthRequest) returns (HealthResponse);
}

message RequestPaymentRequest {
  // Amount in minor units (e.g., cents). Must be > 0.
  int64 amount_minor = 1;
  // ISO 4217 (e.g., "USD", "EUR").
  string currency = 2;
  // Merchant order identifier (opaque string).
  string order_id = 3;
  // Required idempotency key. Same key + same payload returns the same result.
  string idempotency_key = 4;
  // Optional metadata for debugging/demo.
  map<string, string> metadata = 5;
}

message RequestPaymentResponse {
  string payment_id = 1; // server‑generated
  PaymentStatus status = 2;
  string message = 3; // human‑readable summary
  string idempotency_key = 4; // echo back
  google.protobuf.Timestamp created_at = 5;
}

enum PaymentStatus {
  PAYMENT_STATUS_UNSPECIFIED = 0;
  PAYMENT_STATUS_PENDING = 1;
  PAYMENT_STATUS_SUCCEEDED = 2;
  PAYMENT_STATUS_FAILED = 3;
}

message GetPaymentRequest { string payment_id = 1; }

message GetPaymentResponse {
  string payment_id = 1;
  PaymentStatus status = 2;
  int64 amount_minor = 3;
  string currency = 4;
  string order_id = 5;
  string idempotency_key = 6;
  google.protobuf.Timestamp created_at = 7;
  string message = 8;
}

message HealthRequest {}
message HealthResponse { string status = 1; } // e.g., "ok"
```

### Functional requirements

* **Idempotency**: repeated `RequestPayment` with the same `idempotency_key` **must** return the original result without creating a new payment.
* **Validation**: reject non‑positive amounts; reject malformed currency codes; require idempotency key.
* **Persistence**: in‑memory is OK; file/SQLite/postgres optional. Persist enough to make idempotency work across process restarts **or** document the limitation if memory‑only.
* **Observability**: structured logs for each call; simple metrics (counters) are a plus.
* **Error model**: use appropriate gRPC status codes; include helpful messages.

---

## ⚙️ Non‑functional requirements

### Docker

* Multi‑stage build; small final image.
* Healthcheck (container) and graceful shutdown.

### CI (GitHub Actions)

* Jobs: `lint` → `typecheck` (if applicable) → `test` (with coverage gate) → `build` (Docker).
* Fail fast; cache deps where possible.
* Optional: push image to GHCR (if you prefer).

### Style & typing

* **Go**: `gofmt`, `golangci-lint`, minimal `make` targets.
* **Python**: `ruff` (lint+format), `mypy`, `pytest-cov`.
* **TS/Node**: `eslint`, `prettier`, `tsc`, `vitest/jest` + coverage.
* **Java**: `spotless`/`google-java-format`, `spotbugs`/`checkstyle`, `junit` + coverage.

### Tests

* Cover success and typical failure cases (validation, idempotency replay, not‑found).
* Aim for **≥ 80%** line coverage (state your threshold in README and enforce it in CI).
* Include at least one **table‑driven**/parameterized test.

---

## 🤝 AI Collaboration Requirements

We explicitly want to see how you use AI effectively.

Include an `AI_TRACE.md` with entries like:

```markdown
## 2025-09-16 10:12 — Designing idempotency strategy
**Prompt:** "Given a gRPC RequestPayment with idempotency_key, propose a repository pattern in Go..."
**AI Output (excerpt):** code snippet / approach
**Decision:** Accepted repo interface, rewrote storage for clarity.
**Verification:** Added table tests for replay; ensured same response returned.
```

Your `COPILOT.md` should contain:

* Project context summary and domain vocabulary.
* Constraints (e.g., amount in minor units, enum semantics).
* Coding style preferences (naming, error handling, logging).
* **Prompt playlist**: 5–10 high‑leverage prompts you actually used or would use (e.g., “write table‑driven tests for X”, “produce a Makefile for Y”, “explain code smell Z”).
* How to ask the agent for **counter‑examples** and test‑case suggestions.

> You may redact any confidential info. Do not include API keys or personal data.

---

## 📂 Suggested repo layout

```
.
├─ proto/
│  └─ payments.proto
├─ cmd/server/           # main entrypoint
├─ internal/
│  ├─ app/               # service wiring
│  ├─ payments/          # domain logic
│  ├─ storage/           # repository interface + impl (memory, file, etc.)
│  └─ transport/grpc/    # handlers, mapping errors ↔ status codes
├─ pkg/                  # (optional) shared small utilities
├─ tests/                # e2e/integration (spins up server in‑proc)
├─ Dockerfile
├─ docker-compose.yml    # if you add deps
├─ Makefile              # dev ergonomics: build/test/lint/run
├─ .github/workflows/ci.yml
├─ README.md
├─ ARCHITECTURE.md
├─ AI_TRACE.md
└─ COPILOT.md
```

---

## ✅ Acceptance checklist

* [ ] `payments.proto` compiles; service starts; Health returns `ok`.
* [ ] `RequestPayment` enforces idempotency.
* [ ] Validation & clear gRPC error codes.
* [ ] Lint/format/typecheck pass locally and in CI.
* [ ] Tests pass with stated coverage threshold.
* [ ] Docker image builds and runs; documented.
* [ ] AI trace and Copilot guidance included.

---

## 🧰 Starter snippets (optional)

Provide equivalents for your chosen language.

### Example: GitHub Actions (template)

```yaml
name: ci
on:
  push:
  pull_request:
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install ruff
      - run: ruff check . && ruff format --check .
  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install -r requirements.txt
      - run: pytest --maxfail=1 --disable-warnings --cov=.
  build-image:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t payments-svc:ci .
```

### Example: Dockerfile (multi‑stage; Python)

```dockerfile
# build stage
FROM python:3.12-slim AS build
WORKDIR /app
COPY pyproject.toml poetry.lock* ./
RUN pip install poetry && poetry export -f requirements.txt --output requirements.txt --without-hashes
COPY . .

# run stage
FROM python:3.12-slim
ENV PYTHONUNBUFFERED=1
WORKDIR /app
COPY --from=build /app /app
RUN pip install -r requirements.txt
EXPOSE 8080
HEALTHCHECK CMD python -c "import socket; import sys; s=socket.socket(); s.settimeout(1); s.connect(('127.0.0.1',8080)); sys.exit(0)"
CMD ["python", "-m", "payments_server"]
```

> Replace with idiomatic equivalents for Go/TS/Java if you choose those.

---

## 🧗 Stretch goals (pick any)

* **Persistence** beyond memory (SQLite/Postgres with migrations).
* **OpenTelemetry** traces/metrics; request IDs and structured logs.
* **Config** via env vars; 12‑factor alignment.
* **Rate‑limit** per order or idempotency key; basic retry/backoff semantics.
* **Contract tests** or e2e tests that spin up the gRPC server in‑proc.
* **Expose gRPC‑Gateway** to provide JSON/HTTP for demo convenience.

---

## 🔒 Security & compliance hygiene

* No secrets in code, config, or CI logs.
* Add a basic dependency audit step (e.g., `pip-audit`, `npm audit --audit-level=high`, `go list -m -u all` + `govulncheck`).
* Validate all inputs; prefer allow‑lists for currency.

---

## 🕒 Timebox & expectations

* Suggested effort: **3–6 hours** across a few days. Document what you’d do next with more time.

---

## 📮 Submission instructions

1. We provide a **public template repository** with this README.
2. Create a **private repository** using “Use this template”.
3. Implement your solution there and keep a **meaningful commit history** (don’t squash everything into one commit).
4. Add us as **collaborators** (read access) and share a link by email.
5. Include a short note in your README on: decisions you made, trade‑offs, things you’d improve with more time.

> If you prefer, you can send us a tarball/zip. Don’t share any secrets.

---

## 🧮 Scoring rubric (100 pts)

* **Architecture & API design** — 25
* **Code quality & structure** — 20
* **Tests & coverage** — 20
* **DevEx (Docker, CI, style checks)** — 15
* **AI collaboration quality (AI\_TRACE & COPILOT.md)** — 15
* **Security & reliability touches** — 5

**Pass** ≥ 75. **Strong** ≥ 85.

---

## ❓FAQ

* **Can I change the proto?** Minor additions are fine; keep the required fields & semantics.
* **Do I need a database?** Not required. If memory‑only, clearly document idempotency limitations on restart.
* **Can I use generators/frameworks?** Yes, but explain what was generated and why. Make sure you understand and can defend the code.
* **May I use AI extensively?** Yes. Show your process and how you validated outputs.

---

## 🔧 AI‑first guiding principles (for candidates)

1. **Prime your agent**: share the domain, constraints, and definitions (e.g., minor units). Keep a persistent context in `COPILOT.md`.
2. **Decompose**: ask AI for a plan (proto → handlers → storage → tests → CI → Docker). Commit after each step.
3. **Adversarially test**: ask AI for negative cases and edge conditions, then implement tests.
4. **Tight loops**: use short prompts and paste only relevant code; iterate quickly.
5. **Own the code**: refactor AI output; add comments only where they clarify intent.
6. **Verify**: run linters, type‑checkers, tests; don’t trust first drafts.

---

## 🔍 AI\_TRACE.md template

```markdown
# AI Collaboration Trace

> Keep entries short; include why you accepted/rejected AI suggestions.

## 2025-09-16 09:40 — Project skeleton
Prompt:
"Create a Go module layout for a gRPC payments service with idempotency and table‑driven tests."
AI Output (excerpt):
- Proposed `internal/` with `storage` interface
Decision:
- Accepted structure; renamed packages for clarity
Verification:
- Compiles; tests pass

## 2025-09-16 10:55 — Idempotency replay test
Prompt:
"Write tests asserting same idempotency_key returns same payment_id and response."
Decision:
- Adopted test; tightened assertions on timestamps/status
```

---

## 🧭 Maintainer notes (for our public repo)

* Include this document as `README.md`.
* Add an `ISSUE_TEMPLATE` for questions.
* Add a `DISCUSSIONS` Q\&A category if you want public clarifications.
* Provide minimal stubs (empty `proto/` with the schema) but avoid full scaffolds to keep the assessment authentic.

