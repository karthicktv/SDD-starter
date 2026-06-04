
# 🔷 1. Foundation Prompt (Always Start Here)

Use this to force spec-first thinking before any code is written.
You are a senior software architect.

I want to follow spec-driven development. Do NOT write code yet.

First, create a complete specification for the system based on the requirements below.

Include:
- Problem definition (in plain English)
- Functional requirements (clear, testable)
- Non-functional requirements (performance, security, scalability, compliance)
- Assumptions and constraints
- Key architectural decisions (with reasoning)
- Data model (high-level)
- API/event contracts
- Risks and trade-offs
- Definition of Done

Requirements:
[PASTE REQUIREMENTS]

Make it suitable for enterprise use (production-grade, secure, observable).

# 🔷 2. Refinement Prompt (Iterative Spec Improvement)
Review the specification above.

Act as:
- Solution architect
- Security architect
- SRE / platform engineer

Identify:
- Gaps or ambiguities
- Missing non-functional requirements
- Security risks (data exposure, auth, compliance)
- Scalability issues
- Observability gaps

Then:
- Improve the specification
- Add concrete acceptance criteria
- Make it ready for implementation

Keep everything in plain English and structured.

# 🔷 3. API Contract Prompt
Based on the specification, define the API contracts.

Include:
- Endpoints (REST or events)
- Request/response schemas
- Validation rules
- Error handling model
- Authentication and authorization approach
- Rate limiting considerations

Make this:
- Clean
- Versionable
- Production-ready

Avoid implementation details.

# 🔷 4. Data Model & Storage Prompt
Design the data model based on the specification.

Include:
- Entities and relationships
- Key attributes
- Indexing strategy
- Data lifecycle (create, update, archive, delete)
- Constraints and validations

Also explain:
- Why this model was chosen
- Trade-offs vs alternatives

Keep it platform-neutral unless specified.

# 🔷 5. Architecture Design Prompt
Create a high-level system architecture based on the specification.

Include:
- Components and responsibilities
- Interaction flows
- External integrations
- Deployment model (cloud-ready)
- Security boundaries
- Observability (logging, metrics, tracing)

Also include:
- Architecture diagram in Mermaid format
- Justification for design choices
- How this supports scalability and resilience

Make this suitable for a consulting client presentation.

# 🔷 6. Implementation Plan Prompt
Create a step-by-step implementation plan from the specification.

Include:
- Phases (MVP → production)
- Priority order
- Dependencies
- Risks per phase
- Testing strategy
- Definition of Done per phase

Make it realistic for an enterprise delivery team.

# 🔷 7. Code Generation Prompt (Only After Spec is Final)
Now implement the system based strictly on the approved specification.

Constraints:
- Follow the architecture and API contracts exactly
- Include:
  - Clean code structure
  - Error handling
  - Logging and observability hooks
  - Security (input validation, auth checks)
  - Tests

Tech stack:
[Specify e.g. Node.js, Azure, .NET, etc.]

Do NOT deviate from the specification unless explicitly justified.

#🔷 8. Test Strategy Prompt
Create a comprehensive test plan.

Include:
- Unit tests
- Integration tests
- Contract tests
- Performance tests
- Security tests

Also:
- Map tests to requirements
- Identify critical test scenarios
- Suggest automation strategy

# 🔷 9. Production Readiness Prompt (Brilliant for your consulting use)
Evaluate if this system is production-ready.

Check for:
- Scalability
- Resilience
- Security
- Observability
- Cost considerations
- Operational readiness

List:
- Gaps
- Risks
- Recommendations to fix

Answer as if presenting to a client go-live review board.

# 🔷 10. “Consulting Mode” Prompt (Your superpower 💡)
Matches exactly what you’re doing in enterprise architecture work:
Act as a consulting architect.

Review this solution from a client perspective.

Evaluate:
- Business alignment
- Risk exposure
- Compliance (GDPR, public sector, etc.)
- Cost vs value
- Vendor lock-in risks

Suggest:
- Improvements
- Alternatives
- Questions we should ask the client

Keep it executive-friendly.

# 🔷 🔥 Pro Tips (Based on your style + what works best)
From your own internal prompt deck: 
👉 “Vague prompts → shallow output. Clear constraints → production-ready output.” 1
So:
✅ Always include:
	• Context (domain, client, scale)
	• Constraints (security, compliance, cost)
	• Expected outputs (structure, format)
❌ Avoid:
	• “Build me an API”
	• No NFRs
	• No architecture direction

🔷 Bonus: One-Line Master Prompt Pattern
If you want a quick reusable format:
Define → Refine → Design → Validate → Then Code

✅ If you want next step
I can tailor these specifically for:
	• Azure + OpenTelemetry (your DSP observability work)
	• Agentic AI / AI SDLC platforms (what you’re working on now)
	• Kilsaran-style enterprise integrations
Just tell me 👍
