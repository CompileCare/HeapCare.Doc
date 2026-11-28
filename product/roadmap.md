# HeapCare Delivery Roadmap

**Status:** Working product-first delivery plan, updated 2026-11-28. Phase 5 includes a formal enterprise experience uplift, tracked separately from feature delivery. The 2026-11-25 local full-stack acceptance run is recorded in [Local stack acceptance](qa/local-stack-acceptance-2026-11-25.md). This is a milestone order, not a calendar promise. Exit criteria must pass before a phase is called complete.

HeapCare is a hospital and healthcare information/management platform. This roadmap contains only hospital-system capabilities. The [Product Blueprint v1](blueprint-v1.md) is the source for product scope and patient workflows.

## Product execution decisions — 2026-11-28

- **Local-first delivery:** finish local development, local full-stack integration and the prioritised web/product backlog before selecting production hosting, identity provider, region, paid services or production recovery architecture. Production deployment and provider-native backup/PITR remain future release gates; see [deferred production readiness](../operations/heapcare-production-cutover-inventory-2026-11-28.md) and HeapCare.Doc issue #197.
- **Custom domain:** do not purchase or configure a custom domain during current feature development. Revisit domain/DNS/TLS together with the production deployment decision after the product and currently deprioritised backlog are complete.
- **Mobile and OTP:** the current delivery is web-first. Mobile application planning and OTP login are deferred until the web version is complete. Do not add mock OTP flows or treat the current OTP service scaffold as production-ready. Before any mobile OTP implementation, complete its credential-rotation/revocation and security, provider, key-management, abuse-control, privacy and tenant/patient binding gates.
- **Accessibility acceptance:** manual sign-off is tracked in HeapCare.Doc issue #201. The agreed combinations are Windows 11 + Chrome + NVDA; Windows 11 + Firefox + NVDA; Windows 11 + Edge + JAWS; macOS + Safari + VoiceOver; macOS + Chrome + VoiceOver; and Ubuntu Linux + Firefox keyboard-only. Brave receives a separate Chromium-family smoke check. Target WCAG 2.2 Level AA. Automated browser checks are supporting regression evidence, not manual AT or conformance sign-off. Web PR #100 is merged and CI run #46100412610 passed the existing Chromium suite plus public accessibility smoke tests in Playwright Firefox and WebKit on Ubuntu; this adds engine-level regression coverage only and does not complete the OS/browser/screen-reader matrix.

## Phase 1 — Backend foundation

**Status: Completed and merged**

1. API foundation: ASP.NET Core/.NET 10 host, middleware, tenant-context plumbing, health checks, versioned API structure and initial endpoint foundation.
2. Shared observability: structured logging, tracing, metrics, correlation IDs and health instrumentation through HeapCare.Observability.

**Evidence:** HeapCare.Api and HeapCare.Observability repositories have merged implementations. The API currently targets .NET 10 and consumes the shared Observability packages.

## Phase 2 — Developer platform

**Status: Completed and merged — shared client, database operations and caching baseline**

HeapCare.ApiClient provides named/typed client registration, validated options, injectable access-token/tenant/correlation providers, outbound handlers, bounded safe-method resilience and RFC 9457 Problem Details parsing. The correlation header handler validates provider values and replaces untrusted/unsafe caller values.

