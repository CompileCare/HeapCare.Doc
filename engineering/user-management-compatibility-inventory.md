# User-Management Compatibility Inventory

**Status:** Initial source-backed inventory completed 2026-11-28; final exhaustive reader/writer review remains a M0 exit gate.  
**Related:** [ADR-020](../architecture/decisions/ADR-020-user-identity-membership-and-scoped-role-assignments.md), [execution plan](user-management-redesign-plan.md), HeapCare.Doc #228, HeapCare.Api #159.

## Purpose

Identify the existing API, persistence, authorization and Web contracts that depend on one role per TenantMembership and membership-owned scopes. This is the compatibility baseline for the identity and role-assignment migration. It is not a runtime design approval.

## 1. Current persisted model

### TenantMembership

File: HeapCare.Api/src/HeapCare.Infrastructure/Persistence/TenantMembership.cs

- Holds TenantId through TenantOwnedEntity.
- Stores Issuer, Subject, one Role and IsActive.
- Owns TenantMembershipScope records.
- The external principal key is issuer + subject; email is not used.

### TenantMembership constraints

File: HeapCare.Api/src/HeapCare.Infrastructure/Persistence/Configurations/TenantMembershipConfiguration.cs

- Unique index on TenantId + Issuer + Subject.
- Composite alternate key on TenantId + membership Id.
- Role is a required code and currently references TenantRole through TenantId + RoleCode.

### TenantMembershipScope constraints

Files:
- HeapCare.Api/src/HeapCare.Infrastructure/Persistence/TenantMembershipScope.cs
- HeapCare.Api/src/HeapCare.Infrastructure/Persistence/Configurations/TenantMembershipScopeConfiguration.cs

- Scope rows reference one membership through composite TenantId + MembershipId.
- Tenant, Hospital and Department scope shapes are database-checked.
- Hospital and Department references are constrained to the same Tenant and hospital hierarchy.
- Tenant scope is unique per membership and exclusive in service validation.
- FacilitySite/branch/clinic scope does not exist in the runtime model.

### TenantRole and permissions

Files:
- HeapCare.Api/src/HeapCare.Infrastructure/Persistence/TenantRole.cs
- HeapCare.Api/src/HeapCare.Infrastructure/Persistence/Configurations/TenantRoleConfiguration.cs
- HeapCare.Api/src/HeapCare.Infrastructure/Memberships/TenantRoleBootstrapper.cs

Persisted tenant roles and permission rows already support system/custom roles, active state, permission revisions and role-change audit. This foundation should be reused, not replaced.

## 2. Request-time authorization dependency chain

### Identity-to-membership resolution

File: HeapCare.Api/src/HeapCare.Infrastructure/Persistence/TenantMembershipReader.cs

FindActiveMembershipAsync currently:
1. Reads one active TenantMembership matching TenantId + validated Issuer + Subject.
2. Requires the Tenant to be Active.
3. Resolves the one role code on that membership to one active TenantRole.
4. Loads that role's permission grants.
5. Validates permission catalogue/assignability rules and fails closed on unsupported or empty grants.
6. Loads the membership-owned Tenant/Hospital/Department scopes.
7. Returns one TenantMembershipAuthorization containing one MembershipId, one Role, one scope list and one permission list.

ListActiveTenantContextsAsync follows the same one-role assumption and returns one UserTenantContextDto per active TenantMembership with one role, scopes and permissions.

**Migration impact:** both reader methods must return identity/membership plus role-assignment-level authorization data. Do not flatten scopes across assignments.

### Request middleware and contexts

Files:
- HeapCare.Api/src/HeapCare.Api/Middleware/TenantContextMiddleware.cs
- HeapCare.Api/src/HeapCare.Application/Tenancy/TenantMembershipContracts.cs
- HeapCare.Api/src/HeapCare.Application/Tenancy/TenantPermissionContext.cs
- HeapCare.Api/src/HeapCare.Application/Tenancy/ITenantContext.cs

TenantContextMiddleware currently:
- Uses X-Tenant-ID as a selector, not as an authorization credential.
- Requires a validated issuer and subject and resolves the active membership in the selected active Tenant.
- Replaces reserved caller-supplied tenant/role/permission claims with trusted membership-derived claims.
- Adds one tenant role claim and one set of permission claims.
- Populates one IOrganizationScopeContext (MembershipId, Role, Scopes).
- Populates one ITenantPermissionContext containing a flattened IReadOnlySet of permission codes.

**Critical redesign constraint:** a flat permission set may still answer whether a route has a candidate capability, but it cannot be the authority for resource-scope checks when multiple assignments exist. The new request context must preserve assignment identity, role, permissions and scopes together. Authorization must prove that the same active assignment grants both the requested permission and access to the target resource. A broad scope from a different assignment must never be borrowed.

