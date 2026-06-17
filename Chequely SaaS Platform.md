# 1. Project Overview

## Product Vision

Chequely is a cloud-based multi-tenant SaaS platform that enables individuals and organizations to generate, manage, and print bank cheques using pre-configured bank templates.

The platform supports:
- Single cheque generation
- Bulk cheque generation
- PDF export
- Credit-based billing
- Organization user management
- Credit allocation
- Centralized bank template management

The platform shall provide accurate cheque printing aligned to bank-specific cheque layouts while maintaining a simple user experience.
# Business Goals

1. Eliminate manual cheque writing.
2. Reduce cheque printing errors.
3. Support high-volume cheque printing.
4. Monetize through a credit-based billing model.
5. Support both individuals and organizations.
6. Provide bank-specific cheque printing templates.
# 2. User Roles

## Platform Admin

System owner.

Responsibilities:
- Manage bank templates
- Manage pricing
- Manage users
- Manage organizations
- Manage credits
- View reports
## Individual User

Single account holder.

Responsibilities:
- Purchase credits
- Generate cheques
- Generate bulk cheques
- Download PDFs
## Organization Admin

Primary account holder.

Responsibilities:
- Purchase credits
- Invite users
- Allocate credits
- View organization usage
## Organization User

Member of an organization.

Responsibilities:
- Generate cheques
- Generate bulk cheques
- Consume assigned credits
# 3. Functional Requirements

# 3.1 Authentication & Account Management

- **FR-AUTH-001**: System shall support registration as:
	- Individual
	- Organization
- **FR-AUTH-002**: System shall support email verification.
- **FR-AUTH-003**: System shall support secure login.
- **FR-AUTH-004**: System shall support password reset.
- **FR-AUTH-005**: System shall support account deactivation.
- **FR-AUTH-006**: System shall support secure logout.
- **FR-AUTH-007**: System shall maintain active session records.
# 3.2 Organization Management

- FR-ORG-001: Organization Admin shall create organization accounts.
- FR-ORG-002: Organization Admin shall invite organization users.
- FR-ORG-003: Invited users shall activate accounts through invitation links.
- FR-ORG-004: Organization Admin shall deactivate organization users.
- FR-ORG-005: Organization Admin shall view organization users.
- FR-ORG-006: Organization Admin shall edit user details.
- FR-ORG-007: Organization Admin shall view organization credit consumption.

# 3.3 Credit Management

## Wallet System

- FR-CREDIT-001: Every account shall maintain a credit wallet.
- FR-CREDIT-002: Credits shall be consumed only during successful PDF generation.
- FR-CREDIT-003 Credit balance shall be visible to users.
- FR-CREDIT-004: System shall maintain complete credit transaction history.
## Organization Credit Allocation

- FR-CREDIT-005: Organization Admin shall allocate credits to organization users.
- FR-CREDIT-006: Organization Admin shall reclaim allocated credits.
- FR-CREDIT-007: Allocated credits shall not exceed available organization balance.
- FR-CREDIT-008: Organization users shall consume allocated credits only.
- FR-CREDIT-009: System shall maintain credit allocation history.
## Credit Ledger

- FR-CREDIT-010: Every credit transaction shall be recorded.
- FR-CREDIT-011: Credit transactions shall be immutable.
- FR-CREDIT-012: System shall support credit adjustment by Platform Admin.
# 3.4 Payment & Billing

- FR-PAY-001: Users shall purchase credits using supported payment gateways.
- FR-PAY-002: Successful payment shall automatically credit the wallet.
- FR-PAY-003: System shall generate invoices.
- FR-PAY-004: System shall maintain payment history.
- FR-PAY-005: Failed payments shall not create credits.
- FR-PAY-006: Platform Admin shall configure credit pricing.
Example:
- ₹1 = 1 Credit
- ₹5 = 10 Credits
# 3.5 Bank Template Management

## Platform Admin Only

