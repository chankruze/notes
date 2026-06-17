## 1. Project Overview

### Product Vision

A multi-tenant cloud-based HRMS and Payroll platform that enables organizations to manage employees, attendance, leave, payroll, payslips, and employee self-service through organization-specific subdomains.

### Multi-Tenant Architecture
- Shared PostgreSQL database.
- Tenant isolation using `organization_id`.
- Organization-specific subdomains.
- Example:
    - poojapatha.hrms.com
    - abccompany.hrms.com
### User Roles
- Owner
- Admin
- Manager
- Employee
# 2. Functional Requirements

## 2.1 Multi-Tenancy
- FR-MT-001: System shall allow organization registration with a unique subdomain.
- FR-MT-002: System shall prevent duplicate subdomains.
- FR-MT-003: All organization-owned records shall be scoped using `organization_id`.
- FR-MT-004: Users shall only access records belonging to their organization.
- FR-MT-005: Cross-tenant access shall be prevented.
## 2.2 Authentication & Authorization
- FR-AUTH-001: Organization owner shall be able to register an organization.
- FR-AUTH-002: System shall support email verification.
- FR-AUTH-003: System shall support password reset functionality.
- FR-AUTH-004: System shall implement role-based access control.
- FR-AUTH-005: System shall support invitation-based onboarding.
- FR-AUTH-006: Invited users shall activate accounts by setting their password.
- FR-AUTH-007: System shall support secure logout from all active sessions.
## 2.3 Organization Management
- FR-ORG-001: Owner shall manage organization profile information.
- FR-ORG-002: Owner shall upload organization logo.
- FR-ORG-003: Owner shall invite admin and manager users.
- FR-ORG-004: Owner shall deactivate organization users.
- FR-ORG-005: Owner shall transfer organization ownership.
## 2.4 User Management
- FR-USER-001: Authorized users shall create user accounts.
- FR-USER-002: Authorized users shall edit user details.
- FR-USER-003: Authorized users shall deactivate users.
- FR-USER-004: Authorized users shall assign roles.
- FR-USER-005: System shall maintain user activity history.
## 2.5 Employee Management
- FR-EMP-001: Authorized users shall create employee records.
- FR-EMP-002: Employee records shall contain personal and employment details.
- FR-EMP-003: Authorized users shall edit employee records.
- FR-EMP-004: Authorized users shall deactivate employees.
- FR-EMP-005: System shall support employee search and filtering.
- FR-EMP-006: Employees shall have profile pages.
- FR-EMP-007: System shall support employee document storage.
- FR-EMP-008: System shall support future bulk employee import.
## 2.6 Departments & Designations
- FR-DEPT-001: Authorized users shall manage departments.
- FR-DEPT-002: Authorized users shall manage designations.
- FR-DEPT-003: Employees shall be assigned departments and designations.
## 2.7 Attendance Management
- FR-ATT-001: System shall support manual attendance entry.
- FR-ATT-002: Attendance statuses shall include Present, Absent, Half-Day, Leave, and Holiday.
- FR-ATT-003: System shall record check-in and check-out times.
- FR-ATT-004: System shall calculate work duration.
- FR-ATT-005: System shall record overtime hours.
- FR-ATT-006: System shall provide attendance calendar view.
- FR-ATT-007: System shall provide attendance reports.
- FR-ATT-008: System shall support future biometric integration.
## 2.8 Leave Management
- FR-LV-001: System shall support configurable leave types.
- FR-LV-002: Organizations shall define leave policies.
- FR-LV-003: System shall maintain leave balances.
- FR-LV-004: Employees shall apply for leave.
- FR-LV-005: System shall support full-day, half-day, and multi-day leave requests.
- FR-LV-006: Leave requests shall have Pending, Approved, Rejected, and Cancelled statuses.
- FR-LV-007: Managers/Admins shall approve or reject leave requests.
- FR-LV-008: System shall provide leave calendar view.
- FR-LV-009: Approved paid leave shall count as payable days.
- FR-LV-010: Approved unpaid leave shall reduce payable days.
- FR-LV-011: System shall maintain historical leave records.
- FR-LV-012: System shall support future leave encashment.
## 2.9 Holiday Management
- FR-HOL-001: Organizations shall manage holiday calendars.
- FR-HOL-002: Employees shall view upcoming holidays.
- FR-HOL-003: Holidays shall be excluded from leave calculations where applicable.
## 2.10 Salary Structure Management
- FR-SAL-001: System shall support configurable salary components.
- FR-SAL-002: Salary components shall be categorized as Earnings or Deductions.
- FR-SAL-003: Organizations shall configure salary structures.
- FR-SAL-004: Salary structures shall be assignable to employees.
- FR-SAL-005: Salary revisions shall maintain history.
- FR-SAL-006: System shall support PF, ESI, Professional Tax, Advance Recovery, Bonus, OT, and Custom Components.
## 2.11 Payroll Management
- FR-PAY-001: System shall support monthly payroll cycles.
- FR-PAY-002: Payroll calculations shall consider salary, attendance, leave, overtime, and deductions.
- FR-PAY-003: System shall generate payroll previews.
- FR-PAY-004: Authorized users shall approve payroll runs.
- FR-PAY-005: System shall lock finalized payroll runs.
- FR-PAY-006: Locked payroll runs shall require privileged access for modification.
- FR-PAY-007: System shall maintain payroll history.
- FR-PAY-008: System shall support payroll regeneration before lock.
## 2.12 Payslip Management
- FR-PS-001: System shall generate monthly payslips.
- FR-PS-002: Payslips shall be downloadable as PDF.
- FR-PS-003: Employees shall access their own payslips.
- FR-PS-004: Payslips shall preserve historical salary snapshots.
- FR-PS-005: System shall support future payslip email delivery.
- FR-PS-006: System shall provide yearly salary summaries.
## 2.13 Reporting
- FR-REP-001: System shall provide employee reports.
- FR-REP-002: System shall provide attendance reports.
- FR-REP-003: System shall provide leave reports.
- FR-REP-004: System shall provide payroll reports.
- FR-REP-005: System shall provide monthly salary summaries.
- FR-REP-006: System shall provide yearly salary summaries.
- FR-REP-007: Reports shall support filtering and export.
## 2.14 Notifications
- FR-NOT-001: System shall send invitation emails.
- FR-NOT-002: System shall send password reset emails.
- FR-NOT-003: System shall notify users of leave approval or rejection.
- FR-NOT-004: System shall support payroll completion notifications.
## 2.15 Audit Logs
- FR-AUD-001: System shall maintain audit logs for critical actions.
- FR-AUD-002: Audit logs shall record user, timestamp, action, and affected entity.
- FR-AUD-003: Salary modifications shall be auditable.
- FR-AUD-004: Payroll approvals and locks shall be auditable.
- FR-AUD-005: User role changes shall be auditable.
## 2.16 Employee Self-Service Portal
- FR-ESS-001: Employees shall view personal profiles.
- FR-ESS-002: Employees shall view attendance history.
- FR-ESS-003: Employees shall view leave balances.
- FR-ESS-004: Employees shall submit leave requests.
- FR-ESS-005: Employees shall download payslips.
- FR-ESS-006: Employees shall view holiday calendars.
# 3. Non-Functional Requirements

