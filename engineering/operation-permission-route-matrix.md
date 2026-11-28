# Current API Operation-to-Permission Route Matrix

**Status:** Source-verified route inventory, 2026-11-28. This records current endpoint policy metadata; it is not a decision to change role grants.  
**Related:** [ADR-020](../architecture/decisions/ADR-020-user-identity-membership-and-scoped-role-assignments.md), [assignment-aware authorization contract](assignment-aware-authorization-contract.md), [user-management compatibility inventory](user-management-compatibility-inventory.md), HeapCare.Api #159.

## 1. Purpose and interpretation

This document maps current HeapCare.Api route groups to their configured authorization policy and canonical persisted permission code. It supports M0/M2 design and regression planning.

A route policy is a coarse permission gate. The current service layer also checks tenant/hospital/department organization scope using the single-role request context. Under the target multi-role model, both checks must be assignment-aware: the required permission and the resource scope must be granted by the same active MembershipRoleAssignment.

The matrix below describes existing route metadata on HeapCare.Api main; it does not claim that the service's current flat scope context is already assignment-aware.

## 2. Facility administration

Source: HeapCare.Api/src/HeapCare.Api/Endpoints/FacilityEndpoints.cs

| HTTP | Route | Endpoint | Policy | Canonical permission |
|---|---|---|---|---|
| GET | /api/v1/hospitals | ListHospitals | FacilityRead | facility.read |
| GET | /api/v1/hospitals/{id} | GetHospital | FacilityRead | facility.read |
| POST | /api/v1/hospitals | CreateHospital | FacilityWrite | facility.write |
| PUT | /api/v1/hospitals/{id} | UpdateHospital | FacilityWrite | facility.write |
| DELETE | /api/v1/hospitals/{id} | DeleteHospital | FacilityDelete | facility.delete |

Facility descendants (departments, wards, rooms and beds) have their own endpoint contracts and use service-level ownership/scope checks; inventory every descendant mutation/read when migrating those service boundaries.

## 3. Patient registration, search and duplicate review

Source: HeapCare.Api/src/HeapCare.Api/Endpoints/PatientEndpoints.cs

| HTTP | Route | Endpoint | Policy | Canonical permission |
|---|---|---|---|---|
| POST | /api/v1/hospitals/{hospitalId}/patients | RegisterPatient | PatientRegister | patient.register |
| POST | /api/v1/hospitals/{hospitalId}/patients/duplicate-candidates | FindPatientDuplicateCandidates | PatientDuplicateReview | patient.duplicate.review |
| GET | /api/v1/hospitals/{hospitalId}/patients | SearchHospitalPatients | PatientRead | patient.read |
| GET | /api/v1/hospitals/{hospitalId}/patients/{patientId} | GetHospitalPatient | PatientIdentityRead | patient.identity.read |
| GET | /api/v1/hospitals/{hospitalId}/patients/departments/{departmentId}/patients/{patientId} | GetDepartmentPatient | PatientIdentityRead | patient.identity.read |
| GET | /api/v1/hospitals/{hospitalId}/patients/departments/{departmentId}/patients | SearchDepartmentPatients | PatientIdentityRead | patient.identity.read |

Patient search and detail are privacy boundaries. The target scope resolver must pair patient.read / patient.identity.read / patient.register / patient.duplicate.review with the same assignment's Hospital/Department scope for the registration and patient-resource relationship.

## 4. Patient identifiers and MRN

Source: HeapCare.Api/src/HeapCare.Api/Endpoints/PatientIdentifierEndpoints.cs

| HTTP | Route | Endpoint | Policy | Canonical permission |
|---|---|---|---|---|
| POST | /api/v1/hospitals/{hospitalId}/patient-identifier-policy | CreateHospitalPatientIdentifierPolicy | PatientIdentifierPolicyManage | patient.identifier.policy.manage |
| GET | /api/v1/hospitals/{hospitalId}/patient-identifier-policy | GetHospitalPatientIdentifierPolicy | PatientIdentifierPolicyRead | patient.identifier.policy.read |
| POST | /api/v1/hospitals/{hospitalId}/patients/{patientId}/identifiers/mrn | AssignHospitalMedicalRecordNumber | PatientIdentifierAssign | patient.identifier.assign |
| GET | /api/v1/hospitals/{hospitalId}/patients/{patientId}/identifiers | GetHospitalPatientIdentifiers | PatientIdentifierRead | patient.identifier.read |
| GET | /api/v1/hospitals/{hospitalId}/patients/by-mrn/{mrn} | FindHospitalPatientByMrn | PatientIdentifierRead | patient.identifier.read |