- **FR-TEMPLATE-001**: Platform Admin shall create bank templates.
- **FR-TEMPLATE-002**: Platform Admin shall edit bank templates.
- **FR-TEMPLATE-003**: Platform Admin shall activate or deactivate templates.
- **FR-TEMPLATE-004**: Templates shall support coordinate positioning in millimeters.
- **FR-TEMPLATE-005**: Templates shall support configurable fonts.
- **FR-TEMPLATE-006**: Templates shall support versioning.
- **FR-TEMPLATE-007**: Users shall not create or modify templates.
- **FR-TEMPLATE-008**: Users shall only select from available templates.
## Template Configuration

A template shall support:
- Cheque Width
- Cheque Height
- Payee Position
- Amount Position
- Amount In Words Position
- Date Position
- Font Settings
- Character Spacing
# 3.6 Single Cheque Generation

FR-CHEQUE-001: Users shall create single cheques.
FR-CHEQUE-002: Users shall select a bank template.
FR-CHEQUE-003: Users shall enter:
	- Payee Name
	- Amount
	- Amount In Words
	- Date
FR-CHEQUE-004: System shall preview generated cheque.
FR-CHEQUE-005: System shall generate PDF output.
FR-CHEQUE-006: System shall deduct one credit per cheque page generated.
FR-CHEQUE-007: Generated cheques shall be stored in history.
# 3.7 Bulk Cheque Generation

- FR-BULK-001: Users shall upload cheque data via CSV.
- FR-BULK-002: Users shall create bulk cheque batches manually.
- FR-BULK-003: System shall validate uploaded records.
- FR-BULK-004: System shall generate a multi-page PDF.
- FR-BULK-005: One PDF page shall represent one cheque.
- FR-BULK-006: One credit shall be deducted per generated cheque page.
- FR-BULK-007: System shall display estimated credits before generation.
- FR-BULK-008: Generation shall fail if sufficient credits are unavailable.
# 3.8 PDF Generation

- FR-PDF-001: System shall generate print-ready PDF files.
- FR-PDF-002: Generated PDFs shall preserve template alignment.
- FR-PDF-003: PDF generation shall support A4 and custom cheque dimensions.
- FR-PDF-004: Generated PDFs shall be downloadable.
- FR-PDF-005: Users shall be able to re-download previously generated PDFs without additional credit deduction.

# 3.9 History Management

- FR-HISTORY-001: System shall maintain cheque generation history.
- FR-HISTORY-002: System shall maintain bulk generation history.
- FR-HISTORY-003: Users shall filter history by:
	- Date
	- Template
	- Status
- FR-HISTORY-004: Organization Admins shall view organization-wide history.
# 3.10 Reporting

- FR-REPORT-001: Platform Admin shall view total credits sold.
- FR-REPORT-002: Platform Admin shall view revenue reports.
- FR-REPORT-003: Platform Admin shall view total cheques generated.
- FR-REPORT-004: Organization Admin shall view user-level consumption reports.
# 3.11 Audit Logs

- FR-AUD-001: System shall maintain audit logs for:
	- Credit allocation
	- Credit adjustments
	- Template modifications
	- Payment events
	- User invitations
- FR-AUD-002: Audit logs shall include:
	- User
	- Timestamp
	- Action
	- Entity
# 4. Non-Functional Requirements

## Security
- NFR-SEC-001: All APIs shall require authentication.
- NFR-SEC-002: Role-based authorization shall be enforced.
- NFR-SEC-003: Passwords shall be securely hashed.
- NFR-SEC-004: All communication shall use HTTPS.
- NFR-SEC-005: Payment information shall never be stored directly.
- NFR-SEC-006: Credit transactions shall be tamper resistant.
## Performance
- NFR-PERF-001: Page load time shall be under 2 seconds.
- NFR-PERF-002: Single cheque generation shall complete within 2 seconds.
- NFR-PERF-003: 100 cheque PDF generation shall complete within 30 seconds.
- NFR-PERF-004: Credit deduction shall occur atomically.
## Scalability
- NFR-SCALE-001: System shall support 10,000 organizations.
- NFR-SCALE-002: System shall support 100,000 users.
- NFR-SCALE-003: System shall support 10 million cheque records.
- NFR-SCALE-004: PDF generation shall be horizontally scalable.
## Reliability
- NFR-REL-001: Daily automated backups shall be maintained.
- NFR-REL-002: Availability target shall be 99.9%.
- NFR-REL-003: Background jobs shall be retriable.
- NFR-REL-004: PDF generation failures shall not deduct credits.

