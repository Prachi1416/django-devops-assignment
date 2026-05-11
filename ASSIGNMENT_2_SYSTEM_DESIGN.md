Assignment 2 — SaaS Leave Management System Design
1. Data Model Design

The Leave Management System is designed as a multi-tenant SaaS platform where multiple companies use the same application infrastructure while keeping their data logically isolated. Each company is treated as a tenant, and every important business record is associated with a tenant identifier. Since Shemon has a small DevOps team, the best choice is a shared database with a shared schema approach using tenant isolation at the application and database query level.

A shared-schema multi-tenant architecture significantly reduces operational overhead because only one database cluster needs to be maintained, monitored, backed up, and migrated. Separate databases per tenant provide stronger isolation but increase deployment complexity, infrastructure cost, and migration effort. For an early-stage SaaS product with moderate scale expectations, shared schema provides the best balance between scalability, simplicity, and maintainability.

The system contains several core entities.

The Company table represents a tenant. Each company has fields such as company name, timezone, leave configuration settings, and subscription details.

The User table stores authentication and profile information for all users in the platform. Every user belongs to exactly one company through a company_id foreign key. Users have roles such as employee, manager, or HR admin. Role-based access control determines what actions they can perform.

The EmployeeProfile table stores HR-related employee information such as joining date, department, reporting manager, employment status, and leave policy assignment. This table is separated from authentication data to keep business information modular.

The Department table organizes employees within a company. Departments belong to a company, and employees are mapped to departments.

The LeaveType table defines available leave categories such as Casual Leave, Sick Leave, Earned Leave, or Maternity Leave. Since policies vary between companies, leave types are tenant-specific.

The LeavePolicy table stores company-specific leave rules. For example, one company may allow 1.5 leave days accrued monthly while another provides annual lump-sum allocation. Policies also define carry-forward rules, yearly caps, encashment eligibility, and probation restrictions.

The LeaveBalance table stores the current leave balance of each employee for every leave type. This table is updated whenever accrual jobs run or leave requests are approved.

The LeaveRequest table is the central transactional entity. It contains leave start date, end date, reason, leave type, request status, approver information, timestamps, and audit metadata. Each leave request belongs to a single employee and company.

The ApprovalHistory table stores all state transitions for auditability. Instead of overwriting approval data directly in the leave request row, every action such as submitted, approved, rejected, or cancelled is stored as an immutable event. This becomes important for compliance, debugging, and reporting.

The Notification table stores in-app notifications for users. Email delivery logs can also be tracked here or in a separate asynchronous event table.

The relationships are structured as follows:

One company has many users
One company has many departments
One department has many employees
One employee has one reporting manager
One company has many leave types and policies
One employee has many leave balances
One employee creates many leave requests
One leave request has many approval history entries
One leave request triggers many notifications

Tenant isolation is enforced by always filtering queries using company_id. Middleware extracts the tenant from the authenticated user context, ensuring users never access another company’s data.

2. Leave Request Lifecycle

When an employee applies for leave, the frontend sends a request to the Leave Request API with leave dates, leave type, and reason. The backend first validates the request by checking whether the employee belongs to the tenant, whether the leave type is active for that company, whether enough leave balance exists, and whether overlapping leave requests already exist.

Once validation succeeds, a new row is created in the LeaveRequest table with status PENDING_MANAGER_APPROVAL. Simultaneously, an entry is added to the ApprovalHistory table indicating that the request was submitted.

After the leave request is created, the system publishes an internal domain event such as leave_request_submitted. This event is pushed into a background queue system like Celery with Redis or RabbitMQ. The queue worker sends notifications to the reporting manager without slowing down the API response.

The manager receives an in-app notification and email containing the leave details. When the manager opens the request and clicks approve or reject, the backend performs a transactional update.

To prevent race conditions where two managers attempt conflicting actions simultaneously, the system uses optimistic locking or row-level database locking. During approval, the backend executes a query similar to:

UPDATE leave_request
SET status = 'APPROVED'
WHERE id = ? AND status = 'PENDING_MANAGER_APPROVAL';

If the row count affected is zero, it means another action already modified the request. The API then returns an error indicating that the leave request was already processed.

This guarantees that a rejected request cannot later become approved accidentally.

After manager approval, another approval history entry is inserted. The system then emits another event such as leave_request_approved.

A background worker listens for this event and performs the following tasks asynchronously:

Sends HR notifications
Sends confirmation email to employee
Creates in-app notifications
Updates leave balances
Generates audit logs

If the manager rejects the request, the same workflow executes with status REJECTED, except leave balances remain unchanged.

The leave request state is therefore centrally stored in the LeaveRequest table, while the ApprovalHistory table maintains the full workflow timeline for auditing and debugging purposes.

3. Hard Problem — Custom Leave Balance Calculations

One of the most difficult parts of a Leave Management System is supporting highly customizable leave policies because every company follows different HR rules.

The design uses a policy-driven calculation engine rather than hardcoded logic.

Each company defines leave rules inside the LeavePolicy table. Example fields include:

accrual_rate_per_month
accrual_frequency
maximum_balance
carry_forward_limit
carry_forward_expiry_month
negative_balance_allowed
probation_eligibility
weekends_counted
holiday_calendar_id

Instead of calculating balances dynamically during every API request, the system maintains a persistent LeaveBalance table for performance reasons.

A scheduled background job runs daily or monthly depending on policy configuration. The scheduler evaluates all active employees against their assigned policies.

For example, consider a company policy:

Earn 1.5 leave days per month
Maximum yearly balance = 18
Carry-forward expires after March

At the beginning of each month, the accrual worker executes:

Fetch all employees under that policy
Add 1.5 leave days
Ensure balance does not exceed yearly maximum
Apply rounding rules
Insert audit entries into a LeaveLedger table

The LeaveLedger table acts like a bank statement for leave accounting. Instead of storing only the final balance, every transaction is recorded:

Monthly accrual
Leave deduction
Carry-forward
Expiry adjustment
Manual HR correction

This ledger-based approach makes recalculation and auditing much easier.

Carry-forward expiration is handled by a separate scheduled job. On April 1st, the system checks unused carry-forward balances from the previous year and automatically deducts expired days according to company policy.

This architecture is flexible because new leave rules can be introduced without rewriting core approval workflows. Most policy changes become configuration updates rather than engineering changes.

4. Features Deliberately Excluded from v1

The first version of the product should focus on stability, tenant isolation, approval workflows, and reliable notifications. Several advanced features should intentionally be postponed.

The first excluded feature is multi-level approval chains. Some enterprises require workflows such as manager approval followed by director approval and then HR approval. While valuable, this significantly increases workflow complexity, state transitions, escalation handling, and UI logic. A single-manager approval process is sufficient for v1.

The second excluded feature is payroll integration. Synchronizing leave deductions with payroll systems introduces external API dependencies, reconciliation complexity, and compliance risks. Since payroll vendors differ across companies, this integration should be added only after the core leave workflow becomes stable.

The third excluded feature is advanced analytics and forecasting dashboards. Predictive leave trends, department utilization graphs, and burnout analytics are useful for enterprise customers, but they are not essential for validating the product’s core value. Reporting can initially remain simple with basic CSV exports and standard summaries.

By deliberately limiting scope in v1, the engineering team can prioritize reliability, maintainability, and customer onboarding speed instead of prematurely building enterprise-scale complexity.