### API endpoint contracts

Files:
- HeapCare.Api/src/HeapCare.Api/Endpoints/TenantMembershipEndpoints.cs
- HeapCare.Api/src/HeapCare.Api/Endpoints/TenantRoleEndpoints.cs
- HeapCare.Api/src/HeapCare.Api/Endpoints/TenantRegistryEndpoints.cs
- HeapCare.Api/src/HeapCare.Api/Endpoints/UserContextEndpoints.cs
- HeapCare.Api/src/HeapCare.Api/Security/AuthorizationPolicies.cs

Current membership routes expose create/update/deactivate/reactivate. Create and update requests carry one Role plus membership-level scopes. The platform initial-administrator route accepts an existing OIDC issuer and subject. Tenant role endpoints manage role definitions and permission grants. The current-user tenant-context route returns one role per Tenant.

**Migration impact:** retain existing membership routes as compatibility adapters; introduce explicit role-assignment routes and new DTOs. User tenant-context discovery must return a list of assignments per membership rather than a single Role field. Do not silently reinterpret old request payloads.

## 3. Administration service dependencies

### TenantMembershipAdministrationService

File: HeapCare.Api/src/HeapCare.Infrastructure/Memberships/TenantMembershipAdministrationService.cs

Source inspection identifies assumptions that must be migrated:
- Membership create/update writes one Role directly to TenantMembership.
- Role assignment validates one target role against actor permissions.
- Scope containment is checked against the administrator's current IOrganizationScopeContext.
- The service rejects self-modification.
- Last-administrator protection currently protects the last active tenant-wide HospitalAdmin.
- Membership deactivation/reactivation is independent of the external OIDC account.

**Migration impact:** role assignment operations become create/update/revoke operations on individual MembershipRoleAssignment records. Delegation checks must be assignment-aware. Preserve self-modification protections and extend last-admin protection to TenantAdmin and any approved hospital-admin recovery invariant.

### TenantRoleAdministrationService

File: HeapCare.Api/src/HeapCare.Infrastructure/Memberships/TenantRoleAdministrationService.cs

- Role catalogue visibility is bounded by the actor's delegated permissions.
- Custom role creation/update requires tenant-wide membership administration and a reason.
- Custom role permissions cannot exceed the actor's current delegation set.
- Role definitions are separate from assignment instances.

**Migration impact:** retain these role-definition rules. Replace assumptions about a single membership-level role with assignment-level references. Review whether TenantAdmin role administration requires separate system-only controls; do not make TenantAdmin custom-editable by default.

## 4. Web API types and screens

### API contracts

File: HeapCare.Web/src/api/heapcare.ts

Current types expose:
- OrganizationScope union: Tenant | Hospital | Department.
- UserTenantContext with one role, scopes and permissions.
- TenantMembership with one role and scopes.
- Create/update membership inputs with one role and an array of scopes.
- getCurrentUserTenantContexts and tenant-scoped getHospitals.

**Migration impact:** introduce identity, membership, invitation and role-assignment types. Current-user tenant discovery must return assignments. Preserve existing types until the API compatibility adapter is removed.

### Membership administration UI

File: HeapCare.Web/src/features/memberships/TenantMembershipManagement.tsx

Current UI:
- Creates a membership by exact issuer/subject, one selected role and one or more scopes.
- Edits one role and one scope collection per membership.
- Limits scope choices using the current administrator's scope context.
- Deactivates/reactivates the whole membership.

**Migration impact:** display membership lifecycle separately from role assignment lifecycle. Add role-assignment editor with independent scope management; keep membership deactivation as a global tenant-membership action.

### Tenant role administration UI

File: HeapCare.Web/src/features/memberships/TenantRoleManagement.tsx

Manages system/custom role catalogue and permission grants. Keep role-definition management separate from assigning roles to users.

### Platform tenant administration UI

File: HeapCare.Web/src/features/platform/PlatformTenantManagement.tsx

The current PlatformOperator workspace lists and manages Tenant lifecycle, provisions an initial administrator using the current configured OIDC issuer plus an already-created subject, and activates/suspends/closes Tenants. It does not provision IdP users.

The current Web getHospitals function requires a TenantId and sends X-Tenant-ID. Therefore the PlatformOperator tenant list does not currently provide a cross-tenant Hospital directory. A separate read-only portfolio projection is documented in ADR-020 as low priority; it must not create tenant clinical access.

## 5. Existing tests and integrated acceptance surfaces

API integration test files identified in the repository tree:
- TenantMembershipApiTests.cs
- TenantRoleAdministrationApiTests.cs
- TenantRolePersistenceTests.cs
- TenantRegistryApiTests.cs
- UserTenantContextApiTests.cs
- SecurityAuthorizationTests.cs
- PlatformEndpointTests.cs
- PlatformOperatorBootstrapTests.cs
- Capability-specific authorization tests for patient, clinical, billing, appointments/queue, facilities and IPD.