## Maintainability
- NFR-MAIN-001: System shall follow modular domain-driven architecture.
- NFR-MAIN-002: Credit logic shall be isolated from cheque generation logic.
- NFR-MAIN-003: Payment processing shall be isolated from wallet management.
- NFR-MAIN-004: Business-critical workflows shall have automated tests.

## Auditability
- NFR-AUD-001: Credit transactions shall be immutable.
- NFR-AUD-002: Audit logs shall be searchable.
- NFR-AUD-003: Historical cheque records shall be preserved.
# 5. Future Enhancements (Out of Scope for V1)

- FUT-001: API access for ERP integrations.
- FUT-002: Tally integration.
- FUT-003: QuickBooks integration.
- FUT-004: Cheque approval workflow.
- FUT-005: Subscription-based plans.
- FUT-006: Mobile applications.
- FUT-007: WhatsApp delivery of generated PDFs.
- FUT-008: Multi-language cheque generation.
- FUT-009: Bank-specific MICR validation.
- FUT-010: Bulk cheque import from Excel templates.
# 6. Detailed Use Cases

# UC-001 User Registration (Individual)

**Actors**: Individual User
**Preconditions**: User is not registered
### Flow
1. User selects Individual Account
2. User enters personal details
3. User verifies email
4. System creates account
5. Wallet is initialized with 0 credits
### Postconditions
- User account exists
- User can log in
# UC-002 User Registration (Organization)

**Actors**: Organization Admin
### Flow
1. User selects Organization Account
2. Enters organization details
3. Verifies email
4. System creates organization
5. System creates admin account
6. Organization wallet is initialized
### Postconditions
- Organization created
- Admin account created

# UC-003 User Login

**Actors**: All Users
### Flow
1. User enters credentials
2. System validates credentials
3. Session created

**Alternate** Flow: Invalid credentials
**Postconditions**: User authenticated

# UC-004 Invite Organization User

**Actors**: Organization Admin
### Flow
1. Open Users page
2. Click Invite User
3. Enter email
4. Send invitation
5. System sends invitation email
**Postconditions**: Invitation pending
# UC-005 Accept Invitation

**Actors**: Organization User
### Flow
1. Open invite link
2. Set password
3. Activate account
**Postconditions**: User becomes active
# UC-006 Purchase Credits

### Actors
- Individual User  
- Organization Admin
### Flow
1. Open Wallet
2. Select package
3. Make payment
4. Payment verified
5. Credits added

**Postconditions**: Wallet balance increased
# UC-007 Allocate Credits

### Actors
- Organization Admin
### Flow
1. Select organization user
2. Enter credit amount
3. Confirm allocation

### Rules
- Cannot exceed available balance
### Postconditions
- Credits transferred
# UC-008 Reclaim Credits

### Actors
- Organization Admin
### Flow
1. Select user
2. Enter reclaim amount
3. Confirm
### Postconditions
- Credits returned to organization pool

# UC-009 View Wallet Balance

### Actors
- All Users
### Flow
1. Open wallet dashboard
2. View balance
# UC-010 View Credit Ledger

### Actors
- All Users
### Flow
1. Open transactions
2. View history
# UC-011 Select Bank Template

### Actors
- All Users
### Flow
1. Create cheque
2. Select bank template
### Postconditions
- Template loaded
# UC-012 Generate Single Cheque

### Actors
- All Users
### Flow
1. Select template
2. Enter cheque details
3. Preview cheque
4. Generate PDF
### Postconditions
- PDF generated
- 1 credit deducted
# UC-013 Preview Cheque

