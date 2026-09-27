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

**Refresh token persistence — a database table, not another stateless JWT:**
Unlike the short-lived access token, the refresh token has a much longer
lifetime (days/weeks), so it needs a way to be revoked before its natural
expiration — e.g. on logout, or when an admin disables a user's account.
This requires a persistent `refresh_token` table:

```
REFRESH_TOKENS { id, user_id, organization_id, token_hash, revoked, expires_at, created_at }
```

**Why `token_hash`, not the raw token value:** storing the raw refresh
token would mean that anyone who gains read access to this table (an
attacker, or even just database access) could impersonate any user by
presenting their stored token directly — no decryption needed. Instead, a
deterministic hash (e.g. SHA-256) of the token is stored; verifying an
incoming token means hashing it the same way and comparing against the
stored hash, without ever needing to store or recover the original value.
SHA-256 (fast) is appropriate here — unlike password hashing (BCrypt/Argon2,
deliberately slow to resist brute-force), a refresh token is a
high-entropy, randomly generated value, not a human-chosen secret, so
brute-force resistance via slow hashing is not the relevant concern.

**Per-device session metadata (`ip_address`, `user_agent`) — considered and
explicitly excluded from scope:** enabling a "manage your active devices,
sign out a specific device" UI is a legitimate feature, but was deliberately
left out as feature-creep relative to this project's stated depth-over-breadth
goal (Section 20) — it doesn't reinforce the core skills (Spring Security,
JPA, REST API design) this project exists to demonstrate.

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

## 9. Supporting entities — `User.role`, `AssetType`, `MaintenanceRecord.status`

**`User.role` — enum + CHECK, same pattern as `AssetStatus` (Decision 2):**
Roles (`ADMIN`, `MANAGER`, `EMPLOYEE`) are tightly coupled to the permission
matrix defined in code (Section 6) — adding a new role always requires a
corresponding code change to actually grant it meaning, unlike
`LocationType`/`AssetType` which can be extended with a plain data insert.
Same reasoning as `AssetStatus`: enum on the Java side, `CHECK` constraint
on the DB column for defense against writes that bypass the application.

**`AssetType` — global lookup table, not organization-scoped:**
Unlike `Location`/`Asset`/`Assignment` (Decision 5), `AssetType` is a
**global, shared lookup table with no `organizationId`**, following the
same pattern as `LocationType` (Decision 4). The distinguishing question
from the IDOR risk in Decision 5: Decision 5 protects against a user
accessing another organization's *specific data records*. Here, the concern
is different — knowing that a category like "Laptop" *exists* as a type is
not sensitive information, comparable to how any organization can see the
full set of `AssetStatus` values without that being a data leak. Making
`AssetType` organization-scoped would additionally force every organization
wanting a "Laptop" type to create its own duplicate row, instead of sharing
one global definition.

**`MaintenanceRecord.status` — enum + CHECK, but simple ordinal comparison
instead of a transition map:**
The `Reported → IN_PROGRESS → COMPLETED` lifecycle is strictly linear (no
branching, unlike `AssetStatus`), so a full `Map<Status, Set<Status>>`
transition table (Decision 2) would be unnecessary complexity for this
entity. A simple forward-only check (e.g. comparing enum ordinals, rejecting
any transition that would move backward) is sufficient. Still backed by a
DB-level `CHECK` constraint on valid values, same as any other status enum.

**Status:** Decided. Not yet implemented.

## 10. `Asset.locationId` — nullable

**Decision:** `Asset.locationId` is nullable. An asset can exist in the
system (tracked, evidenced) before being physically assigned to a location —
e.g. newly received inventory awaiting placement. Not a diagram default;
deliberately chosen to reflect a real intermediate state in the asset
lifecycle.

**Status:** Decided. Not yet implemented.

## 11. API response shape — custom paged DTO, no wrapper on single-resource responses

**Decision:**
- **List/paginated endpoints** (e.g. `GET /api/v1/assets`) return a custom
  `PagedResponse<T>` DTO — `{ "data": [...], "meta": { totalElements,
  totalPages, number, size } }` — rather than serializing Spring Data's
  `Page<T>` directly.
- **Single-resource endpoints** (e.g. `GET /api/v1/assets/{id}`) return the
  resource object directly, with no wrapping envelope.

**Why not serialize `Page<T>` directly:**
`Page<T>`'s JSON shape is a Spring Data framework implementation detail, not
a contract owned by this API. Serializing it directly ties the API's public
response shape to whatever Spring Data JPA happens to produce in a given
version — a framework upgrade changing that internal shape would silently
break frontend clients relying on it. Defining an owned `PagedResponse<T>`
DTO, populated from `Page<T>` internally, keeps the public contract stable
and independent of the persistence framework's version.

