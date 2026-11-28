# HeapCare Documentation

> **Brand migration status:** The GitHub organization and all 11 repository slugs now use HeapCare. Source namespaces, project paths and package IDs have been migrated in the repositories where implementation exists. Legacy database identifiers, EF migration history, Keycloak realm/client/audience IDs and persistent Docker volume names remain intentionally unchanged pending separately tested compatibility migrations.

HeapCare is an India-focused hospital information and management platform for hospitals, clinics, and healthcare provider organizations. This repository is the product, architecture, workflow, and engineering documentation source of truth.

> **Status:** Foundation documentation — draft v1. Product scope and several architecture decisions remain proposals until reviewed and approved with hospital stakeholders and technical owners.

## Start here
- [Product Blueprint v1](product/blueprint-v1.md) — product intent, user roles, patient journey, capability map, organization/tenant model, MVP and execution sequence.
- [Product vision and scope](product/vision-and-scope.md)
- [MVP scope and non-goals](product/mvp-scope.md)
- [Glossary](product/glossary.md)
- [Hospital module catalogue](product/module-catalogue.md)
- [Delivery roadmap and current implementation status](product/roadmap.md)
- [ADR-017: HeapCare brand migration](architecture/decisions/ADR-017-heapcare-brand-migration.md) — staged identity migration, technical contracts, dependencies and verification gates.
- [Production identity and infrastructure cutover inventory (2026-11-28)](operations/heapcare-production-cutover-inventory-2026-11-28.md) — repository-backed evidence, preserved local compatibility identifiers, unknown production settings and gated cutover sequence.
- [HeapCare migration report (2026-11-27)](architecture/migrations/heapcare-migration-report-2026-11-27.md) — merged repository work, CI evidence, compatibility decisions and remaining closure checklist.
- [HeapCare post-migration full-platform QA report (2026-11-27)](qa/post-migration-full-platform-qa-2026-11-27.md) — full nine-service live startup, 11-repo builds/tests, 27/27 E2E, 24/24 security, observability and defect-remediation evidence.
- [Full-platform E2E validation — 2026-11-26](qa/e2e-validation-2026-11-26.md)
- [Requirements lifecycle](product/requirements-lifecycle.md)
- [Patient registration workflow](product/workflows/patient-registration.md)
- [Staff and provider management workflow](product/workflows/staff-and-provider-management.md)
- [Appointments, walk-ins, check-in and queue workflow](product/workflows/appointments-and-queue.md)
- [Reception and clinical workspace UX foundation](product/workflows/reception-and-clinical-workspace-ux.md)
- [Enterprise experience uplift](product/design/enterprise-experience-uplift.md) — visual quality direction, design-system foundations and the Phase 5 refinement sequence.
- [OPD billing foundation and deferred scope](product/workflows/opd-billing-foundation.md)
- [IPD admission, transfer, discharge and bed-management workflow](product/workflows/inpatient-admission-transfer-discharge-and-bed-management.md) — initial ADT foundation and remaining hospital-policy gates.
- [IPD role and workflow approval worksheet](product/decisions/ipd-role-and-workflow-approval-worksheet.md) — hospital sign-off template; all IPD role grants remain closed pending approval.
- [ADR-018: IPD ADT and bed-management boundaries](architecture/decisions/ADR-018-inpatient-ADT-bed-management-boundaries.md) — accepted initial scope; detailed admission rules and role permissions remain open.
- [ADR-019: Facility-site model and scope inheritance](architecture/decisions/ADR-019-facility-site-model-and-scope-inheritance.md) — proposed design only; not accepted for implementation.
- [ADR-020: User identity, tenant membership and scoped role assignments](architecture/decisions/ADR-020-user-identity-membership-and-scoped-role-assignments.md) — proposed target architecture; preserve existing tenant isolation and migrate memberships without privilege expansion.
- [User-management redesign execution plan](engineering/user-management-redesign-plan.md) — milestones, repository responsibilities, migration sequence, blockers and low-priority platform portfolio visibility.
- [User-management compatibility inventory](engineering/user-management-compatibility-inventory.md) — current API/Web identity, membership, role, scope and authorization dependencies to migrate safely.
- [ADR-014: Appointment scheduling and queue boundaries](architecture/decisions/ADR-014-appointment-scheduling-and-queue-boundaries.md)
- [ADR-013: Staff and provider profile boundaries](architecture/decisions/ADR-013-staff-and-provider-profile-boundaries.md)
- [ADR-012: Verified patient contact data](architecture/decisions/ADR-012-verified-patient-contact-data-and-matching.md)
- [Non-functional requirements](product/non-functional-requirements.md)
- [Architecture overview](architecture/overview.md)
- [API conventions and error contracts](architecture/api-conventions.md)
- [Tenant model](architecture/multi-tenancy.md)
- [ADR-006: Tenant, organization and patient data boundaries](architecture/decisions/ADR-006-tenant-organization-patient-boundaries.md)
- [ADR-007: Reuse HeapCare shared observability](architecture/decisions/ADR-007-shared-observability-reuse.md)
- [Architecture Decision Records](architecture/decisions/)
- [Security and privacy principles](security/security-and-privacy.md)
- [Integration testing strategy](engineering/integration-testing-strategy.md)
- [Engineering standards](engineering/standards.md)
- [Definition of Done](engineering/definition-of-done.md)
- [Existing repository assessment](engineering/repository-assessment.md)
- [Local development setup and troubleshooting](engineering/local-development-setup.md) — Docker Compose, Keycloak/OIDC, private NuGet restore, observed issues, fixes and remaining local acceptance checks.

