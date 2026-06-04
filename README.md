# 📋 How to Use the Specs in This Repository

This repository uses **spec-driven development** — a structured approach where requirements, architecture design, and implementation tasks are formalised as documents before any code is written. This README explains what the specs are, where they live, how they were created, and how to use them to drive implementation.

---

## 🤔 What Is Spec-Driven Development?

Spec-driven development follows a deliberate sequence:

```
Define → Refine → Design → Validate → Then Code
```

The goal is to avoid the most common failure mode in software projects: writing code before the problem is fully understood. By formalising requirements, design decisions, and correctness properties upfront, you get:

- 🎯 A shared source of truth for the whole team
- 🧪 Testable acceptance criteria before a line of code is written
- 🏗️ Architecture decisions documented with their reasoning — not just the outcome
- 📦 An implementation plan that can be executed incrementally with clear checkpoints

---

## 📁 Where the Specs Live

All specs are stored under `.kiro/specs/`. Each feature or workstream has its own subdirectory:

```
.kiro/specs/
├── customer-directory-service/     # The main REST API service
│   ├── .config.kiro                # Spec metadata (type, workflow)
│   ├── requirements.md             # What the system must do
│   ├── design.md                   # How it is built and why
│   └── tasks.md                    # Ordered implementation task list
│
└── azure-waf-policies/             # Azure Policy definitions (WAF pillars)
    ├── .config.kiro
    └── requirements.md
```

### ⚙️ The `.config.kiro` file

Each spec directory contains a `.config.kiro` file that records the spec's metadata:

```json
{
  "specId": "a6c02f4b-b043-437f-a228-1e4951f530b2",
  "workflowType": "requirements-first",
  "specType": "feature"
}
```

| Field | Values | Meaning |
|---|---|---|
| `specType` | `feature`, `bugfix` | Whether this is new functionality or a fix |
| `workflowType` | `requirements-first`, `design-first` | Which document was created first |

---

## 📄 The Three Spec Documents

Every complete spec has three documents. They are always created in order — you do not move to the next until the current one is reviewed and agreed.

### 1. 📝 `requirements.md` — What the system must do

