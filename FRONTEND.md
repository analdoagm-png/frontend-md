# FRONTEND.md

**Owner:** Analdo — Senior Product Designer (agentic/AI-backed workflow)  
**Purpose:** Define what “done” looks like when frontend work hands off to a backend team. Any AI agent should be able to use this file to identify the required artifacts, their canonical locations, and the decisions that need explicit agreement.

If you are an agent working in this repo, read **§6, Instructions for Agents**, before starting.

---

## 1. Core Principle

Storybook shows backend engineers *what* the UI must support. It does not define *how* to build the API. Treat it as the front door to a set of versioned integration artifacts—not a substitute for them.

A backend engineer should be able to open a feature folder and determine:

- which endpoints are needed and when they fire;
- the exact request, response, and error shapes;
- every product state the UI handles, including permission and network failures; and
- how to validate a real API against the frontend expectation.

---

## 2. Deliverables Checklist

For every feature that depends on a backend, review every item deliberately. Mark omitted items as `N/A` with a reason in the PR description or feature README.

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

### 2.1 API Contract

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

### 2.2 Typed Frontend API Models

Store models and fetch logic with the feature, for example `src/features/projects/api.ts`. Types must be generated from, or explicitly aligned to, the approved contract. Keep UI-domain transformations separate from raw transport types where that improves readability.

---

### 2.3 MSW Fixtures in Storybook

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

### 2.4 Integration Map

Maintain this table in the feature README so no one has to reverse-engineer component-to-endpoint dependencies.

| Screen / Component | Endpoint | Trigger | Notes |
|---|---|---|---|
| `ProjectList` | `GET /api/projects` | on mount | cursor-paginated, 20/page |
| `ProjectList` search | `GET /api/projects?q=` | debounced input | 300ms debounce; cancellation behavior documented |
| `ProjectCard` delete | `DELETE /api/projects/:id` | confirm modal | optimistic UI; rollback on error |

---

### 2.5 Auth & Permissions Matrix

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

### 2.6 Error, Retry, and Offline Conventions

- Define field-level versus form-level validation display.
- Map stable backend error codes to user-facing copy; do not render backend messages directly unless explicitly reviewed as safe.
- Specify retry behavior, backoff, cancellation, and which actions require user retry.
- Define timeout/offline handling and when loading transitions to an actionable error.
- Preserve user input on recoverable errors and state focus/announcement behavior for errors, loading, and async updates.
- Capture a safe `requestId`/correlation ID in diagnostics when available; do not log secrets or PII.

---

### 2.7 Event / Analytics Schema

Analytics is separate from the API contract. State the destination and owner: client analytics SDK, server-side event pipeline, or both. Never send secrets or unnecessary PII; document consent and retention requirements when they apply.

| Event name | Trigger | Properties | Destination / fires when |
|---|---|---|---|
| `project_viewed` | Detail navigation | `project_id`, `source` | Client analytics; after detail view renders |
| `search_performed` | Debounced search resolves | `query_length`, `result_count` | Client analytics; never send raw query unless approved |

---

### 2.8 Accessibility and Observability Notes

For each integration, document:

- Announcements, focus management, and keyboard behavior for loading, errors, disabled controls, and async completion
- User-safe error copy and its accessible name/description
- Request IDs, relevant client error logging, dashboard/alert ownership, and what constitutes an integration failure

---

### 2.9 Environment and Handoff Notes

- Required environment-variable names only—never values or secrets
- API base URLs per environment (local/staging/production)
- Local steps for mocked and real-backend modes
- Available staging test accounts/roles, provided through an approved secure channel
- Feature flags, API version, and rollout/rollback notes when applicable

---

### 2.10 Acceptance Checklist

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

## 3. Recommended Repo Structure

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

## 4. Why This Matters for a Senior Product Design Role

This is not documentation hygiene alone. These artifacts extend design work into frontend architecture, API semantics, accessibility, observability, and cross-functional delivery. A design system built this way documents **product states**—loading, empty, permission-denied, offline, and error—not merely visual components.

---

## 5. Operating Rules

- The API contract is authoritative for transport shapes; feature documentation is authoritative for UI behavior and product decisions.
- A change to an endpoint, role gate, or analytics event updates its contract and feature documentation in the same PR.
- Use `N/A — <reason>` rather than silently omitting a deliverable.
- Treat a mismatch found in staging/production as a contract issue: record the observed behavior, request ID if safe, affected version, and decision to change frontend or backend.

---

## 6. Instructions for Agents

1. Before a feature starts, review its contract and `src/features/<feature>/README.md`; do not assume stories alone are sufficient.
2. When adding an endpoint dependency, update the feature Integration Map and the API contract in the same PR.
3. When adding a role/permission gate, update the feature Auth & Permissions Matrix and create the corresponding state coverage.
4. Create MSW fixture coverage for populated, empty, loading, error, unauthenticated, and all applicable forbidden/not-found/offline states. Retain mocks after live integration.
5. Never leave an invented backend response shape undocumented. Record the proposed shape in `contracts/` and flag it for confirmation.
6. If omitting a §2 deliverable, add `N/A — <reason>` to the PR or feature README.
7. Update this file and its changelog only when changing cross-feature policy, locations, or required deliverables; keep ordinary feature specifics in the feature README.

---

## 7. Changelog

| Date | Change |
|---|---|
| 2026-07-18 | Clarified canonical artifact locations; strengthened contract, state, auth, accessibility, observability, and acceptance requirements; retained MSW coverage after live integration. |
| 2026-07-17 | Initial version created |
