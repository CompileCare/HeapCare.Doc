# ADR-020: User Identity, Tenant Membership and Scoped Role Assignments

- **Status:** Proposed — target architecture documented; runtime implementation is not yet approved.
- **Date:** 2026-11-28
- **Decision owners:** HeapCare Product & Engineering, with pilot-hospital operations and security review.
- **Related:** ADR-0004 (tenant registry and trusted membership provisioning), ADR-004 (authentication and authorization), ADR-006 (tenant, organization and patient boundaries), ADR-019 (facility sites and scope inheritance), HeapCare.Doc issue #198.
- **Implementation plan:** [User-management redesign plan](../../engineering/user-management-redesign-plan.md)

## 1. Context and source-verified current state

HeapCare must support one person working for multiple healthcare organizations and holding different responsibilities at different hospitals, clinics, branches or departments. A user-management model tied to one role per tenant membership will not support that requirement cleanly.

The following describes the current implementation, not the target:

1. Authentication uses validated OIDC issuer and subject (iss + sub). Email is not the identity key.
2. TenantMembership currently stores one Role, an active flag and a collection of TenantMembershipScope rows.
3. Membership uniqueness is tenant + issuer + subject. The same external identity can therefore have a membership in more than one Tenant.
4. Membership scopes currently support Tenant, Hospital and Department. FacilitySite/branch/clinic scopes are proposed in ADR-019 and are not implemented.
5. Tenant roles and permission grants are persisted, including custom roles, delegated permission administration and role-change audit.
6. PlatformOperator is a separate global identity registry. A PlatformOperator does not automatically receive tenant membership or clinical access.
7. Platform tenant creation, initial administrator provisioning and tenant lifecycle operations exist. Initial administrator provisioning currently accepts an already-known OIDC issuer/subject; it does not invite or create an identity at the identity provider.
8. No provider-neutral invitation, identity enable/disable, password-recovery or session-revocation integration has been identified in the current application implementation.

Relevant implementation:
- HeapCare.Api/src/HeapCare.Infrastructure/Persistence/TenantMembership.cs
- HeapCare.Api/src/HeapCare.Infrastructure/Persistence/TenantMembershipScope.cs
- HeapCare.Api/src/HeapCare.Infrastructure/Persistence/TenantRole.cs
- HeapCare.Api/src/HeapCare.Infrastructure/Memberships/TenantMembershipAdministrationService.cs
- HeapCare.Api/src/HeapCare.Application/Tenancy/TenantMembershipContracts.cs
- HeapCare.Api/src/HeapCare.Api/Endpoints/TenantRegistryEndpoints.cs

## 2. Decision intent

Evolve the current model toward the following invariant:

> One authenticated identity can belong to multiple customer organizations. Each organization membership can hold multiple independently scoped role assignments. Authorization is resolved centrally from trusted identity, active membership, active role assignment, permission grants, resource ownership and explicit scope.

The redesign is additive and migration-safe. It must preserve the existing tenant boundary, external issuer/subject mapping, role definitions, permission catalogue and fail-closed authorization behaviour.

## 3. Proposed target domain model

### 3.1 UserIdentity

A globally addressable HeapCare identity record for one validated external principal.

- Stable internal identifier.
- Normalized OIDC issuer and subject as the external identity key.
- Lifecycle metadata and timestamps.
- Optional profile/display attributes sourced from a trusted provider, treated as attributes rather than identity keys.
- No local password, password hash, OTP secret or IdP credential storage.

Do not automatically merge identities based on matching email addresses. Linking multiple external identities to one human account requires a separately authenticated, audited account-linking workflow and is not part of the first migration.

### 3.2 TenantMembership

Represents the relationship between one UserIdentity and one Tenant.

- Unique identity + Tenant pair.
- Independent membership status and effective dates.
- Created/invited/activated/deactivated metadata.
- Does not itself imply clinical access.
- Does not store a single authoritative role after the role-assignment migration is complete.