### Actors
- All Users
### Flow
1. Fill cheque details
2. Click Preview
### Postconditions
- Visual preview shown
# UC-014 Download Cheque PDF

### Actors
- All Users
### Flow
1. Open generated cheque
2. Download PDF
### Postconditions
- PDF downloaded
# UC-015 Create Bulk Job

### Actors
- All Users
### Flow
1. Upload CSV
2. Select template
3. Validate data
### Postconditions
- Bulk job created
# UC-016 Generate Bulk PDF

### Actors
- All Users
### Flow
1. Submit bulk job
2. System validates credits
3. Generate PDF
### Postconditions
- Multi-page PDF generated
# UC-017 View Generation History

### Actors
- All Users
### Flow
1. Open history
2. Browse generated cheques
# UC-018 Re-download Historical PDF

### Actors
- All Users
### Flow
1. Open history
2. Select cheque
3. Download PDF
### Postconditions
- No additional credits deducted
# UC-019 Manage Bank Template

### Actors
- Platform Admin
### Flow
1. Create template
2. Configure positions in mm
3. Save
### Postconditions
- Template available
# UC-020 Edit Bank Template

### Actors
- Platform Admin
### Flow
1. Open template
2. Modify coordinates
3. Save new version
# UC-021 Activate Template

### Actors
- Platform Admin
### Flow
1. Select template
2. Mark active
# UC-022 Deactivate Template

### Actors
- Platform Admin
### Flow
1. Select template
2. Mark inactive
# UC-023 View Revenue Dashboard

### Actors
- Platform Admin
### Flow
1. Open dashboard
2. View revenue metrics
# UC-024 View Credit Usage Reports

### Actors
- Platform Admin  
- Organization Admin
### Flow
1. Select date range    
2. Generate report
# UC-025 Adjust Credits

### Actors
- Platform Admin
### Flow
1. Open user wallet
2. Enter adjustment
3. Save
# UC-026 View Audit Logs

### Actors
- Platform Admin
### Flow
1. Open audit logs
2. Search entries
# UC-027 Reset Password

### Actors
- All Users
### Flow
1. Request reset
2. Open email
3. Set new password
# UC-028 Suspend User

### Actors
- Platform Admin
### Flow
1. Select user
2. Suspend account
# UC-029 Export Usage Report

### Actors
- Platform Admin  
- Organization Admin
### Flow
1. Generate report
2. Export CSV
# UC-030 View Organization Consumption

### Actors
- Organization Admin
### Flow
1. Open analytics
2. View user consumption
# 7. RBAC Permission Matrix

|Feature|Individual|Org User|Org Admin|Platform Admin|
|---|---|---|---|---|
|Register Account|✅|❌|✅|❌|
|Login|✅|✅|✅|✅|
|Reset Password|✅|✅|✅|✅|
|Purchase Credits|✅|❌|✅|❌|
|View Wallet|✅|✅|✅|✅|
|View Credit Ledger|✅|✅|✅|✅|
|Generate Single Cheque|✅|✅|✅|✅|
|Generate Bulk Cheque|✅|✅|✅|✅|
|Preview Cheque|✅|✅|✅|✅|
|Download PDF|✅|✅|✅|✅|
|View Own History|✅|✅|✅|✅|
|Invite Users|❌|❌|✅|❌|
|Activate User|❌|❌|✅|❌|
|Deactivate User|❌|❌|✅|✅|
|Allocate Credits|❌|❌|✅|❌|
|Reclaim Credits|❌|❌|✅|❌|
|View Org Usage|❌|❌|✅|❌|
|View Revenue Reports|❌|❌|❌|✅|
|Create Template|❌|❌|❌|✅|
|Edit Template|❌|❌|❌|✅|
|Activate Template|❌|❌|❌|✅|
|Deactivate Template|❌|❌|❌|✅|
|Manage Pricing|❌|❌|❌|✅|
|Adjust Credits|❌|❌|❌|✅|
|View Audit Logs|❌|❌|❌|✅|
|Suspend User|❌|❌|❌|✅|
# 8. Acceptance Criteria

