# User-Management Redesign — Execution Plan

**Status:** Active planning baseline; implementation begins with design and compatibility work.  
**Owner:** HeapCare Product & Engineering.  
**Primary repositories:** HeapCare.Api, HeapCare.Web, HeapCare.Local and HeapCare.Doc.  
**Architecture decision:** [ADR-020 — User Identity, Tenant Membership and Scoped Role Assignments](../architecture/decisions/ADR-020-user-identity-membership-and-scoped-role-assignments.md).  
**Related tracked work:** HeapCare.Doc #198 (facility sites), #197 (deferred production readiness).

## GitHub tracking

- **Architecture epic:** [HeapCare.Doc #228](https://github.com/HeapCare/HeapCare.Doc/issues/228)
- **Compatibility inventory:** [engineering/user-management-compatibility-inventory.md](user-management-compatibility-inventory.md) — API/Web dependency map for current identity, role, scope and authorization consumers.
- **M2 authorization contract:** [engineering/assignment-aware-authorization-contract.md](assignment-aware-authorization-contract.md) — permission and resource scope must be evaluated together on the same active role assignment.
- **Current operation/permission map:** [engineering/operation-permission-route-matrix.md](operation-permission-route-matrix.md) — endpoint-level route policies and canonical permission codes across the existing API modules.
- **API identity and role-assignment migration:** [HeapCare.Api #159](https://github.com/HeapCare/HeapCare.Api/issues/159)
- **API identity lifecycle and invitations:** [HeapCare.Api #160](https://github.com/HeapCare/HeapCare.Api/issues/160)
- **Web administration workflows:** [HeapCare.Web #110](https://github.com/HeapCare/HeapCare.Web/issues/110)
- **Local real-Keycloak and migration acceptance:** [HeapCare.Local #51](https://github.com/HeapCare/HeapCare.Local/issues/51)
- **FacilitySite dependency:** [HeapCare.Doc #198](https://github.com/HeapCare/HeapCare.Doc/issues/198)
- **Portfolio overview:** documented low-priority requirement in ADR-020; intentionally not an implementation issue in the current critical path.

## Objective

Evolve HeapCare's existing OIDC, tenant membership and persisted RBAC foundation so one identity can belong to multiple healthcare organizations and hold multiple independently scoped roles within each organization. Preserve tenant isolation and current effective access during migration.

This is a local-first engineering initiative. Do not select/configure production identity providers, domains, paid services or production infrastructure as part of this work.

## Current verified baseline

- OIDC issuer + subject is the external identity key; email is not used as an identity key.
- TenantMembership currently owns one Role and membership-level Tenant/Hospital/Department scopes.
- A unique tenant + issuer + subject membership allows one identity to join multiple Tenants.
- Tenant roles and permissions are persisted, including custom roles, delegated grants and role-change audit.
- PlatformOperator is a separate global identity and is not a tenant membership.
- Platform tenant lifecycle and initial-admin provisioning exist, but the initial administrator must already have a provider identity.
- No identity-provider invitation, enable/disable, password-recovery or session-revocation application integration is present in the inspected implementation.
- ADR-019 proposes FacilitySite; it is not accepted and no site schema/scope is implemented.

## Milestones and work breakdown

### M0 — Architecture and migration baseline

**Status: Source-backed baseline documented; final discovery remains continuous.**

- [x] Inspect current API identity, membership, scope, role, permission and platform-operator implementation.
- [x] Inspect current Web feature areas for membership and platform administration.
- [x] Record the initial source-backed compatibility inventory for the request-time authorization chain, persistence, endpoints and Web screens.
- [x] Confirm no open PRs in Api, Web, Local or Doc at the start of this workstream.
- [ ] Merge ADR-020 and this execution plan after review.
- [x] Create linked implementation issues in the affected repositories.
- [ ] Inventory every use of legacy membership Role and membership-level scopes.
- [x] Define the initial assignment-aware authorization invariant and privilege-preserving backfill contract in the M2 design document.

**Exit:** proposed target model, source-backed reader/writer inventory, route/permission matrix and migration/backfill invariants are documented. ADR-020 and M2 contract remain proposed pending product/security review. Keep the inventory current whenever new endpoints or services are added.

### M1 — Global identity registry and membership association

**Status: Complete.**

**API evidence:** HeapCare.Api PR #161 merged as `7adf40d014582f29d1a88bb80fd60ddcc2b702f8`. Exact-head API CI run [#46171317470](https://github.com/HeapCare/HeapCare.Api/actions/runs/46171317470) passed 12/12 unit tests and 276/276 PostgreSQL integration tests plus the EF model-drift guard. Docker Integration Verification [#46171317462](https://github.com/HeapCare/HeapCare.Api/actions/runs/46171317462) passed. PostgreSQL Migration Rehearsal [#46171317451](https://github.com/HeapCare/HeapCare.Api/actions/runs/46171317451) passed populated-database upgrade and fresh installation.

**Integrated Local evidence:** HeapCare.Local PR #52 merged as `111f933f624c8dc0c60b86f18b4f532e2515a53c`. Exact-head full-stack run [#46175082165](https://github.com/HeapCare/HeapCare.Local/actions/runs/46175082165) passed both jobs: nine-service Compose startup/telemetry, real Keycloak-issued identity and persisted multi-tenant acceptance, PostgreSQL logical backup and isolated restore, Chromium Authorization Code + PKCE, Firefox/WebKit six-role accessibility, and cleanup of all ephemeral containers, volumes and CI-only secrets. The real-Keycloak acceptance verified that one issuer/subject can be a PlatformOperator and an explicitly provisioned FacilityViewer in Tenant A, while receiving no Tenant B membership and no patient, clinical or IPD privilege expansion.

**Browser-fixture support:** HeapCare.Web PR #111 merged as `5f874752f230d210d63dcc6a65ac2a7b2a836cfe` to verify the separate PlatformOperator and tenant workspaces. PR #112 merged as `520ec89b048958cf253a5823ee011a6483abdf46` to use the dedicated appointment-cancellation local date in the browser fixture while preserving the original booking date.

**Repositories:** HeapCare.Api; HeapCare.Local.

- [x] Add UserIdentity keyed by validated OIDC issuer + subject.
- [x] Add explicit identity links from TenantMembership and PlatformOperator.
- [x] Preserve current identity mappings by exact issuer + subject; never merge by email.
- [x] Keep current issuer/subject columns temporarily for compatibility and diagnostics.
- [x] Add unique issuer/subject, composite identity foreign keys and restrictive delete constraints.
- [x] Test existing identity backfill, multi-tenant identity reuse and distinct issuer handling in PostgreSQL.
- [x] Complete Local real-Keycloak acceptance, including one principal linked to tenant membership and the PlatformOperator registry.
- [x] Complete real-Keycloak PKCE browser acceptance, six-role Firefox/WebKit accessibility, PostgreSQL logical restore and ephemeral environment cleanup.

**Exit:** M1 complete. The global identity backfill and both API and integrated Local acceptance are merged and green. No role, scope or permission authorization behavior was changed by M1.

### M2 — Multiple scoped role assignments

**Status: In progress — additive persistence/compatibility increment is in draft review, not merged.**

**Current API increment:** [HeapCare.Api PR #162](https://github.com/HeapCare/HeapCare.Api/pull/162), branch `feature/m2-membership-role-assignments`, current tested head `301eeff8be1b95e4d2b8dcb78efd6253ca610bac`.

The PR contains:

- Tenant-owned MembershipRoleAssignment and assignment-owned Tenant/Hospital/Department scopes.
- Restrictive composite tenant, membership, role and facility relationships plus lifecycle, effective-range, actor-pair and scope-shape constraints.
- Additive backfill of one active assignment per existing membership, copying its existing role and exact legacy scope rows while retaining legacy columns.
- Compatibility dual-writes for initial-admin provisioning and legacy membership create/update; role changes retain the prior assignment as Revoked.
- Integration coverage for create, role replacement, scope synchronization, deactivation/reactivation and actor attribution.

API CI, Docker Integration Verification and PostgreSQL Migration Rehearsal are required on the exact PR head. PR #162 remains draft and must not merge until all three pass and the backfill/transaction behavior is reviewed.

**Still outstanding in M2:**

- Introduce the immutable assignment-aware request context and one central permission-plus-scope resolver.
- Migrate every API reader/writer and scope-dependent service listed in the compatibility inventory; do not use independently flattened permission and scope unions.
- Add explicit assignment-management contracts, lifecycle operations, delegation validation, last-admin protections and append-only before/after audit.
- Prove positive and negative cases for scope borrowing, sibling hospitals/departments, multiple tenants, membership disablement and role revocation.
- Complete PostgreSQL migration outcome comparisons and Local real-Keycloak integrated acceptance.

TenantAdmin remains gated on an approved permission/delegation matrix. No new clinical grants, IPD permissions or legacy-column removal are part of PR #162.

**Exit:** no privilege expansion in backfill; all role/scope resolver and management tests prove assignment isolation; cross-tenant, cross-hospital and cross-department tests pass; exact-head API, migration and Local acceptance are green.
### M3 — Identity-provider lifecycle and invitations

**Repositories:** HeapCare.Api; HeapCare.Local; possibly HeapCare.OtpService only after its separate security gates.

- Define provider-neutral identity administration contracts and explicit provider capabilities.
- Implement the local Keycloak adapter first.
- Add tenant-bound, short-lived invitation lifecycle and verified acceptance.
- Add provider account enable/disable, password-recovery delegation and session revocation only where provider support can be verified.
- Keep provider calls outside database transactions; reconcile partial failures with pending states and idempotency.
- Do not store passwords or OTP plaintext, and do not treat email delivery as identity verification.

**Exit:** real-Keycloak tests cover invite, accept, expiry, revoke, deactivation, reactivation and provider-operation failure/retry. Unsupported provider capabilities are reported accurately.

### M4 — Tenant and platform administration UI

**Repositories:** HeapCare.Web; HeapCare.Api; HeapCare.Local.

- Platform onboarding: create Tenant, invite the first TenantAdmin and complete activation.
- Tenant user directory: list memberships, invitation status and active role assignments.
- TenantAdmin workflow: create/manage authorized Hospitals and accepted sites; invite HospitalAdmins and delegated users.
- HospitalAdmin workflow: assign roles only within delegated Hospital/Department scope.
- Display effective access explanations without relying on frontend checks for security.
- Preserve responsive, keyboard and accessible interaction patterns.

**Exit:** end-to-end real-Keycloak browser workflows for PlatformOperator, TenantAdmin and HospitalAdmin pass; forbidden cross-scope operations return server-side denials and auditable outcomes.

### M5 — FacilitySite and branch/clinic scope integration (dependent on ADR-019)

**Repositories:** HeapCare.Api; HeapCare.Web; HeapCare.Local; HeapCare.Doc.

- Resolve the open clinic, branch, campus, department, patient-registration, MRN, staff, scheduling, billing and reporting decisions in ADR-019 / issue #198.
- Only after acceptance, add FacilitySite as an additive registry and backfill each existing Hospital to one Hospital-type site.
- Introduce explicit site role scopes without implicit parent/child permission inheritance.
- Add site context to modules incrementally while preserving current HospitalId contracts.
- Test tenant isolation and sibling-site denial for patient, staff, appointment, clinical, billing and audit workflows.

**Exit:** representative single-hospital, multi-hospital, campus/branch and standalone-clinic scenarios pass PostgreSQL and Local real-Keycloak acceptance.

### M6 — Integrated security and migration acceptance

**Repositories:** HeapCare.Api; HeapCare.Web; HeapCare.Local; HeapCare.Doc.

- Fresh database install and populated database upgrade.
- Migration-script reapplication and EF snapshot/model-drift checks.
- PostgreSQL concurrency, uniqueness, admin-lockout and rollback tests.
- Current API contract compatibility and Web role-assignment journeys.
- Real-Keycloak multi-tenant, multiple-role and identity-lifecycle workflows.
- Audit success, denial, before/after and failure-rollback coverage.
- Review threat model, MFA expectations, access review and applicable privacy/legal requirements.
- Retire legacy role/scope columns only in a separate release after all consumers are migrated and rollback is proven.

**Exit:** all acceptance criteria in ADR-020 are evidenced; no open critical/high authorization or migration defect remains.

## Low-priority future capability — platform tenant portfolio overview

**Priority:** Low; deliberately excluded from M0–M6 critical path.

A designated HeapCare owner/operator may later need an informational, read-only view of customer Tenants and their Hospitals/sites.

- Add a separately assigned platform permission such as platform.tenant-portfolio.read.
- Do not hardcode the owner's email, OIDC subject or account identifier in code.
- Show only approved organization metadata: tenant name/slug/status and hospital/site name/type/status, optionally aggregate counts.
- Exclude patient and clinical data, staff personal data, membership subject identifiers, credentials and operational write actions.
- Do not create tenant membership or impersonation access.
- Enforce read-only API authorization and audit each query.
- Design the API projection so this can be delivered later without coupling it to TenantMembership or clinical authorization.

## Existing cross-workstream backlog

| Item | Repository / issue | Next step |
|---|---|---|
| FacilitySite / branch / clinic hierarchy | HeapCare.Doc #198 | Keep open; resolve operational decisions before schema implementation |
| IPD ADT and bed-management | HeapCare.Doc #200 | Continue approved ADT policy, cancellation/discharge and Web gates separately |
| Manual accessibility acceptance | HeapCare.Doc #201 | Complete native OS/screen-reader acceptance after affected UI changes |
| Production identity and infrastructure | HeapCare.Doc #197 | Deferred until local product backlog and production-readiness review |
| OTP security | HeapCare.Mobile #2 / #3 and HeapCare.OtpService | Blocked/deferred; do not depend on OTP for the identity redesign |

## Repository responsibilities

| Repository | Responsibility |
|---|---|
| HeapCare.Api | Identity, membership, role-assignment, invitation contracts/services, persistence, authorization and audit |
| HeapCare.Web | Platform onboarding, organization user directory, invitations and multiple scoped role assignments |
| HeapCare.Local | Keycloak/PostgreSQL integrated acceptance, migration and browser workflows |
| HeapCare.Doc | ADRs, data model, migration plan, permission matrix and decision register |
| HeapCare.OtpService | Not part of initial identity lifecycle unless separately approved after its security gates |

## Delivery rules

- Work in short-lived branches and focused PRs; do not commit directly to main.
- Do not merge without exact-head CI, PostgreSQL integration, migration rehearsal and required Local acceptance.
- Do not change existing role grants or enable new clinical permissions as a side effect of the membership redesign.
- Do not delete or rename legacy columns in the initial migrations.
- Any unexpected privilege expansion, tenant isolation regression or audit inconsistency blocks merge.
- Keep all work local-first; deployment planning remains deferred under issue #197.

## Progress reporting

Update this plan and HeapCare.Doc roadmap when each milestone changes state. Report implemented code, merged PRs, exact CI/Local evidence, remaining blockers and any decisions requiring hospital/product input. A passing build alone is not milestone completion.