A membership can exist without a staff profile. A staff profile can exist without a login identity. Patient identity and patient registration remain separate concepts.

### 3.3 MembershipRoleAssignment

Represents one role held by a membership.

- References one membership and one tenant-owned or approved system role.
- Has independent active/inactive and effective-date lifecycle.
- Records who assigned, changed or revoked it and why.
- Has one or more explicit scope assignments.
- Multiple assignments may coexist for the same membership, including different roles at different hospitals or departments.

Examples:
- HospitalAdmin at Hospital A.
- ClinicalUser at Hospital B / Department Cardiology.
- PatientRegistrar at Clinic C once the site model is accepted.
- TenantAdmin for organization-wide administration.

The effective permission set is the union of permissions from active assignments only for the resource and operation being authorized. A permission from one assignment must not broaden the scope of another assignment. Authorization must evaluate each permission against the scope attached to the assignment that grants it; it must not combine permissions from one assignment with broader scopes from another.

### 3.4 Role and permission definitions

Retain the existing persisted role and permission foundation. System roles remain code-governed where appropriate; tenant custom roles remain tenant-owned, delegated and audited.

Introduce a dedicated TenantAdmin system role with an explicit permission catalogue. It is for organization administration, not a shortcut to every clinical permission. It must not automatically grant unrestricted access to patient records, encounters, invoices or hospital operations.

TenantOwnership or an equivalent commercial/organizational relationship, if required, is separate from permissions. Being an owner, founder, billing contact or legal representative does not automatically grant operational or clinical access.

### 3.5 Scope model

Preserve Tenant, Hospital and Department scope semantics during the first role-assignment migration.

- Tenant scope grants only the permissions explicitly defined for that role at tenant level. It does not mean unrestricted clinical access.
- Hospital scope applies only to that Hospital and resources explicitly owned by it.
- Department scope applies only to that Department within its parent Hospital.
- FacilitySite scope is deferred until ADR-019 is accepted and its operational, patient-visibility, staffing, billing and reporting implications are resolved.
- Parent/child site relationships do not automatically create permission inheritance or patient sharing.

Each role assignment owns its own scopes. Do not store one flattened scope collection on the membership and reuse it across roles.

## 4. User, tenant and identity lifecycle

Keep these lifecycle dimensions independent:

| Dimension | Example states | Meaning |
|---|---|---|
| External identity | Provider-active / provider-disabled / unknown | Whether the identity provider permits authentication |
| Tenant | Provisioning / Active / Suspended / Closed | Whether the customer organization can use HeapCare |
| Tenant membership | Invited / Active / Deactivated | Whether the identity belongs to this Tenant |
| Role assignment | Pending / Active / Revoked / Expired | Which permissions are delegated and for what period |
| Invitation | Pending / Accepted / Expired / Revoked | Whether a tenant invitation can still be redeemed |

Disabling a membership must deny subsequent HeapCare authorization even if the external identity remains enabled. Disabling an external identity must not silently delete memberships or audit history. Session/token invalidation and provider account disablement are separate provider operations; access-token expiry alone must not be represented as immediate session revocation.

## 5. Identity-provider administration abstraction

Introduce a provider-neutral application contract only after the lifecycle and security requirements are approved. Candidate capabilities:

- Invite/provision an identity using a provider-supported invitation flow.
- Read provider identity status.
- Enable/disable a provider identity.
- Revoke provider sessions or refresh-token sessions where supported.
- Initiate password recovery through the provider; HeapCare never handles or stores passwords.
- Report provider capability and operation outcomes without leaking credentials or tokens.

The first adapter should target the existing local Keycloak environment. Do not select a production provider, send real invitations, configure email delivery or create production credentials as part of local development.

The invitation record must bind the intended Tenant, invited address or provider invitation reference, requested role assignments/scopes, expiry, issuer/creator and acceptance state. Email is a delivery/routing attribute, not proof of identity. At acceptance, bind the membership to the authenticated issuer/subject after the provider's verification flow. Invitation secrets, if generated by HeapCare, must be random, short-lived, single-use, stored only as a secure hash, and never logged.