## FR-AUTH-001
- User Registration
### Acceptance Criteria
- User can register as Individual.
- User can register as Organization.
- Duplicate emails are rejected.
- Email verification required.
- Account remains inactive until verification.
## FR-CREDIT-005
- Allocate Credits
### Acceptance Criteria
- Organization Admin can allocate credits.
- Allocation cannot exceed available balance.
- Allocation creates ledger transaction.
- User balance updates immediately.
- Audit log entry created.
## FR-TEMPLATE-001
- Create Bank Template
### Acceptance Criteria
- Only Platform Admin can create template.
- Template stores coordinate values in mm.
- Template supports versioning.
- Template can be activated/deactivated.
## FR-CHEQUE-005
- Generate PDF
### Acceptance Criteria
- PDF generated successfully.
- Template alignment preserved.
- PDF downloadable.
- History record created.
- Credit deducted atomically.
## FR-BULK-004
- Generate Bulk PDF
### Acceptance Criteria
- CSV validated.
- Invalid rows reported.
- PDF contains one cheque per page.
- Credits deducted equal page count.
- Job status tracked.
## FR-PAY-002
- Credit Wallet Recharge
### Acceptance Criteria
- Successful payment credits wallet.
- Failed payment does not credit wallet.
- Ledger transaction created.
- Invoice generated.
## FR-HISTORY-001
- Generation History
### Acceptance Criteria
- History visible to authorized users.
- Search available.
- Date filters available.
- Historical PDFs downloadable.
## FR-AUD-001
- Audit Logs
### Acceptance Criteria
- User recorded.
- Timestamp recorded.
- Entity recorded.
- Action recorded.
- Logs immutable.

# 9. Screen Inventory

## Total Estimated Screens

- Customer Portal: ~20 Screens
- Platform Admin Portal: ~15 Screens
- Shared Components/Modals: ~10 Components
- Total UI Effort:  35–45 screens/views
# A. Public Website

**SCR-PUB-001:** Home Page
- Purpose:
	- Product overview
	- Pricing highlights
	- CTA buttons
- Features:
	- Hero section
	- Benefits
	- Pricing
	- Contact

**SCR-PUB-002:** Pricing Page - Display credit pricing.
- Features:
	- Credit packages
	- FAQ
	- Buy credits CTA

**SCR-PUB-003:** Login Page
- Email
- Password
- Forgot password

**SCR-PUB-004:** Registration Page
- Account Type:
	- Individual
	- Organization

**SCR-PUB-005:** Forgot Password
**SCR-PUB-006:** Reset Password
- Features:
	- New password
	- Confirm password
# B. Customer Portal

**SCR-CUST-001:** Dashboard - Overview page.
- Widgets:
	- Available credits
	- Credits consumed
	- Total cheques generated
	- Recent activity

**SCR-CUST-002:** Wallet - View balance.
- Widgets:
	- Available credits
	- Recharge button

**SCR-CUST-003:** Credit Transactions - Wallet ledger.
- Columns:
	- Date
	- Type
	- Credits
	- Balance

**SCR-CUST-004:** Purchase Credits - Recharge wallet.
- Features:
	- Package selection
	- Payment gateway
# Cheque Management

**SCR-CUST-005:** Cheque List - View all generated cheques.
- Filters:
	- Date
	- Template
	- Status

**SCR-CUST-006:** Create Single Cheque - Enter cheque details.
- Fields:
	- Template
	- Payee Name
	- Amount
	- Amount In Words
	- Date
- Actions:
	- Preview
	- Generate

**SCR-CUST-007:** Cheque Preview - Preview before PDF generation.
- Display:
	- Actual cheque rendering
	- Zoom controls
	- Page ruler/grid (optional)
- Actions:
	- Edit
	- Generate PDF

This screen should show exactly what will be printed.

**SCR-CUST-008:** Generated Cheque Details - View generated cheque.
- Actions:
	- Download PDF
	- Re-download
