### Architecture Style
- Modular Monolith
- Domain Driven Design (DDD Lite)
- Service Object Pattern
- Query Object Pattern
- Event Driven Extensions
- Multi-Tenant SaaS
# High-Level Structure

```text
app/
├── controllers/
├── domains/
├── models/
├── policies/
├── serializers/
├── services/
├── queries/
├── jobs/
├── mailers/
├── events/
├── listeners/
├── forms/
├── validators/
├── concerns/
└── lib/
```
# Domain Structure

```text
app/domains/
├── organizations/
├── accounts/
├── employees/
├── attendance/
├── leave_management/
├── payroll/
├── payslips/
├── reports/
├── notifications/
└── audit_logs/
```

Each domain owns:

```text
domain/
├── services/
├── queries/
├── policies/
├── serializers/
├── forms/
├── validators/
└── events/
```
# Organization Domain

Responsible For:
- Organization creation
- Subdomain management
- Organization settings
- Ownership transfer

```text
organizations/
├── services/
│   ├── create_organization.rb
│   ├── update_organization.rb
│   ├── transfer_ownership.rb
│   └── deactivate_organization.rb
│
├── queries/
│   └── organization_lookup.rb
│
├── serializers/
│   └── organization_blueprint.rb
│
└── policies/
    └── organization_policy.rb
```
# Accounts Domain

Responsible For:
- Users
- Invitations
- Roles
- Permissions
- Authentication

```text
accounts/
├── services/
│   ├── invite_user.rb
│   ├── accept_invitation.rb
│   ├── assign_role.rb
│   ├── revoke_role.rb
│   └── sync_permissions.rb
│
├── queries/
│   └── user_directory.rb
│
├── serializers/
│   ├── user_blueprint.rb
│   ├── role_blueprint.rb
│   └── permission_blueprint.rb
│
└── policies/
```
# Employee Domain

Responsible For:
- Employee lifecycle
- Documents
- Departments
- Designations

```text
employees/
├── services/
│   ├── create_employee.rb
│   ├── update_employee.rb
│   ├── deactivate_employee.rb
│   ├── upload_document.rb
│   └── assign_manager.rb
│
├── queries/
│   ├── employee_search.rb
│   └── employee_directory.rb
│
├── serializers/
│   ├── employee_blueprint.rb
│   └── employee_detail_blueprint.rb
│
├── forms/
│   └── employee_form.rb
│
└── policies/
```
# Attendance Domain

Responsible For:
- Daily attendance
- Work hours
- OT calculations

```text
attendance/
├── services/
│   ├── create_attendance.rb
│   ├── update_attendance.rb
│   ├── approve_attendance.rb
│   ├── calculate_work_hours.rb
│   └── import_biometric_data.rb
│
├── queries/
│   ├── attendance_calendar.rb
│   ├── attendance_summary.rb
│   └── attendance_report.rb
│
├── serializers/
│   └── attendance_blueprint.rb
│
└── policies/
```
# Leave Management Domain

Responsible For:
- Leave requests
- Leave balances
- Holiday calendars

```text
leave_management/
├── services/
│   ├── apply_leave.rb
│   ├── approve_leave.rb
│   ├── reject_leave.rb
│   ├── cancel_leave.rb
│   ├── allocate_leave_balance.rb
│   └── recalculate_leave_balance.rb
│
├── queries/
│   ├── leave_calendar.rb
│   ├── leave_summary.rb
│   └── leave_balance_summary.rb
│
├── serializers/
│   ├── leave_request_blueprint.rb
│   └── leave_balance_blueprint.rb
│
└── policies/
```
# Payroll Domain

Most Critical Domain

Responsible For:
- Salary structures
- Payroll generation
- Payroll locking
- Payroll calculations

