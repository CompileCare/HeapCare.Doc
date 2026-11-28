# Assignment-Aware Authorization Contract

**Status:** Proposed implementation contract for M2; no runtime behavior changed.  
**Date:** 2026-11-28  
**Related:** [ADR-020 — User identity, membership and scoped role assignments](../architecture/decisions/ADR-020-user-identity-membership-and-scoped-role-assignments.md), [M0 compatibility inventory](user-management-compatibility-inventory.md), HeapCare.Api #159, HeapCare.Doc #228.

## 1. Purpose

Define how HeapCare must authorize a user who holds multiple roles within one Tenant without allowing a permission from one role assignment to borrow the scope of another.

The current request context contains one role, one flat permission set and one membership scope set. It is correct for the current one-role membership model but cannot be extended by independently unioning all role permissions and all scopes.

## 2. Core invariant

> A protected operation is authorized only when one active role assignment on the current active tenant membership grants the required permission and that same assignment covers the target resource scope.

For assignment A and requested permission P, a Hospital/Department/Site resource R is permitted only if:

1. The caller has a validated OIDC issuer + subject.
2. The selected Tenant is Active.
3. The caller's TenantMembership in that Tenant is Active.
4. Assignment A belongs to that membership and is Active and effective at the request time.
5. Assignment A references an active TenantRole.
6. That role currently grants P under the canonical permission catalogue and assignability/system-only rules.
7. A scope attached to Assignment A covers R under the explicit scope semantics.
8. The resource itself belongs to the selected Tenant and its parent hierarchy is consistent.
9. Any additional workflow-state and relationship conditions pass.

A permission from Assignment A and a broad scope from Assignment B do not satisfy these conditions. A PlatformOperator is not a TenantMembership and gains no tenant clinical access from platform status.

## 3. Recommended persistence model

### TenantMembership

Keep one row for the identity + Tenant relationship. It owns membership lifecycle, not a single role.

### MembershipRoleAssignment

Recommended fields:

- Id, TenantId, MembershipId, RoleId.
- Status: Pending, Active, Revoked, Expired.
- EffectiveFromUtc and optional EffectiveUntilUtc.
- AssignedByIdentityId / actor issuer+subject snapshot.
- CreatedAtUtc, UpdatedAtUtc, Version.
- Optional reason/reference metadata for administrative change history.

RoleId must reference a TenantRole inside the same Tenant by composite TenantId + RoleId. MembershipId must reference the membership in the same Tenant. Restrict physical deletion; revoke/deactivate instead.

**Recommended initial cardinality:** one active assignment per (TenantId, MembershipId, RoleId). One assignment can carry multiple distinct scopes. This allows a user to hold different roles at different Hospitals while avoiding duplicate overlapping copies of the same role. Whether the product later needs multiple concurrent versions of the same role with different effective periods remains an open decision.

### MembershipRoleAssignmentScope

Move scope ownership from membership to role assignment. Preserve the current shapes during M2:

- Tenant: no HospitalId or DepartmentId.
- Hospital: HospitalId required, DepartmentId null.
- Department: HospitalId and DepartmentId required; Department must belong to that Hospital and Tenant.

Tenant scope is exclusive within one assignment. A redundant Department scope under an already assigned Hospital scope is rejected. Duplicate scope rows are rejected by both application validation and database uniqueness.

FacilitySite scope is not part of M2 until ADR-019 and HeapCare.Doc #198 are accepted.

### Migration mapping

For every existing TenantMembership:

- Create exactly one **Active** MembershipRoleAssignment with its current Role.
- Copy every existing Tenant/Hospital/Department scope to that assignment's scope table, with exact parent identifiers.
- Preserve membership Id, TenantId, issuer, subject, legacy Role and legacy scopes during the compatibility period.
- Keep membership lifecycle independent from assignment lifecycle. An inactive legacy membership receives an Active assignment but remains denied because authorization requires the membership itself to be Active. Reactivating that membership restores only assignments that remain Active and effective; Revoked or Expired assignments stay denied. Do not translate reversible membership deactivation into assignment revocation.
- Do not infer additional roles, sites, hospitals or permissions.
- Compare effective permission/scope outcomes before and after migration; any changed access is a blocker.