- Information:
	- Credits consumed
	- Created date
# Bulk Cheques

**SCR-CUST-009:** Bulk Jobs List - View all bulk jobs.
- Columns:
	- Job Name
	- Status
	- Pages
	- Credits Used

**SCR-CUST-010:** Create Bulk Job - Upload data.
- Features:
	- CSV upload
	- Manual entry

**SCR-CUST-011:** Bulk Validation Screen - Validate uploaded data.
- Display:
	- Total records
	- Valid records
	- Invalid records
- Actions:
	- Fix errors
	- Continue

**SCR-CUST-012:** Bulk Preview - Preview bulk output.
- Display:
	- First cheque
	- Last cheque
	- Page count
	- Credit estimate
- Actions:
	- Generate PDF

**SCR-CUST-013:** Bulk Job Details - View generated batch.
- Actions:
	- Download PDF
	- View metadata
# History

**SCR-CUST-014:** History - Generation history.
- Tabs:
	- Single Cheques
	- Bulk Jobs

**SCR-CUST-015:** Profile Settings: - Manage profile.
**SCR-CUST-016:** Change Password - Security management.
# Organization Features

**SCR-ORG-001:** Organization Dashboard - Organization analytics.
Widgets:
- Credits available
- Credits allocated
- Credits consumed

**SCR-ORG-002:** Organization Users - User management.
Actions:
- Invite
- Deactivate

**SCR-ORG-003:** Invite User - Send invitation.
**SCR-ORG-004:** User Credit Allocation - Allocate credits.
- Display:
	- Available balance
	- Assigned balance

**SCR-ORG-005:** User Usage Report - Track consumption.
- Columns:
	- User
	- Credits Used
	- Cheques Generated
# C. Platform Admin Portal

**SCR-ADM-001:** Admin Dashboard
- Widgets:
	- Revenue
	- Credits Sold
	- Total Users
	- Total Organizations
	- Total Cheques

**SCR-ADM-002:** Users Management - Manage all users.
**SCR-ADM-003:** Organizations Management - Manage organizations.
**SCR-ADM-004:** Credit Adjustments - Manually add/remove credits.
**SCR-ADM-005**: Revenue Reports - Business analytics.
# Bank Template Management

**SCR-ADM-006:** Template List - View templates.
- Columns:
	- Bank
	- Version
	- Status

**SCR-ADM-007:** Create Template - Configure bank template.
- Fields:
	- Name
	- Version
	- Dimensions

**SCR-ADM-008:** Template Coordinate Editor - Configure positions.
- Features:
	- Upload cheque image
	- Visual editor
	- Drag-and-drop fields
	- MM coordinates
- Fields:
	- Payee
	- Amount
	- Amount Words
	- Date

**SCR-ADM-009:** Template Preview - Preview rendered cheque.
- Actions:
	- Save
	- Publish

**SCR-ADM-010**: Template Version History - View revisions.
**SCR-ADM-011**: Pricing Management - Manage credit pricing.
**SCR-ADM-012**: Payment History - View transactions.
**SCR-ADM-013**: Audit Logs - Security tracking.
**SCR-ADM-014**: System Settings - Global configuration.
**SCR-ADM-015**: Reports & Analytics - Operational reports.
# D. Shared Modals / Components

**CMP-001**: nvite User Modal
**CMP-002**: Allocate Credits Modal
**CMP-003**: Credit Adjustment Modal
**CMP-004**: Confirm Generate PDF Modal
- Display pages
- Credits Required
**CMP-005:** Insufficient Credits Modal
**CMP-006:** Payment Success Modal
**CMP-007:** Delete Confirmation Modal
**CMP-008:** CSV Import Wizard
**CMP-009:** PDF Download Modal
**CMP-010:** Template Publish Confirmation
## SCR-CUST-017 Print Calibration Test

**Purpose**: User prints a sample calibration sheet.
**Helps ensure**:
- Printer scaling issues
- Margin issues
- Browser print issues
**Actions**:
- Download calibration PDF
- Verify alignment