```text
payroll/
├── services/
│   ├── generate_payroll.rb
│   ├── approve_payroll.rb
│   ├── lock_payroll.rb
│   ├── unlock_payroll.rb
│   ├── recalculate_payroll.rb
│   └── generate_payroll_snapshot.rb
│
├── calculators/
│   ├── payroll_calculator.rb
│   ├── earnings_calculator.rb
│   ├── deduction_calculator.rb
│   ├── overtime_calculator.rb
│   ├── lop_calculator.rb
│   ├── pf_calculator.rb
│   └── esi_calculator.rb
│
├── queries/
│   ├── payroll_preview.rb
│   ├── payroll_summary.rb
│   └── payroll_history.rb
│
├── serializers/
│   ├── payroll_run_blueprint.rb
│   └── payroll_entry_blueprint.rb
│
└── policies/
```
# Payslip Domain

Responsible For:
- PDF generation
- Download
- Distribution

```text
payslips/
├── services/
│   ├── generate_payslip.rb
│   ├── generate_bulk_payslips.rb
│   └── email_payslip.rb
│
├── builders/
│   ├── payslip_pdf_builder.rb
│   └── salary_summary_builder.rb
│
├── serializers/
│   └── payslip_blueprint.rb
│
└── policies/
```
# Reports Domain

```text
reports/
├── queries/
│   ├── employee_report.rb
│   ├── attendance_report.rb
│   ├── leave_report.rb
│   ├── payroll_report.rb
│   └── salary_report.rb
│
├── exporters/
│   ├── csv_exporter.rb
│   └── xlsx_exporter.rb
│
└── serializers/
```
# Notification Domain

```text
notifications/
├── services/
│   ├── send_invitation.rb
│   ├── send_leave_notification.rb
│   ├── send_payslip_notification.rb
│   └── send_payroll_notification.rb
│
└── mailers/
```
# Audit Domain

```text
audit_logs/
├── services/
│   └── audit_logger.rb
│
├── queries/
│   └── audit_search.rb
│
└── serializers/
```
# Shared Services

```text
app/services/
├── tenant_resolver.rb
├── file_upload_service.rb
├── csv_import_service.rb
├── export_service.rb
├── permission_resolver.rb
└── event_publisher.rb
```
# Serialization Layer

```text
app/serializers/
├── application_blueprint.rb
├── pagination_blueprint.rb
└── error_blueprint.rb
```

All API responses pass through Blueprinter.

Example:

```ruby
EmployeeBlueprint.render_as_hash(employee)
```
# Policies

```text
app/policies/
├── application_policy.rb
├── organization_policy.rb
├── employee_policy.rb
├── attendance_policy.rb
├── leave_request_policy.rb
├── payroll_run_policy.rb
├── payslip_policy.rb
└── report_policy.rb
```

Authorization Flow:

```text
Controller
  ↓
Pundit Policy
  ↓
Permission Layer
  ↓
Role Permissions
```
# Event Architecture

```text
app/events/
├── employee_created.rb
├── employee_deactivated.rb
├── leave_approved.rb
├── payroll_generated.rb
├── payroll_locked.rb
└── payslip_generated.rb
```

```text
app/listeners/
├── create_leave_balance_listener.rb
├── generate_payslip_listener.rb
├── notify_leave_approved_listener.rb
└── sync_biometric_listener.rb
```
# Current Context

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :user
  attribute :organization
  attribute :request_id
end
```
# Background Jobs

```text
app/jobs/
├── payroll_generation_job.rb
├── payslip_generation_job.rb
├── invitation_email_job.rb
├── leave_notification_job.rb
├── report_export_job.rb
└── biometric_sync_job.rb
```

Uses:
- Solid Queue
- Active Job
# API Versioning

```text
app/controllers/api/
└── v1/
    ├── organizations/
    ├── accounts/
    ├── employees/
    ├── attendance/
    ├── leave_management/
    ├── payroll/
    ├── payslips/
    └── reports/
```
# Future Extraction Candidates

If the system grows significantly:
1. Payroll Domain
2. Attendance Domain
3. Notification Domain

can later become Rails Engines or standalone services without major rewrites because boundaries are already established.