**Why not a `{ "data": ... }` wrapper on single-resource responses:**
Considered wrapping every response (list or single) in the same `data`
envelope for structural consistency, but rejected: a TypeScript/frontend
client already knows, from which endpoint it called, whether to expect a
single object or a list — the wrapper adds no information the client
doesn't already have from context, so it would be a cosmetic-only envelope
with no practical benefit. Reserved the envelope specifically for where it
carries real, necessary information (pagination metadata).

**Status:** Decided. Not yet implemented.

## 12. Dynamic filtering — `JpaSpecificationExecutor`, entity-owned Specifications

**Decision:** `AssetRepository extends JpaSpecificationExecutor<Asset>`, with
a dedicated `AssetSpecifications` utility class (static methods like
`hasStatus(...)`, `hasType(...)`, `hasLocation(...)`, `nameContains(...)`
returning `Specification<Asset>`) composed at the service layer via
`Specification.where(...).and(...)` based on which query parameters were
actually supplied.

**Why not a single repository method with every filter as a parameter:**
A method like `findByStatusAndAssetTypeAndLocationAndNameContaining(...)`
cannot represent "any subset of filters, in any combination, some possibly
absent" — Spring Data derived query methods require every parameter to be
supplied. Specifications are composed dynamically at runtime, only adding a
`WHERE` predicate for filters the caller actually provided.

**Where `AssetSpecifications` lives:** inside the `asset` package, not
`common/` — the filtering logic operates on `Asset`-specific fields and is
not a generic, reusable tool across other entities, unlike genuinely
cross-cutting concerns (e.g. `PagedResponse<T>`, Decision 11).

**Status:** Decided. Not yet implemented.

## 13. Global exception handling — HTTP status mapping

**Decision:** A single `@RestControllerAdvice` maps every domain/framework
exception encountered so far to the Section 9 error contract, with the
following HTTP status assignments:

| Exception                                  | HTTP Status | Reasoning |
|---------------------------------------------|:-----------:|-----------|
| `AssetAlreadyAssignedException`             | 409 Conflict | Conflicts with the *current state* of the resource (asset is already actively assigned). |
| `InvalidStatusTransitionException`          | 422 Unprocessable Entity | Request is syntactically valid, but semantically not executable given the current business state (illegal lifecycle transition). |
| `DataIntegrityViolationException`           | 409 Conflict | Mapped uniformly to 409 for simplicity, covering the `Assignment` uniqueness race condition (Decision 1) as the primary expected case. Accepted trade-off: this exception is generic and could in principle be caused by other DB constraint violations (e.g. `NOT NULL`) that might better fit 400 — distinguishing by SQLSTATE/constraint name was considered and deliberately not pursued, as fragile/DB-driver-specific for this project's scope. |
| `ObjectOptimisticLockingFailureException`   | 409 Conflict | Conflicts with a concurrent write — the resource was modified by another transaction since it was read (textbook definition of 409 per HTTP semantics). |

**Status:** Decided. Not yet implemented.

## 14. `Asset` REST endpoints — request DTO shape, PATCH over PUT, soft delete

**Server-controlled fields excluded from request DTOs:**
`CreateAssetRequest` (and update DTOs generally) never include `id`,
`organizationId`, `status`, `version`, `createdAt`/`updatedAt` — fields that
represent ownership, authorization, or system-managed state must never be
client-suppliable, and are instead derived from the security context (JWT)
or set by the server. Allowing `organizationId` in a request body would let
an authenticated user from one organization create data attributed to a
*different* organization — a **Mass Assignment vulnerability**: related to,
but distinct from, IDOR (Decision 5) — IDOR is about referencing/reading an
existing foreign object by ID, whereas mass assignment is about a client
injecting an unauthorized value into a field it should never control on a
new or updated record. General rule applied project-wide: `organizationId`
always comes from the JWT, never from any request body.

```json
// CreateAssetRequest
{
  "assetTag": "LT-482",
  "name": "ThinkPad X1",
  "description": "...",
  "serialNumber": "...",
  "assetTypeId": "uuid",
  "locationId": "uuid | null",
  "purchaseDate": "2026-01-15",
  "purchasePrice": 1200.00
}
```