Written in [EARS pattern](https://alistair.cockburn.us/ears-the-easy-approach-to-requirements-syntax/) (Event-Action-Response-State). Every requirement has:

- A **user story** — who wants it and why
- **Acceptance criteria** — precise, testable conditions using `WHEN / IF / THEN / SHALL`

Example from this repo:

```
### Requirement 2: API Key Authentication

User Story: As a security officer, I want all non-health endpoints protected
by an API key, so that only authorised callers can read or modify customer data.

Acceptance Criteria:
1. WHEN a request is received for any endpoint other than GET /health,
   THE Service SHALL inspect the x-api-key request header.
2. IF the x-api-key header is absent, THEN THE Service SHALL respond with
   HTTP 401 and a JSON body { "error": "Unauthorized", "message": "Missing API key" }.
```

> 💡 **How to use it:** Read `requirements.md` before writing any code. Each acceptance criterion maps directly to a test case. If you cannot write a test for it, the requirement is not specific enough.

---

### 2. 🏗️ `design.md` — How it is built and why

The design document covers:

- 🔀 **Architecture** — layered structure, component responsibilities, request flow
- 🌐 **Network topology** — local Docker Compose and production Kubernetes diagrams
- ☁️ **Azure Landing Zone** — hub-and-spoke VNet, AKS config, security controls, observability pipeline, CI/CD supply chain
- 🗄️ **Data models** — Prisma schema, TypeScript domain types, API request/response shapes
- 🧠 **Key design decisions** — each decision is documented with its reasoning and trade-offs
- ✅ **Correctness properties** — formal statements about system behaviour used to drive property-based tests
- ⚠️ **Error handling** — error hierarchy and handler behaviour
- 🧪 **Testing strategy** — unit, integration, and property-based test approach
- 💰 **Cost estimate** — Azure hosting costs by environment with optimisation options

> 💡 **How to use it:** Read `design.md` before implementing any module. The component responsibilities table tells you exactly what each file should and should not do. The correctness properties section tells you what property-based tests to write.

---

### 3. ✅ `tasks.md` — Ordered implementation task list

The task list breaks the design into concrete, sequenced implementation steps. Each task:

- Has a clear deliverable
- References the requirements it satisfies (e.g. `_Requirements: 2.1, 2.2, 2.3_`)
- Is ordered so that each task builds on the previous one
- Includes **checkpoints** — explicit verification steps before moving on

Tasks use checkbox syntax:

```markdown
- [ ]   Required task — must be completed
- [ ]*  Optional task — can be skipped for MVP (property-based tests)
- [-]   In progress
- [x]   Completed
```

> 💡 **How to use it:** Work through `tasks.md` top to bottom. Do not skip ahead. The checkpoints (tasks 10, 13, 16) are gates — verify the service compiles, tests pass, and coverage meets the threshold before continuing.

---

## 🔀 The Two Spec Workflows

### ✅ Requirements-First (default)

Used when you have clear business needs but the technical approach is open.

```
requirements.md → design.md → tasks.md
```

Start by agreeing what the system must do, then design how to build it, then plan the implementation. This is the workflow used for both specs in this repo.

### 🎨 Design-First

Used when the technical approach is already clear (e.g. you are documenting an existing system or have a strong architectural vision).

```
design.md → requirements.md → tasks.md
```

The design is created first, then requirements are derived from it.

---

## 🚀 How to Create a New Spec

### Step 1 — ✍️ Write a clear prompt

The quality of the spec depends entirely on the quality of the input. A good prompt includes:

- 🏢 **Context** — domain, client type, scale expectations
- 📋 **Functional requirements** — what the system must do, listed explicitly
- 🔒 **Non-functional requirements** — security, observability, reliability, testing, packaging
- 🔧 **Tech stack** — language, framework, persistence, tooling
- 🎯 **Quality gates** — what must pass before the work is done

See `.kiro/specs/Example Prompt` for a complete worked example. See `.kiro/specs/Prompt Guidelines` for a library of 10 reusable prompt patterns covering foundation, refinement, API contracts, data model, architecture, implementation plan, test strategy, production readiness, and consulting review.

---

### Step 2 — 🔀 Choose your workflow

When starting a new spec, you will be asked:

> **Is this a new feature or a bugfix?**

Then (for features):

> **What do you want to start with — Requirements or Technical Design?**

Choose **Requirements** unless you already have a clear technical vision.

---

### Step 3 — 🔍 Review each document before proceeding

The workflow is iterative. After each document is created, review it and provide feedback before the next document is generated. Common things to check:

**📝 In `requirements.md`:**
- Are all acceptance criteria testable? Can you write a failing test for each one?
- Are edge cases covered (empty inputs, duplicates, boundary values)?
- Are non-functional requirements specific enough (e.g. `< 500ms` not `fast`)?

**🏗️ In `design.md`:**
- Does the architecture match the requirements?
- Are all key decisions documented with their reasoning?
- Are the correctness properties specific enough to drive property-based tests?
- Does the cost estimate reflect the actual infrastructure choices?

**✅ In `tasks.md`:**
- Is the ordering logical? Does each task depend only on previously completed tasks?
- Are the checkpoints in the right places?
- Are optional tasks clearly marked?

---

### Step 4 — 🏃 Execute tasks in order

Work through `tasks.md` sequentially. At each checkpoint:

```bash
npm run build    # ✅ must pass before continuing
npm test         # ✅ must pass before continuing
npm run lint     # ✅ must pass before continuing
```

Do not mark a task complete until its acceptance criteria are verifiably met.

---

## 📐 Spec Conventions Used in This Repo

### 🗣️ EARS requirement pattern

All acceptance criteria follow the EARS (Easy Approach to Requirements Syntax) pattern:

| Pattern | When to use |
|---|---|
| `WHEN <trigger>, THE <system> SHALL <response>` | ⚡ Event-driven behaviour |
| `IF <condition>, THEN THE <system> SHALL <response>` | 🔀 Conditional behaviour |
| `THE <system> SHALL <capability>` | 🔁 Ubiquitous behaviour (always true) |
| `WHERE <feature is included>, THE <system> SHALL <response>` | 🔧 Optional feature behaviour |

---

### 🧮 Correctness properties

The design document defines **correctness properties** — formal statements about system behaviour that hold for all valid inputs, not just example inputs. These drive property-based tests using [fast-check](https://github.com/dubzzz/fast-check).

Example from this repo:

```
Property 5: Search results always match the search term

For any set of customers in the database and any non-empty search string,
every customer returned in GET /customers?search=<term> SHALL have a name
or email field that contains the search string (case-insensitively).
No customer whose name and email both do not contain the search string
SHALL appear in the results.

Validates: Requirements 5.6, 5.7
```

Each property test is tagged with a comment:

```typescript
// Feature: customer-directory-service, Property 5: Search results always match the search term
```

---

### 🔗 Requirement traceability

Every task in `tasks.md` references the requirements it satisfies:

```markdown
- [ ] 5.2 Implement src/plugins/auth.ts
  - Register a preHandler hook scoped to a sub-router
  - Compare x-api-key header against config.apiKey
  - _Requirements: 2.1, 2.2, 2.3_
```

This makes it straightforward to verify that every requirement has at least one task implementing it, and every task has a clear reason to exist.

---

## 📊 Specs in This Repository

### 🛠️ `customer-directory-service` — Complete spec (3 documents)

A production-ready REST API for managing customer records.

| Document | Status | Description |
|---|---|---|
| `requirements.md` | ✅ Complete | 16 requirements covering all functional, security, observability, reliability, testing, and packaging concerns |
| `design.md` | ✅ Complete | Layered architecture, network topology, Azure Landing Zone, data models, 6 key design decisions, 10 correctness properties, cost estimate |
| `tasks.md` | ✅ Complete | 16 top-level tasks, 3 checkpoints, 10 optional property-based test tasks |

### ☁️ `azure-waf-policies` — Partial spec (requirements only)

Azure Policy definitions aligned to the five Azure Well-Architected Framework pillars.

| Document | Status | Description |
|---|---|---|
| `requirements.md` | ✅ Complete | 22 requirements across Reliability, Security, Cost Optimisation, Operational Excellence, and Performance Efficiency pillars |
| `design.md` | ⬜ Not started | Bicep/ARM file structure, initiative definitions, assignment templates |
| `tasks.md` | ⬜ Not started | Implementation tasks for policy authoring and deployment |

---

## 📦 Consolidated Documentation

A single consolidated document combining all spec content is available at the repo root:

| File | Format | Description |
|---|---|---|
| `customer-directory-service-full-spec.md` | 📄 Markdown | Full spec — all sections in one file |
| `customer-directory-service-full-spec.docx` | 📝 Word | Word document with embedded architecture diagrams |
| `architecture-inception-canvas.docx` | 🖼️ Word | Single-page landscape canvas summarising the solution |

---

## 📚 Further Reading

- 📖 `.kiro/specs/Prompt Guidelines` — library of 10 reusable prompt patterns for spec-driven development
- 💡 `.kiro/specs/Example Prompt` — the original prompt used to generate the customer-directory-service spec
- 🔗 [EARS Requirements Syntax](https://alistair.cockburn.us/ears-the-easy-approach-to-requirements-syntax/)
- 🔗 [fast-check property-based testing](https://github.com/dubzzz/fast-check)
- 🔗 [Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/)