MRN policy and identifier access must remain hospital-scoped. A patient.identifier permission on one assignment cannot borrow another assignment's hospital scope.

## 5. Staff and provider profiles

Source: HeapCare.Api/src/HeapCare.Api/Endpoints/StaffEndpoints.cs

| HTTP | Route | Endpoint | Policy | Canonical permission |
|---|---|---|---|---|
| POST | /api/v1/staff | CreateStaffProfile | StaffManage | staff.manage |
| GET | /api/v1/staff | ListStaffProfiles | StaffRead | staff.read |
| GET | /api/v1/staff/{staffId} | GetStaffProfile | StaffRead | staff.read |
| PUT | /api/v1/staff/{staffId} | UpdateStaffProfile | StaffManage | staff.manage |
| POST | /api/v1/staff/{staffId}/deactivate | DeactivateStaffProfile | StaffManage | staff.manage |
| POST | /api/v1/staff/{staffId}/reactivate | ReactivateStaffProfile | StaffManage | staff.manage |
| POST | /api/v1/staff/{staffId}/assignments | CreateStaffAssignment | StaffManage | staff.manage |
| POST | /api/v1/staff/{staffId}/assignments/{assignmentId}/end | EndStaffAssignment | StaffManage | staff.manage |

Staff profiles and hospital/department assignments are distinct from login membership. Staff.read/manage must be paired with the assignment scope covering the relevant staff profile or the target assignment's Hospital/Department.

## 6. Availability, appointments and queues

Source: HeapCare.Api/src/HeapCare.Api/Endpoints/AppointmentEndpoints.cs

| HTTP | Route | Endpoint | Policy | Canonical permission |
|---|---|---|---|---|
| POST | /api/v1/hospitals/{hospitalId}/availability-sessions | CreateProviderAvailabilitySession | AvailabilityManage | availability.manage |
| GET | /api/v1/hospitals/{hospitalId}/availability-sessions | ListProviderAvailabilitySessions | AvailabilityRead | availability.read |
| GET | /api/v1/hospitals/{hospitalId}/availability-sessions/{sessionId}/slots | GetProviderAvailabilitySlots | AvailabilityRead | availability.read |
| POST | /api/v1/hospitals/{hospitalId}/appointments | BookAppointment | AppointmentManage | appointment.manage |
| GET | /api/v1/hospitals/{hospitalId}/appointments | ListAppointments | AppointmentRead | appointment.read |
| GET | /api/v1/hospitals/{hospitalId}/appointments/{appointmentId} | GetAppointment | AppointmentRead | appointment.read |
| POST | /api/v1/hospitals/{hospitalId}/appointments/{appointmentId}/cancel | CancelAppointment | AppointmentManage | appointment.manage |
| POST | /api/v1/hospitals/{hospitalId}/appointments/{appointmentId}/reschedule | RescheduleAppointment | AppointmentManage | appointment.manage |
| POST | /api/v1/hospitals/{hospitalId}/appointments/{appointmentId}/check-in | CheckInAppointment | QueueManage | queue.manage |
| POST | /api/v1/hospitals/{hospitalId}/queue/walk-ins | RegisterQueueWalkIn | QueueManage | queue.manage |
| GET | /api/v1/hospitals/{hospitalId}/queue | ListDepartmentQueue | QueueRead | queue.read |
| POST | /api/v1/hospitals/{hospitalId}/queue/{queueEntryId}/{action} | TransitionQueueEntry | QueueManage | queue.manage |

Queue and appointment operations must pair the listed permission with the same assignment's Hospital/Department scope, including the target appointment/session/queue entry ownership.

## 7. OPD encounter and clinical documentation

Source: HeapCare.Api/src/HeapCare.Api/Endpoints/OpdEncounterEndpoints.cs