Provider-specific operations must be idempotent where possible and auditable. Unsupported capabilities must be explicit; do not report a session as revoked when the provider cannot confirm it.

## 6. Administration workflows

### Platform onboarding

1. A trusted PlatformOperator creates a Tenant in Provisioning.
2. Platform administration creates an invitation for the first TenantAdmin through the configured identity-provider adapter.
3. The invited person authenticates through OIDC and accepts the invitation.
4. HeapCare creates or resolves the UserIdentity, creates the TenantMembership and activates the initial TenantAdmin assignment.
5. Tenant activation follows the existing lifecycle preconditions and agreed onboarding checklist.
6. TenantAdmin configures Hospitals and, later, approved FacilitySites.
7. TenantAdmin invites HospitalAdmins and delegates only approved role assignments and scopes.

Preserve a secure recovery path for initial-admin invitations that expire or are sent to the wrong address. Do not automatically promote the tenant creator or the person who initiated an invitation.

### Tenant and hospital administration

- TenantAdmin manages organization configuration, authorized sites and tenant-level membership administration.
- HospitalAdmin manages only memberships, staff and workflows within delegated hospital scopes, subject to explicit permission grants.
- Department-scoped administrators cannot assign permissions or scopes outside their delegation.
- A user may have different role assignments at different hospitals within the same Tenant.
- Staff/provider profile creation and clinical credential/privilege validation remain separate from login provisioning.

### Platform-wide visibility — low priority, informational only

A future, separately permissioned platform portfolio capability may allow one specifically designated HeapCare owner/operator to view a read-only list of customer Tenants and their Hospital/site names and lifecycle statuses.

This is an informational overview, not tenant support impersonation or clinical access.

Guardrails:
- Track it as **low priority / post-core-redesign**; it is not a prerequisite for M1–M6.
- Use a dedicated platform permission such as platform.tenant-portfolio.read, granted only through trusted platform administration. Do not hardcode a person's email, subject or account ID in application code.
- Return a minimal, paginated projection: Tenant name/slug/status and Hospital/site name/type/status/counts where approved.
- Do not return patient identities, registrations, clinical notes, appointments, invoices, staff personal data, membership subject identifiers, credentials or secrets.
- Do not grant tenant membership, tenant context, write operations, impersonation or access to tenant-scoped business APIs.
- Enforce read-only behavior in the API, not only in the UI, and audit every portfolio query.
- A PlatformOperator's existing lifecycle powers must not be mistaken for permission to inspect tenant clinical data.

This capability may be implemented later without changing the identity/membership architecture.

## 7. Authorization and audit requirements

The API remains the security authority. UI hiding is usability only.

Every protected request must evaluate:
1. Validated external identity and issuer.
2. Active internal UserIdentity mapping.
3. Active Tenant and TenantMembership.
4. Active role assignment(s) granting the requested permission.
5. The scope attached to the specific granting assignment.
6. Tenant and resource ownership, including Hospital/Department/Site boundaries.
7. Any additional workflow-state or relationship rule.

Security invariants:
- No client-supplied tenant, role, permission, scope or email value can grant access.
- No PlatformOperator automatically gains tenant clinical access.
- Tenant-wide administrative permissions do not imply patient-data visibility.
- Custom role delegation cannot grant permissions beyond the actor's current delegated set.
- Protect the last active TenantAdmin and any required hospital-level administrator from accidental removal.
- Prevent self-promotion/self-demotion paths that bypass delegation controls.
- Support time-bounded assignments and temporary coverage without editing the underlying user identity.
- Use deny-by-default behavior when an assignment, role, scope or resource mapping is missing or inconsistent.

Membership and role-assignment audit history must record before/after role and scope snapshots, actor issuer/subject, tenant, target identity/membership, reason, timestamp, correlation ID and outcome. Never record passwords, provider tokens, invitation secrets or patient/clinical payloads. Audit events for successful business mutations must commit atomically with those mutations where the existing audit architecture requires it.