Web tests:
- TenantMembershipManagement.test.tsx
- TenantRoleManagement.test.tsx
- PlatformTenantManagement.test.tsx
- Real-Keycloak authenticated and PKCE Playwright suites.

Local Keycloak configuration:
- HeapCare.Local/local/keycloak/compilecare-realm.json
- HeapCare.Local/local/scripts/configure-keycloak.py

**Migration impact:** add PostgreSQL tests for identity backfill, multi-tenant memberships, multiple assignments, assignment/scope isolation, last-admin protection and audit. Local must exercise real Keycloak identities and authenticated UI flows after API and Web contracts are migrated.


## 6. Expanded authorization-consumer map

The following services were inspected directly in HeapCare.Api on 2026-11-28. These consumers currently receive one request-scoped IOrganizationScopeContext and assume its scope collection is the complete authority for the request.

| Service | Current scope-dependent operations | Redesign implication |
|---|---|---|
| AppointmentQueueService | Appointment/queue transitions and department checks through CanManageDepartment | Every operation must check the permission and department scope on the same role assignment |
| AppointmentSchedulingService | Availability, booking and appointment reads/mutations; helper checks tenant/hospital/department scope | Replace union-scope checks with assignment-aware authorization |
| BillingService | Catalogue management, invoice/payment reads and cashier/payment mutations | Billing permissions cannot borrow a broad hospital scope from a different role |
| OpdEncounterService | Encounter create/read and clinical note lifecycle using department access | Clinical permission and department/site assignment must be evaluated together |
| FacilityService | Hospital, department, ward, room and bed listing filtered using IsTenantWide, HospitalIds and DepartmentIds | Each list query must use an operation-specific authorized scope set or centralized resource authorization |
| HospitalFacilityService | Hospital listing and hospital mutation using IsTenantWide and CanManageHospital | Hospital administration must remain bound to the assignment granting the required facility permission |
| DepartmentFacilityService | Department mutation using CanManageHospital | The assignment must grant the required operation and cover the owning hospital |
| StaffManagementService | Staff and assignment filtering plus assignment mutation checks | Staff-read/manage permissions must be paired with the matching hospital/department assignment |
| InpatientService | Admission visibility, placement history and bed assignment/transfer using scopes and CanManageDepartment | IPD permission and hospital/department assignment must be checked together; preserve all existing fail-closed IPD grants |
| AuditEventQueryService | Audit-event query/export scope filtering, currently using tenant-wide scope as a broad query boundary | Audit permission and each audit event's organization scope must be checked together |

### Route-level permission checks

File: HeapCare.Api/src/HeapCare.Api/Security/AuthorizationPolicies.cs

Current policies inspect trusted membership permission claims generated by TenantContextMiddleware. TenantContextMiddleware currently resolves one role, adds one role claim and a flattened set of that role's permissions, then populates the single IOrganizationScopeContext.

This design is correct for the current one-role membership. It is not sufficient for multiple assignments if the middleware simply unions permissions and scopes independently.

**Required target pattern:** the same role assignment must be the source of both the permission and the resource scope. A request with permission P from Assignment A and a Hospital B scope from Assignment B must be denied against a Hospital B resource unless one and the same active assignment grants P and covers Hospital B. Route policies may remain a fast coarse check, but resource authorization must be assignment-aware and authoritative at the service/resource boundary.

The migration must inventory all policy strings and all service scope calls, then use a common authorization abstraction rather than introducing ad hoc checks in each module.

### Application and infrastructure registration

Files:
- HeapCare.Api/src/HeapCare.Infrastructure/DependencyInjection.cs
- HeapCare.Api/src/HeapCare.Infrastructure/Persistence/HeapCareDbContext.cs
- HeapCare.Api/src/HeapCare.Infrastructure/Persistence/TenantModelConfiguration.cs

HeapCareDbContext applies assembly configurations, global tenant query filters to TenantOwnedEntity records, and concurrency tokens to tenant-owned and platform entities. UserIdentity is proposed as a global IPlatformEntity and must not receive tenant query filtering. Membership rows stay tenant-filtered and require an explicit global-identity foreign key with restrictive delete semantics.

### Global identity registry touch points

Files:
- HeapCare.Api/src/HeapCare.Infrastructure/Persistence/PlatformEntities.cs
- HeapCare.Api/src/HeapCare.Infrastructure/Persistence/Configurations/PlatformEntityConfigurations.cs
- HeapCare.Api/src/HeapCare.Infrastructure/Persistence/PlatformOperatorReader.cs
- HeapCare.Api/src/HeapCare.Infrastructure/Tenants/PlatformOperatorBootstrapService.cs

