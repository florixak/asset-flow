# AssetFlow — Design Decisions Log

This document records architectural and domain-modeling decisions made during
Phase 1 (Domain & Architecture), along with the reasoning behind each one.
Format per entry: **Decision**, **Reasoning**, **Alternatives considered**.

Keep this file updated as new decisions are made. If a decision is later
reversed, don't delete the old entry — add a new one and mark the old one as
superseded, so the history of *why* things changed is preserved.

---

## 1. Preventing multiple active `Assignment`s per asset

**Decision:**
Enforce "only one active assignment per asset" using a **partial unique index**
at the database level, combined with a **service-level pre-check** for clean
error messages, unified by a **single global exception handler**.

```sql
CREATE UNIQUE INDEX idx_not_returned_asset
ON assignment (asset_id)
WHERE returned_at IS NULL;
```

- `Assignment.returnedAt = NULL` means the assignment is currently active.
- A plain `UNIQUE(assetId)` was rejected — it would allow only one assignment
  per asset *ever*, blocking reassignment history entirely.
- A `UNIQUE(assetId, returnedAt)` compound constraint was rejected — it fails
  to catch the actual race condition, since Postgres allows unlimited `NULL`
  values in a unique column/columns; two concurrent `NULL` rows for the same
  asset would not violate it.
- A partial index only "watches" rows matching its `WHERE` clause, so returned
  (closed) assignments are invisible to it, while at most one active
  (`returnedAt IS NULL`) row per asset is allowed.

**Why not service-layer check alone:**
A "check if an active assignment exists, then insert" pattern in the service
is vulnerable to a **race condition** (check-then-act): two concurrent
requests can both read "no active assignment" before either has inserted,
and both proceed to insert, corrupting data silently with no error at all.
The DB constraint is checked at actual insert time against the real current
state of the table, so it catches what a point-in-time service check cannot.

**Why not DB constraint alone:**
Relying solely on the DB constraint means every conflict surfaces as a raw
`DataIntegrityViolationException` with no clean business context. A
service-level check (throwing a custom `AssetAlreadyAssignedException`)
handles the normal (non-race) case with a clear, intentional error.

**Exception handling:**
Both exception types — `AssetAlreadyAssignedException` (thrown by the service
check) and `DataIntegrityViolationException` (Spring's translated wrapper
around the Postgres `23505` unique-violation SQLSTATE, thrown when the DB
constraint catches a race) — are handled by the **same** two
`@ExceptionHandler` methods in a single `@RestControllerAdvice`, producing an
identical error response shape. The client cannot (and should not be able to)
tell which path caught the conflict.

**Status:** Decided. Not yet implemented.

---

## 2. `Asset.status` — lifecycle transition validation

**Decision:**
Split enforcement across two layers by *type of rule*:

- **Database:** enforces the *static shape* of the data — `status` is
  `NOT NULL` and constrained to the fixed set of valid enum values via a
  `CHECK` constraint, independent of any Java code:
  ```sql
  status VARCHAR(20) NOT NULL
    CHECK (status IN ('ACTIVE','IN_MAINTENANCE','LOST','RETIRED','DISPOSED'))
  ```
  This protects against writes that bypass the application entirely (manual
  SQL, migrations, other services) — `@Enumerated(EnumType.STRING)` on the
  Java side only protects the path through Hibernate, not direct DB access.

- **Service layer:** enforces *transition legality* (e.g. `RETIRED → ACTIVE`
  is illegal). This cannot live in the database as a simple constraint,
  because a `CHECK` constraint only evaluates the row's value *after* a
  write — it has no access to the *previous* value for comparison. Postgres
  triggers could technically express before/after logic, but were rejected:
  they move business logic out of Java, out of unit tests, and out of Spring
  exception handling, and become harder to maintain as rules grow.

**Where the transition map lives:**
The allowed-transitions map is owned by the `AssetStatus` enum itself (e.g.
`AssetStatus.RETIRED.canTransitionTo(AssetStatus.ACTIVE)`), not by
`AssetService`. Reasoning: the rule is an intrinsic property of the closed
set of statuses, not a concern specific to whichever service happens to use
it — keeping it on the enum avoids duplicating/relocating the rule if other
services (e.g. reporting) need it later.