**`PATCH`, not `PUT`, for updates:**
`PUT` semantically means "replace the entire resource" — a field omitted
from the body is (by REST convention) treated as intentionally cleared.
This creates real risk: if a client omits a field (e.g. due to a UI bug or
a form that doesn't include every field), `PUT` semantics would wipe it,
and the server cannot reliably distinguish "field omitted, keep the old
value" from "field explicitly set to null" without extra machinery.
`PATCH` semantics ("apply only the fields present") avoids this class of
bug by convention: an absent field simply means "leave unchanged," which is
both simpler to implement and matches this project's own Section 7 API
sketch.

**Status changes — separate endpoint, not part of general `PATCH`:**
`status` is deliberately excluded from the general asset-update DTO and
handled via its own endpoint (`PATCH /api/v1/assets/{id}/status`), so it
can go through `AssetStatus.canTransitionTo()` validation (Decision 2) and
be authorized independently from general field edits (matching the
permission matrix in Section 6, which lists "Delete/retire asset" as a
distinct permission from "Edit asset").

**`DELETE` — soft delete via existing status transition, not a physical
row delete:**
`DELETE /api/v1/assets/{id}` does not issue a SQL `DELETE`. A physical
delete would either cascade-delete related `AssetHistory`/`Assignment`/
`MaintenanceRecord` rows (violating the immutable-history requirement,
Section 5.7) or be blocked outright by `ON DELETE RESTRICT` foreign keys —
neither is desirable. Instead, `DELETE` transitions the asset to `RETIRED`
(or `DISPOSED`), reusing the *same* `AssetService.changeStatus()` method
and transition validation as the general status-change endpoint (single
source of truth, same principle as Decision 7) — `DELETE` is a thinner,
semantically-named entry point with a fixed target status and its own
(stricter) authorization rule, not an independent copy of transition logic.
Kept as a distinct endpoint from the general status-change endpoint
specifically because it warrants different authorization (e.g. retiring an
asset may require a stricter role than general status changes).

**Status:** Decided. Not yet implemented.

## 15. `Assignment` return endpoint — `PATCH`, server-derived `returnedAt`

**Decision:**
```
POST  /api/v1/assets/{id}/assignments                      -> create (assign)
PATCH /api/v1/assets/{id}/assignments/{assignmentId}       -> return
```

**Why `PATCH`, not `DELETE`, for returning an asset:**
Unlike `Asset`'s soft-delete (Decision 14), where `DELETE` was justified as
semantically close to "no longer in active circulation," returning an
assignment does not conceptually remove or deactivate the record — the row
remains fully valid and useful as a historical record (who had this asset
before). This is a partial update of a single field (`returnedAt`) on an
otherwise-unchanged, still-relevant row — `PATCH` fits the actual semantics
better than the closer-to-"gone" framing that justified `DELETE` for
`Asset`.

**`returnedAt` is never accepted from the client:**
The `PATCH` request body is empty (or carries only genuinely optional,
non-authoritative fields like a note). `returnedAt` is always set by the
server to the current time when the request is processed, and `returnedBy`
is derived from the JWT security context — never accepted as client input.
Same principle as `organizationId` (Decision 14): any field that is
security- or integrity-sensitive (timestamps of server-recorded actions,
ownership, system-managed state) must never be client-suppliable, since a
client could otherwise backdate or falsify the historical record (e.g.
claiming an asset was returned earlier than it actually was).

**Status:** Decided. Not yet implemented.

## 16. `AssignmentService.assignAsset()` — validation order

**Decision:** `assignAsset(assetId, userId)` performs checks in this order:

1. **Authorization** — does the caller have permission to assign assets?
   Performed first, before touching the database at all — rejecting
   unauthorized requests as early as possible avoids unnecessary DB queries
   and minimizes any information that could otherwise leak through timing
   or error responses before an auth check.
2. **Asset exists and belongs to the caller's organization** —
   `findByIdAndOrganizationId(assetId, orgId)` (Decision 5), not a plain
   `findById`.
3. **Target user exists and belongs to the same organization** — same
   `findByIdAndOrganizationId` pattern applied to `User`. Without this, a
   caller could assign an asset to a user belonging to a *different*
   organization (or, combined with a mismatched asset org, cross-link data
   across organizations) — the same family of risk as Decision 5/14, applied
   to a foreign-key reference supplied in the request body rather than a
   path parameter.
4. **No existing active assignment** (Decision 1's service-level check) and
   **asset status permits assignment** (e.g. not `RETIRED`/`DISPOSED`) —
   these two checks are independent of each other (neither is a logical
   prerequisite for the other) and their relative order does not affect
   correctness; both must pass regardless of order.
5. **Persist** — the new `Assignment` row is inserted, with the DB partial
   unique index (Decision 1) as the final backstop against a race condition
   between concurrent requests reaching step 5 simultaneously.

**Status:** Decided. Not yet implemented.

## 17. `Maintenance` REST endpoints — separate named-action endpoints, deferred `cost`

**Decision:**
```
POST  /api/v1/maintenance                    -> create (Reported)
PATCH /api/v1/maintenance/{id}/start         -> Reported -> IN_PROGRESS
PATCH /api/v1/maintenance/{id}/complete      -> IN_PROGRESS -> COMPLETED
GET   /api/v1/maintenance                    -> list (filtering/pagination, same pattern as Asset)
GET   /api/v1/maintenance/{id}               -> detail
```

**Separate named-action endpoints, not one general `PATCH`:**
A single `PATCH /maintenance/{id}` accepting an arbitrary status change
would require internal branching (`if status == IN_PROGRESS ... else if
status == COMPLETED ...`) — the same kind of branching complexity already
avoided for `AssetStatus` via an enum-owned transition map instead of a
`switch` (Decision 2). Separate, explicitly named endpoints avoid this
structurally, and make each endpoint's responsibility unambiguous.

**`startedAt`/`completedAt` — server-derived, never client-supplied**, same
principle as `returnedAt` (Decision 15) and `organizationId` (Decision 14):
timestamps of server-recorded actions must never be trusted from the client.

**`cost` deferred to the `complete` endpoint, not part of creation:**
The actual repair cost is typically unknown when a maintenance issue is
first reported, and only becomes meaningful once work is completed. `cost`
is therefore part of `CompleteMaintenanceRequest`, not
`CreateMaintenanceRequest`.

```json
// CreateMaintenanceRequest
{ "assetId": "uuid", "title": "...", "description": "...", "priority": "MEDIUM", "assignedTo": "uuid | null" }
// StartMaintenanceRequest -> empty body
// CompleteMaintenanceRequest
{ "cost": 240.00 }
```

**`startMaintenance()` validation order:** organization ownership check of
the `MaintenanceRecord` (`findByIdAndOrganizationId`) is performed first,
alongside authorization, before any other logic — same reasoning as
Decision 16 (reject early, avoid unnecessary work). `MaintenanceService`
does not duplicate `AssetStatus` transition validation; it relies entirely
on `AssetService.changeStatus()` throwing `InvalidStatusTransitionException`
if the asset is not in a state that legally permits `IN_MAINTENANCE` —
consistent with the single-source-of-truth principle established in
Decision 7.

**Status:** Decided. Not yet implemented.

## 18. `Location` and `User` endpoints — applying established patterns

**`Location`:**
```
POST   /api/v1/locations
PATCH  /api/v1/locations/{id}
GET    /api/v1/locations   (?tree=true for hierarchical view)
GET    /api/v1/locations/{id}
DELETE /api/v1/locations/{id}
```
- Validation order for create/patch: authorization + organization ownership
  first; if `parentId` is being changed, run the cycle-detection recursive
  CTE (Decision 4) before persisting; if a new `parentId` is supplied,
  verify it belongs to the *same* organization (same cross-org risk family
  as Decisions 14/16). `Location.organizationId` itself is immutable after
  creation — a node can be reparented within its own organization's tree,
  never moved to another organization's tree.
- `DELETE` performs an actual row delete (unlike `Asset`'s soft delete,
  Decision 14) but only succeeds if the location has no child locations and
  no assets referencing it — otherwise `409 Conflict`. Justified because,
  unlike assets, there is no business need to retain a record that a
  location "used to exist" once it's empty and childless.

**`User` / auth:**
```
POST /api/v1/auth/register     (ADMIN-only — no public self-registration; AssetFlow is an internal tool)
POST /api/v1/auth/login        -> issues access + refresh tokens as HttpOnly cookies (Decision 6)
POST /api/v1/auth/refresh      -> new access token, validated via stored token_hash (Decision 6)
POST /api/v1/auth/logout       -> marks the refresh token `revoked = true`
PATCH /api/v1/users/{id}
PATCH /api/v1/users/{id}/role  (ADMIN-only, separate endpoint — same reasoning as Asset.status/DELETE: different, stricter authorization than general profile edits)
GET  /api/v1/users
```
- `password`/`password_hash` is never included in any response DTO —
  response mappers must explicitly exclude it, since it's easy to forget
  with auto-generated/reflective mapping.
- `login`/`refresh`/`logout` are not organization-scoped by URL/JWT context,
  since no JWT exists yet at that point — `organizationId` is only known
  after the submitted credentials are verified against the `User` record.

**Status:** Decided. Not yet implemented.