| HTTP | Route | Endpoint | Policy | Canonical permission |
|---|---|---|---|---|
| POST | /api/v1/hospitals/{hospitalId}/opd/queue/{queueEntryId}/encounter | CreateOpdEncounterFromQueue | EncounterManage | encounter.manage |
| GET | /api/v1/hospitals/{hospitalId}/opd/queue/{queueEntryId}/encounter | GetOpdEncounterByQueueEntry | EncounterRead | encounter.read |
| GET | /api/v1/hospitals/{hospitalId}/opd/encounters/{encounterId} | GetOpdEncounter | EncounterRead | encounter.read |
| PUT | /api/v1/hospitals/{hospitalId}/opd/encounters/{encounterId}/note | UpdateOpdEncounterNote | EncounterManage | encounter.manage |
| POST | /api/v1/hospitals/{hospitalId}/opd/encounters/{encounterId}/sign | SignOpdEncounter | EncounterSign | encounter.sign |
| POST | /api/v1/hospitals/{hospitalId}/opd/encounters/{encounterId}/amendments | AddOpdEncounterAmendment | EncounterAmend | encounter.amend |
| POST | /api/v1/hospitals/{hospitalId}/opd/encounters/{encounterId}/close | CloseOpdEncounter | EncounterManage | encounter.manage |
| POST | /api/v1/hospitals/{hospitalId}/opd/encounters/{encounterId}/cancel | CancelOpdEncounter | EncounterManage | encounter.manage |

Clinical permissions remain distinct from general tenant administration and staff designation. Clinical resource access must pair encounter permissions with the same assignment's scope.

## 8. Billing and collections

Source: HeapCare.Api/src/HeapCare.Api/Endpoints/BillingEndpoints.cs

| HTTP | Route | Endpoint | Policy | Canonical permission |
|---|---|---|---|---|
| POST | /api/v1/hospitals/{hospitalId}/billing/service-items | CreateBillingServiceItem | BillingCatalogueManage | billing.catalogue.manage |
| PUT | /api/v1/hospitals/{hospitalId}/billing/service-items/{serviceItemId} | UpdateBillingServiceItem | BillingCatalogueManage | billing.catalogue.manage |
| GET | /api/v1/hospitals/{hospitalId}/billing/service-items | ListBillingServiceItems | BillingInvoiceRead | invoice.read |
| POST | /api/v1/hospitals/{hospitalId}/billing/invoices | CreateBillingInvoice | BillingInvoiceManage | invoice.manage |
| GET | /api/v1/hospitals/{hospitalId}/billing/visits/{queueEntryId}/invoice | GetBillingInvoiceForVisit | BillingInvoiceRead | invoice.read |
| GET | /api/v1/hospitals/{hospitalId}/billing/invoices/{invoiceId} | GetBillingInvoice | BillingInvoiceRead | invoice.read |
| GET | /api/v1/hospitals/{hospitalId}/billing/invoices/{invoiceId}/payments | ListBillingInvoicePayments | BillingInvoiceRead | invoice.read |
| POST | /api/v1/hospitals/{hospitalId}/billing/invoices/{invoiceId}/payments | RecordBillingPayment | BillingPaymentManage | payment.manage |
| GET | /api/v1/hospitals/{hospitalId}/billing/payments/{paymentId}/receipt | GetBillingReceipt | BillingInvoiceRead | invoice.read |
| GET | /api/v1/hospitals/{hospitalId}/billing/reports/daily-collection | GetDailyBillingCollection | BillingReportRead | billing.report.read |

The existing service-items list route currently uses BillingInvoiceRead / invoice.read. Keep this mapping as source-verified current behavior until a separate permission-contract decision changes it. Do not silently substitute billing.catalogue.read.

## 9. IPD ADT and bed management

Source: HeapCare.Api/src/HeapCare.Api/Endpoints/InpatientEndpoints.cs

| HTTP | Route | Endpoint | Policy | Canonical permission |
|---|---|---|---|---|
| POST | /api/v1/hospitals/{hospitalId}/ipd/admissions | CreateInpatientAdmission | InpatientAdmit | inpatient.admit |
| GET | /api/v1/hospitals/{hospitalId}/ipd/admissions | ListInpatientAdmissions | InpatientRead | inpatient.read |
| GET | /api/v1/hospitals/{hospitalId}/ipd/admissions/{admissionId} | GetInpatientAdmission | InpatientRead | inpatient.read |
| POST | /api/v1/hospitals/{hospitalId}/ipd/admissions/{admissionId}/bed-placement | AssignInpatientBed | InpatientBedAssign | inpatient.bed.assign |
| POST | /api/v1/hospitals/{hospitalId}/ipd/admissions/{admissionId}/bed-transfer | TransferInpatientBed | InpatientBedTransfer | inpatient.bed.transfer |

The six IPD permissions (including future cancellation/discharge) remain non-assignable under the current policy gate. M2 must not enable them or infer role grants.

## 10. Audit query and export

