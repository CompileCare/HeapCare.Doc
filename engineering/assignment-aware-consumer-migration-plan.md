# M2 Assignment-Aware API Consumer Migration Plan

**Status:** Execution plan — consumer migration not started.  
**Owner:** HeapCare Product & Engineering.  
**Dependencies:** ADR-020 (proposed), [assignment-aware authorization contract](assignment-aware-authorization-contract.md), [user-management compatibility inventory](user-management-compatibility-inventory.md), [current operation/permission route matrix](operation-permission-route-matrix.md), HeapCare.Api #159.  
**Current implementation:** HeapCare.Api draft PRs #162 (assignment persistence), #163 (request snapshot), and #164 (assignment-aware evaluator).

## 1. Goal and non-negotiable rule

Migrate every service that currently consumes the single-role `IOrganizationScopeContext` to assignment-aware authorization without changing the effective access of existing memberships during the first cutover.

For a protected resource operation, one and the same active, effective `MembershipRoleAssignment` must provide:
1. the canonical permission required for the operation; and
2. a scope that covers the target tenant, hospital, department and resource relationship.

Never satisfy an operation by taking a permission from Assignment A and a matching scope from Assignment B. A route-level permission policy may remain a coarse check; the service/resource boundary remains authoritative.

The first cutover must preserve the current five system roles' effective permissions and Tenant/Hospital/Department scope semantics. TenantAdmin remains excluded pending its separately approved organization-administration permission matrix. IPD permissions remain non-assignable. FacilitySite scopes remain deferred pending ADR-019 / issue #198.

## 2. New primitives available in the draft chain

- **PR #162 — persistence/backfill:** one active assignment per existing membership, exact role/scope copy, legacy columns retained, compatibility dual-write. Validated on exact head with API, Docker and PostgreSQL migration workflows.
- **PR #163 — request snapshot:** loads active, currently effective assignments and their own role permissions/scopes; immutable request-scoped context. Validated on exact head with 16 unit and 278 PostgreSQL integration tests plus migration rehearsal. Docker integration is deferred until the dependency branch is retargeted to main.
- **PR #164 — evaluator:** provides same-assignment resource checks and permission-specific scope projection. Draft; validate exact head before accepting it. The evaluator is not wired into existing services.

The evaluator's query-list API is intended for one operation at a time:
- `GetAuthorizedScopes("patient.read")` returns scopes only from assignments that each grant `patient.read`.
- `GetAuthorizedScopes("patient.read", "patient.identity.read")` includes scopes only from assignments that grant both permissions together.
- Do not cache that result across different permission codes or operation types.
- Resource checks use permission-specific `CanAccess...` / `CanManage...` methods, not a broad union of membership scopes.

## 3. Required service-by-service migration

The route matrix is the canonical source for route permission identifiers. Where a service method has no explicit permission argument today, the API contract must name its operation permission before code migration. This plan does not invent new permission codes or alter route policy metadata.