PlatformOperator currently stores issuer + subject directly and is resolved independently. The proposed global UserIdentity registry must reconcile identities found in both TenantMembership and PlatformOperator records without changing platform authorization or giving the platform identity tenant clinical access.

Initial M1 design constraints:
- Backfill one UserIdentity row per distinct exact issuer + subject pair found in both tables.
- Do not merge by email.
- Retain legacy issuer/subject columns during the compatibility period.
- Add nullable UserIdentityId FKs first, backfill, assert no null/unmatched records, then make required in a later migration once the backfill is verified.
- Restrict physical deletion of UserIdentity while memberships, platform operator rows or audit references exist.
- UserIdentity must not receive an IsActive flag that silently overrides independent membership and PlatformOperator lifecycles. External IdP account status is a separate provider-owned state.

### Test and fixture compatibility

Files/fixtures inspected:
- HeapCare.Api/tests/HeapCare.IntegrationTests/FacilityApiFactory.cs
- HeapCare.Api/tests/HeapCare.IntegrationTests/TenantMembershipApiTests.cs
- HeapCare.Api/tests/HeapCare.IntegrationTests/PlatformOperatorBootstrapTests.cs

FacilityApiFactory creates TenantMembership and PlatformOperator records directly for test seeding. Introducing a required UserIdentity FK requires one shared seed helper that creates/resolves the UserIdentity and attaches it to both record types. Avoid each test creating ad hoc identity rows.

The migration test matrix must cover:
- Existing tenant memberships with repeated issuer/subject pairs across several Tenants backfill to one UserIdentity.
- A principal present only as PlatformOperator is backfilled.
- A principal present in both memberships and PlatformOperator maps to exactly one UserIdentity.
- Distinct issuer values with the same subject remain different identities.
- Same email with different issuer/subject remains distinct.
- Duplicate input rows are handled deterministically.
- Existing membership IDs, tenant IDs, roles, scope rows and active states remain unchanged.

Existing authorization test surfaces:
- TenantMembershipApiTests
- TenantRoleAdministrationApiTests
- TenantRolePersistenceTests
- TenantRegistryApiTests
- UserTenantContextApiTests
- SecurityAuthorizationTests
- PlatformEndpointTests
- PlatformOperatorBootstrapTests
- All service-level authorization suites for facilities, staff, patient identifiers, appointments, billing, OPD and IPD.

## 8. M0 findings and remaining closure gates

**Completed source mapping:** persisted membership and scope schema; OIDC resolution; request middleware and permission context; role/membership admin services; platform operator identity; primary tenant/membership/role endpoints; current-user context contract; all ten inspected scope-consuming application services; Web membership/role/platform screens; test seed helper and primary test suites.

**Remaining M0 gates before M2 authorization refactoring:**
- Complete a repository-wide reference scan for any additional membership/scope consumers not listed above.
- Record the canonical operation-to-permission map for every service scope check.
- Specify the assignment-aware authorization API and behavior for overlapping/redundant assignments.
- Finalize TenantAdmin's organization-only permission set and last-admin/recovery invariants.
- Define the compatibility window and rollback trigger for M1/M2 migrations.

M1 identity registry work may proceed additively without changing current request authorization. M2 must not begin permission/scope flattening or grant TenantAdmin until its operation/permission/scope contract and tests are ready.

No runtime code, database schema, role grant or authorization behavior was changed by this inventory.


## 7. Current platform portfolio visibility

Existing global tenant APIs and PlatformTenantManagement allow a persisted PlatformOperator to list/read tenant registry metadata and perform lifecycle operations. The PlatformOperator does not automatically become a tenant member. Hospital data remains behind tenant-scoped hospital endpoints.

Future requirement recorded separately:
- designated platform identity with explicit portfolio-read permission;
- read-only tenant + hospital/site metadata projection;
- no patient, clinical, staff-personal or financial details;
- no tenant impersonation, tenant membership or business writes;
- every query audited;
- low priority and excluded from the identity redesign critical path.

## 9. M0 completion checklist

- [x] Identify the current persisted one-role membership model and its constraints.
- [x] Identify the request-time membership reader and middleware/context dependency chain.
- [x] Identify membership/role administration endpoints and services.
- [x] Identify the principal Web types and membership/role/platform administration screens.
- [x] Identify primary API tests and Local Keycloak acceptance surfaces.
- [ ] Complete exhaustive reader/writer search for all membership/role references, including any non-obvious jobs or integration adapters.
- [ ] Define exact role-assignment/permission evaluation contract and migration diagnostics.
- [ ] Obtain review of TenantAdmin permission matrix, identity lifecycle and migration/rollback sequence.
- [ ] Merge the proposed ADR and plan after review.

No runtime code, database schema, role grant or authorization behavior was changed by this inventory.