**Data structure:** `Map<AssetStatus, Set<AssetStatus>>` as a `static final`
field on the enum (it's fixed configuration, not runtime state — favored
over a `switch` statement because adding a new status can't silently fall
through to a `default` branch unnoticed, and the map is trivially iterable
for parametrized tests covering all legal/illegal transition pairs).

**Status:** Decided. Not yet implemented.

---

## 3. `AssetHistory` recording — domain events, not direct calls

**Decision:**
Business services (`AssignmentService`, `MaintenanceService`, `AssetService`,
etc.) publish domain events (e.g. `AssetAssignedEvent`, `AssetStatusChangedEvent`)
via Spring's `ApplicationEventPublisher` instead of calling
`AssetHistoryRepository.save(...)` directly. A single `AssetHistoryListener`
(`@EventListener` / `@TransactionalEventListener`) listens for all such
events and is solely responsible for writing `AssetHistory` rows.

**Reasoning:**
- **Avoids duplication:** without events, every service that can change
  asset state (6+ so far) would repeat the same "build and save a history
  row" logic.
- **Avoids silent omission:** a new feature/service can easily forget to
  write a history entry if it's a manual step in each service; with events,
  history-writing is centralized and not something each new service author
  needs to remember.
- **Loose coupling:** business services depend only on the generic
  `ApplicationEventPublisher`, not on `AssetHistoryRepository` or the
  `AssetHistory` entity at all — they don't need to know the history
  subsystem exists.
- **Test isolation (single responsibility per test):** `AssignmentServiceTest`
  only needs to verify that the correct event was published with the correct
  data — it does not need to know or verify the shape of an `AssetHistory`
  row. Verifying that a published event correctly produces a history row is
  a separate, independent `AssetHistoryListenerTest`. Without events, a
  single test class would be forced to verify both the business operation
  *and* the history side-effect together, and a future change to
  `AssetHistory`'s shape would require touching tests in every service that
  writes history, instead of just one listener test.

**Status:** Decided. Not yet implemented.

---

## 4. `Location` — single self-referential entity, not `Location`/`Room` split

**Decision:**
Model `Location` as a single, self-referential table instead of two separate
entities (`Location` + `Room`), to support arbitrary nesting depth (matches
the doc's own example of `Headquarters → Building → Room`, and leaves room
for intermediate levels like `Floor` without a schema change).

```sql
CREATE TABLE location (
    id               UUID PRIMARY KEY,
    organization_id  UUID NOT NULL REFERENCES organization(id),
    parent_id        UUID REFERENCES location(id),
    location_type_id UUID REFERENCES location_type(id), -- nullable
    name             VARCHAR(255) NOT NULL,
    description      TEXT
);
```

**`location_type` as a lookup table, not an enum + CHECK:**
Unlike `AssetStatus` (small, stable, code-coupled set of values — enum +
`CHECK` constraint), `LocationType` (BUILDING, ROOM, WAREHOUSE, CAMPUS, ...)
is expected to be extended more casually over time. A lookup table lets a new
type be added via a plain `INSERT`, with no `ALTER TABLE` / migration
required — the right trade-off when the set of values is expected to grow
without needing corresponding code changes, unlike `AssetStatus` where the
values are tightly bound to business logic in code.

**`organization_id` denormalized on every row, not derived by walking `parentId`:**
Even though `organizationId` could in theory be computed by walking up the
`parentId` chain to the root, it is stored directly on every `Location` row
instead. Reasoning: it's needed extremely frequently (e.g. "show me all
assets in my organization" style queries), it enables straightforward
indexing/filtering, and it protects against a broken/missing `parentId`
link accidentally orphaning a subtree from its organization. This is a
deliberate **denormalization for read performance and data integrity**, not
an oversight — the small duplication cost is worth avoiding a recursive
lookup (and its indexing complications) on every query.