## Security
- NFR-SEC-001: All APIs shall require authentication.
- NFR-SEC-002: Tenant isolation shall be enforced at the application layer.
- NFR-SEC-003: Passwords shall be securely hashed.
- NFR-SEC-004: All communication shall use HTTPS.
- NFR-SEC-005: Authorization checks shall be enforced on every protected resource.
- NFR-SEC-006: Sensitive employee and payroll data shall be encrypted at rest where applicable.
## Performance
- NFR-PERF-001: Average page load time shall be under 2 seconds.
- NFR-PERF-002: Payroll generation for 1,000 employees shall complete within 60 seconds.
- NFR-PERF-003: Standard report generation shall complete within 10 seconds.
## Scalability
- NFR-SCALE-001: System shall support at least 1,000 organizations.
- NFR-SCALE-002: System shall support at least 100,000 employee records.
- NFR-SCALE-003: Architecture shall support horizontal scaling.
## Reliability
- NFR-REL-001: Daily automated database backups shall be maintained.
- NFR-REL-002: System availability target shall be 99.9%.
- NFR-REL-003: Failed background jobs shall be retriable.
## Maintainability
- NFR-MAIN-001: System shall follow modular domain-driven architecture.
- NFR-MAIN-002: Payroll logic shall be isolated from attendance and leave modules.
- NFR-MAIN-003: Multi-tenancy shall be centrally enforced.
- NFR-MAIN-004: Business-critical actions shall be covered by automated tests.
## Auditability
- NFR-AUD-001: Audit logs shall be immutable.
- NFR-AUD-002: Audit records shall be searchable and filterable.
## Compliance
- NFR-COMP-001: Payroll records shall be historically preserved.
- NFR-COMP-002: Payslip data shall remain unchanged after payroll lock.
- NFR-COMP-003: Employee financial records shall be retained according to configured retention policies.
# 4. Future Enhancements (Out of Scope for V1)
- FUT-001: Biometric attendance integration.
- FUT-002: Face recognition attendance.
- FUT-003: Mobile applications.
- FUT-004: Leave encashment.
- FUT-005: Employee reimbursement management.
- FUT-006: Expense management.
- FUT-007: Bank payout integration.
- FUT-008: Tax computation and statutory compliance automation.
- FUT-009: Employee performance management.
- FUT-010: Recruitment and applicant tracking system (ATS).