Source: HeapCare.Api/src/HeapCare.Api/Endpoints/AuditEndpoints.cs

| HTTP | Route | Endpoint | Policy | Canonical permission |
|---|---|---|---|---|
| GET | /api/v1/audit/events | ListAuditEvents | AuditRead | audit.read |
| GET | /api/v1/audit/events/export | ExportAuditEvents | AuditRead | audit.read |

Audit reads are privileged. The query boundary must be computed from assignments that grant audit.read and each returned event's tenant/resource scope.

## 11. Tenant membership and role administration

Sources:
- HeapCare.Api/src/HeapCare.Api/Endpoints/TenantMembershipEndpoints.cs
- HeapCare.Api/src/HeapCare.Api/Endpoints/TenantRoleEndpoints.cs
- HeapCare.Api/src/HeapCare.Api/Endpoints/TenantRegistryEndpoints.cs
- HeapCare.Api/src/HeapCare.Api/Endpoints/PlatformEndpoints.cs

| Route group / route | Current policy | Current meaning |
|---|---|---|
| GET/POST/PUT /api/v1/tenant/memberships | TenantMembershipManage | membership.manage |
| POST /api/v1/tenant/memberships/{membershipId}/deactivate | TenantMembershipManage | membership.manage |
| POST /api/v1/tenant/memberships/{membershipId}/reactivate | TenantMembershipManage | membership.manage |
| GET/POST/PUT /api/v1/tenant/roles | TenantMembershipManage | membership.manage (service further restricts role definition changes to tenant-wide admin) |
| POST /api/v1/tenant/roles/{roleId}/deactivate | TenantMembershipManage | membership.manage |
| POST /api/v1/tenant/roles/{roleId}/reactivate | TenantMembershipManage | membership.manage |
| Platform tenant registry/lifecycle routes | PlatformOperator | Persisted PlatformOperator identity; not a tenant permission |
| POST /api/v1/platform/tenants/{tenantId}/initial-administrator | PlatformOperator | Persisted PlatformOperator identity; initial user must already exist in the configured OIDC provider |
| GET /api/v1/platform/operator/status | authenticated user-scoped | Self-only PlatformOperator status |
| GET /api/v1/me/tenant-contexts | authenticated user-scoped | Current principal's own active tenant contexts |

The tenant membership/role group has additional service-layer delegation and tenant-wide restrictions. Do not infer that a route group permission alone grants authority to assign every role or scope.

## 12. M2 implementation rules

1. Retain route-level policy names as coarse, trusted permission gates during compatibility.
2. Replace the single-role request authority with immutable role-assignment context.
3. Add an assignment-aware resource authorization service. For every operation, the required permission and matching scope must be evaluated together on one active assignment.
4. List/read queries must calculate authorized resource scope from assignments that grant that specific permission; do not project all membership scopes independent of permissions.
5. Keep resource Tenant/Hospital/Department ownership validation. Never trust client-supplied role, permission, subject, email or scope.
6. Keep PlatformOperator separate. Platform status must never substitute for a tenant assignment.
7. Backfill exactly one role assignment per old membership using the existing role and exact existing scopes. No extra permissions.
8. Keep IPD permissions unassignable and preserve all existing authorization tests.
9. If a service relies on scope checks without passing a permission code today, update its contract to specify the permission for that operation before M2 implementation.
10. Treat any route with missing permission mapping or service scope validation as a release blocker; do not infer broad access from a tenant-wide context.

## 13. Verification requirements

- Positive/negative API tests for every route policy family and corresponding resource scope.
- A user with HospitalAdmin at Hospital A and ClinicalUser at Hospital B / Cardiology can use only the intended operations in each scope.
- The same user must not borrow Hospital B scope for a patient.read grant that exists only on the Hospital A assignment.
- List endpoints return only authorized resources for the specific permission code.
- Existing role changes, deactivation and custom permissions are observed on the next request.
- Fresh PostgreSQL install, populated migration upgrade, idempotent script and model-drift guard.
- Real-Keycloak Local acceptance after API and Web contracts are migrated.

## 14. Source inventory and limitations

The routes above were checked against endpoint source files on HeapCare.Api main on 2026-11-28. They describe the current route-level policy metadata. They are not a substitute for the service-level authorization/ownership checks mapped in the [compatibility inventory](user-management-compatibility-inventory.md). Facility descendant endpoints and any newly introduced routes must be added to this matrix as they are discovered or implemented.