HeapCare.Database establishes PostgreSQL/EF migration ownership, deployment/release, backup/restore and operations runbooks. HeapCare.Caching has a .NET 10 HybridCache package with optional Redis, validated configuration, explicit tenant/global cache-key namespaces and a tested sample (Caching PR #2; restore/build/test/pack passed).

**Follow-up release hardening:** publish/inspect packages and validate them from consuming applications; expand OpenAPI response annotations and contract validation as modules are introduced. Provider-native scheduled backups/PITR and production recovery drills remain; the repository now includes guarded logical backup/restore helpers and PostgreSQL 16 Docker verification.

## Phase 3 — Database, identity and security

**Status: Foundation and identity-context milestones implemented and merged — persistence, OIDC/tenant RBAC, request audit, tenant registry/lifecycle, trusted membership provisioning, organization scopes and current-user tenant discovery are implemented. Audit and production operational hardening continue in parallel with Phase 5.**

### Implemented foundation

1. PostgreSQL is the active provider in HeapCare.Api; EF Core 10 is the persistence and migration mechanism.
2. Tenant-aware facility persistence, query filters, scoped uniqueness, composite tenant foreign keys, restrictive deletes, audit timestamps and concurrency foundations are implemented.
3. The InitialFacilityHierarchy migration and model snapshot cover Hospital → Department → Ward → Room → Bed.
4. Integration fixtures apply versioned migrations to an isolated test database; CI checks model drift.
5. OIDC/JWT Bearer authentication validates signature, issuer, audience, lifetime, expiry and subject. The validated issuer and OIDC subject are used as the external identity key.
6. Tenant-scoped routes resolve an active membership matching the validated identity and tenant; a tenant header alone grants no access.
7. Facility permissions are derived from persisted membership roles; caller-supplied role/permission claims cannot elevate privileges.
8. Append-only tenant-scoped request audit events and PostgreSQL-backed tests are merged.
9. Global tenant registry and lifecycle are implemented. Tenants have unique normalized slugs and Provisioning/Active/Suspended/Closed states.
10. Trusted initial administrator provisioning and tenant/hospital/department membership scopes are implemented, including protections against self-modification and removing the last active tenant-wide HospitalAdmin (API PR #13).

### Remaining work — recommended order

1. **Audit and tenant-isolation hardening:** API PR #24 (merge commit `700785110f26ce17f2aee5dc456ec8965a37668f`) covers anonymous access denial and malformed tenant-context rejection. API PR #25 (merge commit `ffb5c128932de12356b9e907aab5f8414477c7c0`) covers privileged initial-administrator provisioning. API PR #26 (merge commit `dbf0f0cc3af5da90b29255fbca37fbb419b78134`) adds audited current-user tenant-context discovery by validated issuer/subject, with active membership and tenant filtering. API PR #44 verifies metadata-only audits for clinical and billing actions; PR #45 verifies denied cross-department clinical reads; PR #46 verifies cross-tenant patient-read denial and audit attribution; PR #47 verifies cross-tenant billing confidentiality; PR #48 verifies OPD encounter confidentiality; PR #49 verifies appointment and queue isolation; and PR #50 verifies staff-profile and assignment isolation. API PR #51 adds patient identifier/MRN confidentiality coverage (present on API main at `222c5569fa3019ee62b865cb14e3f3c2fad6978e`; CI #798/#799 and Docker #166 passed). API PR #53 adds hospital-local duplicate-candidate isolation and audit coverage (present on API main at `8dadb4cb1865769d67d4d5e6a81309364c543301`; CI #806/#807 and Docker #169 passed). API PR #54 adds PostgreSQL integration coverage for tenant-membership role/scope updates, deactivation/reactivation and rejected self-demotion, asserting metadata-only audit attribution and unchanged persisted administrator state; it was merged on 2026-11-26 after hosted CI and Docker/PostgreSQL verification passed. API PR #55 addresses local acceptance finding SEC-002 by validating hospital visibility before patient search and department collection listing, with PostgreSQL regression coverage for tenant-wide foreign-hospital and out-of-scope requests; it was merged on 2026-11-26 after .NET CI and Docker Integration Verification passed (runs 45739052864 and 45739052930). API PR #61 and #62 add cross-hospital and same-hospital cross-department OPD denial audit coverage; API PR #64 adds audit attribution for department-scoped patient search/detail denials; API PR #65 adds audit attribution for department-scoped invoice and payment-history denials; API PR #66 adds same-hospital cross-department isolation coverage for all ten appointment/queue operations, including availability, booking, read, cancel, reschedule, check-in, queue listing/transition and walk-in registration, with rejected audit attribution and resource identifiers. API PR #67 adds PostgreSQL failure injection proving patient registration and its hospital registration roll back with HTTP 503 when audit-event persistence fails; CI and Docker verification passed. API PR #68 adds an endpoint-registration guard asserting every authenticated route carries tenant, user or platform-operator audit metadata, including metadata inherited from route groups; Release build, EF model-drift, integration tests and Docker verification passed. API PR #69 strengthens that guard by asserting route discovery is non-empty, preventing a false-positive pass when no endpoint sources are exposed to the test host; Release build and Docker verification passed. Continue with additional sensitive record access and material clinical/business actions. Controlled tenant-wide audit query/export is implemented in API PRs #71/#72; define retention/deletion, export monitoring, operational access reviews and privacy/legal approval before customer activation.
2. Continue tenant isolation and authorization tests across the synchronous routes and persistence paths that exist today. API PR #90 covers facility read/create/delete outcomes for all five persisted tenant roles; PR #92 adds audited billing denials for FacilityViewer and ClinicalUser; PR #93 adds audited scheduling denials for those roles. All three PRs passed hosted CI and Docker Integration Verification and are merged. API PR #94 adds staff profile/assignment denials for FacilityManager, FacilityViewer, PatientRegistrar and ClinicalUser; it merged on 2026-11-27 as `e72af18ac6b70aef462d8d554e9f6d66c030cd63` after hosted PR CI, push CI and Docker Integration Verification all passed. API PR #95 adds PostgreSQL-backed patient identifier/MRN role-denial coverage: FacilityManager, FacilityViewer and ClinicalUser are denied policy and identifier operations, while PatientRegistrar retains identifier read/assignment but cannot read or manage the hospital policy. It merged on 2026-11-27 as `62e73a0577d0dd10e69a1d480290d3c1e3210f30` after hosted PR CI, push CI and Docker Integration Verification passed. These are focused capability slices, not the complete per-capability role matrix. Require the same tenant/organization binding, metadata-only audit and cross-tenant regression tests when background jobs, messaging consumers, cache integrations, file/object storage, export pipelines, search indexes or support tooling are introduced; do not create those subsystems solely to satisfy this checklist.
3. A disposable PostgreSQL 16 CI rehearsal now validates base-to-upgrade, fresh install and idempotent migration-script reapplication (API PR #73); production migration rehearsal against representative data and an approved release process remain. Provider-neutral logical backup/restore helpers and Docker verification are implemented (Database PR #4). Database PR #6 merged as `9ee88a71eb167b8d005e4f0966baf98d2193987f`, adding a provider-neutral production recovery plan and isolated restore-drill evidence template. Its PostgreSQL Backup Restore Verification workflow run `45863881845` passed shell validation and the disposable PostgreSQL 16 Docker backup/restore regression. This proves the repository helper workflow only; it does not configure a provider or prove production recovery. Provider-native physical backups/PITR, encrypted off-site storage, retention, approved RPO/RTO, monitoring and successful provider-native restore drills remain release gates. Define signing-key rotation using the [OIDC signing-key rotation runbook](../security/oidc-signing-key-rotation-runbook.md), production secrets, encryption, retention/deletion, incident response and access reviews.
4. The initial application threat model is recorded in [Threat model](../security/threat-model.md). Validate it with the pilot hospital, complete suitable independent penetration testing and qualified privacy/legal review for applicable Indian requirements. Do not claim regulatory compliance from implementation or documentation alone.

**Authorization review artifacts:** the API policy and endpoint inventory is recorded in [authorization policy inventory](../security/authorization-policy-inventory.md), and the current source-derived grants are recorded in the [persisted role-to-permission matrix](../security/persisted-role-permission-matrix.md). The matrix is an implementation snapshot, not product-owner approval. It must be reconciled with approved hospital/reception/clinical responsibilities and complete positive/negative PostgreSQL integration coverage before pilot approval.

**Phase 3 exit criteria:** reviewed migration install/upgrade paths; authenticated identity bound to active trusted tenant membership; organization-scope authorization at relevant tenant entry points; append-only audit coverage for approved sensitive/privileged actions; cross-tenant tests for synchronous and asynchronous paths; and reviewed production operational controls. Core security and membership capabilities are implemented; operational and cross-cutting validation remains before calling this phase production-ready.

## Phase 4 — Core healthcare domain

**Status: Partially implemented — outpatient patient, staff/provider, appointment, queue, OPD and billing foundations are merged. The initial IPD ADT/bed-management API foundation is also merged: persistence, guarded admission creation, paginated/detail reads, same-hospital bed assignment/transfer and audit/isolation hardening. All IPD routes remain permission-gated with no existing role grants. Real-Keycloak negative authorization acceptance has passed in Local CI. The complete hospital-approved admission-to-discharge workflow, cancellation/discharge policy and services, six-role permission mapping and Web IPD workspace remain outstanding; see ADR-018 and issue #200.**

### Implemented

1. Hospital → Department → Ward → Room → Bed hierarchy with CRUD, bounded pagination, tenant-scoped relationships and persistence constraints. Backend CRUD and the permission-aware Web administration workspace are implemented and merged. Web PR #34 merged as `c8a5a5ec9a31e85cab05d60d05e8852616319be3`; final hosted CI runs 45744960504 and 45744951178 passed 67 Vitest tests, all 14 Chromium E2E tests, production build and Docker smoke. Authenticated local-stack acceptance remains pending.
2. Tenant-owned patient master and hospital registration, with hospital-scoped search, authorization and PostgreSQL-backed tests (API PR #14).
3. Hospital-configured MRN policy and concurrency-safe, idempotent MRN assignment/lookup (API PR #15). Patient GUID remains the internal identity; MRN is hospital-local.
4. Required patient registration and MRN workflows are documented in ADR-008/009 and the patient-registration workflow.

### Remaining delivery order

1. **Organization boundary accepted:** a Tenant represents the customer/hospital organization and may operate one or multiple hospitals, branches/campuses and clinics; no separate HospitalGroup entity is required. The current Hospital record remains a physical hospital. Facility/site modelling for branches and clinics, their scope grants and within-tenant patient visibility remain active design work; see ADR-006.
2. **Patient identity — normalized candidate matching completed; contact design is next.** Patient master, hospital registration, hospital-local MRN and advisory duplicate matching are merged through API PRs #14–#17. The normalized matcher is on API main at c603dc602565b69f25b24b4738e45f24c4090362; post-merge build, EF model-drift check, 12 unit tests and 58 PostgreSQL integration tests passed. Patient-owned phone/email contact lifecycle, protected storage, verification, permissions and matching limits are designed in ADR-012, merged through documentation PR #19. No contact API/schema/runtime implementation is included. Implementation remains gated on approved security, privacy, key-management and retention decisions; contact corroboration may only be considered later for existing name/DOB candidates. Cross-hospital discovery, guardian relationships, emergency reconciliation and governed identity workflows remain separate.
3. **Staff and provider management — initial foundation merged:** tenant-owned profiles, effective-dated hospital/department assignments, bounded staff search and scoped staff permissions are implemented in HeapCare.Api PR #18. Follow [ADR-013](../architecture/decisions/ADR-013-staff-and-provider-profile-boundaries.md) and the [staff/provider workflow](workflows/staff-and-provider-management.md). Profile-to-membership linking, credential verification and clinical privileging are deliberately not included; profiles and designations do not independently grant access or clinical authority.
4. **Appointments, walk-ins, check-in and queue — implemented and merged:** HeapCare.Api PR #20 delivers appointment rescheduling with persisted lineage, idempotent check-in, walk-in registration, department/day queue tokens and queue lifecycle transitions. CI, EF model snapshot validation and PostgreSQL Docker integration verification passed before squash merge (commit `c968d9441affaa21070063081b2bc785c848e547`). Follow [ADR-014](../architecture/decisions/ADR-014-appointment-scheduling-and-queue-boundaries.md) and the [appointments and queue workflow](workflows/appointments-and-queue.md).
5. Patient-owned contact storage and verification following [ADR-012](../architecture/decisions/ADR-012-verified-patient-contact-data-and-matching.md), only after security/privacy/provider/key-management and retention decisions are approved. Initially allow only corroboration of existing exact-DOB/name candidates.
6. **OPD encounter and clinical documentation — implemented and merged:** HeapCare.Api PR #21 adds queue-linked encounters, scoped ClinicalUser permissions, draft notes, signed-note immutability, append-only amendments and a bounded lifecycle. CI, EF model-drift validation, PostgreSQL integration tests and Docker Integration Verification passed before squash merge (commit `723a107c0de40e95cf28f1c12a772f1ab2a75c73`). Follow [ADR-015](../architecture/decisions/ADR-015-opd-encounter-and-clinical-documentation-boundaries.md) and the [OPD encounter workflow](workflows/opd-encounter-and-clinical-documentation.md). Production activation remains gated on clinician, privacy, role and retention validation.
7. **Minimal OPD billing — API and Web pilot foundation implemented and merged:** HeapCare.Api PR #23 delivered the initial catalogue, visit-linked invoice, payments, receipts and daily collection report. API PR #39 added explicit `Issued`, `PartiallyPaid` and `Paid` settlement states with a PostgreSQL migration; API PR #40 added scoped invoice-by-visit lookup; API PR #41 added regression coverage for duplicate issuance, department isolation and concurrent payment overage; API PR #43 added department-scoped payment history for receipt recovery. Web PRs #10–#11 deliver the permission-driven billing workspace, payment history, receipt re-open and print. Web PR #12–#13 add browser E2E coverage for billing recovery and the full walk-in → clinical → billing journey. API PR #44 verifies metadata-only audit events for clinical/billing actions and confirms no clinical note or payment reference is stored. API CI #767/Docker #158, API CI #771/Docker #159, Web CI #154, Web CI #168 and Web CI #175 passed. Statutory tax, discounts/waivers, refunds/credit notes, insurance/TPA, gateway/accounting integrations, deposits/advances and advanced cashier reconciliation remain deferred pending pilot decisions. Follow [OPD billing foundation](workflows/opd-billing-foundation.md).
8. **IPD ADT and bed management — API foundation merged; positive hospital workflow remains gated:**
   - **PR #134:** tenant/hospital-scoped admission and bed-placement persistence, same-hospital registration FK, composite facility relationships, filtered uniqueness for active admissions/placements/bed occupancy, retained placement history and PostgreSQL migration.
   - **PR #135:** physical-delete guards and dedicated IPD permissions/contracts; no existing role grants.
   - **PR #136:** scoped admission read, first-bed assignment and same-hospital transfer services with department checks and concurrency protection.
   - **PR #137:** read, bed assignment and transfer HTTP routes behind dedicated permissions; every current tenant role and PlatformOperator remains denied.
   - **PR #138:** denied admission-read audit attribution.
   - **PR #139:** guarded admission creation for a patient registered at the exact Hospital and Tenant, requiring Tenant-wide or Hospital-wide scope.
   - **PR #140:** permission-gated paginated admission list; Department-only visibility is based on current placement and history is filtered by department.
   - **PR #141:** Hospital-scoped admission-list isolation from sibling Hospitals under the same Tenant.
   - **PR #142:** successful admission-create audit resource is the generated Admission ID, not the parent Hospital ID.
   - **PR #143:** bed-transfer attempts to sibling Hospitals or foreign Tenants return non-disclosing 404 and preserve the original placement.
   - **PR #144:** concurrent transfers from one active placement produce one success and one conflict, with one active destination and retained movement history.
   - **PR #145:** transfer into an occupied bed returns 409 and preserves both original placements.
   - **PR #146:** denied bed assignment/transfer HTTP requests have metadata-only audit attribution and no placement side effects.
   - **PR #147:** admission detail is isolated across sibling Hospitals and foreign Tenants.
   - **PR #148:** successful bed assignment and transfer commit placement history and audit events in the same request transaction.
   - **PR #149:** Department-scoped readers see only placement history in their authorized department.
   - **PR #150:** audit-insert failure returns 503 and rolls back bed assignment/transfer changes.
   - **PR #151:** successful admission creation commits the admission and metadata-only audit event together; audit-insert failure returns 503 and rolls back the admission. This is test-only and introduces no runtime, schema, migration or permission change.
   - **Local PR #46:** real-Keycloak negative authorization covers every then-current IPD route for all five tenant roles and platform-only PlatformOperator.
   - **Local PR #48:** current-main nine-service verification covers IPD schema, real-Keycloak negative authorization, observability, PostgreSQL logical backup/restore and Chromium/Firefox/Playwright WebKit acceptance. This is automated Linux CI evidence, not positive IPD workflow approval or manual accessibility certification.
   - **Policy documentation:** HeapCare.Doc PR #222 adds the [IPD Hospital Policy Decision Worksheet](workflows/ipd-hospital-policy-decision-worksheet.md), covering admission sequencing, mandatory data, cancellation/discharge boundaries, bed readiness/reservation, six-role permissions and site-level scope. All options remain proposals pending hospital approval.

   **Validation evidence:** API PR #151 passed Release build, EF model-drift guard, 12/12 unit tests and 259/259 PostgreSQL integration tests. Both build/test workflows and Docker Integration Verification passed. The PR did not change the schema or migration chain.

   **Remaining:** hospital approval of admission-before-bed-assignment semantics, mandatory admission data/reasons, six-role positive/negative permission mapping, cancellation/discharge eligibility and reasons, and bed reservation/readiness rules; then implement lifecycle services/routes, run positive real-Keycloak acceptance, and build the Web IPD workspace. No IPD role grants have been introduced. Medication administration, nursing/clinical orders, pharmacy, diagnostics, inventory, patient portal and payer integrations remain separately gated. See [ADR-018](../architecture/decisions/ADR-018-inpatient-ADT-bed-management-boundaries.md), [ADR-019](../architecture/decisions/ADR-019-facility-site-model-and-scope-inheritance.md), the [IPD workflow](workflows/inpatient-admission-transfer-discharge-and-bed-management.md), the [policy worksheet](workflows/ipd-hospital-policy-decision-worksheet.md) and HeapCare.Doc issues #198/#200.

### Deferred workforce, credentialing and HR roadmap — retained for later

These capabilities are intentionally outside the initial staff-profile increment and the first outpatient MVP. They remain planned product areas and must not be dropped from future planning:

1. **Workforce identity:** approved staff-profile-to-membership linking and the employee self-service foundation. A profile is not an account; linking requires a verified identity and an attributable approval trail.
2. **Credentialing and clinical privileging:** qualification and credential evidence, issuing-body verification, licence validity/expiry, renewal and facility-specific privileges. Provider designation alone never authorizes clinical practice; clinical permissions must depend on verified credentials and active privileges.
3. **Workforce operations:** rosters, shifts, attendance, leave and related approval workflows, designed as explicit workforce capabilities rather than extra staff-profile fields.
4. **HR and finance:** compensation, payroll, payslips and payroll integration. Prefer integration with established payroll providers before building a full payroll engine unless pilot requirements justify otherwise.
5. **Secure employee records:** sensitive identifiers and employment documents in a dedicated, encrypted and access-restricted employee-record/document capability, with purpose limitation, audit and retention controls. Do not expose these fields in ordinary staff APIs or telemetry.
6. **Cross-tenant workforce:** governed multi-hospital/cross-tenant staff sharing and assignments only after ownership, consent, legal basis, authorization, audit and tenant-isolation rules are explicitly approved.

**Provisional pilot defaults to revisit with the hospital:** collect only operationally necessary profile fields (staff code, name, designation and only the contact details required for the workflow); permit concurrent assignments only where explicitly allowed by policy and validate conflicting effective dates; keep profile create/update/deactivate permissions with HospitalAdmin initially and make account linking a separately restricted action; prefer a tenant-configurable designation catalogue; require verified identity plus an auditable approval process before membership linking; and confirm applicable Indian legal requirements and the pilot hospital's retention policy before collecting employment records or sensitive identifiers. These are planning defaults, not legal advice or final product approvals.

**Next engineering hardening increment:** extend audit and tenant/organization-scope verification to additional sensitive clinical record reads and material clinical/business actions. API PR #61 covers cross-hospital OPD encounter denials; PR #62 covers same-hospital cross-department encounter read/update/sign/create denials; PR #64 covers department-scoped patient list/detail denials; PR #65 covers department-scoped invoice and payment-history denials; PR #66 covers same-hospital cross-department appointment and queue operations. PR #68 verifies all authenticated routes are included in the audit middleware contract. These tests validate actor/tenant audit attribution. This remains a Phase 3 production-readiness gate; controlled audit query/export, retention and operational access policies are still outstanding. Proposed guardrails and implementation gates are documented in [ADR-016: Audit review access, export and retention governance](../architecture/decisions/ADR-016-audit-review-access-and-retention-governance.md); hospital/compliance validation is required before customer-facing audit browsing or export is implemented.

Each domain workflow requires hospital-user validation, permissions, audit, tenant scope, acceptance criteria, migration and automated tests.

## Phase 5 — User experience

**Status: In progress — authenticated reception, patient registration/MRN, appointment booking, walk-in arrival, queue operations, OPD clinical workspace, permission-driven feature gating, OIDC silent renewal and the initial billing workspace are implemented and merged. Browser E2E now covers billing recovery, the full walk-in-to-billing path, appointment booking/check-in and successful OIDC silent renewal.**

**Latest integrated accessibility/security regression evidence — 2026-11-28:** Local PR #45 merged read-only six-role authenticated accessibility smoke coverage on Chromium, Firefox and Playwright WebKit. Its exact-head full-stack run [#46106532457](https://github.com/HeapCare/HeapCare.Local/actions/runs/46106532457) passed the nine-service stack, real-Keycloak PKCE Chromium journeys, Firefox/WebKit role baselines, PostgreSQL logical backup/restore and cleanup. Local PR #46 then added real-Keycloak denial assertions for every IPD route for all five persisted tenant roles and the platform-only PlatformOperator; exact-head full-stack run [#46114200037](https://github.com/HeapCare/HeapCare.Local/actions/runs/46114200037) passed the nine-service stack, persisted-role verification, backup/restore, Chromium PKCE, Firefox/WebKit baseline and cleanup. These are automated regression checks only. Native Windows/macOS desktop testing, Microsoft Edge/Apple Safari, NVDA/JAWS/VoiceOver, manual keyboard workflows, contrast and zoom/reflow measurement and qualified WCAG 2.2 AA review remain outstanding under issue #201.

The first UX foundation increment is documented in [Reception and clinical workspace UX](workflows/reception-and-clinical-workspace-ux.md). It maps reception, queue and clinical tasks to the implemented patient, appointment, queue and OPD contracts; defines screen responsibilities, access/data boundaries, accessibility and failure states. Billing is now included in the delivered OPD slice; patient self-service and unimplemented clinical capabilities remain deferred.

The local developer environment is maintained in `HeapCare.Local`. Its setup and troubleshooting are documented in [Local development setup](../engineering/local-development-setup.md). The first-run platform bootstrap and hospital onboarding changes are merged: API PR #32 adds the Development-only one-time persisted platform-operator bootstrap; Local PR #4 restores explicit profile/email OIDC scopes and corrects sibling build contexts; Web PR #8 adds the guided first tenant, initial administrator, activation, hospital and department workflow. Their CI/PR work is merged; the remaining acceptance item is hands-on Docker/OIDC smoke validation in the developer Mac environment, which cannot be confirmed by GitHub CI alone.

The organization has a dedicated `HeapCare.Web` repository, separate from the public-facing `HeapCare.Site`. The authenticated React + TypeScript workspace shell is implemented and merged (Web PR #1, merge commit `af37ecba2d8224474964f9052b394cb9dca6fd6b`). The first reception vertical slice is also implemented and merged (Web PR #2, merge commit `c83a5583d81be91e5f123d836aef5f8d51cad067`): it consumes the current-user tenant-context contract, allows explicit tenant and API-filtered hospital selection, and supports hospital-scoped patient name search with pagination. The API context contract is merged in HeapCare.Api PR #26. Patient results remain in component memory, requests use no-store, and the API revalidates every permission and scope. Patient registration with a hospital-local duplicate-candidate review and MRN assignment is merged in [HeapCare.Web PR #3](https://github.com/HeapCare/HeapCare.Web/pull/3), with paired documentation in [HeapCare.Doc PR #30](https://github.com/HeapCare/HeapCare.Doc/pull/30). The UI uses the existing patient registration, duplicate-candidate and identifier APIs; it does not add contact capture, patient merge or cross-hospital discovery. The appointment and arrival vertical slice is merged in [HeapCare.Web PR #4](https://github.com/HeapCare/HeapCare.Web/pull/4), with paired documentation in [HeapCare.Doc PR #31](https://github.com/HeapCare/HeapCare.Doc/pull/31). It delivers selectable patient results, server-published availability and booking, walk-in registration, check-in confirmation with server-issued queue token, and department queue visibility. The queue operations UI is merged in [HeapCare.Web PR #5](https://github.com/HeapCare/HeapCare.Web/pull/5), with paired workflow and screenshot evidence documentation in [HeapCare.Doc PR #32](https://github.com/HeapCare/HeapCare.Doc/pull/32). It provides authorized call, recall, start, complete, cancel and no-show actions using the server-side queue transition contract. The OPD clinical workspace is merged in HeapCare.Web PR #9. It provides clinical queue discovery, queue-linked encounter creation/resume, draft note entry, signing, append-only amendments, cancellation and closure using the existing clinical APIs. API PRs #35–#38 added the related Reception/HospitalAdmin permission corrections, department-scoped patient search and detail reads, and server-derived tenant permission discovery. Web PR #9 uses those permission identifiers for module/action visibility and enables OIDC silent session renewal; the API remains the authorization boundary. ClinicalUser receives patient.identity.read to verify only the patient linked to an encounter without granting general patient search, plus scoped facility/queue read permissions to discover departments and operational tokens; it does not receive queue mutation permissions. Follow the [HeapCare experience quality bar](design/experience-quality-bar.md) for visual hierarchy, accessibility, responsive behavior and operational feedback. Continue Phase 3 audit/tenant-scope and production-operational hardening in parallel as a release-readiness track. Mobile remains deferred until validated use cases and identity/API contracts justify it.

### Latest Phase 5 delivery and next increment — 2026-11-25

The initial OPD vertical slice now includes reception and patient registration/MRN, appointments and queue operations, department-scoped clinical identity/encounter access, permission-driven feature gating, OIDC session renewal and the OPD billing workspace. Current delivery evidence: HeapCare.Api PRs #35–#53 and HeapCare.Web PRs #9–#26, with paired workflow documentation. API PR #44 verifies metadata-only audit for patient identity reads, encounter creation/note/sign/close and payment capture while excluding note text/payment references (CI #771, Docker #159); PR #45 verifies denied cross-department clinical reads are audited (CI #775, Docker #160); PR #46 verifies cross-tenant patient-read denial and audit attribution (CI #779, Docker #161); PR #47 verifies cross-tenant billing confidentiality (CI #783, Docker #162); PR #48 verifies OPD encounter confidentiality (CI #786/#787, Docker #163); PR #49 verifies appointment and queue isolation (CI #790/#791, Docker #164); PR #50 verifies staff-profile and assignment isolation (CI #795/#796, Docker #165); PR #51 verifies patient identifier/MRN confidentiality (CI #798/#799, Docker #166); and PR #53 verifies tenant isolation in duplicate-candidate search (CI #806/#807, Docker #169). Web PR #12 tests billing recovery/permission gating (CI #168); PR #13 tests the walk-in → queue → clinical → billing journey (CI #175); PR #14 tests appointment booking/check-in (CI #181); PRs #15–#18 cover successful OIDC renewal, slot-conflict recovery and re-authentication after renewal failure/expiry (CI #185, #191, #195 and #199); PR #19 adds visible keyboard focus, reduced-motion support and narrow-viewport overflow checks (CI #210). Web PRs #20–#26 extend keyboard-only Chromium coverage across reception search, clinical encounter entry, billing visit/invoice/payment/receipt workflows, appointment booking/check-in, queue call/start and patient registration/MRN assignment. The final relevant browser checks passed on Web CI #214, #218, #246, #247, #234, #254 and #257. The cross-tenant MRN and duplicate-search tests are present on API main at commits `222c5569fa3019ee62b865cb14e3f3c2fad6978e` and `8dadb4cb1865769d67d4d5e6a81309364c543301`; the keyboard patient-registration test is present on Web main at `4dc3a991eafe625d80bae887886283aa722fef94`.

**Next recommended increment:** execute the authenticated-workspace accessibility review across PlatformOperator, HospitalAdmin, FacilityManager, FacilityViewer, PatientRegistrar and ClinicalUser workflows using the acceptance matrix in [Phase 5 authenticated accessibility acceptance](../qa/phase5-authenticated-accessibility-acceptance.md). The current automated suite covers the public workspace and keyboard-focused OPD journeys; it is not a full authenticated accessibility or WCAG 2.2 AA assessment. Continue in parallel with Phase 3 operational hardening: verify end-to-end keyboard completion, focus order, dialog behavior, semantic labels/live announcements, contrast and screen-reader output at desktop, tablet and mobile sizes. PRs #19–#26 provide an automated and keyboard-navigation baseline, not a complete WCAG 2.2 AA conformance claim. In parallel, continue Phase 3 operational hardening: migration deployment/rehearsal, backup/restore automation and recovery drills, signing-key rotation, secret management, encryption, retention/deletion, incident response and access reviews. Require equivalent tenant/organization binding, audit and isolation tests when background jobs, exports, caches or other asynchronous integrations are introduced; do not create those subsystems solely to satisfy a checklist. Exercise the local Docker/OIDC onboarding path on the developer Mac before declaring environment acceptance; GitHub CI cannot replace this environment-specific smoke test. Keep staff/provider UI gated on hospital-user workflow validation; do not invent workforce policy. Do not start GST/tax, refunds, credit notes, TPA, gateway or accounting integration until pilot policy decisions are recorded.


## Active release-readiness work — status 2026-11-26

The local acceptance findings and follow-up PRs have been rechecked. The GitHub Actions runner-allocation issue that previously produced `runner_id=0` and zero-step jobs is resolved. All 11 HeapCare repositories are now public, hosted jobs receive runners and execute actual tests and Docker verification, and issue [HeapCare.Api #56](https://github.com/HeapCare/HeapCare.Api/issues/56) is closed as completed.

### Merged after executed verification

- **HeapCare.Api PR #54 — membership audit coverage:** merged after the PostgreSQL integration suite passed. It verifies membership role/scope updates, deactivation/reactivation, rejected self-demotion and metadata-only audit attribution.
- **HeapCare.Api PR #55 — inaccessible hospital collection contract:** merged as `07fb9cea2dc6f706fc5c1a6fff67850f585b79d9`. CI run 45739052864 passed restore, Release build, EF model-drift validation and unit/integration tests; Docker Integration Verification run 45739052930 passed.
- **HeapCare.Database PR #2 — logical backup/restore:** merged as `97bafc795ddae67168c88462d35ecc9fcfc30e27` after PostgreSQL Docker backup/restore verification run 45738452765 passed. This remains logical backup/restore tooling, not physical backup/PITR or a production recovery drill.
- **HeapCare.Caching PR #5 — tenant cache isolation tests:** merged after the tenant/global namespace isolation workflow passed.
- **HeapCare.Local PR #5 — local Npgsql GSSAPI warning:** merged; the connection-string setting remains scoped to the local Docker configuration.
- **HeapCare.Local PR #7 — Keycloak silent renewal:** merged as `6fc76effa728a36529ddcc25eaf051cab8412068`. Local CI run 45738522025 passed realm validation, Docker Keycloak import smoke test and Compose validation. The workflow also tests reconciliation of an existing client with a stale redirect list without deleting its realm volume/users.
- **HeapCare.Local PR #6 — observability overlay:** merged after Docker runtime smoke runs 45740981547 and 45740976578 passed. The smoke started Collector, Prometheus, Jaeger, Elasticsearch and Kibana, sent synthetic OTLP metrics/traces plus a synthetic log, and verified queryability/readiness. A real HeapCare.Api exporter-to-backend check remains local post-merge acceptance.
- **HeapCare.Doc PR #50:** merged as `dffe467a98854b9d688ac75c45bd1a15fba69faf`, paired with Local PR #5.
- **HeapCare.Web PR #27 — cross-platform OPD browser test fixes:** merged after browser E2E and Docker smoke verification passed.
- **HeapCare.Web PR #28 — local Playwright report configuration:** merged after Web CI passed.
- **HeapCare.Web PR #35 — tenant membership administration:** merged as `b8e4a4ec8864884f5e71dc070a319a5f4df07185`.
- **HeapCare.Web PR #36 — platform tenant lifecycle workspace:** merged as `54589758db04848d25e2e078f3619c1932cc645a`. Hosted CI runs 45747601458 and 45747606834 passed 85 Vitest tests across 9 files, 14 Chromium E2E tests, production build and Docker smoke. The workspace supports platform-only tenant creation, initial HospitalAdmin provisioning, activation, suspension, reactivation and terminal closure. It is visible only to persisted platform operators; the API independently authorizes every global operation.

### Completed Web experience and facility PR stack

All eight Web implementation PRs #29–#36 have been merged into `main` in dependency order, using merge commits to preserve the stacked visual/experience branch history; the tenant and platform administration slices were then merged on top:

- **Web PR #29 — enterprise visual foundation:** merged as `37f30508a23add4a8af0918307a524dd9f9f4bdb`; hosted CI 45738737945 passed Vitest, build, Chromium E2E and Docker smoke.
- **Web PR #30 — Reception patient selection:** merged as `f0e548737ac05a1886640be742ed64a32f892e38`; hosted CI 45738814930 passed.
- **Web PR #31 — appointment slot selection:** merged as `0c10cafacb43183b47bc2f18ea0deadf8a90a95e`; hosted CI 45738824899 passed.
- **Web PR #32 — queue-row accessibility:** merged as `68a8c9798a29df34031e2e13d7857550f887546b`; hosted CI 45738835427 passed.
- **Web PR #33 — clinical queue-row accessibility:** merged as `23cf6dcf503fc4e781e9b343d244ea9665566e29`; hosted CI 45739127780 passed, including the OPD journey browser test and Docker smoke.
- **Web PR #34 — permission-aware Hospital → Department → Ward → Room → Bed workspace:** merged as `c8a5a5ec9a31e85cab05d60d05e8852616319be3`. Final CI runs 45744960504 and 45744951178 passed 67 Vitest tests, 14 Chromium E2E tests, production build and Docker smoke on head `b36c7115003384eee8f809511a0b6808b36e76f2`.
- **Web PR #35 — tenant membership administration:** merged as `b8e4a4ec8864884f5e71dc070a319a5f4df07185` after 75 Vitest tests, 14 Chromium E2E tests, production build and Docker smoke passed.
- **Web PR #36 — platform tenant lifecycle workspace:** merged as `54589758db04848d25e2e078f3619c1932cc645a` after 85 Vitest tests, 14 Chromium E2E tests, production build and Docker smoke passed.

### Antigravity acceptance findings — disposition

- **AUTH-005:** code/configuration correction merged in Local PR #7; hosted Keycloak Docker reconciliation validation passed. Repeat the full local sign-in and token-renewal acceptance on the developer Mac after pulling the merge.
- **SEC-002:** corrected and merged in API PR #55; PostgreSQL integration and Docker verification passed. Repeat the two real-stack acceptance cases on the updated local stack.
- **Mocked macOS Playwright failures:** corrected and merged in Web PR #27 using deterministic native-select selection and platform-specific Select All behavior. Hosted Linux browser E2E passed; this is not a claim that a separate macOS runner was executed.
- **Visual foundation regressions:** PR #29 now tests skip-link-first focus and mobile navigation width; hosted browser and Docker checks passed. Authenticated screenshot evidence remains outstanding.
- **Clinical queue accessible-name mismatch:** corrected in PR #33's OPD E2E selector; hosted browser and Docker checks passed.

The eight Not Run scenarios describe the original tested build, not the current main branch. Facility hierarchy CRUD, tenant membership administration, platform tenant lifecycle, staff/provider UI, MRN policy/search UI and appointment availability/list/reschedule/cancel UI are now implemented and merged. Authenticated local-stack acceptance remains pending. Remaining unimplemented/deferred areas include inpatient ADT/bed occupancy/MAR/discharge; LIS/RIS/PACS and pharmacy; inventory; notifications and patient portal; HR/payroll; and ABDM/TPA/insurance integrations. Deliver these in dependency order only after product, security, privacy and operational requirements are recorded. Do not implement patient contact verification, clinical credentialing, payroll, insurance or other sensitive subsystems merely to eliminate the Not Run count.

## Release gates and working rules

- Product workflow and ownership precede module/repository implementation.
- Work in short-lived branches and focused PRs; do not commit directly to main.
- Review root causes and existing architecture before changing implementation.
- Every PR includes related docs, tests, security/privacy implications, migration/recovery notes and CI evidence.
- A phase is complete only when its exit criteria are met and reviewed.
- Keep the modular core simple; introduce separately deployed services only when documented needs justify them.


## Phase 5 experience quality uplift — new priority

**Status: In progress — PRs #27–#36 are merged. The shared visual foundation, Reception, appointment, queue, clinical accessibility, facility hierarchy, tenant membership administration and platform tenant lifecycle slices are now on Web main. Authenticated real-application screenshot evidence remains a post-merge product acceptance item, not an open PR or merge blocker.** The 2026-11-25 local baseline remains 48/50 executed checks (96%), with AUTH-005 and SEC-002 corrected in merged PRs #7 and #55, and eight scenarios historically marked Not Run in that original build; facility hierarchy, tenant membership and platform tenant lifecycle are now implemented, while six deferred scenario groups remain unimplemented. Public-repository Actions runner allocation is working again. This is an improvement to the quality of the existing hospital workspace, not a change to the hospital product scope or backend authorization model.

The current authenticated UI has delivered the first operational workflows, but feature completeness alone is not the product quality bar. The workspace must evolve into a polished, cohesive, enterprise-grade hospital application that feels consistent across Reception, Appointments, Queue, Clinical, Billing, Facilities, Memberships and the separate Platform tenants control plane. The first shell/design-system slice in [HeapCare.Web PR #29](https://github.com/HeapCare/HeapCare.Web/pull/29) includes shared visual tokens, permission-driven active navigation, a patient-registration entry when authorized, a keyboard skip-to-content link, and an explicit active-hospital/workspace heading. Hosted CI passed and the visual foundation merged in `37f30508a23add4a8af0918307a524dd9f9f4bdb`. Real authenticated screenshot evidence remains pending as defined by the screenshot-evidence guide, now tracked as post-merge product acceptance. The facility hierarchy workspace merged in [PR #34](https://github.com/HeapCare/HeapCare.Web/pull/34), tenant Memberships in [PR #35](https://github.com/HeapCare/HeapCare.Web/pull/35), and the platform tenant lifecycle workspace in [PR #36](https://github.com/HeapCare/HeapCare.Web/pull/36). Their hosted CI/Docker checks passed; local authenticated acceptance and screenshot evidence remain required. See [Facility management](workflows/facility-management.md), [Tenant membership administration](workflows/tenant-membership-administration.md), [Platform tenant lifecycle](workflows/platform-tenant-lifecycle.md), [Enterprise experience uplift](design/enterprise-experience-uplift.md) and the existing [experience quality bar](design/experience-quality-bar.md).

### Delivery sequence

1. **Experience audit and baseline:** review every existing authenticated route and workflow at desktop, tablet and narrow viewports; record information hierarchy, density, navigation, responsive behavior, accessibility and consistency issues. Use real application captures with synthetic/local data only; do not fabricate screenshots or introduce mock business data.
2. **Design foundation:** establish shared design tokens for typography, color, spacing, radii, elevation, focus and semantic status; consolidate reusable page headers, buttons, form controls, cards/panels, tables/lists, status chips, alerts, empty/loading/error states and confirmation patterns.
3. **Application shell:** refine navigation hierarchy, active tenant/hospital/department context, page title and action regions, session controls, breadcrumbs where useful, responsive navigation and consistent content width. Preserve permission-driven module visibility and API-authoritative authorization.
4. **Module-by-module refinement:** improve Reception/patient search and registration, Appointments, Queue, Clinical and Billing in focused vertical slices. Prioritize high-frequency staff tasks, readable operational density, predictable keyboard interaction and clear recovery from API errors. The merged appointment-slot refinement in [HeapCare.Web PR #31](https://github.com/HeapCare/HeapCare.Web/pull/31) extracts a reusable selection grid, exposes selected/available/unavailable state, formats times in the provider session timezone and prevents changes during in-flight or confirmed booking state. The merged queue-row refinement in [HeapCare.Web PR #32](https://github.com/HeapCare/HeapCare.Web/pull/32) extracts the row presentation, announces the server-returned status and labels the action group while preserving the existing state/action/permission matrix and required cancellation reason. The merged Clinical UI increment in [HeapCare.Web PR #33](https://github.com/HeapCare/HeapCare.Web/pull/33) extracts the clinical queue row, announces the server-returned clinical queue state and labels the open/resume action by token; identity confirmation and all clinical note lifecycle operations remain unchanged. Facility hierarchy administration is delivered by merged PR #34.
5. **Visual and accessibility verification:** validate desktop, tablet and 375px layouts; complete keyboard-only workflows; review focus order, semantic labels, contrast, status announcements and reduced-motion behavior. Add targeted component/browser tests and attach real screenshots to the paired documentation where safe.

### Design and engineering constraints

- Do not trade clinical clarity, data minimization, accessibility or speed for decoration.
- Reuse existing API contracts and server-returned state; do not invent dashboards, KPIs, clinical data, permissions or workflows.
- The API remains authoritative for identity, tenant/facility scope, validation and business transitions.
- Do not place patient/clinical data in URLs, telemetry, browser storage or screenshots used outside the approved synthetic/local test context.
- Keep visual primitives shared and feature code modular; avoid a wholesale rewrite of the large application component while workflow coverage is still being established.
- Prefer small, reviewable PRs with paired documentation, tests and before/after evidence from the running application.

### Exit criteria for the uplift

The existing workflows share a consistent design language; core tasks are discoverable and efficient; layouts are usable at the supported desktop/tablet/narrow widths; keyboard-only completion and focus behavior are verified; loading, empty, validation, conflict, forbidden and server-error states are coherent; WCAG 2.2 AA is treated as a target for review rather than an unverified compliance claim; and the complete test suite/build/Docker smoke checks pass before each UI slice is merged.

The uplift is incremental. It does not block ongoing Phase 3 security/operational hardening, and it does not authorize new hospital capabilities or changes to authentication and authorization.


## Local acceptance evidence — 2026-11-25

The full Docker Compose acceptance run is documented in [Local stack acceptance](qa/local-stack-acceptance-2026-11-25.md). It is a local QA result, not production/pilot sign-off. The two failed real-stack cases were corrected in API PR #55 (SEC-002, merged after CI/Docker verification) and Local PR #7 (AUTH-005, merged after Keycloak Docker validation). The three macOS-only mocked browser-test failures were addressed in Web PR #27 (merged after hosted browser/Docker verification). Facility hierarchy CRUD UI is merged in Web PR #34, tenant membership administration UI in Web PR #35 and the platform tenant lifecycle workspace in Web PR #36; authenticated local-stack acceptance remains pending for all three. The remaining Not Run scenarios remain planned or unimplemented scope and must not be represented as test failures or completed features. Repeat the local full-stack acceptance on the updated merged commits before pilot sign-off.



### Tenant access administration — 2026-11-26

Tenant-scoped membership administration is implemented and merged in [HeapCare.Web PR #35](https://github.com/HeapCare/HeapCare.Web/pull/35), merge commit `b8e4a4ec8864884f5e71dc070a319a5f4df07185`. The workspace is permission-driven by `membership.manage`, uses the validated OIDC issuer and exact subject, supports role and multi-scope assignment, and provides confirmed deactivate/reactivate operations. The API remains authoritative for scope containment, self-management rejection, last-tenant-wide-HospitalAdmin protection and metadata-only audit.

The workflow is documented in [Tenant membership administration](workflows/tenant-membership-administration.md). ADR-0004 now records the implemented tenant registry, separate persisted platform-operator authorization and membership lifecycle rules.

**Completed:** [HeapCare.Web PR #36](https://github.com/HeapCare/HeapCare.Web/pull/36) merged as `54589758db04848d25e2e078f3619c1932cc645a`. The global platform tenant lifecycle workspace uses the merged self-status endpoint from API PR #57 and the activation guard from PR #58. It is visible only to persisted platform operators; all global operations remain protected by the PlatformOperator API policy. **Next:** authenticated local Keycloak/API/PostgreSQL acceptance for tenant creation, initial administrator provisioning, activation, suspension, reactivation and terminal closure.


### Scheduling concurrency and cross-hospital audit — 2026-11-26

Appointment scheduling now maps PostgreSQL exclusion/unique conflicts and deadlocks (SQLSTATE 23P01, 23505 and 40P01) to HTTP 409 rather than leaking an internal-server error. Concurrent booking keeps exactly one successful reservation; the competing caller must refresh availability and choose again.

- [HeapCare.Api PR #60](https://github.com/HeapCare/HeapCare.Api/pull/60) merged as `b31832ef8a416373532bcfe46b9e171dcc9890e7`. Hosted runs 45748869864 and 45748869951 passed Release build, EF model-drift, 12 unit tests, 97 PostgreSQL integration tests and Docker verification.
- [HeapCare.Api PR #61](https://github.com/HeapCare/HeapCare.Api/pull/61) merged as `ac8ef92aa8063321bc07f838c4e512945f8f2182`. The cross-hospital OPD denial test verifies five unauthorized operations return 404, emit rejected audit metadata with validated actor/tenant attribution, and leave the owner's clinical note unchanged. Hosted runs 45749260863 and 45749260926 passed the same Release/model-drift/unit/integration/Docker gates.
- [HeapCare.Api PR #62](https://github.com/HeapCare/HeapCare.Api/pull/62) merged as `ceac287c3d03d92fadc408a4c20f76cb4e223b9e`. Department-scoped ClinicalUser isolation now verifies five same-hospital, cross-department encounter operations return 404, are audited with validated actor/tenant attribution, and cannot mutate or sign the owning department's note. Hosted CI runs 45749980410 and 45749962698 passed Release build, EF model-drift, 12 unit tests and 98 PostgreSQL integration tests; Docker run 45749980403 passed.

The appointment and cross-hospital security workflows are documented in [Appointments and queue](workflows/appointments-and-queue.md) and [ADR-014](../architecture/decisions/ADR-014-appointment-scheduling-and-queue-boundaries.md). Authenticated local Keycloak/API/PostgreSQL acceptance remains required before pilot sign-off.

- [HeapCare.Api PR #64](https://github.com/HeapCare/HeapCare.Api/pull/64) merged as `8b51c3e0058b5daa5215ed5d4234fbc554078ca2`. Department-scoped PatientRegistrar and ClinicalUser denials for hospital-wide patient search, cross-department patient detail and search now assert rejected audit metadata with validated issuer/subject, requested tenant and 404 status. Hosted CI runs 45750443111 and 45750425883 passed Release build, EF model-drift, 12 unit tests and 98 PostgreSQL integration tests; Docker run 45750443058 passed.

- [HeapCare.Api PR #65](https://github.com/HeapCare/HeapCare.Api/pull/65) merged as `7660a2761b2a638c8e42461a4c3a6c86ab073073`. The department-scoped billing integration test now asserts that unauthorized visit-invoice and payment-history reads return 404 and produce rejected Billing audit records with validated actor/tenant attribution. Hosted CI runs 45750909327 and 45750890352 passed Release build, EF model-drift, 12 unit tests and 98 PostgreSQL integration tests; Docker run 45750909307 passed.


## Reception MRN search update — 2026-11-26

HeapCare.Web PR #37 is merged as `150a61a20e824f3e82f4450bc76e536a390417fd`. Reception now supports exact hospital-scoped MRN lookup alongside patient-name search. The MRN mode is visible only to memberships with `patient.identifier.read`; lookup uses the selected tenant and hospital, disables caching/credential forwarding through the existing API client, and does not persist patient search terms or results in browser storage. Switching tenant/hospital or losing identifier-read permission clears the MRN mode and search state. Hosted Web `build-test` runs 45752346956 and 45752342891 passed on head `a9a44eaf0d1f38d6336fc23dc2478bf050b77acd`, including Vitest, Chromium E2E, production build and Docker smoke.

**Follow-up:** MRN policy administration was delivered in Web PR #38; appointment availability/list/reschedule/cancel workflows were delivered in API PR #70 and Web PR #39; staff/provider management was delivered in Web PR #40. The remaining Phase 5 work is cross-module usability polish and authenticated local acceptance with synthetic records. Desktop/tablet/mobile screenshots and keyboard/accessibility evidence remain outstanding; hosted CI is not a substitute.


## Hospital MRN policy administration — 2026-11-26

HeapCare.Web PR #38 merged as `23e84efaaa02ed61d18696fd8a9318db2df1862b`. The selected-hospital MRN policy workspace reads the existing policy and allows one-time initial configuration using a prefix and sequence length. Access is permission-gated by `patient.identifier.policy.read` and `patient.identifier.policy.manage`; creation is exposed only to policy managers. The UI reflects the API's immutable-after-creation contract and uses tenant/hospital-scoped, no-store requests. Hosted Web run 45753221432 passed Vitest, production build, Chromium E2E and Docker smoke; run 45753227661 also passed. Local authenticated acceptance remains outstanding.

**Delivered:** appointment management in API PR #70 and Web PR #39, followed by staff/provider management in Web PR #40. Continue using API-enforced permissions and synthetic data. Do not implement patient contact verification or clinical credential/privileging without the required security/product decisions. Authenticated local acceptance remains outstanding.


## Appointment management workspace — 2026-11-26

**API listing foundation completed:** [HeapCare.Api PR #70](https://github.com/HeapCare/HeapCare.Api/pull/70) merged as `47f8f038603d4c03a923b0a712e24467b6808f5d`. Added paginated appointment listing scoped by tenant, hospital, department and availability-session local date, with validated status filtering and bounded pagination. The endpoint uses the existing `appointment.read` authorization policy and returns scheduling metadata without patient names or contact data. Hosted Release build, EF model-drift, unit/integration tests and Docker verification passed (runs 45754993339 and 45754993230).

**Web workspace completed:** [HeapCare.Web PR #39](https://github.com/HeapCare/HeapCare.Web/pull/39) merged as `13f56f80cb7cf66a373a862fbe0152024ce26c63`. Hospital administrators can publish provider availability only for active providers assigned to the selected hospital/department on the chosen local date. The appointment workspace supports date/department/status filtering, pagination, explicit two-step cancellation and rescheduling that retains the original patient and API lineage. Appointment times are rendered using the hospital session time zone. Hosted Vitest, production build, Chromium E2E and Docker smoke passed on head `251365ffcea7351847192b7adf45b896e179fff1` (run 45755338598).

**Delivered:** staff/provider directory and effective-dated assignment management are complete in Web PR #40. The remaining acceptance pass must cover staff profile lifecycle, department assignments, availability publishing, appointment listing, department isolation, cancellation, reschedule lineage and queue check-in using synthetic records. Hosted CI is not local acceptance.


## Staff and provider management workspace — 2026-11-26

[HeapCare.Web PR #40](https://github.com/HeapCare/HeapCare.Web/pull/40) merged as `3d54ab958ab8129f526a690a25d4ce56bc17cf5a`. The tenant-scoped staff workspace supports staff profile creation and updates, selected-hospital directory search, effective-dated department assignment creation/end, and profile deactivation/reactivation. Deactivation closes open-ended active assignments through the API. UI access is gated by `staff.read` and mutations by `staff.manage`; provider designation is descriptive and does not imply credential verification or clinical privileging. Hosted Vitest, TypeScript/production build, Chromium E2E and Docker smoke passed (runs 45755742307 and 45755757928).

**Current Phase 5 follow-up:** the workspace now includes reception/MRN, scheduling, workforce, OPD, billing, facility administration, membership management, platform tenant lifecycle and the hospital operations overview (Web PR #42). Focus next on authenticated end-to-end acceptance, responsive visual evidence, keyboard/accessibility review and cross-module workflow polish. Credential verification/privileging and patient contact verification remain separate security/product approval gates.


## Workspace visual refresh — 2026-11-26

[HeapCare.Web PR #41](https://github.com/HeapCare/HeapCare.Web/pull/41) merged as `c0cc89c06a847f4a93ce46b20e120199a048dc34`. Refreshed the shared design tokens, typography, surfaces, focus-visible treatment, responsive desktop/tablet/mobile navigation, sticky workspace shell and reduced-motion behavior. The change is CSS-only and retains module authorization and workflow behavior. Hosted Vitest, production build, Chromium E2E and Docker smoke passed (runs 45756076512 and 45756064645). Visual acceptance screenshots at desktop/tablet/mobile sizes and keyboard accessibility review remain outstanding on the local authenticated environment.

**Delivered:** the hospital operations overview is complete in Web PR #42. Continue with authenticated local acceptance, responsive visual evidence, keyboard/accessibility review and cross-module workflow polish. Operational hardening now includes controlled audit query/export, migration rehearsal CI and logical backup/restore verification; production retention, key rotation, secrets/encryption, provider-native physical backup/PITR and recovery sign-off remain. The initial OPD pilot is feature-rich but not production-approved until local acceptance and security/operational sign-off are complete.


## Hospital operations overview — 2026-11-26

[HeapCare.Web PR #42](https://github.com/HeapCare/HeapCare.Web/pull/42) merged as `3243b658067b94f3a462d35527c58bff64653648`. Added a permission-aware hospital overview showing selected-date appointment totals and queue-entry totals scoped to the selected department, plus hospital-wide daily collection and payment count in the configured hospital time zone. Quick links are exposed only for the corresponding module permissions; the dashboard does not display patient names, MRNs or other PHI. Requests are guarded against stale results when tenant, hospital, department or date changes. Hosted Vitest, production build, Chromium E2E and Docker smoke passed on head `b138ee3d99f76caa43adde575e12f809e64f4bf8` (run 45756983905).

**Next priorities:** finish local end-to-end acceptance and visual/accessibility evidence; then harden operational controls (audit retention/export governance, key rotation, secrets/encryption, migration rehearsal and tested physical backup/recovery). After these controls are underway, sequence remaining hospital modules such as IPD, pharmacy, LIS, RIS/PACS, inventory and integrations. The platform is not production-approved until security, privacy, local acceptance and recovery sign-offs are complete.


## Controlled audit query and export — 2026-11-26

[HeapCare.Api PR #71](https://github.com/HeapCare/HeapCare.Api/pull/71) merged as `da0880979b25cc12a2ccfd18419a6368baa1d2de`. Added tenant-wide, permission-gated audit event listing and CSV export. Queries require a validated tenant context and tenant-wide organization scope; hospital-scoped administrators are denied tenant-wide audit access. Listing supports bounded UTC date ranges (maximum 90 days), pagination (maximum 100 rows/page), and action/resource filters. CSV export is bounded to 31 days and 5,000 rows and neutralizes spreadsheet formula prefixes. Exported fields are audit metadata only; request/response bodies and clinical payloads are excluded. PostgreSQL integration tests cover query/export, tenant isolation, tenant-wide scope authorization, date limits and formula-injection handling. Hosted Release CI and Docker integration verification passed on the merged commit.

**Operational follow-up:** audit retention, authorized access review, export access logging/monitoring, retention enforcement and privacy/legal review remain required before production approval. The query/export capability does not itself implement retention or a production backup/recovery process.


## Audit export hardening — 2026-11-26

[HeapCare.Api PR #72](https://github.com/HeapCare/HeapCare.Api/pull/72) merged as `0fc8cfee33290986c798bf0285dc6cf6861a9422`. CSV export now neutralizes formula values even when preceded by whitespace and prefixes leading tab/CR/LF control characters. PostgreSQL integration coverage confirms a control-character-prefixed actor subject is neutralized and that the export endpoint itself is recorded as an `ExportAuditEvents` audit event. Release CI and Docker integration verification passed. This supplements the bounded tenant-wide audit query/export delivered in PR #71; retention policy, access review and production governance remain outstanding.


## PostgreSQL migration rehearsal and integration-test isolation — 2026-11-26

[HeapCare.Api PR #73](https://github.com/HeapCare/HeapCare.Api/pull/73) merged as `e95e23dee5a08fe6805ae64da1c8cdefda759597`. Added a PostgreSQL 16 CI rehearsal that applies the base revision to an isolated upgrade database, then applies the PR revision's idempotent migration script to both the upgraded database and a fresh database. The script is applied twice to verify idempotency, and the complete EF migration histories must match. The rehearsal caught and corrected a missing SQL statement terminator in the audit-trigger migration; the trigger behavior is unchanged. Integration tests now run serially because several fixtures reset the same shared test database. Release CI, Docker integration verification and migration rehearsal passed. This is a disposable CI check, not a production-data migration rehearsal or release approval.


## Local development credential bootstrap hardening — 2026-11-26

[HeapCare.Local PR #8](https://github.com/HeapCare/HeapCare.Local/pull/8) merged as `6784962a9120546952329e9f63d92aeee9c6bd4c`. Fresh macOS and Windows local setups now generate independent random 256-bit Keycloak administrator and PostgreSQL passwords in the ignored `.env`; the macOS `.env` is restricted to owner read/write. Compose requires an explicit PostgreSQL password and no longer falls back to a checked-in weak value. Existing `.env` credentials are not silently rotated because persistent PostgreSQL/Keycloak volumes retain their own credentials; legacy values are warned/rejected with guidance for coordinated rotation. Local CI passed shell syntax, PowerShell parsing, Keycloak Docker import/reconciliation, Compose validation and observability Docker smoke tests. This is local-development hardening, not a production secrets-management or key-rotation solution.


## Logical backup and restore verification — 2026-11-26

[HeapCare.Database PR #4](https://github.com/HeapCare/HeapCare.Database/pull/4) merged as `245c42fcc4e19a51a391b7203087de88a144cf3d`. The existing PostgreSQL 16 Docker backup/restore suite now additionally verifies the archive's SHA-256 sidecar, manifest format/source database metadata and mode `0600` on the archive and both sidecars. The test also exercises successful restore, source/target overwrite guards and corrupted-archive rejection. The Docker verification workflow passed. This validates provider-neutral logical backup helpers only; production physical backups/PITR, encryption/key ownership, off-site storage, retention, RPO/RTO and scheduled restore drills remain open.


## Keyboard skip-link verification — 2026-11-26

[HeapCare.Web PR #43](https://github.com/HeapCare/HeapCare.Web/pull/43) merged as `73764e1bb14ce5de6a84cced5e9072603b796e73`. The Chromium accessibility suite now verifies the skip link is the first keyboard stop, has a visible focus indicator, and activates with Enter to move focus to the main workspace landmark. Vitest, production build, Chromium E2E and Docker smoke passed (Web CI runs `45762291461` and `45762285239`). This is a targeted keyboard-navigation regression test, not a complete WCAG 2.2 AA audit; authenticated local-stack keyboard/screen-reader review and real responsive screenshots remain outstanding.


## Weekly synthetic logical backup/restore regression — 2026-11-26

[HeapCare.Database PR #5](https://github.com/HeapCare/HeapCare.Database/pull/5) merged as `5e53afec375549f81fe05b9f6c8a9db733a24297`. The existing PostgreSQL 16 Docker logical backup/restore verification now also runs every Monday at 03:17 UTC, in addition to relevant PRs, pushes to `main` and manual dispatch. It uses disposable PostgreSQL and synthetic probe rows to validate archive/checksum/manifest, isolated restore, overwrite guards and corrupted-archive rejection. This is scheduled CI regression coverage only; it does not access production, prove provider-native physical backup/PITR or off-site storage, or replace a production restore drill. Provider selection, encryption/key ownership, retention, RPO/RTO and accountable recovery drills remain open.


## Local infrastructure smoke helper — 2026-11-26

[HeapCare.Local PR #9](https://github.com/HeapCare/HeapCare.Local/pull/9) merged as `d9e52044051b3fadb15e8ed3f679f0346ad31952`. Added `bash local/scripts/verify-stack.sh` to verify the four Compose services are running, PostgreSQL readiness, Web/API/OpenAPI reachability and Keycloak OIDC discovery/issuer consistency. Local CI passed shell syntax validation, Keycloak Docker import/reconciliation, Compose validation and the observability Docker smoke pipeline. The script does not perform real user authentication, tenant/RBAC checks or patient workflow acceptance; it must be run against the developer's local Docker stack before local environment acceptance can be closed.


## Audit permission spoofing regression — 2026-11-26

[HeapCare.Api PR #75](https://github.com/HeapCare/HeapCare.Api/pull/75) merged as `649d32636b9bc4bd5114dc08ca27974e50ce4d71`. PostgreSQL integration tests verify PatientRegistrar, ClinicalUser, FacilityManager and FacilityViewer memberships cannot list or export tenant audit events, even when a signed test token carries forged audit.read, tenant-role and standard role claims. This protects the rule that HeapCare permissions are derived from persisted membership, not caller-supplied token role/permission claims. Release CI and Docker integration verification passed.


## Product delivery extension — UI architecture, pilot gates and deferred hospital modules (2026-11-26)

This section complements the milestone checklist above. It distinguishes current implementation, remaining acceptance work and intentionally deferred product scope. Do not mark a milestone complete from PR count or hosted CI alone.

### Immediate priority: stabilization and acceptance
- [x] Merge Web response-security headers with Docker-verified assertions while retaining `X-Frame-Options: DENY` (Web PR #46). Content-Security-Policy remains deferred until deployment-specific OIDC/API origins are allow-listed.
- [ ] Run the merged one-click local Docker launcher on the developer Mac and record service health.
- [ ] Complete authenticated Keycloak/API/PostgreSQL regression with synthetic tenants, users, roles and patient records.
- [ ] Verify cross-tenant and cross-hospital denial using persisted membership and changed resource identifiers.
- [ ] Verify real API logs, metrics and traces reach Elasticsearch/Kibana, Prometheus and Jaeger through the shared Collector; inspect redaction and export errors.
- [ ] Capture authenticated desktop/tablet/mobile screenshots and keyboard/accessibility evidence.
- [ ] Repeat local acceptance after fixes; record PASS/FAIL/BLOCKED/NOT TESTED and retain all skipped scenarios.
- [ ] Complete backup/PITR and restore, migration/rollback, secrets/key rotation, audit retention, security and pilot acceptance gates.

### Web experience architecture milestone
**Goal:** move from the interim single Workspace/hash-section composition to independent route-level module workspaces under a shared authenticated shell.

- [x] Introduce route-based module layouts and stable URLs; retain safe redirects/deep links from existing hash URLs during migration (Web PR #64).
- [ ] Extract overview, reception and patient registration into independent route workspaces.
- [ ] Extract appointments and queue workspaces without changing their API lifecycle semantics.
- [ ] Extract clinical/OPD, retaining patient identity confirmation, signing, closing and append-only amendment safeguards.
- [ ] Extract billing, facilities, staff, memberships and patient identifier policy.
- [ ] Keep platform tenant lifecycle in a separate platform-operator control plane.
- [ ] Use permission-aware navigation and user-friendly route guards while retaining API-side authorization as the only security boundary.
- [ ] Add route-level loading, empty, error, forbidden and not-found states; verify refresh, browser history, session expiry and deep links.
- [ ] Verify desktop/tablet/mobile, keyboard-only navigation, screen-reader semantics and WCAG 2.2 AA target criteria.
- [ ] Capture real authenticated screenshots with synthetic data; mocked tests are not local acceptance evidence.

Implementation must be staged in focused Web PRs. Do not combine this with backend/domain changes or a big-bang rewrite.

### Deferred implementation phases — retained scope

| Phase | Planned module | Major capabilities | Status |
|---|---|---|---|
| 6 | IPD / inpatient | ADT, admission, bed reservation/occupancy, ward/nursing, observations, care plans, handover, medication administration, discharge | Deferred |
| 7 | Pharmacy | Formulary, prescription validation, dispensing, batches/expiry, stock, returns and supplier workflows | Deferred |
| 8 | LIS | Diagnostic test catalogue/orders, specimen collection/barcodes, worklists, processing, result validation, critical-result escalation | Deferred |
| 9 | RIS/PACS | Imaging orders/scheduling, modality worklists, DICOM, reports, image storage/access and retention | Deferred |
| 10 | Inventory/procurement | Stores, requisitions, purchase orders, vendors, receipts, stock movement, assets and maintenance | Deferred |
| 11 | Advanced revenue cycle | Statutory invoice/tax rules, deposits, discounts, refunds, credit notes, insurance/TPA, claims, payment/accounting integrations and reconciliation | Deferred |
| 12 | Workforce/HR | Verified account linking, credentialing, clinical privileges, rostering, attendance, leave, employee records and payroll integrations | Partially scaffolded; extended capabilities deferred |
| 13 | Patient engagement/interoperability | Patient/guardian portal and mobile, self-booking, consent-aware communications, ABDM/ABHA, FHIR/HL7, DICOM and external integrations | Deferred |
| 14 | Advanced analytics/intelligent assistance | Permissioned operational/financial analytics, exports and governed AI-assisted workflows with human oversight | Deferred |

### Deferred scope guardrails
- Patient contact verification, identity merge/correction, guardian and emergency reconciliation require approved privacy, identity, consent, retention and security design.
- Clinical credentialing and privileges require verified evidence and qualified clinical governance; a staff designation is not authority.
- Advanced billing, GST/statutory treatment, refunds, insurance and claims require pilot finance policy and qualified review.
- ABDM/ABHA, FHIR/HL7 and DICOM integrations require validated use cases, partner agreements, approvals and conformance/security testing.
- Autonomous diagnosis, triage or prescribing is not in scope.
- Full ERP/payroll, broad mobile delivery, multi-region active-active and microservice extraction remain gated by validated demand and architecture decisions.

### Dependency order
Stabilize and validate the outpatient journey → route-based Web architecture and OPD completion → security/operations/pilot gates → IPD → pharmacy → LIS/RIS-PACS → inventory/procurement → advanced revenue cycle → workforce/patient engagement → interoperability/analytics. Reorder only with documented pilot evidence and explicit dependency review.

Every completed capability must update the blueprint, module catalogue, workflow documentation, API contracts, tests and release evidence. Keep future work visible as Deferred rather than silently removing it from scope.


## Stabilization evidence update — 2026-11-26 (post full-platform regression)

This is an evidence update to the original Antigravity report. It records code/CI remediation separately from developer-machine acceptance. A green hosted workflow does not prove that the user's nine-container stack or authenticated browser session has been revalidated.

### Defects addressed in code and merged

| QA item | Remediation merged | Verification | Remaining acceptance |
|---|---|---|---|
| API-001 — malformed requests returning 500 | API PR #76 preserves the 4xx status carried by `BadHttpRequestException` and adds malformed audit-query/empty-body integration cases | Release CI, PostgreSQL integration, Docker test and migration rehearsal passed | Re-run the live local API request cases |
| API-002 — concurrent patient booking deadlock | API PR #77 adds a transaction-scoped tenant/patient advisory lock before the overlap check; the exclusion constraint remains the final guard; the race test now runs eight concurrent requests | Release CI, PostgreSQL integration, Docker test and migration rehearsal passed | Re-run repeated booking race in the local full stack |
| WEB-001 — wrong default workspace | Web PR #45 makes Reception the default for staff with patient-search access | Vitest, TypeScript/build, Chromium E2E and Docker smoke passed | Authenticated local browser visual confirmation |
| WEB-002 — 390px horizontal overflow | Web PR #45 adds narrow-screen layout constraints and an authenticated 390×844 Playwright regression | Chromium E2E and Web CI passed | Confirm the actual Mac/browser viewport and capture screenshots |
| WEB-003 — cancellation contrast | Web PR #45 adds high-contrast destructive styling and a component assertion | Vitest and Web CI passed | Visual confirmation in the local authenticated workspace |
| OBS-001 — broken API OTLP traces/metrics | Observability PR #4 fixes the exporter to OTLP/gRPC and publishes package 1.0.1; API PR #78 consumes it and adds Npgsql instrumentation; Local PR #11 configures port 4317 and Local PR #13 runs the complete nine-service Compose stack and `verify-stack.sh` in hosted CI | Observability, API and Local CI passed. Local PR #13 verified all nine services, API traces in Jaeger including a `postgresql` operation, and ASP.NET Core request metrics through the OTLP Collector exporter | **Hosted integration verified.** Still run the merged launcher and `verify-stack.sh` on the developer Mac and retain local evidence; this is separate from authenticated browser acceptance |
| OBS-002 — missing Kibana data view | Local PR #12 adds idempotent `HeapCare Logs` provisioning for `heapcare-logs-*` with `@timestamp` and wires it into stack verification | Python syntax and Docker observability CI, including Kibana data-view API assertion, passed | Verify the view and Discover experience in the local Kibana instance |
| WIN-001 — Windows local stack missing observability | Local PR #19 brings the Windows one-click launcher and stop command to the same nine-service Compose overlay as macOS, with PostgreSQL/API/Keycloak/observability readiness checks and Kibana data-view provisioning | PowerShell syntax/static launcher checks and full nine-service Docker, persisted tenant/RBAC, telemetry and real PKCE browser workflow passed in run `45815836344` | Actual Windows Docker Desktop execution remains a host-specific acceptance item |

### Security and operations — current state

- Web PR #46 adds response headers (`nosniff`, Referrer-Policy, `X-Frame-Options: DENY`, CORP and Permissions-Policy); Docker smoke asserts them. This is defence-in-depth, not a complete browser security policy.
- SEC-001 code-level hardening is merged in API PR #79: per-subject/per-IP rate limiting, RFC 9457 429 responses with Retry-After, HSTS outside Development, trusted forwarded-header processing and the Kestrel banner disabled. Configure ReverseProxy:KnownProxies with the exact trusted ingress proxy IP addresses before deployment. The built-in limiter is per API process/instance, not a fleet-global quota; a multi-replica deployment needs gateway/shared quota enforcement for strict global limits. Tune the defaults against measured hospital traffic.
- SEC-002 is partially addressed: Web PR #47 commits npm's generated lockfile, switches CI and Docker to npm ci, and makes npm audit blocking. The final PR CI passed and the dependency audit reported zero vulnerabilities for that lockfile. At the time of this dependency-audit PR, CSP remained deferred pending exact production Keycloak/API origins. Subsequent Web PRs #56, #57 and #61 implemented an enforced strict policy; PR #62 added browser acceptance against the built Nginx container. Exact production origins and ingress validation remain open. Do not claim complete browser security or supply-chain compliance from the dependency audit alone.
- The previous NU1902 on the OTLP exporter version 1.12.0 was addressed by the Observability 1.0.1 update to exporter 1.15.3.
- GAP-001 (Audit Review UI) is **deferred roadmap scope** and remains planned for a later phase. API audit search/export capability does not equal a usable, permission-controlled Audit Review workspace. Do not start customer-facing audit browsing, export UX or broader audit permissions until the hospital/compliance approvals and ADR-016 governance decisions are complete.

### Acceptance gates still open

- [ ] Execute the merged `HeapCare.Local` macOS one-click startup against current `main`, without deleting persistent Docker volumes.
- [ ] Execute the merged `HeapCare.Local` Windows one-click startup on Windows 10/11 Docker Desktop and verify the nine services, Keycloak sign-in, Kibana data view and retained volumes; Linux CI does not replace Windows-host acceptance.
- [x] Run the full `verify-stack.sh` against all nine services in hosted Docker CI; Local PR #13 verified API-to-Jaeger traces including an Npgsql `postgresql` operation, plus Collector-fed ASP.NET Core request metrics.
- [ ] Run the merged `verify-stack.sh` on the developer Mac and retain the local Jaeger/Collector evidence; hosted CI does not replace the local environment check.
- [x] Verify real Keycloak-issued access tokens against the API issuer/audience/subject validation and exercise two PostgreSQL-persisted tenant memberships with different roles, hospital data isolation, cross-tenant membership mutation denial, and membership revocation/reactivation in Local PR #14 hosted Docker CI.
- [x] Verify the real browser Authorization Code + PKCE sign-in/callback and tenant switching against the full nine-service Docker stack in Local PR #15 hosted CI, using the actual Keycloak login page, Web client, API and persisted tenant memberships.
- [x] Verify real reception patient registration and search for HospitalAdmin in tenant A, including duplicate review and persisted API creation; verify FacilityViewer in tenant B has no registration UI and receives HTTP 403 for a direct patient-registration API request (Local PR #16 hosted full-stack CI).
- [x] Verify FacilityViewer in tenant B receives HTTP 403 for both department-scoped patient search and patient identity detail using a real Keycloak-issued token and a synthetic tenant-B department; the corrected route assertions passed both Local CI jobs, including full nine-service Docker, persisted tenant/RBAC checks and real PKCE browser acceptance (Local PR #18, workflow run `45813953115`).
- [ ] Run the merged one-click stack on the developer Mac and repeat browser sign-in, silent renewal/session lifecycle and role-based workflows there; capture desktop/tablet/390px screenshots, console/network evidence and keyboard/accessibility review.
- [x] Verify the eight-way concurrent appointment booking race repeatedly against PostgreSQL: `ConcurrentAppointmentBookingAllowsOnlyOneWinnerAcrossRepeatedWaves` executes three successive eight-request waves, asserting one `201 Created`, seven `409 Conflict` responses per slot, and persisted appointment consistency. API `main` CI run `45781581764` passed all 117 PostgreSQL integration tests (zero skipped).
- [ ] **Deferred — Audit Review UI:** complete hospital-approved requirements, named-reader permission model, search/export workflow, retention/legal-hold policy and acceptance criteria before implementation. Keep this in the roadmap; it is not part of the current engineering increment.
- [ ] Configure exact production CSP origins and validate real OIDC/API browser flows behind the selected ingress; complete secret rotation, backup/PITR/restore, audit retention and production ingress configuration.
- [ ] Configure exact ReverseProxy:KnownProxies ingress addresses and validate forwarded scheme/client-IP behavior in the target deployment; load-test and tune per-instance rate limits, and add gateway/shared quotas if multi-replica global limits are required.
- [ ] Complete hospital pilot acceptance, security/privacy review and applicable legal/regulatory assessment.

### Release decision

The code-level remediation above reduces the original defect set. OBS-001 has hosted full-stack evidence from Local PR #13; Local PR #14 verifies real Keycloak-issued tokens and persisted two-tenant authorization/revocation; Local PR #15 verifies browser Authorization Code + PKCE sign-in/callback and tenant switching; and Local PR #16 verifies the real reception patient registration/search workflow plus FacilityViewer's server-side 403 denial against the hosted Docker stack. The platform remains **CONDITIONAL FAIL / HOLD NEW FEATURE WORK** until developer-Mac execution, browser/session renewal on that machine, broader appointment/queue/clinical/billing workflow acceptance, actual hospital-user visual/accessibility review and remaining deployment/security/operational gates are evidenced. Hosted CI is not a substitute for the developer's local installation or final user-facing visual acceptance. Keep new hospital modules deferred while these gates remain open.


### Security and reproducible-build follow-up — 2026-11-26

The subsequent remediation is now merged:
- API PR #79 (merge commit 403b633) adds the rate limiter (600/minute per authenticated subject, 120/minute per anonymous IP by default), 429 ProblemDetails/Retry-After, trusted forwarded-header processing with a one-hop/symmetric-header policy, HSTS outside Development and disables the Kestrel server banner. Its final hosted checks passed 12 unit tests, 117 PostgreSQL integration tests, Docker verification and migration rehearsal.
- Web PR #47 (merge commit 0732352) commits the npm-generated v3 lockfile and uses npm ci in both Web CI and the Docker build. The PR's blocking npm audit, Vitest, TypeScript/production build, Chromium E2E and Docker/header smoke all passed. The generating CI audit reported zero vulnerabilities.

These changes close the original API rate-limit code gap and the Web lockfile/ENOLOCK auditability gap at repository/CI level. They do not close deployment acceptance: operators must set the exact trusted proxy IPs, calibrate per-instance rate limits, define a shared quota strategy if strict multi-replica limits are required, and complete CSP policy testing for the actual production OIDC/API origins.


### Full nine-service hosted Docker evidence — 2026-11-26

Local PR #13 merged as `bc609fc67f9656e444bc486a47fa047999bd33fa`. Its PR CI run `45785200869` passed both jobs:
- Full nine-service Compose job: all nine services running; PostgreSQL readiness; Web/API readiness and OpenAPI; Keycloak OIDC discovery; Kibana Logs data view provisioning; API metrics endpoint.
- Real application telemetry: Jaeger contained HeapCare.Api operations including the Npgsql `postgresql` database operation; ASP.NET Core request-duration metrics reached the OpenTelemetry Collector Prometheus exporter.
- The isolated observability pipeline, Keycloak import/reconciliation, configuration and shell validation also passed.
- The uniquely named CI Compose project, volumes and generated local secrets were removed successfully.

This closes the **hosted integration verification** portion of OBS-001. It does not prove the developer's Mac installation, authenticated sign-in/tenant workflows, actual production ingress configuration, or browser visual/accessibility acceptance. Those remain explicit gates.


### Real Keycloak identity and persisted multi-tenant acceptance — 2026-11-26

Local PR #14 merged as `02888b4994fed2c2e8aa119e3202cf0470205849`. Its PR CI run `45792658639` passed both Local jobs, including the full nine-service Docker stack and the real authenticated tenancy script. The acceptance log confirms:
- Real access tokens issued by the HeapCare Keycloak realm passed the API's configured issuer, audience and subject validation.
- A platform operator was bootstrapped and two tenants were provisioned in PostgreSQL.
- The same synthetic Keycloak user had a persisted HospitalAdmin membership in tenant A and a persisted FacilityViewer membership in tenant B; the user-context endpoint returned both roles/permission sets.
- Hospital records remained isolated between the two tenants; cross-tenant reads and cross-tenant membership updates were denied; the FacilityViewer could not create a hospital.
- Deactivating the tenant-B membership immediately denied tenant-B access while tenant-A access remained; reactivation restored tenant-B access.
- The full-stack run also continued to verify the nine services, API-to-Jaeger/Npgsql traces, Collector-fed request metrics and Kibana Logs data view.

The test uses a CI-only temporary Keycloak client with direct grants. The production-style local Web client remains Authorization Code + PKCE; the test does **not** exercise browser login, refresh/session behavior or real Web UI flows.

**Keycloak 26.2.5 configuration correction:** the local realm now assigns the built-in `basic` client scope to `compilecare-web`. The local reconciler verifies that its `oidc-sub-mapper` emits the `sub` claim in access tokens. It also assigns that scope to the temporary CI user client. This is required so the API can resolve persisted tenant memberships by validated issuer and subject. Validate the equivalent subject mapper/scope configuration in any separately managed production Keycloak realm; the local realm definition does not change a production identity provider.

The hosted real browser Authorization Code + PKCE sign-in/callback and role-based tenant switching are verified in Local PR #15. The remaining gate is to run the merged stack on the developer Mac and repeat browser/session renewal there, then complete role-based hospital-user visual review, responsive screenshots and keyboard/screen-reader accessibility acceptance. Keep those checks separate from the hosted browser and API acceptance.


### Real browser Authorization Code + PKCE acceptance — 2026-11-26

Local PR #15 merged as `d6e1362642c8d1553c9680020395a92b88005144`. The successful Local PR CI run was `45795779615`, attempt 2. Both Local workflow jobs passed. The full-stack job:
- Built and started the nine-service Compose stack, then passed the real Keycloak-issued identity and PostgreSQL-persisted two-tenant API checks.
- Installed Chromium and ran the dedicated Web Playwright suite against the actual Docker Web application at `http://localhost:8080`.
- Completed a real Authorization Code + PKCE redirect to the local Keycloak login page and returned through the Web OIDC callback.
- Verified the same synthetic user could select tenant A (HospitalAdmin) and tenant B (FacilityViewer), saw CI Hospital A/B respectively, and had the correct role-based navigation (patient registration only for HospitalAdmin; facility read navigation for FacilityViewer).
- Observed real API hospital-list requests carrying a bearer authorization header and the selected `X-Tenant-ID`; the test recorded only whether a bearer header was present and each tenant ID, never the token itself.
- Removed the unique CI Compose project and volumes, plus the mode-0600 temporary browser credential file.

This closes the **hosted browser OIDC/tenant-switch acceptance** gate. It does not close developer-Mac acceptance, repeat browser/session renewal on that machine, authenticated hospital-user visual review, screenshots, keyboard/screen-reader accessibility, or production identity-provider/ingress/CSP verification.


### Repeated concurrent appointment booking evidence — 2026-11-26

The existing PostgreSQL integration test `AppointmentSchedulingApiTests.ConcurrentAppointmentBookingAllowsOnlyOneWinnerAcrossRepeatedWaves` is already the required stress regression. It creates six half-hour slots, then executes three successive waves of eight simultaneous booking attempts against a single slot, using a different registered synthetic patient for each wave. Every wave asserts exactly one successful booking and seven conflicts, with the final appointment list matching the three successful records.

API `main` workflow run `45781581764` passed on merge commit `403b633418eba5aa82b5c34384d969b8a217be9a`: all 117 PostgreSQL integration tests passed, zero skipped. This closes the hosted repeatable concurrency-regression gate; it does not replace any further load testing required by a deployment or hospital pilot.


### Real reception patient registration and search acceptance — 2026-11-26

Local PR #16 merged as `35880086060049bf3ebf49e48e7e5f205ec98b69`. Successful Local PR CI run `45807187585`, attempt 2, passed both workflow jobs. The full-stack job checked out HeapCare.Web main at `300afedcf2e0fdcd8e33ef3d2b7b63b28efa133b` (Web PR #50) and verified:

- The real Keycloak Authorization Code + PKCE browser sign-in/callback and persisted two-tenant membership selection.
- In HospitalAdmin tenant A, the browser completed the patient registration form, performed the hospital-local duplicate review, acknowledged the synthetic new-patient result, submitted registration through the real API and searched for the newly registered patient in Reception.
- The browser test observed real patient API method/path, tenant header and bearer-token presence only; it did not capture or log access tokens or patient request payloads.
- In FacilityViewer tenant B, patient registration navigation was absent. The backend acceptance script also submitted a valid synthetic patient-registration request directly to the selected hospital using the FacilityViewer token and asserted HTTP 403.
- The same run retained the real API issuer/audience/subject checks, PostgreSQL-persisted membership roles, tenant data isolation, cross-tenant membership mutation denial, membership revocation/reactivation, and nine-service telemetry checks.

This closes the **hosted reception patient registration/search and FacilityViewer patient-registration authorization** gates. It does not complete appointments, queue, OPD/clinical or billing browser workflows; it does not prove local Mac/browser session renewal, hospital-user visual/accessibility acceptance, or production configuration. All records used in CI are synthetic.

### FacilityViewer patient-data authorization regression — 2026-11-26

The subsequent authorization regression slice is merged and hosted-verified:

- Web PR #51 merged as `cc63739d0bbc6094f98afaf2295bfb6937f714f9`. The real PKCE browser acceptance now verifies that FacilityViewer cannot see Reception navigation, Patient registration navigation, or the "Find a registered patient" workspace.
- Local PR #16 merged as `35880086060049bf3ebf49e48e7e5f205ec98b69`. It proves a real Keycloak-issued FacilityViewer token receives HTTP 403 when directly attempting patient registration against the tenant-B hospital.
- Local PR #17 merged as `8deec532fbe52327b373b5f386d4e72859b859a2`. Its real Keycloak/PostgreSQL acceptance script additionally requires HTTP 403 for patient registration, patient search, patient identity detail and duplicate-candidate review. All attempted patient-like values are synthetic; no patient record is created for the denied cases.
- Local workflow run `45810119901` passed both jobs: full nine-service Docker startup, real Keycloak identity and persisted tenant checks, all FacilityViewer negative API assertions, real Authorization Code + PKCE browser acceptance, telemetry verification, tooling validation and isolated environment cleanup.

These results close the **hosted FacilityViewer patient read/write/duplicate-review authorization regression** for the current patient API surface. They do not imply that every future patient endpoint is automatically covered: new routes must retain explicit positive and negative permission tests, tenant/organization scope tests and audit metadata.

### Current continuation checkpoint — 2026-11-26

The latest connected GitHub review found no remaining open PRs across Web, API, Database, Observability, ApiClient, Caching, OtpService, Mobile, Site and Documentation. Local PR #17 was merged after both hosted workflow jobs passed. The only open-PR listing briefly returned a stale snapshot for #17; the direct PR record confirms it is closed and merged.

**Next engineering priority:** continue Phase 3 security/operational hardening without bypassing ADR-016. The existing audit query/export API remains provisional. Do not implement customer-facing Audit Review UI, broaden audit scope, add bulk export, or invent retention rules until the pilot hospital's named-reader, scope, retention, legal-hold and export approvals are recorded. Safe engineering work may continue on test coverage, documentation, threat-model traceability and deployment-readiness automation that does not activate unapproved customer access.

Developer-Mac execution, local browser/session renewal, authenticated visual/accessibility review (owned by Antigravity/user), production CSP and trusted-proxy configuration, provider-native backup/PITR and restore drills, and pilot/security/privacy sign-off remain outstanding. The hosted evidence is strong progress, not a production-readiness declaration.


## Full-platform E2E validation and remediation — 2026-11-26

**Status: Local full-stack baseline completed with CONDITIONAL PASS; first defect-remediation batch merged. This is not hospital-pilot or production approval.** The detailed evidence and defect register are recorded in [Full-platform E2E validation](../qa/e2e-validation-2026-11-26.md).

Antigravity started the official macOS launcher in 109 seconds, built the API/Web images, reconciled Keycloak and verified all nine running containers. The live baseline recorded 58/59 full-stack checks (34/35 browser E2E and 24/24 security/RBAC/rate-limit checks), zero blocked checks and seven explicitly deferred/not-implemented areas. Observability evidence included API/Npgsql traces, Collector-fed metrics and synthetic logs in Elasticsearch/Kibana. The local latency measurements are a development baseline only.

The following Web remediation is merged:

- [PR #53](https://github.com/HeapCare/HeapCare.Web/pull/53), `e62d97a`: DEF-001 provider-directory refresh after staff mutations; DEF-002 same-tenant selection no-op; DEF-003 operation-neutral facility conflict copy; DEF-004 Chromium Unicode Sets-compatible MRN prefix pattern. CI run `45827629738` passed.
- [PR #54](https://github.com/HeapCare/HeapCare.Web/pull/54), `0a3bb14`: DEF-005 Overview refresh on re-entry and tab-scoped tenant/hospital selection restoration, accepted only when still present in the authenticated membership and API-filtered hospital lists. CI run `45828086411` passed 113 unit tests, 16 Chromium E2E tests, production build, real-OIDC suite discoverability and Docker smoke.
- [PR #55](https://github.com/HeapCare/HeapCare.Web/pull/55), `256c6df`: DEF-007 billing E2E synchronization with the successful queue API response. CI run `45828251071` passed 108 unit tests, all 16 Chromium E2E tests, production build, real-OIDC suite discoverability and Docker smoke.

DEF-006 is reclassified rather than treated as one code defect: CSP remains a deployment-specific gate until exact production Keycloak/API origins are known; Audit Review UI remains deferred under ADR-016; and OtpService runtime integration remains deferred/not implemented. Do not broaden audit permissions or implement customer-facing audit browsing/export UX before hospital/compliance approval.

**Next:** rerun Antigravity's full local E2E and visual evidence collection against the combined current Web main; review restored context with multiple/revoked memberships; complete hospital-user keyboard/screen-reader review; validate the Windows 10/11 launcher on a real host; and close production CSP/trusted-proxy, provider-native backup/PITR, RPO/RTO, restore-drill and pilot/security/privacy gates. The original local report remains the historical baseline; new validation must use a new dated evidence directory.

### Post-remediation local acceptance — 2026-11-26 (supersedes the prior rerun gate)

**Status: VALIDATION PASS FOR IMPLEMENTED OPD MVP SCOPE. Not hospital-pilot or production approval.**

The post-remediation Antigravity run was completed against the combined current Web main after PRs #53–#55. The detailed evidence is recorded in [Post-remediation E2E validation](../qa/post-remediation-e2e-validation-2026-11-26.md).

- Live browser E2E improved from 34/35 to **41/41**; security/RBAC/tenant isolation remained **24/24**; combined live checks improved from **58/59 to 65/65**, with zero failures and zero blocked checks.
- DEF-001 through DEF-005 and DEF-007 are verified fixed. DEF-005 includes separate Overview refresh and persisted/validated tenant+hospital context checks.
- Web CI evidence: 113/113 unit tests and 16/16 Playwright specs using five workers.
- All nine local containers were running. Prometheus had two scrape jobs up and 140 HTTP metric series; Collector exposed 1,037 HTTP metric lines with zero Collector error-log lines; Jaeger contained 50 API traces/101 spans including PostgreSQL spans; sampled Elasticsearch secret-leak search returned zero hits.
- Local performance sample: 167 API requests, p50 13 ms, p95 51 ms, max 111 ms; this is not production capacity certification.
- Execution summary: 41/41 browser checks passed, zero failed/blocked, seven not-implemented areas. Six console errors and 12 network entries were recorded and classified in the QA report; these are not described as zero console/network errors.

**DEF-006 remains open as a grouped release backlog item:** production CSP must be designed for exact deployed Keycloak/API origins; customer-facing Audit Log Viewer UI remains deferred under ADR-016 despite the reported API/CSV export implementation; OtpService runtime integration remains a skeleton and patient contact verification remains gated by ADR-012.

The previous checklist item requiring a fresh combined-main local E2E rerun is now complete. Remaining gates are authenticated hospital-user keyboard/screen-reader and accessibility review, Windows 10/11 launcher acceptance, production TLS/HSTS/CSP/trusted-proxy/KMS configuration, Elasticsearch/Kibana authentication and TLS, provider-native encrypted PITR and restore drills with defined RPO/RTO, independent penetration/privacy/legal review, and hospital pilot approval. Do not implement Audit Review UI or OTP/contact verification before their respective governance decisions.

### Web Content Security Policy hardening — 2026-11-26

Web PR #56 is merged as `71d6557dbc8d9fc34add12b9061805e5d050c250`; CI run `45838673174` passed unit tests, build, browser E2E, real-OIDC suite discoverability and Docker security-header smoke. Web PR #57 is also merged; CI run `45840521245` adds negative startup tests proving HTTP origins are rejected without the local-only opt-in and missing required CSP origins fail closed. Web PR #61 is merged as `cd2dd1230be808e12f90bb07717a8ae7495d3298`; CI run `45851992088` removes `unsafe-inline` from `style-src`, leaving `style-src 'self'`. The current policy blocks objects and framing, restricts scripts and styles to same-origin, and does not allow inline scripts, inline styles or `unsafe-eval`.

HeapCare.Local PR #20 is merged as `792196a2d19afeb128e8bf40205977d6b05e607a`. It supplies the local API/Keycloak origins in `local/.env.example`, scopes the HTTP exception to the local-development Compose configuration, and extends `verify-stack.sh` and CI assertions. Latest Local workflow run `45839528532` passed both jobs: configuration/Keycloak/observability validation and the full nine-service Docker stack, persisted multi-tenant acceptance and real Keycloak Authorization Code + PKCE browser acceptance. The full-stack run exercised the enforced CSP with the local API and Keycloak origins. An earlier run `45839463966` also passed the full-stack and real-OIDC acceptance using the explicit .env values.

**Still open for production:** supply the exact deployed HTTPS API and identity-provider origins, validate the enforced policy through real OIDC login, callback, silent renewal, API calls and static asset loading in the target ingress/browser environment, and confirm no additional third-party origins are required. Web PR #62 is merged as `d02a4d9d72a30203692650cffaf52b8476ae4d4d`; CI run `45853231713` passed all 20 browser E2E tests against the built Nginx container with the strict no-inline CSP, verified the generated security headers, and verified wildcard, insecure HTTP and missing-origin configurations fail closed. The Docker browser suite initially exposed a hard-coded Vite-port CORS origin in the test fixtures; the mocks now echo the actual request origin and the full suite passes. This uses synthetic data and mocked API/OIDC endpoints, not the production identity provider. CSP implementation does not close TLS/HSTS, trusted-proxy, KMS, Elasticsearch/Kibana authentication/TLS, backup/PITR, independent security review or pilot approval.

### API production CORS configuration hardening — 2026-11-26

HeapCare.Api PR #81 is merged as `2af5bfc2c9e576cd92010b49e622922f6b29a8e5`. All three hosted workflows passed: CI run `45841339836` (Release build, EF model-drift check and complete unit/integration suite), Docker Integration Verification `45841339388`, and PostgreSQL Migration Rehearsal `45841339328`.

The API now validates `Cors:AllowedOrigins` during startup outside Development. Configured origins must be explicit HTTPS origins without wildcard hosts, credentials, paths, query strings or fragments; trailing slashes are normalized and duplicates removed. HTTP remains available only for Development. An empty list is valid for same-origin Web/API deployment. Integration tests cover accepted, rejected, normalized, Development and same-origin cases.

**Still open for deployment:** the hosting operator must provide the exact production Web origin (and any additional browser origins actually required), set the allowlist and test preflight and authenticated API requests through the selected ingress. This code change does not choose or configure production domains, TLS certificates, HSTS at the edge or trusted proxy IPs.

### Production PostgreSQL TLS enforcement — 2026-11-26

HeapCare.Api PR #82 is merged as `dc07866a064855dce72880bc24c1cc13c393be40`. The API now fails startup outside Development and the isolated Testing host if `ConnectionStrings:CompileCare` is missing or PostgreSQL is not configured with Npgsql `SSL Mode=VerifyFull`. This validates the certificate chain and server hostname. Development keeps its local plaintext PostgreSQL setup; the Testing exception is limited to the isolated integration-test host.

All three final hosted workflows passed: CI `45843760937` (Release build, EF model-drift and full test suite), Docker Integration Verification `45843760927`, and PostgreSQL Migration Rehearsal `45843760942`. The Docker suite also exercises the production HSTS/forwarded-header middleware using the non-Development Testing environment and a synthetic, non-connected database configuration.

**Still open for deployment:** select the hosting provider, securely mount its trusted CA if required, configure the real PostgreSQL hostname and `VerifyFull` connection string, and prove a successful TLS connection with hostname validation from the deployed API network. Encryption-at-rest, KMS ownership/rotation, encrypted backups/PITR and restore drills remain separate gates.

### Automated Web accessibility regression baseline — 2026-11-26

Web PR #58 is merged as `a480b0eea462afcf955b44da278a3817ef3ca6bf`; CI run `45844753971` passed 113 unit tests and all 19 Chromium Playwright specs. Web PR #59 subsequently added an authenticated HospitalAdmin workspace regression for accessible names on visible controls and labels on visible form fields; CI run `45845404253` passed 113 unit tests and all 20 Playwright specs. Web PR #60 then verified that keyboard Tab reaches the authenticated skip link and activating it moves focus to the main landmark; CI run `45846148079` passed 113 unit tests and all 20 Playwright specs. The existing keyboard skip-link/focus and responsive overflow checks are retained. The accessibility suite now also asserts one main landmark and one level-one heading, accessible names for public-shell links/buttons, associated labels for form fields, alternative text attributes on images, and reduced-motion transition durations.

This is an automated public-shell baseline only. It does not constitute WCAG 2.2 AA conformance or validate authenticated role-specific clinical workflows with screen readers. Keep the manual hospital-user keyboard, screen-reader, zoom/contrast and assistive-technology review as an explicit pilot gate; extend automated coverage as authenticated fixtures become available.



### Production CORS origin validation — 2026-11-26

HeapCare.Api PR #84 is merged as `e72feea2daf633d82cefa876e753536b0af4c3ac`. API startup now validates configured `Cors:AllowedOrigins` outside Development: only explicit HTTPS origins are accepted; wildcards, credentials, paths, queries, fragments and HTTP are rejected. Trailing slash normalization and duplicate removal are covered. Development retains the local loopback HTTP configuration. Same-origin deployments may omit the CORS list.

All three hosted workflows passed: CI `45851265491` (Release restore/build, EF model-drift check and full test suite), Docker Integration Verification `45851265298`, and PostgreSQL Migration Rehearsal `45851265489` (upgrade and fresh-database rehearsal). Tests cover valid HTTPS origins, invalid/insecure origins, Development HTTP and same-origin empty configuration.

**Still open:** set the exact deployed Web origin in `Cors:AllowedOrigins` only for cross-origin deployments; verify CORS and CSP together through the actual OIDC callback, silent renewal and API workflows behind the selected ingress. This validation does not configure TLS termination, edge HSTS or trusted proxy addresses.

### API HSTS and trusted forwarded-header hardening — 2026-11-26

HeapCare.Api PR #83 is merged as `afcafdce72ca6aff0e904a1a343911930040cb1d`. The API now emits a one-year HSTS max-age on non-Development requests only after the request scheme is trusted as HTTPS. `includeSubDomains` and `preload` remain disabled pending domain-wide review. The existing forwarded-header policy remains restricted to explicit `ReverseProxy:KnownProxies` IPs, one hop and symmetric headers.

All three hosted workflows passed: CI `45846891145`, Docker Integration Verification `45846891086`, and PostgreSQL Migration Rehearsal `45846891178`. Integration tests assert the exact one-year HSTS duration, absence of subdomain/preload directives, and that an untrusted forwarded scheme cannot enable HSTS.

**Still open:** the API does not terminate TLS. The selected ingress must install valid certificates, redirect HTTP to HTTPS, set edge HSTS, and provide the exact trusted proxy IPs. Do not enable HSTS preload or includeSubDomains until every affected hostname is confirmed HTTPS-only.

### Backup, restore and disaster-recovery operational baseline — 2026-11-26

Added `operations/backup-restore-disaster-recovery.md` as the required deployment-independent operational baseline. It inventories PostgreSQL, identity, object/document storage, key custody, configuration/secrets, audit logs and rebuildable indexes; defines the decisions required for RPO/RTO, backup frequency, retention, geographic resilience and restore-drill cadence; and sets minimum encryption, access, isolation, validation and approval controls. It explicitly distinguishes local migration rehearsal from provider-native backup/PITR and a successful disaster-recovery exercise.

**Still open and release-blocking:** select the hosting provider and database/storage topology; approve per-deployment RPO/RTO and retention; configure provider-native encrypted backups and WAL/PITR where required; establish KMS/HSM custody and break-glass recovery; and complete a successful isolated restore drill. Tenant-level recovery is not implemented or evidenced and must not be promised. The starting recommendation is a monthly restore drill, subject to formal owner approval.

### Strict no-inline CSP and production-container browser acceptance — 2026-11-26

Web PR #61 removed the last `unsafe-inline` CSP allowance after inspection of the current React components, entry HTML and CSS assets found no inline style attributes or inline script blocks. The Nginx response now restricts scripts and styles to `'self'`. CI run `45851992088` passed unit tests, build, browser E2E, real-OIDC suite discoverability and Docker security-header smoke.

Web PR #62 added a dedicated Playwright configuration and Compose override so the same 20 browser workflow tests run against the built static image served by Nginx under the actual enforced CSP. CI run `45853231713` passed all 20 tests, all CSP response-header assertions and all fail-closed startup checks (wildcard, HTTP without local opt-in, and missing required origins). Test fixtures now derive their mocked CORS allow-origin from the actual browser request, so both Vite and Docker-served origins are covered.

This closes the repository/CI production-container browser acceptance gap for the test origins. It does not validate actual production hostnames, TLS termination, trusted proxy configuration, the hospital's real identity provider, third-party origin inventory or production ingress. Those remain deployment gates.

### Development CORS loopback hardening — 2026-11-26

HeapCare.Api PR #85 is merged as `5b368f53a108dface6e43844d1be28b62bc3c509`. CORS validation no longer bypasses checks in Development: only explicit HTTP(S) loopback origins are accepted (`localhost`, `*.localhost`, IPv4/IPv6 loopback). Remote hosts, wildcards, credentials, paths, queries and fragments are rejected in every environment. Origins are normalized and deduplicated. HeapCare.Local's `http://localhost:8080` origin remains supported.

All three hosted workflows passed: CI `45853970275` (Release build, EF model-drift check and 161 passing unit/integration tests: 12 unit + 149 integration), Docker Integration Verification `45853970325`, and PostgreSQL Migration Rehearsal `45853970298`. The tests cover local hostnames, `.localhost`, IPv4/IPv6 loopback and negative remote/wildcard/path/credential cases.

This closes the Development CORS validation bypass at code/CI level. Production still requires the exact deployed Web origin to be configured when cross-origin, and actual CORS/CSP/OIDC flows must be verified behind the selected ingress.

### Nginx version disclosure hardening — 2026-11-26

Web PR #63 is merged as `b2e0916b38ed5fe8e031bbf7c5b754b133a6b538`. The Nginx server configuration sets `server_tokens off`, and Docker CI fails if the response exposes `Server: nginx/<version>`. CI run `45854681476` passed dependency audit, unit tests, production build, all browser E2E tests, real-OIDC suite discoverability and Docker smoke, including the strict-CSP browser suite and version-header assertion.

This reduces version disclosure only; it does not replace timely Nginx base-image patching or production ingress hardening.

### Production ingress and network-security baseline — 2026-11-26

Added `operations/production-ingress-and-network-checklist.md` to translate the existing API/Web controls into deployment acceptance requirements. It covers exact hostnames, TLS/HTTPS/HSTS, trusted proxies, CORS/CSP alignment, route exposure, private database/identity/observability networks, egress, edge rate limiting and evidence/sign-off.

**New explicit deployment gate:** `/metrics` is mapped without endpoint-level authorization metadata and is intended for Prometheus only. The selected ingress/network must prevent public access and permit only the approved monitoring source or an authenticated internal scrape path. Health endpoints must be restricted to approved orchestrator/monitoring sources. No production ingress or network policy has yet been selected or validated.

### Prometheus metrics source-network restriction — 2026-11-26

HeapCare.Api PR #86 is merged as `1a5a722a3342cf5fbaf439418717570189e808ea`. Outside Development and Testing, API startup now requires explicit `Observability:Metrics:AllowedNetworks` CIDRs. A middleware guard runs after trusted forwarded-header processing and returns 404 for `/metrics` requests whose effective remote address is outside those networks. Missing, malformed, whitespace-ambiguous and unrestricted `/0` CIDRs are rejected. HeapCare.Local Development and isolated Testing retain their existing scrape behavior.

All hosted checks passed on the merged change: CI `45856072221` (Release build, EF model-drift check, 12 unit tests and 162 integration tests), Docker Integration Verification `45856072556`, and PostgreSQL Migration Rehearsal `45856072217`.

**Still open:** select the actual production monitoring source CIDRs, configure exact trusted proxy IPs, and enforce the route as private at the selected ingress. This is a network allowlist, not user authentication. Health endpoint exposure and production ingress behavior remain deployment-specific acceptance gates; no actual hosting topology has been validated.

### Metrics allowlist address-family compatibility — 2026-11-26

API PR #88 is merged as `d67b2d77ae6d4f1db56de3150b77d0284a388984`. The metrics guard now normalizes IPv4-mapped IPv6 remote addresses before comparing against configured IPv4 CIDRs and explicitly denies requests with no remote address. Regression tests cover both cases; the configured CIDR allowlist is unchanged.

All hosted checks passed: CI `45856806161` (12 unit and 164 integration tests), Docker Integration Verification `45856806071`, and PostgreSQL Migration Rehearsal `45856806255`.


### Patient identifier organization-scope regression — 2026-11-27

HeapCare.Api PR #96 is merged as `41533bf7dfe34cc16fdd3f90aa99c1b3726ecc7f`. PostgreSQL-backed integration coverage verifies that a hospital-scoped HospitalAdmin can operate only within its assigned hospital, while cross-hospital policy/identifier access and department-scoped access to hospital-wide MRN operations are denied with non-disclosing 404 responses. The tests assert rejected audit attribution, no patient/MRN data in denied responses, and unchanged policy/identifier state after denied writes. Hosted PR CI, push CI and Docker Integration Verification all passed on head `853a24c5377b3ba94bfa14281b439f50f5508fb8`.

This closes a focused patient-identifier organization-scope regression gap; it does not establish exhaustive organization-scope coverage for every domain or production acceptance.


### OPD encounter role-denial regression — 2026-11-27

API PR #97 merged as `ac99bd91602e03d3e6cd483a8bdba0971395eca7`. PostgreSQL-backed integration coverage verifies that FacilityManager, FacilityViewer and PatientRegistrar memberships cannot read or mutate OPD encounters. The regression exercises direct and queue-linked reads, encounter creation, note updates, signing, amendments, closure and cancellation. Each denied operation must return 403, disclose no clinical note content, leave the encounter unchanged, and create a denied audit event attributed to the validated tenant, issuer and subject. Hosted Release build/test, push build/test and Docker Integration Verification all passed on head `19e9781bbe0a15c5a27fea4355de73def42c6844`.

This closes the identified non-clinical role-denial slice for the OPD encounter endpoints. It does not replace clinical workflow acceptance, clinical governance review, or exhaustive scope validation for every future clinical capability.


## Platform stabilization and public website update — 2026-11-27

### Authorization regression coverage

- [HeapCare.Api PR #98](https://github.com/HeapCare/HeapCare.Api/pull/98) merged as `71fb901eafdde5e557c7b3257070317cf8c9c390`. FacilityManager was added to the negative appointment/availability and billing matrices; FacilityManager and FacilityViewer are both denied patient registration and hospital patient search. Release CI, push CI and Docker Integration Verification passed.
- [HeapCare.Api PR #99](https://github.com/HeapCare/HeapCare.Api/pull/99) merged as `45dfdcb726a85df04a1daf9e76d96dfdd25a2014`. HospitalAdmin is explicitly verified to have no implicit clinical encounter access. The PostgreSQL-backed test exercises encounter read by ID and queue, creation, note update, sign, amendment, close and cancel; it verifies no clinical note disclosure, no state mutation and denied audit attribution. Release CI, push CI and Docker Integration Verification passed.
- [HeapCare.Api PR #100](https://github.com/HeapCare/HeapCare.Api/pull/100) merged as `0a8f326cea2be23984a68e90e100188b311b7a4b`. Hospital- and department-scoped HospitalAdmin memberships are denied tenant-wide audit query and CSV export; the tests retain the persisted audit-read grant and verify the service-level tenant-wide scope boundary. Release CI, push CI and Docker Integration Verification passed.

### Route-level authenticated Web workspaces

[HeapCare.Web PR #64](https://github.com/HeapCare/HeapCare.Web/pull/64) merged as `1a73684e7ac3c4660cf343855499a594792401c3`. Existing permission-aware URLs now render one module workspace at a time under the shared authenticated shell. The change covers overview, reception, patient registration, appointments, queue/walk-ins, clinical, billing, facilities, staff, MRN policy, memberships and platform tenant lifecycle. Module components remain mounted while inactive routes are hidden so local form state survives navigation. Legacy hash links and non-module accessibility anchors are preserved. Vitest, TypeScript/production build, Chromium E2E, real-OIDC suite discovery and Docker/CSP smoke passed after updating end-to-end journeys to navigate between routes.

**Remaining Web architecture work:** extract the inline reception and patient-registration markup into independent page components, continue component ownership extraction for each module, and execute authenticated local accessibility acceptance. The route conversion is an incremental architecture milestone, not completion of the full Web refactor.

### Public HeapCare website

- [HeapCare.Site PR #3](https://github.com/HeapCare/HeapCare.Site/pull/3) merged as `74094e98012f5356cb3b03ff5bd040157ffdc87d`: redesigned public homepage/navigation, full module catalogue with implementation/planned status labels, technology page, About/Contact, metadata and responsive design.
- [HeapCare.Site PR #4](https://github.com/HeapCare/HeapCare.Site/pull/4) merged as `d43021891268af376aa1cd3349bbb4ca0ce018d9`: direct dependency/version inventory, additional tooling and explicit distinction between current technologies and planned ABDM/ABHA, FHIR, HL7 and DICOM integrations.
- [HeapCare.Site PR #5](https://github.com/HeapCare/HeapCare.Site/pull/5) merged as `6e293ace962e7633e120c44ec9c887157370175b`: provider-neutral .NET 8 container, non-root runtime, port 8080, /health endpoint and container build CI.

The repository build and container build pass. The custom domain's live deployment is **not verified**: the repository has no provider-specific deployment workflow and no hosting-provider deployment connection has been confirmed. Do not report the new build as live until the chosen provider deploy and domain health checks are completed.


### Route workspace extraction and full-stack integration — 2026-11-27

**Web route-component extraction is complete for the current implemented hospital workspaces:**

- [Web PR #65](https://github.com/HeapCare/HeapCare.Web/pull/65) — patient registration extracted into a dedicated workspace with duplicate-review and acknowledgement behavior retained.
- [Web PR #66](https://github.com/HeapCare/HeapCare.Web/pull/66) — reception patient search extracted with permission-gated MRN lookup.
- [Web PR #67](https://github.com/HeapCare/HeapCare.Web/pull/67) — appointment booking and queue/walk-in flows extracted into dedicated workspaces.
- All three PRs passed Vitest, TypeScript/production build, Chromium E2E, real-OIDC suite discovery and Docker/CSP checks before merge. API orchestration and workflow state remain in the authenticated workspace controller; route components own presentation and delegate actions.

**Public-site technology and runtime verification:**

- [Site PR #6](https://github.com/HeapCare/HeapCare.Site/pull/6) — exact internal package versions, OpenTelemetry instrumentation, container image tags and scripting tools.
- [Site PR #7](https://github.com/HeapCare/HeapCare.Site/pull/7) — browser standards, npm/Git/Chromium, Google Fonts, jsDelivr and Simple Icons inventory; technology cards have logo images with alternative text.
- [Site PR #8](https://github.com/HeapCare/HeapCare.Site/pull/8) — CI starts the .NET 8 Docker container, probes /health and smoke-tests the homepage, features, technology, About and Contact routes; container cleanup is verified.
- [Site PR #10](https://github.com/HeapCare/HeapCare.Site/pull/10) — final direct .NET package inventory, including the ASP.NET Core observability companion, dotnet-ef and cryptography/XML dependencies.

**Cross-repository full-stack evidence:**

[HeapCare.Local run #45946047300](https://github.com/HeapCare/HeapCare.Local/actions/runs/45946047300) passed after Web PR #67. The full nine-service stack, OIDC discovery, Prometheus/Jaeger/Elasticsearch/Collector readiness, Kibana data-view provisioning, API/Npgsql traces, Collector-fed API metrics, real Keycloak-issued tokens, PostgreSQL-persisted tenant memberships, tenant isolation, cross-tenant membership mutation denial, membership revocation/reactivation and real Authorization Code + PKCE browser sign-in all passed. The browser suite reported 1 passed test.

Exact checked-out commits recorded by the run:

- HeapCare.Local: `13e815acbeb9a195ba749bb50814f97876e7e70a`
- HeapCare.Api: `0a8f326cea2be23984a68e90e100188b311b7a4b`
- HeapCare.Web: `1b7da7aa8e4d5e3ebae79f17986e44560d68edd9`
- HeapCare.Observability: `935f6629d5abb92296385ec2e0c1ec04060b3fb7`

**Remaining acceptance gates:** manual authenticated role/workflow execution, UI visual review, keyboard-only and screen-reader review, contrast/zoom/reflow, Mac/Windows launcher acceptance, production ingress/TLS, provider-native PostgreSQL PITR/restore, signing-key rotation, clinical governance and pilot sign-off remain open. Hosted Linux CI is not a substitute for those activities.

**Backlog reconciliation:** API issue #4 (obsolete roadmap tracker) and Site issue #2 (fake demo login on the informational site) are closed with rationale. Mobile issues #2/#3 remain blocked pending OTP service security/provider decisions and a consuming mobile application. Site issue #9 tracks the public-site deployment and remains blocked until the hosting provider/service, domain purchase and DNS authority are confirmed. No legacy or candidate domain is verified as owned or configured.


## HeapCare brand migration — 2026-11-27

The proposed product brand is now **HeapCare**. The migration is tracked in [ADR-017: HeapCare brand migration](../architecture/decisions/ADR-017-heapcare-brand-migration.md). This is a staged identity migration, not a product rewrite. Initial README branding changes are open as draft PRs across all 11 repositories. Source namespaces, project/assembly names, NuGet IDs, database identifiers, OIDC realm/client IDs, Docker/Compose names, deployment settings, organization/repository names and external URLs have not yet been changed. The `heapcare.in` domain has not been purchased or configured. Do not use it for production URLs, email addresses, redirect URIs, CORS or CSP until domain ownership, DNS and HTTPS are established. Complete the dependency inventory, coordinated source/configuration migration, administrative GitHub renames and full CI/Docker/integration/E2E/local-stack verification before closing this milestone.

## HeapCare identity migration — execution update (2026-11-27)

HeapCare is the product-facing brand. The following technical migrations are merged: HeapCare.ApiClient (PR #4), HeapCare.Caching (PR #6) and HeapCare.Observability (PR #5). Their latest migration-branch build/test/pack or build/test/format checks passed; the Observability 1.0.1 packages were published to GitHub Packages.

Public Site and authenticated Web branding changes have passed their build checks. Local launcher names have been updated and the local tooling/Keycloak/Compose/observability validation checks pass. The Local full-stack workflow is being updated to consume the new HeapCare Observability package identities and the API migration branch.

The API branch now references HeapCare.Observability and HeapCare.Observability.AspNetCore, with NuGet source mapping updated for the new package family. Its latest CI and migration rehearsal are still being validated. The API source namespaces, project/folder paths and solution name have not yet completed the full HeapCare rename.

Do not rename the existing Keycloak realm/client IDs, PostgreSQL database/role, EF migration history, connection-string key or persistent Docker volume names until a tested non-destructive identity/data migration is ready. The GitHub organization and repository names remain unchanged because the connected GitHub capability does not expose the required administrative rename operation. The heapcare.in domain is not purchased or configured; no external DNS, email, certificate, OIDC callback or production origin is assumed.

Remaining gates: finish API namespace/project/solution migration; publish and verify HeapCare.ApiClient and HeapCare.Caching packages; update all downstream consumers; complete Local full-stack Docker, API integration and browser E2E verification; update all cross-repository documentation/links; then perform administrative GitHub renames and the separately gated external identity/domain migration.

## HeapCare migration progress — 2026-11-27

HeapCare is the accepted product-facing brand. The migration is now in implementation rather than planning only. Observability, API Client, Caching, Site, Local, Database documentation, Mobile scaffold, OTP scaffold and the central documentation PR have been merged. The Web workspace identifier migration is in PR #69 and the API solution/runtime migration is in PR #101. API package restore/build and EF migration rehearsal have passed on a recent PR head; Docker integration, Web Docker/CSP and Local full-stack verification are being rechecked after the latest compatibility updates.

**Preserved intentionally:** GitHub organization/repository slugs, support email/domain, production Keycloak realm/issuer/client/audience, PostgreSQL database/role/schema, EF migration history and persistent Docker volumes. These are external or persisted contracts, not safe global-replacement targets. The API now prefers `ConnectionStrings:HeapCare` with a legacy `ConnectionStrings:CompileCare` fallback. Local session-storage keys are migrated without losing tenant/hospital selection.

**Administrative blockers:** GitHub organization/repository renames are not exposed by the connected GitHub actions in this session. HeapCare domain/DNS/email/certificates have not been purchased/configured. New private HeapCare NuGet packages need package read grants for production consumer repositories. Final completion requires those administrative tasks plus green latest-head and post-merge verification.


### Merge completion — 2026-11-27

The API technical migration (PR #101) and Web frontend identifier migration (PR #69) are now merged, alongside the previously merged Site, Local, Database documentation, Observability, API Client, Caching, Mobile/OTP scaffold and central documentation changes. The API final PR-head checks passed: .NET restore/build/test, EF pending-model-change detection, Docker integration verification, and legacy-to-HeapCare PostgreSQL migration rehearsal. The Web final PR-head checks passed: npm audit, unit tests, production build, browser E2E, Docker build and CSP smoke test. Post-merge API restore/build/test and EF checks passed; Web audit/unit/build/browser E2E/Docker checks passed; the Local main-branch nine-service full-stack, persisted multi-tenant, Keycloak PKCE and telemetry run passed.

The source/project migration is therefore merged across all 11 repositories. Remaining work is limited to external/admin identity changes and final post-merge verification: GitHub organization/repository slugs, domain/email/DNS/certificates, production Keycloak realm/client/audience, private package read grants, and data-safe database/persistent-volume renaming where genuinely required.


## HeapCare migration verification refresh — 2026-11-27 15:30 IST

The source/project migration PRs are merged: API #101, Web #69 and Site #12, along with the previously merged shared libraries, Local, Database documentation, scaffolds and migration documentation. Latest post-merge API, Web, Site, Database and Observability CI runs are green.

The latest Local full-stack run identified a Keycloak import filename compatibility defect. The realm export file had been renamed to `heapcare-realm.json`, but its internal realm ID intentionally remains `compilecare`; Keycloak requires the import filename to match that realm ID. The correction restores `local/keycloak/compilecare-realm.json` and updates the Docker Compose mount, Keycloak reconciliation script default and Local CI validation/import paths. The realm, issuer, OIDC client and audience IDs remain unchanged to protect existing authentication state. The post-fix Local validation and full-stack workflow is running; do not mark this check green until its completed result is confirmed.

A previous full-stack baseline did pass the nine-service environment, persisted tenant workflows, real Keycloak PKCE browser acceptance and API telemetry. The current rerun is specifically validating the merged HeapCare checkout/path/package changes plus the corrected Keycloak import contract.

The repository source trees for API, Web, Site, Database, Observability and documentation contain no remaining HeapCare-named file paths. Local retains only `local/keycloak/compilecare-realm.json`, intentionally named to match the preserved legacy realm ID. GitHub organization/repository slugs remain unchanged because repository/organization administration is not exposed by the connected GitHub actions. No HeapCare domain, DNS, email, certificate or production origin is configured. Production Keycloak migration, package read grants and any persisted database/volume rename remain separate, non-destructive release gates.


### Post-fix Local full-stack verification — 2026-11-27 15:35 IST

Local run [#45978681883](https://github.com/HeapCare/HeapCare.Local/actions/runs/45978681883) has completed successfully. Both jobs passed. The full-stack job confirms the nine-service Compose stack, real Keycloak-issued identities, persisted multi-tenant workflows, real Authorization Code + PKCE browser acceptance and API telemetry. The validation job confirms macOS/Windows launchers, Keycloak realm import, OpenTelemetry Collector configuration, Compose configuration and the Docker observability pipeline.

The Keycloak import filename was restored to `compilecare-realm.json` to match the intentionally preserved `compilecare` realm ID. The import filename is the only remaining HeapCare-named file path in the Local repository and is a Keycloak compatibility requirement, not an unfinished product-facing label.

Follow-up Local run [#45979293490](https://github.com/HeapCare/HeapCare.Local/actions/runs/45979293490) is now checking the merged API `main` branch and the `HeapCare.Local` checkout directory. Do not treat this follow-up as complete until both jobs finish green.


### Latest HeapCare CI synchronization — 2026-11-27 15:45 IST

- Web run [#45979824743](https://github.com/HeapCare/HeapCare.Web/actions/runs/45979824743) passed after the real-OIDC browser suite was switched to the `HEAPCARE_BROWSER_TEST_USER_FILE` contract used by Local CI.
- The CI-only synthetic Keycloak client, synthetic user prefix/email, browser credential artifact names and isolated Compose project/container names now use HeapCare identifiers. The existing local realm, issuer, production Web client, API audience and persistent database/volume identifiers remain unchanged.
- Local CI now uses a single concurrency group to cancel superseded full-stack runs and checks out the merged API `main` branch into `HeapCare.Api`, Web into `HeapCare.Web`, Observability into `HeapCare.Observability`, and this repository into `HeapCare.Local`.
- The latest Local workflow run [#45980070858](https://github.com/HeapCare/HeapCare.Local/actions/runs/45980070858) is executing against those final identifiers. Its shell/PowerShell, secret-placeholder and Keycloak JSON validation steps have passed; Docker realm-import smoke, Docker observability smoke and the nine-service stack are still running. The earlier full-stack baseline #45978681883 passed.


## Administrative HeapCare rename verification — 2026-11-27

The GitHub organization login is now `HeapCare` (organization ID `228320081`), and all 11 repositories are verified under the new owner with HeapCare repository slugs: HeapCare.Api, HeapCare.ApiClient, HeapCare.Caching, HeapCare.Database, HeapCare.Local, HeapCare.Mobile, HeapCare.Observability, HeapCare.OtpService, HeapCare.Site, HeapCare.Web and HeapCare.Doc. Repository IDs were preserved across the rename. The organization-scoped open-PR search returned no open PRs at this checkpoint.

This administrative rename does not change persisted or external identity contracts. Keep the existing Keycloak realm/client/audience identifiers, PostgreSQL database/role names, EF migration history and persistent Docker volume names until a separately rehearsed, reversible migration is approved. Update package access grants, external integrations, clone URLs and local remotes as part of post-rename validation. Do not treat GitHub URL redirects as proof that all consumers have migrated.


## HeapCare post-rename CI integration findings — 2026-11-27

The first post-rename CI inspection found stale organization references in the API workflow's shared Observability checkout and in the API Client/Caching NuGet publish endpoints. The two package repositories' Release build, test and pack stages passed, but their publish jobs failed with HTTP 403 because they still pushed to `nuget.pkg.github.com/HeapCare`. Updated the API and Docker Integration Verification workflows to checkout `HeapCare/HeapCare.Observability`, and updated API Client and Caching publish feeds to `https://nuget.pkg.github.com/HeapCare/index.json`. The affected push-triggered workflows were started by those commits and were still running at the time of this note; package publication is not considered restored until those new runs pass and package consumption is verified from a clean restore.

The Local CI launcher checks still exercise the legacy `compilecare` Keycloak realm, OIDC client/audience, database names and persistent volume names. These remain deliberately preserved until a coordinated identity/data migration has a tested rollback and user-preserving cutover. The Local workflow is currently running after the README/launcher documentation update. The Web CI run for the current HeapCare README commit passed. The Observability workflow was triggered by its migration-status README update and is pending verification.

## HeapCare migration verification addendum — 2026-11-27

This addendum supersedes earlier execution notes above where they describe the GitHub organization/repository rename or package publication as still pending.

### Administrative rename and CI remediation
- The GitHub organization is now **HeapCare**, and all 11 repository slugs use the HeapCare name. Repository history and IDs were preserved.
- Post-rename CI references were corrected in `HeapCare.Api/.github/workflows/ci.yml`, `HeapCare.Api/.github/workflows/docker-test.yml`, `HeapCare.ApiClient/.github/workflows/ci.yml` and `HeapCare.Caching/.github/workflows/ci.yml`.
- Verified successful main-branch runs: API #45984268087, API Client #45984273706, Caching #45984278571 and Observability #45984332659.
- API Client run #45984273706 passed both build/test and the **Publish to GitHub Packages** job. Caching run #45984278571 passed build/test/pack and the **Publish to GitHub Packages** job. These runs verify publication from CI; package-read access for every production consumer/deployment identity still requires consumer-side validation.
- Additional successful checks include Web #45983835225, Local tooling #45983816999 and Database PostgreSQL backup/restore verification #45970305638. The public Site latest recorded run #45972538371 passed.
- The organisation-wide open-PR search returned no open PRs at this checkpoint.

### OTP service security hold
- `HeapCare.OtpService` main currently contains only a scaffold README and `.gitignore`; the earlier OTP implementation PR #1 was closed **without merge**. Its feature branch remains separate from main.
- PR #1 explicitly recorded unresolved security/architecture blockers: OTP hashing at rest, verification attempt limits, recipient/IP abuse controls, atomic distributed cooldown/rate limiting, request DTO and recipient validation, endpoint authorization/error review, concurrency coverage and expiry/resend/delivery-failure tests.
- PR #1 also recorded that an SMTP credential had previously been committed to Git history and must be revoked/rotated. Removing it from the current configuration is insufficient; confirm rotation with the credential owner and follow the repository's secret-incident process before any OTP implementation is reused.
- OTP delivery is therefore **deferred and not production-ready**. Do not merge or revive the old feature branch without a fresh security review and passing tests.

### Remaining release gates
- Validate package restore from each actual consuming repository and deployment identity using least-privilege package access.
- Complete the full post-rename local Docker/OIDC/tenant acceptance against the current main branches, not only individual repository CI.
- Preserve Keycloak realm/client/audience IDs, PostgreSQL identifiers, EF migration history and persistent Docker volumes until a separately tested identity/data migration is approved.
- Do not claim production readiness from green CI alone; privacy/legal review, production secrets and key management, provider-native backups/PITR, restore drills, deployment configuration and hospital pilot acceptance remain separate gates.


## Phase completion execution directive — 2026-11-27

This directive is authoritative for sequencing and supersedes earlier notes that suggest starting optional performance/capacity work or other new features ahead of the approved phases. Complete the original Phase 3, Phase 4 and Phase 5 scope before proposing out-of-scope additions.

### Execution order and current status

1. **Phase 3 — Database, identity and security: IN PROGRESS; release readiness not complete.** Continue in this order: finish tenant/organization and audit coverage for existing sensitive workflows; obtain pilot-approved audit reader roster, role mapping, retention/deletion/legal-hold policy and export governance; complete representative-data migration/release rehearsal with rollback evidence; define and execute provider-native encrypted backup/PITR and isolated restore drills after a hosting/database provider is selected; complete production signing-key/secret/encryption custody and rotation runbooks; then incident response, privileged access review, monitoring and qualified privacy/legal review. Do not invent retention, RPO/RTO or compliance claims. Owner-dependent approvals and provider configuration remain explicit blockers.
2. **Phase 4 — Core healthcare domain: PARTIALLY COMPLETE.** Complete the validated group/branch onboarding model and end-to-end tenant/hospital/department provisioning first. Finish the approved initial OPD workflow acceptance and defects. Patient contact storage/verification remains gated by ADR-012 security, privacy, key-management and retention decisions. Workforce identity linking, credential verification/clinical privileging and HR/payroll remain governed sub-roadmap items. Proceed through the already documented hospital-domain sequence only after requirements and acceptance criteria exist; every increment requires permissions, tenant scope, audit, data model, migration, tests and operational design. Do not invent clinical, workforce or financial policy.
3. **Phase 5 — User experience: IN PROGRESS; manual authenticated acceptance outstanding.** Execute the six-role acceptance matrix against the current merged Local/API/Web stack using synthetic identities/data. Complete keyboard-only workflows, focus/dialog behavior, accessible names/announcements, responsive checks, contrast, zoom/reflow and a real screen-reader pass. Capture sanitized authenticated screenshots for PlatformOperator, HospitalAdmin, FacilityManager, FacilityViewer, PatientRegistrar and ClinicalUser. Fix defects in small PRs with regression tests and paired documentation. Do not claim WCAG 2.2 AA conformance without a complete qualified evaluation.

### New verified evidence — 2026-11-27

HeapCare.Local run [#45993951443](https://github.com/HeapCare/HeapCare.Local/actions/runs/45993951443) passed on main commit `a2daae6a0f2cbb3eeea07d583ddf0459297b6eae`. The Local tooling validation and full nine-service Docker stack/API telemetry jobs both passed. The full-stack job verified real Keycloak-issued identities, persisted multi-tenant workflows and real Authorization Code + PKCE browser acceptance. This is automated full-stack evidence; it does not replace manual six-role acceptance, screen-reader review or local desktop validation.

### Scope control

- Do not start optional features, load/performance harnesses, mobile/OTP integration, patient portal or external integrations ahead of phase exit criteria.
- Keep OTP blocked until the historical SMTP credential is confirmed revoked/rotated and the implementation passes a fresh security review; do not reuse the old feature branch as-is.
- Preserve existing production identity/database/volume contracts unless a separate, rehearsed, reversible migration is required and approved.
- Every implementation slice needs tests, security/privacy implications, migration/rollback notes where applicable and updated documentation.
- A phase is complete only when its documented exit criteria and evidence are satisfied. Green CI alone is not phase completion or production approval.

**Out-of-scope work remains paused until Phases 3–5 exit criteria are met.**


## Parallel execution and phase checkpoint — 2026-11-27

### Completed in this execution
- Web PR #71 merged after both CI runs passed: unit tests, production build, Chromium browser E2E, real-OIDC suite discoverability and Docker smoke validation. The change adds regression coverage that membership-list failures are announced through an alert and rejected membership creation preserves the entered OIDC subject while announcing an actionable conflict.
- Documentation PR #168 merged. It records the source-verified current tenant-to-hospital hierarchy and explicitly leaves HospitalGroup/Branch semantics as an open product decision; no speculative schema change is authorized.
- The organization-wide open-PR search returned no open pull requests immediately after these merges.

### New independent validation work in progress
- API PR for audit pagination stability: add a PostgreSQL-backed integration test proving records with identical OccurredAtUtc values are ordered by descending ID consistently across page boundaries. This protects the documented deterministic offset-pagination contract. CI must pass before merge.
- Phase 5 remains open for authenticated six-role manual acceptance, keyboard/focus behavior in authenticated workflows, responsive/zoom checks and an actual screen-reader pass. Existing public-shell E2E checks and Web CI are not substitutes for those manual checks.

### Live pending-task register
| Phase | Status | Remaining work / gate |
|---|---|---|
| 3 — Database, identity and security | In progress | Finish sensitive workflow scope/audit coverage; obtain pilot-approved audit reader/role, retention/deletion/legal-hold and export governance; migration/rollback rehearsal with representative data; select provider and prove encrypted backup/PITR plus isolated restore; production key/secret custody and rotation; incident response, privileged-access review, monitoring and qualified privacy/legal review |
| 4 — Core healthcare domain | Partially complete | Confirm tenant/hospital/department onboarding and hospital-group/branch product model; finish OPD acceptance/defects; keep patient contact storage gated by ADR-012; define governed workforce identity/credentialing/privileging and HR/payroll sub-roadmap |
| 5 — User experience | In progress | Six-role authenticated acceptance using synthetic identities/data; keyboard-only workflows; focus/dialog behavior; accessible announcements; responsive, contrast, zoom/reflow and screen-reader validation; sanitized role-based evidence |

### Execution controls
- Independent repository tasks are developed concurrently where they do not share files or depend on unmerged changes. CI is monitored while independent work proceeds; no task is marked complete until its own checks and integration evidence are verified.
- No separate coding-agent/background-worker facility is exposed in this session. GitHub API operations and independent repository updates can be orchestrated concurrently, but this does not imply persistent workers.
- Owner/provider/legal/pilot decisions remain blockers and will not be invented. Preserve existing Keycloak, database, migration-history and persistent-volume identifiers until a rehearsed reversible migration is approved.
- OTP remains on security hold pending confirmation that the historical SMTP credential is revoked/rotated and a fresh security review. No production-readiness or WCAG 2.2 AA conformance claim is made from CI alone.


## Additional verified phase evidence — 2026-11-27

### Phase 3 — API security and audit regression coverage
The following test-only API changes were merged after both the API CI and Docker integration verification jobs passed on their respective PR heads:

- **PR #111:** PostgreSQL-backed stable audit pagination ordering when events share the same timestamp.
- **PR #112:** malformed, empty-GUID and repeated `X-Tenant-ID` values are rejected with HTTP 400 Problem Details.
- **PR #113:** CSV export neutralizes all spreadsheet formula prefixes (`=`, `+`, `-`, `@`) including a formula preceded by whitespace.
- **PR #114:** audit queries reject non-UTC offsets and future end times.
- **PR #115:** department-scoped HospitalAdmin cannot discover, read or mutate staff profiles assigned outside the department; permitted in-scope reads remain available and denied attempts are audited.

These are focused regression tests over existing implementation behavior. They do not approve audit governance, retention/legal hold, export purpose, production key custody or hosting-provider backup policy.

**API PR #116 merged:** CSV export quoting/escaping for commas, embedded double quotes and newlines is covered by a PostgreSQL-backed regression test. API CI and Docker integration verification both passed on the final PR head.

### Phase 5 — authenticated workspace regression coverage
- **Web PR #71 and #73** are merged with CI green: membership error announcements and named lifecycle confirmation grouping.
- **Web PR #74 and #75** are merged with CI green: mocked appointment, billing, OPD and session-renewal fixtures now use HeapCare demo names; the legacy `compilecare-web` OIDC client identifier remains intentionally unchanged in test session storage.

These automated checks do not replace the manual six-role authenticated accessibility matrix, real screen-reader pass, contrast/zoom/reflow checks or WCAG 2.2 AA evaluation.

### Product-facing identity cleanup
- **Site PR #14 and #15** are merged with Site CI green. The retired HeapCare support address is no longer published; the public site accurately states that the official HeapCare support contact is not yet published.
- **Docs PR #168–#174** are merged, including current tenant/hospital hierarchy documentation, ADR owner terminology, migration execution status and Phase 5 membership accessibility evidence.

### Execution rule
CI is treated as a parallel validation lane. Independent API, Web and documentation changes were started while other workflows were running. Merge decisions were made only after each PR's own required checks passed and the exact PR head was verified.


### Patient duplicate-review role denial — 2026-11-27

- **API PR #117 merged** as `0407dcfeefdd16d63299e74ba8cc27a9e3ef5e86`. PostgreSQL-backed integration coverage verifies that FacilityManager and FacilityViewer cannot invoke patient duplicate-candidate review. The regression asserts HTTP 403, no submitted patient-name disclosure in the response, and denied audit attribution with tenant, validated issuer/subject, hospital resource and status.
- This closes a focused negative permission case for the existing duplicate-review capability. It does not constitute complete role/capability or organization-scope coverage, and it does not approve the product role model.


## Phase execution update — 2026-11-27 (API PR #118)

### Phase 3 — duplicate-review authorization regression

HeapCare.Api PR [#118](https://github.com/HeapCare/HeapCare.Api/pull/118) merged as `56bfd4018b596834686eb00fb72d425dae10842d` after the hosted API CI and Docker/PostgreSQL integration verification workflows passed on the exact PR head `42135b92862d4f537ef68aa5404f4b36d3ca66bd`. The new PostgreSQL-backed integration test verifies that a persisted `ClinicalUser` cannot invoke patient duplicate-candidate review: the API returns HTTP 403, the response does not disclose submitted patient names, and the denied audit event is attributed to the tenant, validated issuer/subject and hospital resource. This complements PR #117's FacilityManager and FacilityViewer denials. The implementation grants duplicate review to HospitalAdmin and PatientRegistrar; this test-only increment changes no runtime permission. The role design still requires pilot/product-owner approval and complete positive/negative endpoint and organization-scope coverage.

### Phase 4 — next validation focus

Continue validating the existing hospital onboarding and OPD workflows against the merged Local/API/Web stack. The hospital-group/branch model remains an explicit product decision; do not add schema or provisioning semantics until the owner-approved model exists. Verify tenant → hospital → department provisioning, then execute the synthetic-data OPD journey and close reproducible defects with API/Web regression tests and paired documentation. Patient contact storage remains gated by ADR-012 approvals. Workforce identity, credential verification and clinical privileging remain separately governed sub-roadmap items.

### Phase 5 — next validation focus

The existing Playwright accessibility suite currently verifies the public shell (skip link, focus visibility, landmarks, accessible names, image alt text, reduced motion and narrow viewport overflow). It does not establish authenticated six-role accessibility acceptance. Continue with authenticated acceptance for PlatformOperator, HospitalAdmin, FacilityManager, FacilityViewer, PatientRegistrar and ClinicalUser, capturing sanitized evidence for keyboard workflows, focus/dialog behavior, announcements, responsive layouts and zoom/reflow. A real screen-reader pass and qualified WCAG 2.2 AA evaluation remain separate requirements; do not claim conformance from automated checks alone.

### Parallel execution constraints

Independent API authorization, Local integration, Web UX and documentation work may proceed concurrently when they do not modify shared files or depend on unmerged changes. CI is monitored per PR and merges require green required checks on the exact PR head. Provider selection, production identity/key custody, retention/legal-hold policy, pilot role approval and hospital-group/branch semantics remain owner-dependent gates; no production or clinical policy is inferred.


## Phase execution update — 2026-11-27 (API #119–#120, Local #30 and Web #76)

### Phase 3 — duplicate-review authorization, scope and audit evidence

- **HeapCare.Api PR #119** merged as `bb3ab9868b58de247033b7a2695878c10e218902`. PostgreSQL-backed coverage proves a persisted HospitalAdmin can review duplicate candidates and receives the matching hospital-local candidate. Hosted API CI and Docker/PostgreSQL integration verification passed.
- **HeapCare.Api PR #120** merged as `25f4b300ce1bd85bb99a4db57f16affeee45f617`. A hospital-scoped HospitalAdmin can review candidates in the assigned hospital; a same-tenant request for another hospital receives a non-disclosing 404. Both the successful and rejected requests are asserted in the audit store with tenant, issuer, subject, action, resource, outcome and status attribution. API CI and Docker integration verification passed.
- These changes add test evidence only. They do not change role grants or approve which roles should have duplicate-review responsibility. The current persisted implementation continues to grant that capability to HospitalAdmin and PatientRegistrar; role approval remains a product/clinical-owner gate.

### Phase 4 — authenticated duplicate-review acceptance

- **HeapCare.Local PR #30** merged as `981a63809db69861ec316d8bcb82a8d8ae5fee7d`. The isolated nine-service acceptance workflow now obtains real Keycloak-issued tokens for the persisted tenant roles and exercises duplicate review through the API: HospitalAdmin and PatientRegistrar can retrieve the synthetic hospital-local candidate; FacilityManager, FacilityViewer and ClinicalUser are denied without patient-name disclosure.
- The full Local run [#46031858710](https://github.com/HeapCare/HeapCare.Local/actions/runs/46031858710) passed Local tooling validation and full nine-service Docker/API telemetry verification, including the real Authorization Code + PKCE browser journey.
- This closes a specific API workflow acceptance slice; it does not complete the OPD/appointment/billing end-to-end acceptance or constitute live clinical/patient data validation.

### Phase 5 — authenticated role UI regression

- **HeapCare.Web PR #76** merged as `8dcb5417bf9b690d7bcdff97ae96e929ef9beafa`. Playwright now verifies the authenticated workspace's main landmark, keyboard skip-link focus, accessible control names and labelled form fields for all five persisted tenant-role contexts.
- The role contexts in PR #76 are intentionally UI-mocked. Real OIDC browser acceptance remains a separate lane, and manual six-role accessibility, real screen-reader, zoom/reflow, contrast and qualified WCAG 2.2 AA evaluation remain outstanding.


## Parallel phase execution update — 2026-11-27 (HeapCare migration continuation)

### Phase 3 — database, identity and security hardening

- **HeapCare.Api PR #122 merged** as `cab471452c6c1904fde0f7b407001c065e64dd00` after the Release build, EF Core model-drift check, full API test suite and Docker/PostgreSQL integration verification passed on the exact PR head. The new regression proves that an audit CSV export includes requesting-tenant actor metadata and excludes another tenant's actor metadata.
- **HeapCare.Api PR #123 merged** as `9a4a517add68c5af08fc914533b387e6883440be`. PostgreSQL-backed coverage verifies that a department-scoped PatientRegistrar can transition a queue entry inside the assigned department, receives a non-disclosing 404 for a queue entry in another department, and leaves the denied entry in `Waiting`. Both allowed and rejected transitions are asserted in the audit store with validated tenant, issuer, subject, action, hospital resource, outcome and HTTP status. Release build, EF model-drift, complete API tests and Docker/PostgreSQL verification passed on exact head `69c73091e18c142f2240b64efe1c511f04dc1be8`.
- These tests harden existing authorization and audit behavior; they do not change role grants or approve the product role model. Audit retention/legal hold, export-purpose governance, operational access reviews, privacy/legal approval, production key custody and provider-native backup/PITR remain separate release gates.

### Phase 4 — hospital workflows and real OIDC acceptance

- **HeapCare.Web PR #79 merged** as `0e4310b3ff1701da6f7e3518b9f3ad1efadf5251`. The real-Keycloak ClinicalUser browser journey now consumes a synthetic in-progress walk-in, requests the live department/date queue, opens the linked encounter, retrieves department-scoped patient identity and requires identity confirmation. The test intentionally does not enter or sign a clinical note.
- **HeapCare.Local PR #31 merged** as `bddfa886e0c35e29cda0e5bda7a435a75e2f1a63`. It creates separate runner-temporary Keycloak identities for tenant roles, synthetic tenant/hospital/department records and an in-progress OPD walk-in. It verifies with real-issued tokens that PatientRegistrar and ClinicalUser can read the synthetic department queue while FacilityManager is denied. Its full browser acceptance was subsequently completed in Local #36 / run #46046213095.
- The earlier Local full-stack run `46035812237` built and started all nine services and passed persisted-role API checks, but the browser test expected the empty clinical note's `Review and sign` button to be enabled. The application correctly kept signing disabled because no note content had been entered. **HeapCare.Web PR #81 merged** to correct this test expectation without entering, saving or signing clinical content.
- The later Local PR #36 full-stack run `46046213095` passed on its exact head, including the nine-service stack, persisted-role API workflow, logical backup/restore and 11 real-OIDC browser tests. It supersedes the earlier in-progress and failed attempts described above; Local PR #31 is merged.
- The later Local #36 run includes the real-OIDC PatientRegistrar appointment booking/check-in and synthetic billing/receipt UI assertions. Hospital pilot validation, role approval and operational sign-off remain separate gates.

### Phase 5 — authenticated workspace quality and accessibility

- **HeapCare.Web PR #80 merged** as `829e93ac19a96cb1d0016500313fa957630e8d14`. Real-Keycloak browser checks cover HospitalAdmin, FacilityManager, FacilityViewer, PatientRegistrar, ClinicalUser and PlatformOperator. For each identity, the suite checks a single main landmark and level-one heading, first-stop skip-link focus and visible outline, focus transfer to main content, and page-level overflow at 768px, 375px and 320px.
- **HeapCare.Web PR #81 merged** as `2d35fd4b72ce21ca5e44ca5ba31c5434349ab044`. The ClinicalUser browser test now asserts that signing remains disabled for an empty note after patient identity confirmation. It remains non-mutating and does not create or sign clinical content.
- Web main CI `46036812116` passed unit tests, production build, Chromium E2E, real-OIDC suite discovery and Docker/CSP smoke on the merged main head. The new authenticated accessibility spec still needs execution in the full nine-service real-Keycloak environment; that acceptance is part of the active Local #31 run and is not yet claimed as passed.
- Automated browser coverage does not replace manual keyboard workflow review, screen-reader testing, contrast and zoom/reflow review, desktop/tablet evidence, or a qualified WCAG 2.2 AA assessment. No conformance claim is made.

### Parallel execution and release constraints

- Phase-specific merged evidence in this increment: API PRs #122, #123 and #124; Web PRs #79, #80, #81, #85, #86, #87 and #88; Local PRs #31, #33, #34, #35 and #36. Local #36 run #46046213095 passed full nine-service/real-OIDC verification. Local #37 is the follow-up replay for Web #88's appointment availability live-region update.
- Continue API security, Local integration, Web experience and documentation work in parallel when repository/file dependencies allow. Merge only after required checks pass on the exact current head. Never infer cross-repository acceptance from a component-only CI pass.
- Preserve database, migration-history, Keycloak realm/client, OIDC issuer/audience and persistent-volume identifiers. Production backup/PITR, encryption-key custody, provider/region selection, isolated restore drill, retention/legal-hold policy, pilot role approval and hospital-group/branch semantics remain explicit owner-dependent gates.


---

## Phase execution update — 2026-11-27 (Local PRs #33–#35; Web PR #85)

This update supersedes the earlier Phase 4 note that Local PR #31 was still open. The accepted baseline is now the merged Local main head exercised by run [#46041076667](https://github.com/HeapCare/HeapCare.Local/actions/runs/46041076667).

### Phase 3 — authorization and database recovery evidence

- **HeapCare.Local PR #33** merged after the exact-head Local run passed. Its real-Keycloak API acceptance verifies that ClinicalUser cannot read a synthetic cashier invoice or record a payment, and FacilityViewer cannot read the invoice. These are negative authorization checks against runner-local synthetic records; no runtime grants were broadened.
- **HeapCare.Local PR #34** merged the disposable PostgreSQL logical backup/restore smoke step. CI creates a custom-format `pg_dump` archive, restores it into a separate empty database using `pg_restore`, and compares every application table's row count. The current-main Local run passed this step.
- This proves the local CI dump/restore path only. It does not prove provider-native backups, WAL archiving/PITR, key recovery, encrypted off-site custody, cross-region recovery, tenant-level restore or production RPO/RTO. Those remain release gates.

### Phase 4 — provider, availability and OPD workflow validation

- **HeapCare.Local PR #35** merged after Local tooling validation and the complete nine-service real-Keycloak run passed. The synthetic API setup creates a provider staff profile and effective department assignment, publishes a future Asia/Kolkata availability session, and verifies PatientRegistrar can discover the session and read slots. FacilityManager cannot publish availability, ClinicalUser cannot read it, and PatientRegistrar cannot create staff profiles.
- The same Local run also passed persisted-role API acceptance, the existing synthetic OPD/billing workflows, and the real Authorization Code + PKCE browser suite against HeapCare.Web main.
- **HeapCare.Web PR #86** adds the paired browser journey that searches/selects a synthetic PatientRegistrar patient, loads the published slot, books an appointment and checks the patient in. The follow-on Local #36 run #46046213095 subsequently passed the exact real-Keycloak browser acceptance, including patient search, provider slot selection, booking and check-in. The later Web #88 live-region UI change remains subject to the separate Local #37 replay. Appointment scheduling remains staff-assisted and production activation remains subject to hospital policy validation.

### Phase 5 — authenticated accessibility and quality

- **HeapCare.Web PR #85** merged the six-role real-Keycloak accessible-name and form-label checks. The current Local full-stack run passed the authenticated browser suite against the resulting Web main.
- Automated checks establish a useful regression baseline, not WCAG 2.2 AA conformance. Manual keyboard review, screen-reader testing, contrast, zoom/reflow and qualified accessibility evaluation remain outstanding.

### Current release boundary

Phases 3–5 have stronger merged implementation and CI acceptance evidence, but none is declared production-ready solely from these tests. Product/clinical role approval, privacy/legal review, operational backup/PITR, secrets and key custody, retention/legal hold, hosting/region selection, hospital scheduling policy and a production-like recovery drill remain explicit gates.

---

## Final Phase 3–5 execution record — 2026-11-27

This section supersedes earlier pending notes in this roadmap and records only changes that are merged and independently verified.

### Phase 3 — tenant security, audit and database recovery

- API #122 added PostgreSQL-backed audit CSV tenant isolation: requesting-tenant actor metadata is included while another tenant's actor metadata is excluded.
- API #123 merged as 9a4a517add68c5af08fc914533b387e6883440be. A department-scoped PatientRegistrar can transition an in-scope queue entry but gets a non-disclosing 404 for a peer department. The denied queue entry remains Waiting and both allowed/denied audit attribution is checked.
- API #124 merged as 745085912684d54a2c2f6ca830ade712d76d9e63. Eight simultaneous PostgreSQL-backed appointment check-in requests all return the same queue-entry ID and token; a direct database query confirms exactly one appointment-linked queue row remains. Release build, xUnit analyzers, EF model-drift, full tests and Docker/PostgreSQL checks passed.
- Local #33 merged as bb252b944b84e1d431d0cbd23459059e12167732. Real Keycloak synthetic identities verify ClinicalUser cannot read cashier invoices or record payments and FacilityViewer cannot read invoices.
- Local #34 merged as 704aa3e24682e5c386d4975df32c5c2b33bac709. The full-stack workflow creates a disposable PostgreSQL custom-format logical backup, restores to a separate empty database and compares application-table row counts. This is a logical dump/restore smoke test, not provider-native backup, WAL/PITR, cross-region DR, tenant-level restore or production RPO/RTO evidence.

### Phase 4 — provider, availability, appointment, queue and billing workflows

- Local #35 merged as 44ec59dacdfeb3810785ce4e843032a69dd86f10. Real Keycloak/API acceptance creates a synthetic provider profile and effective department assignment, publishes future availability in Asia/Kolkata, and verifies PatientRegistrar can discover the session and slot. FacilityManager cannot publish availability, ClinicalUser cannot read it and PatientRegistrar cannot create provider staff profiles.
- Web #86 merged as 3a029ad55f053223a9612d6038e30a6cb1e7e125; Web #87 merged as 3b81a70e9113382067c9002421bbdcd528b2a22c. The PatientRegistrar browser workflow searches a synthetic patient using the API's field-based family-name query, selects a future published slot, books the appointment and checks the patient in.
- Local #36 merged as 085448ba36bc05338aafab604c7e9d5787bbf3d4. Full-stack run #46046213095 passed both tooling validation and the full nine-service job. The full-stack job passed real Keycloak persisted-role API acceptance, PostgreSQL logical backup/restore, cleanup and the real Authorization Code + PKCE browser suite (11 tests). Recorded cross-repository SHAs: API 9a4a517add68c5af08fc914533b387e6883440be, Web 3b81a70e9113382067c9002421bbdcd528b2a22c, Observability 1a06382e4eeed3e0070a0def2b89b8d11512dd52.
- API #124 is a test-only follow-up merged after the integrated run. It passed Release build, EF model-drift, full tests and Docker/PostgreSQL verification; no API runtime behavior or authorization grant changed.

### Phase 5 — authenticated accessibility baseline

- Web #80 and #85 merged the six-role real-Keycloak workspace checks. Local #36's full-stack browser run passed all six identities: HospitalAdmin, FacilityManager, FacilityViewer, PatientRegistrar, ClinicalUser and PlatformOperator. The checks cover main landmark/heading, first-stop skip-link focus, visible focus outline, focus transfer, responsive overflow, accessible names and form labels.
- Web #88 merged as 554012ff0cafdab86381addd8c77985fe897dccd. Appointment availability now announces provider-session count and available-slot count through polite status semantics, with component regression tests. Its hosted Web pipeline passed Vitest, production build, Chromium E2E, real-OIDC suite discovery and Docker/CSP smoke.
- **Local #37** merged as `e0364bf081bc57c4a9ba2053d1dd36a30f33fbf4`. Full-stack run [#46047530266](https://github.com/HeapCare/HeapCare.Local/actions/runs/46047530266) passed both Local tooling validation and the full nine-service job against Web `554012ff0cafdab86381addd8c77985fe897dccd` (including Web #88), API `745085912684d54a2c2f6ca830ade712d76d9e63`, and Observability `1a06382e4eeed3e0070a0def2b89b8d11512dd52`. The real Keycloak Authorization Code + PKCE browser suite completed 11 passing tests, including all six role accessibility cases and the PatientRegistrar appointment, booking and check-in path. This verifies the Web #88 availability announcement change in the integrated stack.

### Remaining release gates

- Manual keyboard-only completion across all six role workflows, screen-reader review, desktop/tablet visual checks, contrast, zoom/reflow and qualified WCAG 2.2 AA evaluation remain Not Run. No conformance claim is made.
- Provider credentials and clinical privileges require hospital-side verification; the synthetic provider profile proves scheduling API behavior only.
- Scheduling, cancellation, rescheduling, no-show policy, check-in window, hospital timezone and retention require pilot-hospital approval.
- Production backup/PITR, encryption/key custody, retention/legal hold, restore cadence, production RPO/RTO, hosting/region and tenant-level recovery remain owner-dependent gates.

### Final Local replay after Web #88 — 2026-11-27

Local PR #37 merged as e0364bf081bc57c4a9ba2053d1dd36a30f33fbf4. Full-stack run #46047530266 passed Local validation and the full nine-service Docker/observability workflow. Exact cross-repository heads were API 745085912684d54a2c2f6ca830ade712d76d9e63, Web 554012ff0cafdab86381addd8c77985fe897dccd, and Observability 1a06382e4eeed3e0070a0def2b89b8d11512dd52. The real Keycloak Authorization Code + PKCE suite completed 11 passing tests, including six persisted-role accessibility checks and the PatientRegistrar booking/check-in workflow. This replay includes Web #88's provider-session and available-slot status announcements.

### Clinical note lifecycle — real Keycloak acceptance — 2026-11-27

- **Local PR #38 merged** as `fae841e12223e82ff6e7a8c0dfdf571222f744d6`. Full-stack run [#46050151054](https://github.com/HeapCare/HeapCare.Local/actions/runs/46050151054) passed both tooling validation and the full nine-service Docker/observability job.
- The real Keycloak-issued ClinicalUser workflow registered a separate synthetic patient, created an in-progress queue visit and encounter, saved a draft, signed it with signer attribution, rejected a post-sign overwrite with HTTP 409, closed the encounter, and appended an attributed amendment. It re-read the encounter and proved the original signed assessment stayed unchanged. PatientRegistrar was denied encounter read access.
- The browser manifest exposed the second synthetic encounter. The real-OIDC ClinicalUser workspace reopened it and verified patient identity, read-only signed note fields, absence of the Review and sign action, and visible append-only amendment history. The original empty-note encounter remained separate and continued to prove signing is disabled until the required fields exist.
- The integrated run completed 11 real-OIDC browser tests, including the six persisted-role workspace checks and the clinical, registration, appointment and billing journeys.
- Exact cross-repository heads for run #46050151054: API `745085912684d54a2c2f6ca830ade712d76d9e63`, Web `baab2966baeafd8f07e94bf45ab02f8ef3ecc1e9`, Observability `1a06382e4eeed3e0070a0def2b89b8d11512dd52`.
- All clinical note values in this added test were explicitly synthetic CI placeholders in an ephemeral database. No real patient history, examination findings, diagnosis, treatment or care plan was used; this does not authorize production clinical use.


---

## Appointment cancellation and lifecycle concurrency — 2026-11-28

### Merged API behavior

- **HeapCare.Api PR #125** merged as `92f0765e42feca47508bd46518ac41521123318f`. Check-in, cancellation and rescheduling use the same PostgreSQL transaction-scoped advisory lock keyed by tenant, hospital and appointment. Each request verifies its scope first, then locks and rereads the appointment before making a lifecycle decision.
- A booked appointment can be cancelled by an authorized `appointment.manage` user. Once a queue entry exists, cancellation is rejected with HTTP 409; the appointment remains Booked and the existing queue row is preserved.
- Repeated cancellation, check-in after cancellation and reschedule after cancellation are rejected with HTTP 409. Authorization denials remain 403 and audit as `Denied`; rejected business transitions audit as `Rejected`.
- PostgreSQL integration coverage races cancellation against first check-in and permits only two consistent outcomes: cancellation wins, leaving Cancelled with no queue entry; or check-in wins, leaving Booked with exactly one appointment-linked queue entry. Release build, xUnit analyzers, EF model-drift, integration suite, migration rehearsal and Docker/PostgreSQL checks passed on the exact API PR head.

### Merged Web behavior

- **HeapCare.Web PR #91** merged as `254521e578bff33b4ba0d6198b9601f2f1ed0496`. The appointment-management workspace handles HTTP 409 by returning the UI to the first cancellation-confirmation step, preserving the Booked status and displaying actionable refresh guidance. It does not optimistically mark the appointment Cancelled.
- Its real-Keycloak PatientRegistrar browser journey loads a separate synthetic appointment, confirms cancellation through the workspace, verifies HTTP 200 and the refreshed Cancelled state, and checks the appointment list uses a patient reference rather than exposing the synthetic family name.

### Current integrated acceptance status

- **HeapCare.Local PR #39** merged on 2026-11-28 as `28a2123c5f0dc9b25a8f1ead0c08cb59bb4d51f9` from tested head `855b6f4a19c6bb5cd56e5dfabb539ef35aa0fb65`. Exact-head pull-request run [#46086998413](https://github.com/HeapCare/HeapCare.Local/actions/runs/46086998413) passed both jobs: tooling validation and full nine-service Docker/API telemetry verification. It checked out API main `92f0765e42feca47508bd46518ac41521123318f` and Web main `009055abee9de8791c156c0e8e35dcdfafe865ec`.
- The real-Keycloak browser suite completed **11/11 tests**: all six authenticated role accessibility-baseline cases (HospitalAdmin, FacilityManager, FacilityViewer, PatientRegistrar, ClinicalUser and PlatformOperator), plus the persisted-workspace and Authorization Code + PKCE scenarios. The previous declaration-order defect was fixed in merged **Web #92**; the ARIA IDREF whitespace-tokenization defect was fixed in merged **Web #94**.
- The isolated PostgreSQL logical backup/restore recreated a separate target and verified matching row counts for all **26 application tables**. The run also verified nine-service readiness, CSP, OIDC issuer/audience, API role/tenant checks, OpenTelemetry traces/metrics and the HeapCare Kibana data view. This is CI logical-dump/restore evidence only—not provider-native backup/PITR or a production disaster-recovery drill.
- The appointment cancellation API/UI acceptance is integrated and merged. Manual six-role keyboard completion, real screen-reader testing, contrast, zoom/reflow and qualified WCAG 2.2 AA evaluation remain **Not Run**; automated tests are not a conformance claim.
- The latest merged production cutover inventory is **Docs #185**, merged as `7d0f683c2172926f7a55238c7128431bdedec480`. It preserves the boundary between local compatibility identifiers and unknown production identity/database/domain/backup infrastructure.

### Owner-dependent policy boundary

No-show action, grace period, cancellation reason taxonomy, scheduling role mapping, hospital check-in policy and retention still require pilot-hospital approval. The CI workflow uses synthetic users and appointments only; it does not authorize live clinical use.


## Current HeapCare identity and phase checkpoint — 2026-11-28

This section is the current status and supersedes any earlier 2026-11-27 migration sentences below or above it that describe the GitHub administrative rename as still pending.

### Product and repository identity

- The GitHub organization and all 11 repository slugs now use **HeapCare**. Application namespaces, project/folder paths and shared package identities are migrated in the repositories where implementation exists.
- Current-use architecture, security, engineering, product and operational guides are being aligned to HeapCare paths and naming. Dated audit/QA records retain their original names where they are historical evidence.
- The official support email, production website domain, DNS authority, TLS certificate and live hosting service are not verified. `heapcare.in` remains unpurchased/unconfigured; do not invent a support address or public URL.

### Technical compatibility identifiers — intentionally retained

- The local Keycloak realm/issuer path remains `compilecare`; local Web client `compilecare-web`, API audience `compilecare-api` and client scope `compilecare-api-audience` remain unchanged.
- API request-local authorization claims now use the `heapcare:tenant_id`, `heapcare:tenant_role`, `heapcare:permission` and `heapcare:validated_issuer` namespace. The former `compilecare:*` equivalents are deny-only compatibility inputs: the API strips them at authentication ingress and regression tests prove forged legacy and new claim names cannot grant permissions. This does **not** rename the local Keycloak realm/issuer, OIDC client, audience or scope identifiers listed above; those remain identity contracts requiring a separate continuity plan. The server-created membership identity type is `HeapCareTenantMembership`.
- The API prefers `ConnectionStrings:HeapCare` while retaining `ConnectionStrings:CompileCare` as a compatibility fallback. Local Docker Compose project and persisted Keycloak/PostgreSQL/observability volume names retain their existing identifiers.
- These values are not public product branding. Do not rename them without an approved identity/data mapping, isolated rehearsal, backup/restore proof and reversible cutover. The exact local/CI values and safe migration sequence are documented in [the production cutover inventory](../operations/heapcare-production-cutover-inventory-2026-11-28.md).

### Phase status and remaining owner gates

- **Phase 3:** PostgreSQL/EF persistence, OIDC issuer/subject validation, tenant/organization RBAC, platform tenant lifecycle and membership administration are implemented. Append-only metadata audit, tenant-wide `audit.read` query/CSV export with bounded windows and formula-injection neutralization are implemented; audit persistence failure is fail-closed, and the 503 Problem Details response now uses `about:blank` + `code: audit_unavailable` (API PR #126, merge `a02e4e8`). API Client #6 is merged with optional `ProblemDetails.Code`; API PR #127 is merged with both HeapCare Observability packages at 1.0.2 (`aaeaa40`). Still open: named audit-reader/retention/legal-hold/export-monitoring approvals; production IdP/key custody, external user lifecycle, periodic access reviews, incident response, provider-native backup/PITR, recovery objectives and privacy/legal/security sign-off (tracked in [production readiness gate #197](https://github.com/HeapCare/HeapCare.Doc/issues/197)). `HeapCare.OtpService` remains a scaffold on security hold pending a fresh design/review and provider-owner confirmation that its historical SMTP credential has been revoked/rotated. API PR #131 (`ffc3aa0`) moves server-generated request-local tenant/role/permission/validated-issuer claims to `heapcare:*` and strips legacy claim names; API PR #132 (`f2121a4`) adds a PostgreSQL-backed route-discovery regression guard requiring explicit authorization or explicit anonymous metadata on every `/api/` route. Both passed exact-head CI and Docker integration verification.
- **Phase 4:** the first integrated hospital workflows are implemented across facility hierarchy, patient registration/MRN/duplicate review, staff/provider profile and assignment foundations, appointments/availability, walk-ins/check-in/queue, OPD encounter/clinical note lifecycle and the initial billing/payment/receipt workflow. The initial **IPD ADT and bed-management technical foundation is now implemented**: guarded admission creation, paginated admission list/detail, first-bed assignment and same-hospital transfer are merged behind dedicated permissions. PostgreSQL regressions verify sibling-Hospital/cross-Tenant transfer destination denial (API PR #143), concurrent transfers from one active placement (API PR #144), transfer into an occupied bed leaves both placements unchanged (API PR #145), and metadata-only audit attribution for denied bed assignment/transfer requests (API PR #146). All four passed exact-head API build/test and Docker verification. Local PR #46 passed real-Keycloak negative authorization across all five persisted tenant roles and the platform-only PlatformOperator (all five current IPD routes return 403). This does not approve positive role grants or establish a positive IPD workflow. Admission/bed sequencing, mandatory admission data, six-role permission mapping, cancellation/discharge policy and the Web IPD workspace remain gated. Pharmacy, LIS, RIS/PACS, inventory, workforce/HR, patient portal and payer integrations remain separately deferred behind hospital policy and product decisions. The latest integrated Local #43 run ([#46095789248](https://github.com/HeapCare/HeapCare.Local/actions/runs/46095789248)) re-executed existing synthetic tenant/hospital, patient reception, appointment/check-in, queue, clinical and billing/receipt journeys through real Keycloak identities. Hospital hierarchy approval is tracked in [gate #198](https://github.com/HeapCare/HeapCare.Doc/issues/198); IPD and deferred domain/clinical workflow approval is tracked in [gate #200](https://github.com/HeapCare/HeapCare.Doc/issues/200).
- **Phase 5:** the authenticated workspace and core role workspaces are implemented. Latest exact-head integrated Local PR #43 run [#46095789248](https://github.com/HeapCare/HeapCare.Local/actions/runs/46095789248) passed both jobs and all 11 real-Keycloak browser tests on Local PR merge-ref tree `080197c88f5d9ca5a3f2691cc8fcc504e74e6ac6`, with API `f2121a4062afb06b69d1be4c8b1d7640cca55fbc`, Web `62b3687f1ef77d84986113810740904112064589` and Observability `84bc2ec84de9a1378ac376491d6f6e38dfdcabb6`. All six persisted roles now verify forced-colors keyboard focus on the skip link and on the authenticated Sign out action (`:focus-visible` plus visible outline). Manual keyboard-only workflow completion, native Windows High Contrast, real screen-reader, dialog/focus, contrast, zoom/reflow and qualified WCAG 2.2 AA evaluation remain Not Run (tracked in [manual accessibility acceptance #201](https://github.com/HeapCare/HeapCare.Doc/issues/201)); no WCAG conformance is claimed.
- **Observability dependency alignment — merged 2026-11-28:** HeapCare.Observability and HeapCare.Observability.AspNetCore `1.0.2` were published to GitHub Packages after PR #7 merged as `84bc2ec84de9a1378ac376491d6f6e38dfdcabb6` and publish workflow [#46089332169](https://github.com/HeapCare/HeapCare.Observability/actions/runs/46089332169) passed. The libraries now embed their NuGet README and use stable OpenTelemetry 1.15.x core/hosting/ASP.NET Core/HTTP/runtime/SQL Client packages. `OpenTelemetry.Exporter.Prometheus.AspNetCore` and `OpenTelemetry.Instrumentation.Process` remain pinned prerelease upstream dependencies because the current metrics implementation uses them; NU5104 warnings are documented and visible. API PR #127 merged as `aaeaa40af74ac3b55ec0c5f4dbed52aeb52da9e9`, aligning both central package references and CI ephemeral feeds. Local PR #41 merged as `2bb4fb6cc50f322951a595ddd28cf9934ee18ac5`, aligning the full-stack ephemeral feed. Exact-head integrated Local run [#46089858978](https://github.com/HeapCare/HeapCare.Local/actions/runs/46089858978) passed tooling and the nine-service stack against API `aaeaa40`, Web `19087321764122944af819b402e28c225994ad45`, and Observability `84bc2ec`; it verified API-to-Jaeger Npgsql spans, Collector-fed metrics, Kibana `heapcare-logs-*`, PostgreSQL logical restore of 26 tables and 11/11 real-Keycloak browser tests. This remains CI logical restore evidence, not provider-native PITR/DR. The post-merge Local main run is independently tracked in GitHub.
- The same Local run verified an isolated PostgreSQL logical backup/restore with matching row counts across 26 application tables. It is not provider-native backup/PITR or a production DR drill. Production domain/ingress, signing-key custody, provider-native recovery, clinical governance and pilot approval remain independent release gates.



### Audit fail-closed Problem Details contract — 2026-11-28

API PR #128 merged as `2262653171bc8b527993b78c06999eceb6ddc5ea`. Exact-head PR CI [#46091353956](https://github.com/HeapCare/HeapCare.Api/actions/runs/46091353956) and Docker Integration Verification [#46091353955](https://github.com/HeapCare/HeapCare.Api/actions/runs/46091353955) passed. The PostgreSQL integration suite passed **210/210 tests** and the unit suite passed **12/12**.

The audit-store failure regression now checks the RFC 9457 response's HTTP 503 status, `about:blank` type, canonical title, stable `audit_unavailable` code, exact request path in `instance`, safe generic detail and absence of the synthetic PostgreSQL exception text. It also checks the caller's `X-Correlation-ID` is preserved and the Problem Details distributed `traceId` is independently present; those identifiers are intentionally distinct. The existing scenario still proves patient/hospital registration writes roll back and no success audit is recorded when the audit insert fails.

This is a test-coverage hardening change; no runtime response behavior or authorization policy changed. The post-merge API main CI run is tracked separately at [#46091639424](https://github.com/HeapCare/HeapCare.Api/actions/runs/46091639424).


## Phase 3–5 continuation checkpoint — 2026-11-28

- **Phase 3 — API authorization claim namespace:** HeapCare.Api PR #131 merged as `ffc3aa0087af6acd5c1a54e7a29cf7ec82808b09`. API-generated tenant, role, permission and validated-issuer request claims now use the `heapcare:*` namespace. Legacy `compilecare:*` claim types are removed from inbound authenticated identities and are never treated as grants. Exact-head checks passed for API CI, Docker Integration Verification and PostgreSQL Migration Rehearsal. Keycloak realm/client/audience compatibility identifiers are unchanged; this change is deliberately limited to server-local claim types.
- **Phase 5 — forced-colors focus:** Web PR #96 merged as `7611875b18afb81d90e64c34d454e41c4a3bd2fa`. The design system preserves visible keyboard focus in forced-colors mode, and Playwright emulates forced-colors and checks the focus outline. Native Windows high-contrast verification remains a manual host check.
- **Remaining distinction:** automated browser checks are evidence for specific tested behaviors only. Manual keyboard-only role workflows, screen-reader review, contrast review, zoom/reflow and qualified WCAG 2.2 AA evaluation remain not run. Production identity/key custody, provider-native backup/PITR, approved RPO/RTO and privacy/security/hospital approvals remain independent release gates.


### Integrated Phase 3–5 verification — 2026-11-28 (Local PR #42)

- Local PR #42 merged as `a48723643762a56c2ca5cf3a4fa7f6693d70e72c`; exact-head full-stack run [#46095042936](https://github.com/HeapCare/HeapCare.Local/actions/runs/46095042936) passed both the nine-service/telemetry job and Local tooling/observability validation.
- Tested trees: Local PR merge ref `0fe5c72dd36d23aaa48927ef1335762c2e3b3604` (PR head `bcd7259465cf168e2e1fcf042aaf01349ccb4e5a`); API `f2121a4062afb06b69d1be4c8b1d7640cca55fbc`; Web `9dff5b506eef0de2955c6a27d70f4b30838686b0`; Observability `84bc2ec84de9a1378ac376491d6f6e38dfdcabb6`.
- Real Keycloak issuer/subject and persisted multi-tenant workflows passed; all 11 Authorization Code + PKCE browser tests passed, including six authenticated accessibility-baseline cases for all persisted roles. Existing synthetic workflows covered reception registration/search, appointments/check-in, queue, OPD clinical encounter and billing/receipt. Forced-colors emulation checks visible keyboard focus after sign-in for every role.
- PostgreSQL logical backup/isolated restore passed with matching row counts across 26 application tables. This does not prove provider-native physical backups, PITR or production DR. CI-only secrets and volumes were successfully cleaned up.
- Remaining gates are not marked complete: manual keyboard-only role workflows, native Windows High Contrast, screen reader, contrast, zoom/reflow, qualified WCAG 2.2 AA, production IdP/key custody, provider-native recovery/RPO/RTO, audit retention/legal hold/export governance, privacy/security sign-off and hospital-governance decisions.


### Authenticated action keyboard-focus replay — 2026-11-28

Local PR #43 merged as `a486fdb39792bc82f8e24f7dcf690afa53166f29`; exact-head run [#46095789248](https://github.com/HeapCare/HeapCare.Local/actions/runs/46095789248) passed both jobs. Browser suite completed 11/11 real-Keycloak tests, including the six-role skip-link and authenticated Sign out `:focus-visible` checks. The nine-service stack, real tenant/membership workflows, API-to-Jaeger Npgsql telemetry, Collector/Prometheus/Jaeger/Elasticsearch/Kibana verification, 26-table logical restore row-count comparison and cleanup also passed. Tested trees: Local merge ref `080197c88f5d9ca5a3f2691cc8fcc504e74e6ac6`; API `f2121a4062afb06b69d1be4c8b1d7640cca55fbc`; Web `62b3687f1ef77d84986113810740904112064589`; Observability `84bc2ec84de9a1378ac376491d6f6e38dfdcabb6`. The PostgreSQL check remains CI logical restore only. Manual keyboard workflows, native Windows High Contrast, real screen reader, contrast, zoom/reflow and qualified WCAG 2.2 AA remain open; production identity, provider PITR/RPO/RTO, privacy/security and clinical governance gates are unchanged.


### Phase 4 appointment availability concurrency guard — 2026-11-28

HeapCare.Api PR #133 merged as `0fc0aecd8b80f06e10ce8f797a2b37f3f712c35f`. It adds a PostgreSQL integration regression that sends eight concurrent availability-publication requests for the same tenant/provider/time interval and asserts exactly one creation, seven conflicts and one persisted session. The test exercises the existing provider/time exclusion constraint and service conflict mapping; no scheduling runtime or schema change was made. Exact-head API CI [#46096519916](https://github.com/HeapCare/HeapCare.Api/actions/runs/46096519916) passed **212 integration tests and 12 unit tests**, EF model drift detection and Release build; Docker Integration Verification [#46096519987](https://github.com/HeapCare/HeapCare.Api/actions/runs/46096519987) also passed. This hardens the existing availability publication workflow; it does not close the separate hospital-group, clinical policy or production-readiness gates.


### IPD admission creation — merged 2026-11-28

HeapCare.Api PR #138 merged as `f28a94ce92d6c1fbf8c29cf4fb565e897bd73700`, adding PostgreSQL-backed verification that a denied IPD admission read retains the validated Tenant and OIDC actor in the metadata-only audit event while recording the denied permission outcome. The exact-head build/test workflows and Docker Integration Verification passed.

HeapCare.Api PR #139 merged as `727285013da9448fb032b91ed1bd2b06b74de429`, implementing guarded inpatient admission creation:
- `POST /api/v1/hospitals/{hospitalId}/ipd/admissions` is protected by the dedicated `inpatient.admit` permission.
- The selected patient must already have a HospitalPatientRegistration at the exact requested Hospital inside the current Tenant.
- Only Tenant-wide or Hospital-wide organization scope may create an admission; Department-only scope is denied.
- The service creates an Active admission with a server-generated UTC timestamp and no bed placement. Bed assignment remains a separate operation.
- A second active admission for the same patient/Hospital returns conflict; the existing PostgreSQL filtered unique index remains the final concurrency arbiter.
- Existing tenant roles, PlatformOperator and forged external role/permission claims do not receive admission permission. No role-to-IPD grants were changed.

**Exact-head validation on PR #139:** API Release build and model-drift checks passed; unit tests **12/12** and PostgreSQL integration tests **240/240** passed; PostgreSQL migration rehearsal and Docker Integration Verification passed. Evidence: API CI [#46112235715](https://github.com/HeapCare/HeapCare.Api/actions/runs/46112235715), second API CI [#46112231273](https://github.com/HeapCare/HeapCare.Api/actions/runs/46112231273), migration rehearsal [#46112235635](https://github.com/HeapCare/HeapCare.Api/actions/runs/46112235635), and Docker verification [#46112235716](https://github.com/HeapCare/HeapCare.Api/actions/runs/46112235716).

**Still open:** admission mandatory fields/reason catalogue, hospital approval of admission-before-bed-assignment semantics, the six-role positive/negative IPD permission matrix, cancellation and discharge rules, bed reservation/readiness policy, Local real-Keycloak replay, and the Web IPD workspace. Do not describe the complete IPD workflow as finished from admission creation alone. No billing, MRN, clinical-state, medication, nursing or other clinical behavior was added.


### IPD admission-list read model — merged 2026-11-28

HeapCare.Api PR #140 merged as `6448b05de80bbe5f9bbb0e0452b00cdc5f1550cc`. It adds the permission-gated `GET /api/v1/hospitals/{hospitalId}/ipd/admissions` endpoint with bounded pagination and optional case-insensitive Active/Cancelled/Discharged status filtering.

The Application/Infrastructure service applies the existing Tenant query filter and HospitalId boundary. Tenant-wide/Hospital-wide membership scope can list admissions for the requested Hospital. Department-only scope sees only admissions whose **current** bed placement is in an assigned department; unplaced admissions and admissions currently elsewhere are excluded. Placement history in the response is restricted to the caller's visible departments so older movements elsewhere are not disclosed. No schema/migration or role-grant changes were made.

**Exact-head validation:** API Release build and EF model-drift guard passed; **12/12 unit tests and 244/244 PostgreSQL integration tests** passed; PostgreSQL Migration Rehearsal [#46112870907](https://github.com/HeapCare/HeapCare.Api/actions/runs/46112870907) and Docker Integration Verification [#46112870941](https://github.com/HeapCare/HeapCare.Api/actions/runs/46112870941) passed.

The API now has technical foundations for admission persistence/creation, admission list/detail reads, bed assignment and same-hospital bed transfer. All IPD permissions remain ungranted to existing tenant roles and PlatformOperator. Full user-enabled/end-to-end ADT remains incomplete pending the hospital-approved six-role matrix, required admission data/reason catalogue, admission/bed sequencing policy, cancellation/discharge rules, Local real-Keycloak verification and Web workspace. Do not treat the API service foundations as clinical or operational policy approval.


### IPD hospital-scope and audit hardening — merged 2026-11-28

HeapCare.Api PR #141 merged as `57a8489b77ee72e0e34cca89401f75736785090f`. A PostgreSQL integration regression creates two Hospitals under the same Tenant with separate registrations/admissions and proves that Hospital A scope lists only Hospital A admissions, receives a non-disclosing 404 when requesting Hospital B's list, and Hospital B scope sees only its own admission. Exact-head CI passed Release build, EF model-drift guard, **12/12 unit tests and 245/245 PostgreSQL integration tests**, PostgreSQL migration rehearsal and Docker Integration Verification.

HeapCare.Api PR #142 merged as `22a6dae84b5a8a7c6b549975a321507d308f8b0a`. Successful IPD admission creation now writes the newly created Admission ID into request-local audit context; the middleware prefers that identifier over the parent Hospital ID in the create route. Existing route-derived resource IDs remain the fallback. The regression verifies the explicit created-resource ID takes precedence. Exact-head CI passed Release build, EF model-drift guard, **12/12 unit tests and 246/246 PostgreSQL integration tests**, PostgreSQL migration rehearsal and Docker Integration Verification.

These increments strengthen the existing Tenant-versus-Hospital boundary and metadata-only audit attribution. They introduce no schema, migration, role-grant or clinical workflow change. Local PR #46 is merged and its exact-head nine-service real-Keycloak run [#46114200037](https://github.com/HeapCare/HeapCare.Local/actions/runs/46114200037) passed the negative authorization matrix: all five persisted tenant roles and the platform-only PlatformOperator received HTTP 403 for all five current IPD routes. This closes only the negative-access integration gate; it does not approve role grants or prove a positive admission-to-discharge journey.

### IPD transfer, occupied-bed and denied-operation audit verification — merged 2026-11-28

- **API PR #143** (32e7b56b7969d0b80b8194b3973949ff4865259a): PostgreSQL service tests reject transfer destinations in a sibling Hospital under the same Tenant and in a different Tenant with non-disclosing 404 responses, leaving the original placement unchanged.
- **API PR #144** (c718f0ccd8816158d6740faef5a27656da89dc0a): a deterministic PostgreSQL race test proves one of two simultaneous transfers from the same active placement succeeds, the competing request receives 409, and exactly one active destination placement remains.
- **API PR #145** (b7663ee29f231c55f4775b7a473af71eb05bc80b): transfer to a bed occupied by another active admission returns 409; both existing active placements remain unchanged.
- **API PR #146** (39c40bb7ccdb899b23684b18595293c331a79078): denied bed assignment and transfer requests are covered through HTTP/PostgreSQL integration tests. Audit records include validated Tenant/OIDC actor, endpoint/action, admission resource ID, route, denied outcome, HTTP status and correlation ID; denied requests create no placement.
- Exact-head PR #145 and #146 validation each passed 12/12 unit tests and 249/249 PostgreSQL integration tests, Release build, EF model-drift guard, PostgreSQL migration rehearsal and Docker Integration Verification. PR #146 also passed the additional API build/test workflow.
- Existing Local PR #46 verified all five persisted tenant roles and platform-only PlatformOperator are denied the five current IPD routes with real Keycloak tokens. This remains a negative-access proof only; no IPD permission is granted.

**Still gated:** positive six-role IPD authorization, hospital decision on admission-before-bed sequencing and mandatory admission data, cancellation/discharge eligibility/reasons, bed reservation/readiness policy, positive real-Keycloak ADT workflow and the Web IPD workspace. Do not claim full IPD workflow or clinical approval.

### IPD detail isolation and successful bed-operation audit verification — merged 2026-11-28

- **API PR #147** (`031dfab96a6e7f98dc34fd613b36dade4809c004`): PostgreSQL service regressions deny admission-detail reads for a Hospital-scoped member of a sibling Hospital and for a Tenant-wide member from another Tenant. Both return non-disclosing 404 responses without an admission DTO. Exact-head Release build/test, Docker Integration Verification and additional build/test passed.
- **API PR #149** (`8c9da9b48aac855f51f26a0be9f9c419b243185b`): after a same-Hospital transfer from Department A to Department B, a Department B-only reader sees the current placement but not the previous Department A movement history. Exact-head checks passed Release build, EF model-drift guard, 12/12 unit tests, 253/253 PostgreSQL integration tests, Docker Integration Verification and additional build/test.
- **API PR #148** (`74c0a607e3fcb1f1cd4e983822bb690b6684b95c`): successful first-bed assignment and bed transfer are exercised through the real `AuditEventMiddleware` transaction using synthetic, explicitly scoped test actor context. Both business placement changes and metadata-only audit events commit together. Assertions cover validated Tenant/OIDC actor, endpoint action, IPD resource type, admission ID, route, outcome/status and correlation ID. Exact-head checks passed Release build, EF model-drift guard, 12/12 unit tests, 252/252 PostgreSQL integration tests, Docker Integration Verification and additional build/test.
- **Local PR #48** (`668cee45f46d9313d968ad64b0385f43ae3c797d`): full nine-service run [#46116355864](https://github.com/HeapCare/HeapCare.Local/actions/runs/46116355864) passed with API `39c40bb7ccdb899b23684b18595293c331a79078`, Web `afb61cfbd77aab837ed65dcb30043519c27a5684` and Observability `84bc2ec84de9a1378ac376491d6f6e38dfdcabb6`. It verifies the IPD schema, all five current IPD route denials for the five tenant roles plus platform-only PlatformOperator, OTel traces/Npgsql spans/Collector metrics, logical backup/restore of 28 PostgreSQL tables with matching row counts, 11/11 Chromium OIDC tests, 6/6 Firefox and 6/6 Playwright WebKit authenticated accessibility tests, and stack/secret cleanup.

**Scope boundary:** PRs #147–#149 are test-only and make no runtime, schema, migration, policy or permission changes. The Local full-stack run checked out API main at PR #146 (`39c40bb7`); subsequent API PRs #147–#149 add tests only. The Local evidence proves negative authorization and Linux CI/browser coverage, not positive IPD workflow, manual accessibility certification, production recovery or clinical approval.

**Still gated:** hospital-approved six-role positive permissions and scopes; admission-before-bed sequencing and mandatory fields/reason catalogue; cancellation/discharge eligibility and reasons; bed reservation/readiness policy; successful real-Keycloak HTTP workflow and Web IPD workspace; manual Windows/macOS/Ubuntu assistive-technology accessibility acceptance. All six IPD permission identifiers remain ungranted.

### IPD audit-failure rollback verification — merged 2026-11-28

HeapCare.Api PR #150 merged as `4b07eb11c5063288751e1e8a507ce9dc31aafda3`. PostgreSQL failure-injection integration tests exercise the real metadata-only `AuditEventMiddleware` + InpatientService transaction:
- A successful first-bed assignment followed by an audit-event insert failure returns 503 and leaves no placement or audit event.
- A successful same-Hospital transfer followed by an audit-event insert failure returns 503 and rolls back both the previous-placement closure and new-placement creation, preserving the original active placement/history.
- The synthetic PostgreSQL trigger/function are removed in `finally`.

Exact-head checks passed Release build, EF model-drift guard, 12/12 unit tests, 257/257 PostgreSQL integration tests, Docker Integration Verification and the additional build/test workflow. The only database error logs are the two expected synthetic failure injections.

This verifies IPD bed-operation audit transaction failure handling, complementing API PR #148 success-path audit verification and #138/#146 denied-path audit verification. No runtime, schema, migration, role-grant or hospital-policy change was made. All six IPD permission identifiers remain ungranted.

## Tenant-persisted RBAC acceptance and role-workspace browser follow-up — 2026-11-28

### Phase 3 — persisted roles and permission assignments completed

The canonical permission catalogue, persisted role administration, Web management workspace and Local real-Keycloak acceptance are merged:

- HeapCare.Api PR #155 — canonical permission catalogue and exact current system-role grants; merge commit `062a16bf76ad8973a60674eb1b8b19403d61347f`.
- HeapCare.Api PR #157 — tenant-persisted system/custom roles and permission rows, compatibility backfill, fail-closed request-time permission resolution, delegation checks, immutable system roles, append-only before/after audit history, active-role membership validation and membership-reactivation revalidation; merge commit `73f4737ff5c4c36ad29616e306c1fc037a9c40fa`.
- HeapCare.Web PR #102 — role/permission administration workspace with delegated permission selection and reasoned role lifecycle actions; merge commit `77c72a90548f88579721bbcd967ac541e1516dcf`.
- HeapCare.Doc PR #225 — permission matrix and authorization-policy inventory reconciled with the merged implementation; merge commit `e5fd64f4cf2367ec0543ce9c479e01635ccf87ea`.
- HeapCare.Local PR #49 — real-Keycloak custom-role acceptance; merge commit `d0228d78eb735599323fce7770e71de9e26a297d`.

**Exact Local validation:** workflow run [#46158083003](https://github.com/HeapCare/HeapCare.Local/actions/runs/46158083003) completed successfully on PR #49 head `6d1840ac40d7ad507545f480330572fca94d1173`. Both the full nine-service stack job and the tooling/observability validation job passed. It verified:
- Real Keycloak-issued tenant identities, baseline roles, persisted custom-role create and membership assignment, exact effective permissions, permission edits taking effect on the next request, and inactive-role membership reactivation being denied until the role is restored.
- A delegated custom membership administrator cannot create a role with permissions outside their grant set or use a gated IPD permission.
- The protected `membership.role.assign.system` permission and all six IPD permission identifiers remain excluded from the assignable catalogue.
- PostgreSQL logical backup and isolated restore; real Authorization Code + PKCE browser acceptance on Chromium; authenticated six-role accessibility baselines on Firefox and WebKit; stack and CI-secret cleanup.

API issue #156, “Implement persisted tenant roles and permission assignments,” was closed as completed after this evidence was merged. No production identity integration or provider deployment was part of the acceptance.

### Phase 5 — actual browser flow for role management

HeapCare.Web PR #103 added a real-Keycloak Playwright scenario using the synthetic Tenant A HospitalAdmin to open the Roles & permissions workspace, create a custom role with an audit reason, add a delegated facility permission, and deactivate/reactivate the unassigned role with reasons. It supplements the existing component tests and Local API acceptance with an authenticated UI journey.

- PR: [HeapCare.Web #103](https://github.com/HeapCare/HeapCare.Web/pull/103), merged as `3134b8bf9bb65a0d8dbeafa521c8ec10220038e5`.
- Exact-head Web CI run [#46158890530](https://github.com/HeapCare/HeapCare.Web/actions/runs/46158890530) passed build/test and the Firefox/WebKit accessibility jobs.
- The first Local browser replay found a strict-mode ambiguity because both the shell `h1` and workspace `h2` use the same accessible heading text; the authenticated route rendered correctly, and the failure occurred before create/edit/lifecycle actions. Web PR #104 scoped the assertion to the role component region and merged as `32e00251cfaef8b06d1e2d18c5b74adf8e3bcac4`; exact-head Web CI run [#46159723577](https://github.com/HeapCare/HeapCare.Web/actions/runs/46159723577) passed all three jobs.
- HeapCare.Local PR #50 was refreshed to retest current Web main. Local run [#46159955402](https://github.com/HeapCare/HeapCare.Local/actions/runs/46159955402) passed the full nine-service stack, persisted-role API acceptance, PostgreSQL backup/restore and tooling, but the Chromium browser suite found a second duplicate-text locator before role actions: the seeded custom role name appears in more than one node. Web PR #105 scopes that assertion to the matching role card and is open pending Web CI. A further exact-head Local replay is required after #105 merges; the role-management browser lifecycle remains unaccepted.

### Unchanged gates

- IPD remains policy-gated by HeapCare.Doc #200. All six IPD permission identifiers remain non-assignable and ungranted; no positive IPD workflow or discharge/cancellation policy is inferred.
- Branch, site and clinic modeling remains pending product/hospital operational agreement in HeapCare.Doc #198. Do not create a FacilitySite schema or additional inherited scope until approved.
- Native assistive-technology testing and qualified WCAG 2.2 AA review remain open in HeapCare.Doc #201. Chromium/Firefox/Playwright WebKit are not substitutes for NVDA, JAWS, VoiceOver or the native OS combinations in that issue.
- Production identity, secret/key custody, provider-native PITR/DR and deployment decisions remain deferred under HeapCare.Doc #197. This execution remains local-first.