## Delivery status

1. **Phase 1 — Backend foundation: completed.** HeapCare.Api provides the ASP.NET Core/.NET 10 API foundation, tenant context, middleware, health checks and versioned endpoint structure. HeapCare.Observability provides the shared logging, tracing, metrics and correlation foundation.
2. **Phase 2 — Developer platform: implemented.** HeapCare.ApiClient, HeapCare.Database and HeapCare.Caching provide typed API access, PostgreSQL/EF migration and operations ownership, and tenant-aware caching. Package-consumer verification, provider-native backup/PITR and production operational hardening remain release tasks.
3. **Phase 3 — Database, identity and security: core milestones implemented; production hardening remains open.** PostgreSQL/EF Core 10, OIDC/JWT issuer-subject identity, persisted tenant memberships, five tenant roles, organization-scope authorization, separate PlatformOperator tenant lifecycle, initial administrator provisioning, membership administration, append-only request audit and controlled audit query/export are implemented. Remaining work includes production IdP and signing-key custody/rotation, external IdP user lifecycle, periodic access reviews, retention/deletion policy, provider-native encrypted backup/PITR, incident response, penetration testing, and privacy/legal and operational approval. The HeapCare.OtpService repository is still a scaffold; OTP delivery/security must not be treated as production-ready.
4. **Phase 4 — Core healthcare domain: the first integrated hospital workflows are implemented.** Hospital hierarchy, patient master/registration, hospital-local MRN and duplicate review, staff/provider profiles and assignments, availability/appointment booking, walk-ins/check-in/queue, OPD encounter and clinical-note lifecycle, and the initial billing/payment/receipt workflow are implemented in the API and authenticated workspace. **IPD ADT/bed-management is in progress:** admission and bed-placement persistence, admission creation, admission read and paginated admission list, bed assignment and same-hospital transfer foundations are merged behind dedicated permissions that remain ungranted to all existing roles. The complete user-enabled/end-to-end ADT workflow, cancellation/discharge, approved six-role permission matrix, Local real-Keycloak verification and Web workspace remain incomplete. MAR, nursing, clinical orders, diagnostics, pharmacy/inventory, broader workforce/HR, patient portal, and insurance/TPA/gateway/tax integrations remain outside the initial IPD scope or deferred pending separate hospital policy, privacy and product decisions.
5. **Phase 5 — Authenticated user experience: implemented baseline with manual accessibility acceptance still open.** HeapCare.Web includes the authenticated workspace shell and Reception, patient registration/MRN, appointments, queue, clinical, billing, staff/provider, tenant-membership and PlatformOperator control-plane workspaces. Exact-head real-Keycloak full-stack CI run [#46086998413](https://github.com/HeapCare/HeapCare.Local/actions/runs/46086998413) passed 11 browser tests, including all six authenticated role accessibility-baseline cases, against merged API/Web heads recorded in the roadmap. Manual keyboard completion, real screen-reader testing, focus/dialog review, contrast, zoom/reflow and qualified WCAG 2.2 AA evaluation remain Not Run; automated checks are not a conformance claim.

A merged repository or passing build does not imply that every hospital capability is delivered or that the platform is ready for production. See the [delivery roadmap](product/roadmap.md) for the detailed evidence and remaining release gates.

## Documentation governance

Important decisions must be recorded as ADRs. Documents distinguish implemented facts, accepted decisions, proposals, assumptions and open questions. Update affected requirements, workflows, API contracts and tests when scope or behaviour changes.

HeapCare is exclusively a hospital and healthcare systems product. Unrelated products and repositories are outside this documentation and delivery scope.

## Contributing

Use short-lived branches, focused commits, reviewable pull requests, automated tests and documentation updates. Follow [Engineering Standards](engineering/standards.md) and [Definition of Done](engineering/definition-of-done.md).

## Operations and security

- [Backup, Restore and Disaster-Recovery Operational Baseline](operations/backup-restore-disaster-recovery.md)
- [Production Ingress and Network-Security Acceptance Checklist](operations/production-ingress-and-network-checklist.md)
- [Security and Privacy](security/security-and-privacy.md)
- [Application Threat Model](security/threat-model.md)