## 8. Backward-compatible migration strategy

No destructive schema migration is authorized by this proposal. Use staged, reviewable changes:

1. **Baseline and invariant tests:** inventory every reader/writer of legacy membership Role, membership scopes, authorization context, user-tenant discovery, last-admin checks, role administration, audit and Web contracts.
2. **Identity registry:** add UserIdentity keyed by normalized issuer + subject; backfill distinct pairs from existing memberships and PlatformOperator records without merging by email. Preserve legacy issuer/subject columns during compatibility.
3. **Role assignment tables:** add MembershipRoleAssignment and assignment-owned scope rows. Backfill exactly one assignment for each existing membership using its current role and exact current scopes. Preserve role IDs/codes, custom grants, active state and audit lineage.
4. **Dual-read/compatibility boundary:** introduce a single authorization resolver that can compare legacy and new representations during transition. Do not run two independently maintained authorization implementations. Add consistency diagnostics and fail closed on mismatches.
5. **API contracts:** introduce explicit role-assignment commands and queries. Keep existing membership routes as compatibility adapters until all consumers are migrated; do not silently reinterpret their payloads.
6. **Web migration:** update membership details, user directory and role management to display and manage multiple assignments while preserving scope/delegation UX.
7. **Provider lifecycle:** add invitations and provider operations behind explicit configuration and local Keycloak tests. Keep external provider calls outside database transactions; use pending states, idempotency and reconciliation for partial failures.
8. **Retire legacy columns:** only after API, Web, Local, data backfill, audit and rollback acceptance prove no legacy reader/writer remains. This must be a separate migration and release decision.

Every migration requires EF model snapshot verification, empty-database install, populated-database upgrade, migration-script reapplication check, PostgreSQL concurrency/uniqueness tests, API compatibility tests and Local real-Keycloak acceptance. Preserve current tenant IDs, membership IDs, OIDC subjects, hospital IDs, role definitions and audit history.

## 9. Acceptance criteria

- One identity can belong to multiple Tenants without duplicate credentials.
- One membership can hold multiple active roles at different Hospital/Department scopes.
- Permissions from one role assignment cannot borrow the scope of another assignment.
- A TenantAdmin can perform explicitly granted organization administration without automatically viewing patient or clinical information.
- A HospitalAdmin cannot manage sibling Hospitals without an explicit assignment.
- Department-only scope cannot read or mutate sibling departments.
- Existing memberships are backfilled with exactly their current effective role and scope; there is no privilege expansion.
- Deactivation, role revocation and tenant suspension deny subsequent requests.
- Invitation acceptance binds the verified OIDC identity, never email alone.
- Provider disable and session revocation outcomes are accurately represented.
- Last-admin/delegation protections and before/after audit records are covered by PostgreSQL tests.
- Existing tenant isolation and all current application workflows remain passing.
- The optional portfolio capability is implemented only as separately permissioned, read-only metadata access and remains outside the core redesign critical path.

## 10. Open decisions before runtime implementation

- Whether multiple OIDC providers are required within one deployment and how a human can securely link identities.
- Final TenantAdmin permission grants and the distinction between owner, TenantAdmin and billing contact.
- Invitation delivery channel, expiry, resend, withdrawal and wrong-address recovery policy.
- Which roles may invite users and assign each system/custom role.
- Whether multiple identical role assignments on overlapping scopes should be rejected, merged or allowed with distinct effective dates/reasons.
- Required MFA/authentication assurance for PlatformOperator and TenantAdmin.
- Hospital-level last-administrator invariant and break-glass recovery.
- Acceptance of ADR-019 site types and scope inheritance.
- Audit retention and privacy/legal review for applicable Indian requirements.

Until these decisions and the staged migration plan are reviewed, this ADR is a design baseline, not authorization to change runtime schema or grant new permissions.