| Service | Permission-specific operations to migrate | Required target behavior |
|---|---|---|
| `AppointmentQueueService` | `queue.read`, `queue.manage`; queue transition and check-in | Pair the requested queue permission with the assignment scope covering the queue entry's Hospital and Department. |
| `AppointmentSchedulingService` | `availability.read/manage`, `appointment.read/manage` | Booking, availability, rescheduling, cancellation and reads must prove permission and scope on one assignment; list/slot filters use the matching permission's authorized scopes. |
| `BillingService` | `invoice.read/manage`, `payment.manage`, `billing.report.read`, `billing.catalogue.manage` | Invoice, payment, catalogue and report queries/mutations each use their own route-mapped permission and the same assignment's Hospital/Department coverage. Preserve the current service-items list mapping to `invoice.read`. |
| `OpdEncounterService` | `encounter.read/manage/sign/amend` | A clinical permission must be paired with the assignment covering the linked queue entry/encounter's Hospital and Department. Keep clinician credential/privilege checks separate. |
| `FacilityService` | `facility.read/write/delete` by operation | Each Hospital/Department/Ward/Room/Bed query or mutation derives authorized scopes using the permission for that operation; a read scope must not authorize writes or deletes. |
| `HospitalFacilityService` | `facility.read/write/delete` | Hospital listing uses `facility.read`; create/update use `facility.write`; delete uses `facility.delete`. The same assignment must own both the required grant and tenant/hospital scope. |
| `DepartmentFacilityService` | `facility.read/write/delete` by operation | Department mutation and ownership validation must use the permission required for that exact endpoint and the matching parent Hospital/Department scope. |
| `StaffManagementService` | `staff.read/manage` | Staff profile list/detail and staff assignment create/end each use the appropriate permission and assignment scope covering the staff resource or target assignment's Hospital/Department. Staff records remain distinct from login memberships. |
| `InpatientService` | `inpatient.read/admit/bed.assign/bed.transfer` | Maintain fail-closed behavior because all IPD grants are currently non-assignable. Resource code may use the evaluator, but no existing role may receive IPD grants until separate policy approval. |
| `AuditEventQueryService` | `audit.read` | Compute query bounds only from assignments granting `audit.read`; each returned event must be covered by the same qualifying assignment and its tenant/resource scope. |
| `PatientService` | `patient.read`, `patient.identity.read`, `patient.register` | Patient search, detail and registration must enforce hospital-registration ownership plus same-assignment permission/scope. Department endpoints require exact Department scope unless a Tenant/Hospital-scoped assignment covers the resource as defined by current semantics. |
| `PatientIdentifierService` | `patient.identifier.read/assign`, `patient.identifier.policy.read/manage` | MRN policy and identifier read/assign operations remain hospital-scoped; separate policy codes must not inherit one another's scope. |
| `PatientDuplicateDetectionService` | `patient.duplicate.review` | Candidate discovery and review require this permission and the same assignment's Hospital scope for the candidate registration. |

## 4. Migration sequence

### M2-A — Complete the compatibility scan

- Search all API projects for `IOrganizationScopeContext`, `CanAccessHospital`, `CanManageHospital`, `CanAccessDepartment`, `CanManageDepartment`, `IsTenantWide`, `HospitalIds`, `DepartmentIds`, direct `Scopes` and current permission context use.
- Search every endpoint policy string against the route matrix; record routes without a canonical permission.
- Inspect jobs, exports, background operations and event handlers that may bypass HTTP middleware.
- Record operation permission, resource owner lookup, scope level, list-query filter and audit outcome for every call site.
- No service should be treated as complete from a class-level inspection alone; every scope-dependent call site must be tracked.

### M2-B — Add operation-aware query and resource adapters

- Use immutable assignment snapshots only.
- For list operations, call the permission-specific scope projection with the canonical operation permission. Apply the resulting scope filter inside the database query before materialization/pagination.
- For single-resource operations, resolve the resource's Tenant/Hospital/Department ownership first, then call the evaluator with that operation's permission and exact owner IDs.
- For compound actions, require all permissions on the same assignment; do not combine a set of required permissions across assignments.
- Keep route-level authorization claims as a coarse fast path only. Do not treat a union of assignment permission claims as sufficient resource authorization.
- Keep PlatformOperator routes and tenant authorization separate.
- Retain legacy route contracts and DTOs until Web consumers are migrated.

### M2-C — Migrate consumers in dependency order

1. Facilities and resource ownership helpers (establish canonical Tenant/Hospital/Department lookup behavior).
2. Patient search/detail/registration, duplicate review and MRN (privacy-sensitive boundaries; complete negative cross-hospital tests).
3. Staff profiles and staff assignments.
4. Availability, appointment booking/read/mutation, queue and OPD encounter workflows.
5. Billing invoices, payments, catalogues and reports.
6. Audit queries/exports.
7. IPD read and operations only as a fail-closed compatibility migration. Do not grant IPD permissions.