The existing role permission tables remain authoritative. A custom-role permission edit continues to affect active assignments on the next authorized request, consistent with the current persisted RBAC behavior.

## 4. Request context contract

Replace the single-role authorization context with an immutable request-local collection of role assignments. The shape should preserve assignment, role, permission and scope together, for example:

- MembershipId and TenantId.
- For each assignment: AssignmentId, RoleId/Code, active/effective state.
- The permission codes currently granted by that role.
- The exact scope rows attached to that assignment.

Do not expose one flattened scope list as a security authority. Do not flatten permissions and scopes separately. 

**Implementation progress (draft, dependent on HeapCare.Api #162):** HeapCare.Api [PR #163](https://github.com/HeapCare/HeapCare.Api/pull/163) adds a data-only `ITenantRoleAssignmentContext`. The membership reader loads only active, currently effective assignments whose TenantRole is active and whose own permission grants pass the canonical permission-catalogue validation. Each snapshot retains its own role, permissions and scope rows. Middleware populates this snapshot alongside the current legacy membership context.

This increment deliberately adds **no authorization methods** to the new context and changes no route policy, claim, service-level scope check or clinical permission. The current legacy authorization remains authoritative until the scope policy is approved and every consumer is migrated. Keep the snapshot out of authorization decisions until that follow-up is complete.


### Required authorization API semantics

The application authorization abstraction should expose operations equivalent to:

- HasAnyAssignmentPermission(permissionCode): a coarse route gate only; it cannot authorize a resource.
- CanAccessTenant(permissionCode, tenantId).
- CanAccessHospital(permissionCode, hospitalId).
- CanAccessDepartment(permissionCode, hospitalId, departmentId).
- Queryable authorized scope sets for a permission, e.g. authorized Hospital IDs and Department IDs, for list/read queries.

Each method must search for one assignment that contains the requested permission and has a scope covering the resource. Scope-set query helpers must filter the assignments by the requested permission before projecting their scopes.

When a use case requires multiple permissions, each required permission must independently authorize the same target resource. A separate assignment may satisfy another permission only if its own scope also covers that same resource; scopes are never borrowed across assignments.

### Scope semantics

- Tenant scope covers Tenant-owned resources only for permissions granted by the role on that assignment. It does not itself create permissions.
- Hospital scope covers resources under exactly that Hospital, subject to each module's resource-parent relationship.
- Department scope covers resources belonging to exactly that Department in the stated Hospital.
- No parent/child or sibling-site inheritance is implied.
- A patient record remains subject to its registration/clinical resource relationship and explicit patient-visibility rules; membership alone does not create cross-hospital sharing.

**Compatibility rule:** an existing tenant-wide HospitalAdmin assignment keeps exactly the effective permissions that its current role and Tenant scope grant today. Do not silently replace those users with TenantAdmin or narrow/expand their access in the backfill.

## 5. Route-level and resource-level authorization

Route policy checks remain useful for coarse denial and endpoint discovery, but they are not sufficient resource authorization.

A route policy may verify that the request has at least one active assignment with the required permission. The application service must then verify that the granting assignment's scope covers the specific Hospital/Department/site/resource being read or changed.

For list endpoints, the SQL query must be restricted to resources covered by assignments that grant the list permission. Do not use a flat set of all member HospitalIds/DepartmentIds when that set includes scopes from assignments that do not grant the requested permission.

For mutations, validate resource ownership and assignment-aware permission/scope before changing data. Continue using non-disclosing 404/403 patterns and metadata-only audit behavior where currently required.

## 6. Existing service migration map

All of the following current services consume the single IOrganizationScopeContext and must migrate to permission-specific scope authorization. A common application authorization abstraction should be introduced; do not add custom role-scope logic independently to each service.

| Service | Permission families that must be paired with resource scope |
|---|---|
| FacilityService / HospitalFacilityService / DepartmentFacilityService | facility.read, facility.write, facility.delete |
| PatientService | patient.read, patient.identity.read, patient.register |
| PatientIdentifierService | patient.identifier.read, patient.identifier.assign, patient.identifier.policy.read/manage |
| PatientDuplicateDetectionService | patient.duplicate.review |
| StaffManagementService | staff.read, staff.manage |
| AppointmentSchedulingService | availability.read/manage, appointment.read/manage |
| AppointmentQueueService | queue.read, queue.manage |
| OpdEncounterService | encounter.read/manage/sign/amend |
| BillingService | billing.catalogue.manage, invoice.read/manage, payment.manage, billing.report.read |
| InpatientService | inpatient.read/admit/bed.assign/bed.transfer and later cancel/discharge |
| AuditEventQueryService | audit.read |

This is the initial source-backed list from M0 inventory; complete a final reference scan before changing the request context.

## 7. TenantAdmin permission design

TenantAdmin is a separate system role, not a rename or alias for HospitalAdmin.

Its proposed responsibility is organization administration: tenant configuration, authorized hospital/site registry administration, user/membership administration, role assignment where explicitly delegated, and approved aggregate organization metadata.

It must not automatically include patient.read, patient.identity.read, patient.register, clinical encounter permissions, IPD permissions, or billing/payment permissions. A person who needs clinical or financial work receives separate assignments for those responsibilities.

The exact TenantAdmin grant list and whether it may assign every system role require product/security approval before implementation. In particular, membership.role.assign.system is an explicit privilege and must not be inferred merely from being a TenantAdmin.

TenantOwnership, legal representative, billing contact and platform owner are relationships or contact designations, not roles that grant application permissions.

## 8. Test acceptance matrix

### Positive tests

- One identity belongs to two Tenants and receives independent assignments in both.
- One membership has HospitalAdmin at Hospital A and ClinicalUser at Hospital B / Department Cardiology.
- An assignment with patient.read + Hospital A scope can search/view only Hospital A records.
- A second assignment with encounter.read + Hospital B / Cardiology scope can read only matching encounters.
- A user with two distinct scoped assignments can use each authorized workflow without changing login identity or membership.
- Role permission updates affect subsequent requests through the persisted role source.

### Critical negative scope-borrowing tests

- Assignment A grants patient.read at Hospital A. Assignment B grants facility.read at Hospital B. patient.read must not become available at Hospital B.
- Assignment A grants staff.manage at Department X. Assignment B grants staff.read at Hospital Y. staff.manage must not authorize changes at Hospital Y.
- A read permission on a broad Tenant-scoped administrative assignment cannot borrow a clinical scope from an assignment that grants no such clinical permission.
- Department A permission/scope cannot access Department B even when the same user has another role scoped to Department B but that role lacks the requested permission.
- Inactive/expired assignment, inactive role, inactive membership, Suspended/Closed Tenant and mismatched resource Tenant all deny.
- Revoked role must deny on the next request without waiting for a stale client claim.
- Forged role/permission claims, tenant headers and email values do not affect resolution.

### Regression and migration

- Backfill creates exactly one initial assignment per membership and copies exact scopes.
- Existing effective permissions remain identical for every seeded pre-migration user.
- Preserve self-modification protections, delegated custom-role checks, system-role assignment gates, last-admin protections and audit success/denial/rollback coverage.
- Fresh install, populated upgrade, idempotent migration script re-application, EF snapshot guard, PostgreSQL concurrency and Local real-Keycloak acceptance.
- Existing API, Web and role-management journeys remain passing.

## 9. Open decisions before M2 runtime implementation

- Confirm one assignment per membership + role as the initial uniqueness rule.
- Approve the precise TenantAdmin grant list and administrator recovery invariant.
- Specify whether organization-level administrative read operations are covered by Tenant scope, distinct from clinical/financial data.
- Confirm each operation's canonical permission code and resource boundary for the thirteen scope-consuming services.
- Define temporary assignment expiry/revocation semantics and audit reason requirements.
- Decide whether any permission should support multiple-resource checks or a two-person/approval workflow.
- Accept ADR-019 before introducing FacilitySite scopes.

Until these decisions and test cases are agreed, this document is a proposed contract. M1 identity registry work can proceed independently; M2 must not flatten current permissions/scopes or enable new role grants ahead of the contract.