**Preventing cycles in `parentId` — service layer, not a DB constraint:**
Same reasoning pattern as `AssetStatus` transitions: a `CHECK` constraint
evaluates a single row in isolation and cannot inspect other rows or walk a
chain of ancestors, so cycle prevention cannot be expressed as a static DB
constraint (a trigger could technically do it, but is rejected for the same
maintainability reasons as before). Before assigning `nodeId`'s parent to
`newParentId`, the service walks *upward* from `newParentId` toward the root
and rejects the change if `nodeId` appears anywhere in that ancestor chain
(that would make `nodeId` its own ancestor). Because each node has exactly
one parent (a tree, not a general graph), this is a simple linear walk, not
a full DFS with a visited-set.

To avoid N sequential `SELECT` queries (one per level of depth) when doing
this walk, use a single **recursive CTE** query walking upward via
`parent_id`, mirroring the same tool used for walking a subtree *downward*
(e.g. "all locations under Headquarters"):

```sql
WITH RECURSIVE ancestors AS (
    SELECT id, parent_id FROM location WHERE id = :newParentId
    UNION ALL
    SELECT l.id, l.parent_id
    FROM location l
    JOIN ancestors a ON l.id = a.parent_id
)
SELECT id FROM ancestors;
```
If `nodeId` appears in the result set, the reparenting is rejected.

**Status:** Decided. Not yet implemented.

---

## 5. Organization data isolation — repository-level filtering, not RLS

**Decision:**
Enforce organization isolation via explicit `...AndOrganizationId` repository
query methods (e.g. `findByIdAndOrganizationId(id, organizationId)`),
applied consistently across every repository, rather than Postgres
Row-Level Security (RLS).

**The risk being prevented (IDOR):**
Without organization scoping, an endpoint like `findById(id)` would let any
authenticated user fetch any record by ID regardless of which organization
owns it — a textbook **Insecure Direct Object Reference (IDOR)**
vulnerability. Authorization must check *ownership*, not just
*authentication*.

**Alternatives considered:**

- **Controller/service-level check after fetch** (`if (asset.getOrgId() !=
  currentOrgId) throw Forbidden`) — rejected as the primary mechanism: it
  requires a manual check to be remembered on every single endpoint/service
  method, the same structural weakness identified for `AssetHistory`
  (easy to forget on a newly added endpoint, and here the cost of forgetting
  is a cross-tenant data leak rather than a missing log entry).
- **Postgres Row-Level Security (RLS)** — a policy like
  `USING (organization_id = current_setting('app.current_org_id'))` would
  enforce isolation automatically at the database level, even protecting
  against a hand-written native `@Query` that forgets to filter by
  organization (a gap `findByIdAndOrganizationId()` alone does not cover).
  However, it was **considered and deliberately rejected** for this project:
  - It requires a `SET app.current_org_id = ...` command to be issued
    reliably at the start of every request (e.g. via a Servlet `Filter`),
    since the setting lives on the DB *session/connection*, not the Java
    `HttpRequest`.
  - Because Spring Boot uses connection pooling (HikariCP), a DB connection
    is reused across unrelated requests. If the per-request `SET` is ever
    skipped or not properly cleared, a connection can carry a **stale
    `current_org_id` from a previous, different tenant's request** into the
    next one — reproducing the exact same IDOR risk this mechanism was
    meant to prevent, but via a subtler, infrastructure-level bug instead of
    a missing application-level check.
  - For a solo portfolio project, RLS adds meaningful operational complexity
    (correct session variable lifecycle management, risk of a false sense
    of security if misconfigured) without a proportionate reduction in risk,
    given that `findByIdAndOrganizationId()` applied consistently already
    covers the realistic risk surface. RLS remains a valid, stronger option
    for a larger/team project and was evaluated on its merits, not simply
    unconsidered.

**Consistency safeguard:**
- A shared `OrganizationScopedRepository<T, ID> extends JpaRepository<T, ID>`
  interface defines the standard `findByIdAndOrganizationId` /
  `findAllByOrganizationId` methods once, so every entity repository gets a
  consistent, correctly-scoped API by extension rather than reinventing it.