Each module PR must include positive and negative tests for its own permission codes, Tenant/Hospital/Department boundaries, same-user multi-role behavior and route-to-service permission consistency. No module PR may independently enable new role grants.

### M2-D — Coordinated authorization cutover

- Complete the reader/writer inventory and every service consumer before switching the runtime authority.
- Add before/after parity tests that run the old membership role/scope representation and the backfilled assignment representation against the same synthetic dataset. Existing effective outcomes must be identical.
- Test a user with different grants at Hospital A and Hospital B; verify no role/scope borrowing for patient, clinical, staff, appointment, billing, audit or facility data.
- Verify custom-role permission changes, assignment revoke/expiry, membership deactivation/reactivation and tenant suspension are reflected on the next request.
- Switch middleware/service resource authorization as one coordinated release. If parity or cross-scope behavior differs, keep the legacy runtime path and block rollout.
- Do not remove legacy Role or membership scopes in this migration. Their retirement is a later, separately rehearsed release.

## 5. Mandatory negative and positive test matrix

| Scenario | Expected result |
|---|---|
| Assignment A grants `patient.read` at Hospital A; Assignment B grants unrelated `facility.write` at Hospital B | Patient read at Hospital B is denied. |
| Assignment A grants `staff.read` at Hospital A; another assignment grants `staff.manage` at Hospital B | Staff write in Hospital A is denied; Staff read at Hospital B is denied. |
| Two assignments independently grant `appointment.read` at Hospitals A and B | Appointment list includes only resources in A and B, no sibling tenant/hospital records. |
| One assignment grants `invoice.read`; another assignment grants `payment.manage` | A compound action requiring both on one assignment is denied unless one active assignment grants both. |
| Assignment is future-dated, expired, revoked or its role becomes inactive | It contributes no effective permissions or scopes. |
| Membership is deactivated while its assignment remains Active | Request is denied by membership lifecycle. Reactivation restores only still-active/effective assignments. |
| Tenant is Suspended | Tenant-scoped request is denied regardless of assignment state. |
| Department A scope attempts Department B resource in same Hospital | Denied unless the same permission-granting assignment also covers B or a valid broader scope. |
| Hospital A scope attempts sibling Hospital B resource | Denied unless the same permission-granting assignment covers B. |
| PlatformOperator has no tenant assignment | No tenant clinical/resource access is inferred. |
| IPD route has a known but non-assignable permission | Remains denied to all current tenant roles. |
| List route has no canonical permission mapping | Build/release gate fails; no broad scope fallback is allowed. |

## 6. Cross-cutting controls and test evidence

- **Audit:** capture success/denial, actor, tenant, assignment ID(s) considered, resource owner IDs, permission code and correlation ID. Do not log patient or clinical payloads.
- **Delegation:** assignment creation/revocation must verify the actor can grant the exact role and scope; self-promotion remains denied; last-active-admin recovery is preserved.
- **Database:** PostgreSQL integration tests for all tenancy boundaries and concurrent role/assignment lifecycle changes.
- **Migration:** fresh install, populated database upgrade, idempotent script, model snapshot guard, backfill cardinality/scope parity, rollback rehearsal.
- **Local:** real Keycloak token and browser workflow acceptance after the API/Web contract migration.
- **Web:** current UI is not a security boundary; hidden controls do not replace API denies.
- **Accessibility:** preserve keyboard/responsive behavior; manual OS/screen-reader acceptance remains separately tracked in issue #201.

## 7. Explicit non-goals

- No TenantAdmin grant until its organization-only permission/delegation matrix is approved.
- No Platform portfolio/tenant overview implementation (low priority).
- No FacilitySite/branch/clinic scope until ADR-019 / issue #198 decisions are accepted.
- No IPD permission enablement, role grants, discharge/cancellation policy decisions or clinical workflows under this authorization migration.
- No production identity-provider or deployment configuration; remain local-first under issue #197.
- No destructive legacy-column/scope removal.
