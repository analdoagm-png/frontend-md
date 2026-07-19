# FRONTEND.md

**Owner:** Analdo — Senior Product Designer (agentic/AI-backed workflow)  
**Purpose:** Audit the health of a repository’s frontend-to-backend integration. Any AI agent should be able to inspect the available evidence, identify risks and missing capabilities, recommend practical improvements, and give backend teams an actionable list of gaps the product designer cannot resolve alone.

If you are an agent working in this repo, read **§8, Instructions for Agents**, before starting.

---

## 1. Core Principle

Storybook shows backend engineers *what* the UI must support. It does not define *how* to build the API. Treat it as one source of audit evidence alongside runtime code, contracts, tests, fixtures, and environment documentation—not a substitute for them.

A backend engineer should be able to open a feature folder and determine:

- which endpoints are needed and when they fire;
- the exact request, response, and error shapes;
- every product state the UI handles, including permission and network failures; and
- how to validate a real API against the frontend expectation.

---

## 2. Audit Workflow

Audit the repository that is actually present. Do not award a passing result because an ideal artifact *could* exist, and do not guess behavior that is not evidenced in code, documentation, fixtures, or a verified running environment.

1. **Set the scope.** Identify the product surfaces, feature folders, API clients, contracts, Storybook/MSW setup, tests, environment files, and relevant CI configuration. State what was unavailable or out of scope.
2. **Trace real integrations.** For each meaningful user flow, map screen/component → client call → endpoint/operation → response model → UI state. Search for direct calls as well as shared clients, generated clients, GraphQL operations, and server-state libraries.
3. **Assess the audit domains in §3.** Assign each domain one result: **Verified**, **Partial**, **Missing**, **Not applicable**, or **Not verifiable**. “Not verifiable” means the repository does not provide enough evidence; it is not a pass.
4. **Validate behavior when feasible.** Run existing tests, type checks, Storybook, or documented local flows. Report commands that could not run and why. Never treat a passing mock-only state as proof that a real API matches it.
5. **Publish an evidence-based report.** If repository writes are in scope, use `docs/frontend-backend-audits/YYYY-MM-DD.md`; otherwise return the same report in the audit response. Every finding should cite a file/path, command result, observed state, or explicitly state that the evidence is absent.

### Finding format

| Priority | Finding | Evidence | Product / delivery impact | Recommended next step | Owner | Verification |
|---|---|---|---|---|---|---|
| P1 | Project deletion has no documented error shape | `src/features/projects/api.ts`; no contract found | UI cannot reliably explain failures or rollback | Define the error envelope and add 403/500 fixtures | Backend + frontend | Contract test and Storybook states pass |

- **P0** — security, data loss, privacy, or a release-blocking integration failure
- **P1** — a material product state, contract, authorization, or reliability gap likely to break users or delivery
- **P2** — an important maintainability, testability, accessibility, or observability weakness
- **P3** — a low-risk improvement or documentation refinement

Recommendations must be specific, testable, and assigned to **Product Design**, **Frontend**, **Backend**, or **Shared**. Do not label a finding “backend” without explaining the missing capability, its impact, and the decision or artifact needed.

---

## 3. Audit Domains and Expected Artifacts

For every backend-dependent feature, assess every item deliberately. Record the audit result and evidence; mark omitted items as `N/A` with a reason in the report or feature README.

- [ ] **API contract** (OpenAPI or concise endpoint document)
- [ ] **Typed frontend API models** generated from, or validated against, that contract
- [ ] **MSW fixtures and Storybook states**
- [ ] **Integration map**
- [ ] **Auth and permissions matrix**
- [ ] **Error, retry, and offline conventions**
- [ ] **Event / analytics schema**
- [ ] **Accessibility and observability notes**
- [ ] **Environment and handoff notes**
- [ ] **Integration acceptance checklist**

### Canonical locations

Each backend-backed feature owns its integration documentation at `src/features/<feature>/README.md`. Put its `api.ts`, `handlers.ts`, fixtures, and stories in that same feature folder. The machine-readable API contract lives in `contracts/<feature>.openapi.yaml`; its README may contain the human-readable fallback. Root `mocks/` only contains shared MSW setup and truly cross-feature handlers.

---

### 3.1 API Contract

The API contract is the source of truth for request and response semantics. Store it in `contracts/` and link to it from the feature README.

For every endpoint, include:

- Path, method, API version, auth requirement, and required headers
- Request parameters/body, defaults, validation rules, and idempotency behavior where relevant
- Success response and every supported error response, including a stable machine-readable error `code`
- Pagination, sorting, filtering, and caching semantics
- Status codes and the condition that triggers each one
- Field conventions: nullability, date/time timezone/format, IDs, money, locale, and enums
- Deprecation/migration plan for a breaking change, when applicable

Use OpenAPI YAML/JSON whenever the project can support it. Otherwise, use a markdown table as a temporary minimum and record the planned owner/date for formalization. Prefer generated TypeScript types and clients from OpenAPI. If generation is not available, add a contract-validation test so handwritten types cannot silently drift.

Define a consistent error envelope. For example:

```ts
type ApiError = {
  code: string;          // stable, machine-readable
  message?: string;      // safe diagnostic text; never render directly by default
  fieldErrors?: Record<string, string[]>;
  requestId?: string;
};
```

---

### 3.2 Typed Frontend API Models

Store models and fetch logic with the feature, for example `src/features/projects/api.ts`. Types must be generated from, or explicitly aligned to, the approved contract. Keep UI-domain transformations separate from raw transport types where that improves readability.

---

### 3.3 MSW Fixtures in Storybook

Every meaningful backend outcome gets a deterministic Storybook story and MSW handler. Minimum set per feature:

- Populated/default
- Empty
- Loading, with an artificial delay
- Validation error and unexpected server error (`500`)
- Unauthenticated (`401`) and forbidden (`403`), where the feature is role-gated
- Not found (`404`), when users can navigate directly to a resource
- Offline/network failure and retry behavior, where applicable
- Mutation pending, success, and rollback behavior, where applicable
- A feature-relevant edge case (long content, partial data, pagination boundary, etc.)

Fixtures should be realistic enough to serve as contract examples, but must not contain production data or secrets. **Keep MSW handlers after the real endpoint ships**: they support Storybook, deterministic tests, offline development, and regression coverage. Update and label fixtures with the contract version instead of removing them.

---

### 3.4 Integration Map

Maintain this table in the feature README so no one has to reverse-engineer component-to-endpoint dependencies.

| Screen / Component | Endpoint | Trigger | Notes |
|---|---|---|---|
| `ProjectList` | `GET /api/projects` | on mount | cursor-paginated, 20/page |
| `ProjectList` search | `GET /api/projects?q=` | debounced input | 300ms debounce; cancellation behavior documented |
| `ProjectCard` delete | `DELETE /api/projects/:id` | confirm modal | optimistic UI; rollback on error |

---

### 3.5 Auth & Permissions Matrix

Define what is hidden, disabled, or server-rejected for each role. Record it in the feature README.

| Role / state | Can view | Can edit | Can delete | UI treatment when denied |
|---|---|---|---|---|
| Admin | ✓ | ✓ | ✓ | — |
| Editor | ✓ | ✓ | ✗ | Delete control hidden |
| Viewer | ✓ | ✗ | ✗ | Edit control disabled; explanation available to assistive tech |
| Expired session | ✗ | ✗ | ✗ | Redirect to login; preserve return URL (`401`) |
| Authenticated but unauthorized | feature-specific | ✗ | ✗ | Explain or remove action; server rejects action (`403`) |

Be explicit about **hidden**, **disabled**, and **rejected**. Also distinguish `401` (authentication required/expired), `403` (authenticated but forbidden), and `404` used to avoid disclosing inaccessible resources. The server remains authoritative even when the UI hides a control.

---

### 3.6 Error, Retry, and Offline Conventions

- Define field-level versus form-level validation display.
- Map stable backend error codes to user-facing copy; do not render backend messages directly unless explicitly reviewed as safe.
- Specify retry behavior, backoff, cancellation, and which actions require user retry.
- Define timeout/offline handling and when loading transitions to an actionable error.
- Preserve user input on recoverable errors and state focus/announcement behavior for errors, loading, and async updates.
- Capture a safe `requestId`/correlation ID in diagnostics when available; do not log secrets or PII.

---

### 3.7 Event / Analytics Schema

Analytics is separate from the API contract. State the destination and owner: client analytics SDK, server-side event pipeline, or both. Never send secrets or unnecessary PII; document consent and retention requirements when they apply.

| Event name | Trigger | Properties | Destination / fires when |
|---|---|---|---|
| `project_viewed` | Detail navigation | `project_id`, `source` | Client analytics; after detail view renders |
| `search_performed` | Debounced search resolves | `query_length`, `result_count` | Client analytics; never send raw query unless approved |

---

### 3.8 Accessibility and Observability Notes

For each integration, document:

- Announcements, focus management, and keyboard behavior for loading, errors, disabled controls, and async completion
- User-safe error copy and its accessible name/description
- Request IDs, relevant client error logging, dashboard/alert ownership, and what constitutes an integration failure

---

### 3.9 Environment and Handoff Notes

- Required environment-variable names only—never values or secrets
- API base URLs per environment (local/staging/production)
- Local steps for mocked and real-backend modes
- Available staging test accounts/roles, provided through an approved secure channel
- Feature flags, API version, and rollout/rollback notes when applicable

---

### 3.10 Acceptance Checklist

Copy this block into the feature PR or README when connecting a real endpoint:

```text
- [ ] Contract is linked, versioned, and response types are generated from or validated against it
- [ ] Real API reproduces success, empty, loading, error, unauthenticated, and applicable forbidden/not-found/offline states
- [ ] Pagination, filtering, sorting, caching, and cancellation behave as documented
- [ ] Authentication and permission behavior matches the matrix; the server enforces authorization
- [ ] Error codes map to approved frontend copy; retry and rollback behavior works
- [ ] Accessibility behavior for async states is verified
- [ ] Analytics events contain only approved properties and reach the intended destination
- [ ] Safe diagnostics include the expected request/correlation ID where available
- [ ] MSW fixtures remain available and match the current contract version
- [ ] Any omitted deliverable is marked N/A with a reason
```

---

### 3.11 Backend Guidance Register

When an audit finds a gap that Product Design or Frontend cannot resolve from the repository, add it to this register. Its purpose is to make backend teams aware of what is missing and give them enough context to either address it or guide the designer toward a workable solution.

Use it for missing or undocumented endpoints, response semantics, authorization rules, error codes, rate limits, caching, server-owned analytics, staging access, and observability. Do **not** use it for a purely frontend implementation task that the repository already enables.

| Area lacking | Evidence and impact | Backend action or decision needed | What the designer/frontend needs | Owner / status |
|---|---|---|---|---|
| Contract semantics | `Project.status` is rendered but its allowed values and null behavior are undocumented | Publish or confirm enum values, nullability, and a versioned contract | Approved labels and fallback behavior | Backend — open |
| Authorization model | UI hides delete, but no source defines whether the server returns 403 or 404 | Confirm enforcement and resource-disclosure policy | Correct denied state and test fixtures | Backend + security — open |
| Diagnostics | API errors expose no safe correlation ID | Return or document a request ID in error responses | Supportable error state and bug-report path | Backend — open |

Phrase each item as a request that can be answered: **what is missing, why it matters, what decision or artifact is needed, and how the answer will be verified**. Backend may supply the capability, name the owner, document an existing behavior, or propose an alternative. Record that response and update the audit finding rather than leaving a permanent workaround.

---

## 4. Product Design Response Plan

Work through audit findings in priority order. Do not start lower-priority polish while a higher-priority finding leaves users unsafe, blocks delivery, or makes product behavior unknown.

### P0 — Contain user harm and block unsafe release

1. Confirm the evidence, affected flow, users, environments, and severity with Frontend and Backend.
2. Stop or constrain the unsafe path: remove a harmful action, gate a feature, provide a safe fallback, or pause release. Do not design around a security, privacy, data-loss, or authorization failure as if it were cosmetic.
3. Open a Shared incident/task with a named decision-maker and a verification condition.
4. Add any server-side dependency to the Backend Guidance Register, then update UI states, copy, and test/Storybook coverage once the decision is made.
5. Close only after the real integration is verified—not merely a mock or design prototype.

### P1 — Resolve product and contract ambiguity

1. Reproduce or trace the flow and write the exact user-facing failure: trigger, expected result, actual/unknown result, and impact.
2. Specify the intended product behavior: success, loading, empty, denied, validation, error, offline, and recovery states as applicable.
3. Identify the owner. Product Design owns experience and decision framing; Frontend owns client implementation; Backend owns server behavior and semantics; use Shared when the contract requires both.
4. Update feature documentation, interaction states, acceptance criteria, and fixtures. Add unresolved server behavior to the Backend Guidance Register with the required decision or artifact.
5. Agree on a verification method—contract test, staging scenario, API response sample, Storybook state, or accessibility check—and retest before closing.

### P2 — Strengthen reliability, accessibility, and maintainability

1. Group related findings into a small improvement slice rather than scattering isolated cleanup tasks.
2. Prioritize improvements that prevent recurring support work: typed/validated contracts, realistic fixtures, accessible async feedback, correlation IDs, and clear ownership.
3. Add the missing documentation or test coverage, and ensure the change has an owner and definition of done.
4. Schedule it in the next appropriate iteration; escalate to P1 if evidence shows active user or release impact.