- Enforced via an **ArchUnit** test in CI that fails the build if service
  code calls unscoped `findById`/`findAll` directly on a repository, rather
  than relying purely on code review discipline to catch it.
- Any custom native `@Query` is treated as requiring explicit review for an
  `organization_id` filter, since ArchUnit can catch missing scoping methods
  but not silently-wrong SQL inside a `@Query` string.

**Status:** Decided. Not yet implemented.

---

## 6. Authentication mechanism — stateless JWT in an HttpOnly cookie

**Decision:**
Use stateless JWT (short-lived access token + longer-lived refresh token)
rather than server-side sessions, with the access token delivered to the
browser as an **HttpOnly, Secure, SameSite=Strict** cookie rather than
stored in `localStorage` and manually attached via an `Authorization:
Bearer` header from JavaScript.

**Why stateless JWT over server-side sessions:**
A server-side session requires the server to hold per-user state, which
becomes a problem once there is more than one backend instance (e.g. behind
a load balancer, or multiple Docker containers) — sessions would need to be
shared (e.g. via Redis) or pinned to a specific instance (sticky sessions).
A stateless JWT can be verified by any instance independently, with no
shared session store required, at the cost of not being able to unilaterally
revoke a token before it naturally expires (mitigated by keeping the access
token's lifetime short and relying on a refresh token for renewal).

**Why HttpOnly cookie over `localStorage`:**
`localStorage` is readable by any JavaScript running on the page, including
malicious script injected via an XSS vulnerability (e.g. unescaped
user-generated content) — such a script can trivially exfiltrate a token
stored there. An `HttpOnly` cookie is invisible to JavaScript entirely
(`document.cookie` cannot see it), so an XSS vulnerability alone is not
sufficient to steal the token.

**CSRF risk introduced by cookies, and its mitigation:**
Unlike a manually-attached `Authorization` header, cookies are attached
*automatically* by the browser to every request targeting the cookie's
domain, **regardless of which site initiated the request** — the browser
only checks the destination domain, not the origin of the request. Without
mitigation, a malicious site could trigger a request (e.g. a hidden form or
background `fetch`) against the AssetFlow API, and the browser would
faithfully attach the user's valid session cookie — a Cross-Site Request
Forgery (CSRF) attack requiring no knowledge of the token itself.

Mitigated via the cookie's `SameSite` attribute set to `Strict`: the cookie
is never sent on any cross-site-initiated request, including a user
clicking a link to the app from an external site or email (which `Lax`
would still permit for safe navigation). `Strict` was chosen over `Lax`
specifically because AssetFlow is an internal organizational tool, not a
consumer-facing product where users are expected to arrive via external
links/emails and expect to be immediately authenticated — for that use case,
prioritizing CSRF protection over that convenience is the right trade-off.
`Secure` ensures the cookie is only ever transmitted over HTTPS.

**Status:** Decided. Not yet implemented.

---

## 7. Maintenance workflow — synchronous service call, not a domain event

**Decision:**
Starting/completing maintenance updates both `MaintenanceRecord.status` and
`Asset.status` within a single `@Transactional` service method
(`MaintenanceService.startMaintenance()` calls `AssetService.changeStatus()`
directly), rather than via a domain event/listener as used for
`AssetHistory` (Decision 3).

**Why `@Transactional`, and specifically why atomicity matters here:**
Without wrapping both updates in one transaction, a failure between the two
writes (e.g. an exception validating the `Asset` status transition, or a
crash) would leave `MaintenanceRecord.status = IN_PROGRESS` while
`Asset.status` remains unchanged — e.g. still `ACTIVE` — producing a
contradictory state (asset appears available while maintenance is actively
being performed on it). `@Transactional` ensures atomicity: if any part of
the method fails, all writes performed so far in that method are rolled
back, not just prevented going forward.

Note: Spring's default rollback behavior only triggers automatically for
unchecked `RuntimeException`/`Error` — a checked exception (e.g. an
`IOException` or `SQLException`) is, by default, still committed unless
`@Transactional(rollbackFor = ...)` explicitly requests otherwise. Any
custom exceptions used here that must trigger a rollback should either
extend `RuntimeException` or be explicitly listed in `rollbackFor`.

**Why direct service call instead of a domain event (contrast with `AssetHistory`):**
The distinguishing question is whether the second effect is a *core part of
the same business invariant* or an *independent, optional side-effect*:
- `AssetHistory` (Decision 3): recording history is observability — if it
  were delayed or failed independently, the primary operation (e.g. an
  assignment) would still be valid and meaningful on its own. Appropriate
  for a domain event.
- `Asset.status` changing when maintenance starts: this is **not optional**
  — `MaintenanceRecord.status = IN_PROGRESS` without a corresponding
  `Asset.status = IN_MAINTENANCE` is an inconsistent, meaningless state (an
  asset undergoing maintenance must not appear available for assignment).
  Both writes must succeed or fail together, within the same transaction
  boundary — which requires a direct, synchronous call, not an
  asynchronous/decoupled event.

**No duplicated transition validation in `MaintenanceService`:**
`MaintenanceService.startMaintenance()` does not pre-check whether the
asset's current status legally permits a transition to `IN_MAINTENANCE`.
It simply calls `AssetService.changeStatus(assetId, IN_MAINTENANCE)` and
lets `InvalidStatusTransitionException` propagate (triggering the
transaction rollback) if the transition is illegal (e.g. asset is
`RETIRED`/`DISPOSED`). Duplicating that check in `MaintenanceService` would
create a second place holding knowledge that should live solely in
`AssetStatus` (Decision 2) — the same single-source-of-truth principle
applied there.

**Status:** Decided. Not yet implemented.

---

## 8. Optimistic locking — `@Version` on mutable entities (promoted from optional to Phase 3 baseline)

**Decision:**
Add a `@Version` (integer/long) column to entities subject to concurrent
updates — starting with `Asset` — rather than treating optimistic locking as
an optional advanced feature (as originally listed in Section 16). Promoted
to a Phase 3 baseline concern given the concurrency issues already
identified in this project (e.g. two managers concurrently updating the
same `Asset`'s location and status).

**The problem — lost update, a distinct category from Decision 1's race condition:**
Two concurrent requests reading the same `Asset` row, each independently
passing their own status-transition validation, then both issuing an
`UPDATE` — the second `UPDATE` silently overwrites the first with no error
at all, since a plain SQL `UPDATE` has no built-in way to detect that the
row changed since it was read. This is the **lost update problem**, and is
a different category of concurrency issue from the `Assignment` uniqueness
race condition (Decision 1): that was two *INSERTs* competing over
row-existence (solved by a partial unique index), whereas this is two
*UPDATEs* competing over the current value of an *existing* row. A
`@Version` column does not help with the `Assignment` case at all — an
`INSERT` creates a new row with a new primary key and has no "previous
version" to compare against. Each tool addresses a distinct concurrency
category; both are needed.

**Why an incrementing integer, not a timestamp:**
A timestamp-based check was considered and rejected: timestamps have finite
resolution, so two updates close enough in time could receive an identical
value and a conflict could go undetected; timestamps are also vulnerable to
clock drift across multiple backend instances (relevant given the
stateless, horizontally-scalable design in Decision 6). A plain
auto-incrementing integer `version` column has no such ambiguity — it
increases by exactly 1 on every update, deterministically, independent of
system clocks.

**Mechanism:**
Hibernate appends the version to the `UPDATE`'s `WHERE` clause (e.g.
`UPDATE asset SET ..., version = 4 WHERE id = 5 AND version = 3`). If
another transaction already advanced the version, this `UPDATE` affects
zero rows; Hibernate detects the row-count mismatch (1 expected, 0 actual)
and throws `OptimisticLockException` (wrapped by Spring as
`ObjectOptimisticLockingFailureException`). This exception is handled by
the same shared `@RestControllerAdvice` mechanism established in Decision 1,
via its own `@ExceptionHandler`, returning a clear "this record was
modified by someone else, please reload and retry" response rather than a
raw error.

**Status:** Decided. Not yet implemented.

---
