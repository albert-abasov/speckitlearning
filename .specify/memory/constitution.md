<!-- Sync Impact Report
VERSION BUMP: 1.0.0 (Initial Constitution)
RATIFIED: 2026-07-31
SECTIONS: Core Principles (5), Security Requirements, Technology Standards, Development Workflow, Governance
PRINCIPLES: MVP-First Delivery, API-WebAssembly Contract, Security by Design, Test-Driven Quality, Incremental Architecture
STATUS: Complete - all placeholders filled from project materials and user directive
-->

# RSS Feed Reader MVP Constitution

## Core Principles

### I. MVP-First Delivery

Every feature decision MUST prioritize getting a working MVP into production rather than perfecting
for the final state. For the RSS Feed Reader:

- Start with subscription management UI only (add URL, display list) with in-memory storage
- Defer feed fetching, parsing, persistence, and advanced features to post-MVP phases
- Each phase (MVP → Extended-MVP → Post-MVP) is a complete, shippable increment
- MUST NOT block MVP progress on future phases; design MUST accommodate future needs without
  requiring rewrites
- Rationale: Delivers user-facing value quickly; gathering real usage feedback informs architecture
  for persistence, feed processing, and advanced features

### II. API-WebAssembly Contract Clarity

All communication between ASP.NET Core backend (port 5151) and Blazor WebAssembly frontend (port 5213)
MUST be explicit, version-aware, and documented:

- Define all API endpoints in REST style; document request/response payloads (shape, validation rules)
- Blazor client code MUST NOT assume implicit server behavior; all feed operations and storage decisions
  MUST be explicitly specified in the contract, not inferred
- Version API endpoints (e.g., `/api/v1/subscriptions`) to support future changes without breaking
  existing clients
- MUST include clear error responses (HTTP status codes + error bodies) for client-side error handling
- Rationale: Enables independent frontend/backend development; reduces coupling; clarifies MVP vs.
  extended/post-MVP boundaries

### III. Security by Design (Data & Network)

Feed subscription handling MUST incorporate security practices from MVP onwards, not as an afterthought:

- URL input validation MUST reject obviously malformed URLs; in MVP, store safely without fetching
- Store URLs as-is in subscription state; MUST NOT execute, parse, or fetch URLs during MVP
- When Extended-MVP adds feed fetching (System.ServiceModel.Syndication), MUST validate URLs before HTTP
  requests and handle network errors gracefully
- MUST sanitize feed content (titles, links) before rendering in Blazor; use HTML encoding by default
- Future SQLite storage MUST use parameterized queries; in-memory MVP MUST avoid string concatenation for
  data handling
- Rationale: Prevents injection, network-based attacks, and information disclosure as features expand;
  establishes secure baseline before persistence and network I/O layers are added

### IV. Test-Driven Quality (Unit & Integration)

Code quality is non-negotiable; testing strategy scales with feature complexity:

- MVP: Unit tests for subscription management logic (add, display); xUnit MUST be used
- Extended-MVP and later: Integration tests for feed fetching, error cases, and persistence
- Every public API method MUST have at least one unit test; tests MUST verify both happy path and
  documented error conditions
- MUST NOT merge code with test coverage below 70%; coverage gaps MUST be documented with rationale
- Rationale: Maintains code correctness as features add complexity; xUnit integration supports
  background service and EF Core testing in future phases

### V. Incremental Architecture

Code structure MUST support adding features (feed fetching, persistence, refresh, sorting) without
major refactoring:

- Separate concerns: Subscription data model, API contract layer, UI components, and future storage layer
  MUST be independently testable and modifiable
- In-memory storage (MVP) MUST use a defined interface; future EF Core + SQLite implementation replaces
  it without changing API contract or UI code
- Blazor components MUST receive data via props/parameters, not fetch directly; this allows UI to work
  identically whether backend uses in-memory or database storage
- Abstract feed operations (fetch, parse, error handling) into a service layer; MVP can stub it; later
  phases implement it with System.ServiceModel.Syndication
- Rationale: Prevents MVP compromises from blocking post-MVP work; reduces refactoring cost; code remains
  maintainable as complexity grows

## Security Requirements

- URL input MUST be validated (well-formed URI check); reject if malformed
- In MVP and beyond, MUST NOT execute shell commands, file operations, or network requests based on user
  input without explicit allowlisting
- Feed data (when added in Extended-MVP) MUST be treated as untrusted; sanitize before storage and display
- Credentials (authentication tokens, API keys for future feed sources) MUST NOT be embedded in code;
  use configuration/secrets management from day one
- HTTP communication between frontend and backend MUST use HTTPS in production; cross-origin (CORS) policy
  MUST be restrictive and documented

## Technology Standards

- Backend: ASP.NET Core Web API (.NET latest stable); runs on port 5151
- Frontend: Blazor WebAssembly; runs on port 5213
- Storage: In-memory (MVP); EF Core + SQLite (Extended-MVP onward)
- Feed Processing: System.ServiceModel.Syndication (Extended-MVP, requires external fetch layer)
- Testing: xUnit for unit/integration tests; code MUST compile without warnings
- Development: C#, cross-platform (Windows, macOS, Linux via dotnet CLI)
- Logging: Structured logging (Microsoft.Extensions.Logging) from MVP; rotate and archive logs in production

## Development Workflow

- All features MUST be defined in `/tasks.md` before coding begins; task description includes API contract
  (input/output), acceptance criteria, and test cases
- Code review MUST verify: (a) feature matches task description, (b) tests pass, (c) test coverage ≥ 70%,
  (d) no hardcoded secrets, (e) error messages are user-friendly and logged
- CI/CD MUST run unit tests on every PR; code MUST compile successfully and all tests MUST pass before merge
- Versioning: Use SemVer (MAJOR.MINOR.PATCH); MAJOR increments for MVP → Extended-MVP → Post-MVP transitions
  or breaking API changes; MINOR for feature additions; PATCH for bug fixes
- Issues related to future phases (Post-MVP) MUST be tagged `future-phase` and MUST NOT block MVP development

## Governance

This constitution supersedes all other development practices and style guides for the RSS Feed Reader project.
Amendments require:

1. **Documentation**: Change rationale, affected sections, and migration steps (if any)
2. **Discussion**: At least one team member review and sign-off
3. **Version Bump**: Update CONSTITUTION_VERSION following SemVer rules (see Development Workflow)
4. **Logging**: Record amendment in git commit message and this document's "Last Amended" date

All PRs and code reviews MUST verify compliance with Core Principles and Security Requirements.
Deviations MUST be documented in the PR with explicit justification and team consensus.

**Version**: 1.0.0 | **Ratified**: 2026-07-31 | **Last Amended**: 2026-07-31