### P3 — Capture low-risk refinements

1. Record the improvement with evidence and the smallest useful recommendation.
2. Bundle it with nearby feature work, documentation maintenance, or a planned audit cycle.
3. Do not let P3 items obscure unresolved P0–P2 findings in the audit summary.

### Product Designer closure checklist

- [ ] The finding has evidence, priority, owner, and user/delivery impact.
- [ ] The intended UI behavior and acceptance criteria are documented.
- [ ] Backend-owned unknowns appear in the Backend Guidance Register rather than in an invented design assumption.
- [ ] The chosen implementation/decision is reflected in contracts, states, and fixtures where applicable.
- [ ] The finding was verified in the agreed environment and its audit status was updated.

---

## 5. Recommended Repo Structure

```text
contracts/
├── projects.openapi.yaml
└── projects.README.md

src/
├── components/                 # Pure UI; no feature API knowledge
│   ├── button/
│   ├── header/
│   └── footer/
├── features/
│   ├── projects/
│   │   ├── README.md           # integration map, auth matrix, handoff notes
│   │   ├── api.ts              # contract-aligned types, client, transforms
│   │   ├── fixtures.ts         # version-labelled mock data
│   │   ├── handlers.ts         # feature MSW handlers
│   │   └── ProjectList.stories.tsx
│   └── profile/
└── mocks/
    ├── browser.ts              # shared MSW setup
    └── handlers.ts             # shared handlers only
```

Feature-owned mocks, fixtures, stories, models, and documentation should evolve together. A real backend connection changes the runtime client configuration; it does not remove the feature’s test and Storybook contract coverage.

---

## 6. Why This Matters for a Senior Product Design Role

This is not documentation hygiene alone. These artifacts extend design work into frontend architecture, API semantics, accessibility, observability, and cross-functional delivery. A design system built this way documents **product states**—loading, empty, permission-denied, offline, and error—not merely visual components.

---

## 7. Operating Rules

- The API contract is authoritative for transport shapes; feature documentation is authoritative for UI behavior and product decisions.
- A change to an endpoint, role gate, or analytics event updates its contract and feature documentation in the same PR.
- Use `N/A — <reason>` rather than silently omitting a deliverable.
- Treat a mismatch found in staging/production as a contract issue: record the observed behavior, request ID if safe, affected version, and decision to change frontend or backend.
- An audit result must cite evidence. Absence of a contract, test, fixture, or documented behavior is a finding or `Not verifiable`, not an assumption that it exists.
- Product Design should resolve UI and product-decision gaps it owns; move backend-owned unknowns to the Backend Guidance Register rather than inventing an API behavior.

---

## 8. Instructions for Agents

1. Start with §2. Inventory the repository and state the audit scope and missing evidence before making conclusions.
2. Trace each material user flow from UI to its backend dependency; do not assume Storybook stories alone represent runtime behavior.
3. Assess every §3 domain as Verified, Partial, Missing, Not applicable, or Not verifiable, with evidence and priority for each gap.
4. Recommend concrete fixes with an owner and a verification method. Separate Product Design, Frontend, Backend, and Shared work.
5. When required backend behavior is absent, undocumented, inaccessible, or requires a server-side decision, add an item to the §3.11 Backend Guidance Register. Do not invent a response shape to close the finding.
6. When adding an endpoint dependency, update the feature Integration Map and the API contract in the same PR.
7. When adding a role/permission gate, update the feature Auth & Permissions Matrix and create the corresponding state coverage.
8. Create MSW fixture coverage for populated, empty, loading, error, unauthenticated, and all applicable forbidden/not-found/offline states. Retain mocks after live integration.
9. If omitting a §3 deliverable, add `N/A — <reason>` to the report, PR, or feature README.
10. Update this file and its changelog only when changing cross-feature audit policy, locations, or required deliverables; keep ordinary feature specifics in the feature README.

---

## 9. Changelog

| Date | Change |
|---|---|
| 2026-07-18 | Added a priority-ordered Product Design Response Plan for turning audit findings into containment, resolution, improvement, and verification work. |
| 2026-07-18 | Reframed the playbook as an evidence-based repository audit, added finding priorities and ownership, and added the Backend Guidance Register for gaps Product Design cannot resolve alone. |
| 2026-07-18 | Clarified canonical artifact locations; strengthened contract, state, auth, accessibility, observability, and acceptance requirements; retained MSW coverage after live integration. |
| 2026-07-17 | Initial version created |
