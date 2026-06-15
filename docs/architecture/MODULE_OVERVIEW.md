# Module Overview — ERP Accounting System

> This document describes all 21 modules of the system. Each entry covers purpose, key entities, key operations, inter-module dependencies, and the API route prefix.

---

## 1. Core

**Purpose:** Provides the foundational infrastructure shared by every other module, including base classes, tenant scoping, fiscal period management, and cross-cutting middleware. It has no user-facing business logic of its own but is a hard dependency for all other modules.

**Key Entities:**
- `Company` — top-level tenant; holds fiscal year config, base currency, timezone
- `Branch` — subdivision of a company; enables cost-center-level data isolation
- `FiscalPeriod` — calendar month or quarter within a fiscal year; carries `open`/`closed` status
- `BaseModel` — abstract Eloquent model with UUID PKs, soft deletes, and tenant scopes
- `BaseRepository` — injects `company_id`/`branch_id` global scopes into all queries

**Key Operations:**
- Resolve and inject tenant context (company + branch) into every authenticated request
- Enforce fiscal period locking via `FiscalPeriodOpen` middleware on all write routes
- Provide `BaseRepository::scopeTenant()` applied automatically to all module repositories
- Manage fiscal period CRUD and status transitions (open → closed)
- Supply shared pagination, filtering, and sorting infrastructure

**Dependencies on Other Modules:** None — Core is the root dependency.

**API Route Prefix:** `/api/v1/core`

---

## 2. Auth & RBAC

**Purpose:** Handles user authentication using Laravel Sanctum SPA tokens and enforces role-based access control via `spatie/laravel-permission`. It manages user identities, roles, permissions, and login audit events.

**Key Entities:**
- `User` — application user with `HasRoles` trait; belongs to one or more companies
- `Role` — named role (e.g., `admin`, `accountant`, `viewer`, `auditor`)
- `Permission` — granular permission string (e.g., `journal_entry.create`)
- `PersonalAccessToken` — Sanctum token with expiry and last-used tracking

**Key Operations:**
- Issue and revoke Sanctum API tokens on login/logout
- Assign and remove roles and permissions per user per company
- Validate `Bearer` tokens on every authenticated request
- Enforce permission gates on all controller actions via `$this->authorize()`
- Revoke expired tokens via scheduled `RevokeExpiredTokens` job

**Dependencies on Other Modules:** Core (tenant scoping, base models)

**API Route Prefix:** `/api/v1/auth`

---

## 3. Company

**Purpose:** Manages the top-level tenant entity — the company — including its profile, settings, fiscal year configuration, and base currency. Every other data record is ultimately scoped to a company.

**Key Entities:**
- `Company` — name, registration number, tax ID, base currency, fiscal year start month, timezone, logo
- `CompanySettings` — JSON settings bag for feature flags and preferences

**Key Operations:**
- Create and configure a company (provisioning)
- Update company profile and settings
- Configure fiscal year start month and default currency
- Upload and retrieve company logo via S3-compatible storage
- Retrieve company settings for use in other module configurations

**Dependencies on Other Modules:** Core, Auth

**API Route Prefix:** `/api/v1/companies`

---

## 4. Branch

**Purpose:** Models branches or cost centers within a company, enabling multi-location accounting where each branch can maintain its own transaction records while consolidating at the company level.

**Key Entities:**
- `Branch` — name, code, address, phone, email, currency, status (active/inactive); belongs to `Company`

**Key Operations:**
- Create, update, activate, and deactivate branches
- Seed a default chart of accounts for a new branch from a company template
- List branches available to the authenticated user (respects user-branch assignments)
- Switch active branch context in the user session

**Dependencies on Other Modules:** Core, Company

**API Route Prefix:** `/api/v1/branches`

---

## 5. General Ledger

**Purpose:** The financial core of the ERP — maintains the chart of accounts, records double-entry journal entries, and provides the trial balance and account balances that feed all financial reports. Every financial transaction in every other module ultimately posts to the GL.

**Key Entities:**
- `Account` — chart-of-accounts node with type (Asset/Liability/Equity/Revenue/Expense), normal balance (Dr/Cr), parent account for hierarchy
- `JournalEntry` — header record with date, reference number, description, fiscal period, and status (`draft`/`posted`/`reversed`)
- `JournalEntryLine` — individual debit or credit line linked to an account; `debit_amount` and `credit_amount` fields
- `AccountBalance` — denormalized running balance cache updated on each posting

**Key Operations:**
- Create and maintain a hierarchical chart of accounts
- Create journal entries in `draft` status with any number of debit/credit lines
- Post a journal entry (status → `posted`); enforced DB constraint ensures sum(debits) = sum(credits)
- Reverse a posted entry by creating an equal and opposite journal entry
- Compute trial balance for any date range using PostgreSQL window functions
- Retrieve account hierarchy via recursive CTE query

**Dependencies on Other Modules:** Core, Auth, Company, Branch, FiscalPeriod (via Core)

**API Route Prefix:** `/api/v1/gl`

---

## 6. Accounts Payable

**Purpose:** Manages the full lifecycle of vendor invoices from receipt to payment, including approval workflows, payment scheduling, and automatic GL posting. It tracks the company's outstanding obligations to suppliers.

**Key Entities:**
- `Vendor` — supplier master: name, tax ID, payment terms, default AP account, bank details
- `VendorInvoice` — invoice header with vendor, date, due date, status (`draft`/`approved`/`paid`/`cancelled`)
- `VendorInvoiceLine` — line item with amount, GL account, tax code
- `VendorPayment` — payment record linked to one or more invoices via allocation

**Key Operations:**
- Create and maintain vendor master records
- Enter vendor invoices in draft; route through approval workflow
- Post approved invoices to GL (Debit: Expense account, Credit: Accounts Payable)
- Record vendor payments and allocate against outstanding invoices
- Generate aged creditors report (30/60/90/120+ day buckets)
- Send payment due reminder notifications

**Dependencies on Other Modules:** Core, Auth, GeneralLedger, TaxManagement, Documents

**API Route Prefix:** `/api/v1/ap`

---

## 7. Accounts Receivable

**Purpose:** Manages customer invoicing and cash collection, tracking the money owed to the company by its customers. It handles invoice generation, receipt recording, and aging analysis for collections management.

**Key Entities:**
- `Customer` — customer master: name, tax ID, credit limit, payment terms, default AR account
- `CustomerInvoice` — invoice with customer, issue date, due date, status; auto-numbering
- `CustomerInvoiceLine` — line item with description, quantity, unit price, tax code, GL account
- `CustomerReceipt` — receipt record that allocates collected cash against open invoices

**Key Operations:**
- Create and maintain customer master records
- Generate and send customer invoices (PDF via Documents module)
- Post invoices to GL (Debit: Accounts Receivable, Credit: Revenue)
- Record customer receipts and allocate to outstanding invoices
- Compute AR aging buckets for collections follow-up
- Send automated invoice reminder emails (queued via Horizon)

**Dependencies on Other Modules:** Core, Auth, GeneralLedger, TaxManagement, Documents, Reporting

**API Route Prefix:** `/api/v1/ar`

---

## 8. Cash Management

**Purpose:** Tracks petty cash funds and internal cash accounts, recording all cash inflows and outflows and maintaining real-time cash balances. It supports inter-account cash transfers and daily cash reconciliation.

**Key Entities:**
- `CashAccount` — named cash fund (petty cash, till, safe) with a linked GL account and current balance
- `CashTransaction` — individual cash receipt or payment with amount, payee/payer, and category

**Key Operations:**
- Create and configure cash accounts linked to GL accounts
- Record cash receipts and disbursements with automatic GL posting
- Transfer cash between accounts with dual-sided GL entry
- View real-time cash balances per account
- Daily cash-up reconciliation (entered cash vs. system balance)

**Dependencies on Other Modules:** Core, Auth, GeneralLedger, Branch

**API Route Prefix:** `/api/v1/cash`

---

## 9. Bank Reconciliation

**Purpose:** Reconciles the company's bank accounts against its GL cash accounts by importing bank statements and matching bank transactions to GL entries. It identifies unmatched items and produces the formal bank reconciliation statement.

**Key Entities:**
- `BankAccount` — bank account with IBAN/account number, linked GL account, current statement balance
- `BankStatement` — imported statement for a specific period with a closing balance
- `BankStatementLine` — individual transaction from the bank statement (date, description, amount)
- `BankReconciliation` — reconciliation record with matched/unmatched summary and sign-off status

**Key Operations:**
- Import bank statement CSV or OFX files and parse into statement lines
- Auto-match bank statement lines to GL journal entry lines using amount, date, and description heuristics
- Manually match or unmatch individual items
- Record bank charges and interest not yet in the GL directly from the reconciliation screen
- Produce and lock the formal bank reconciliation statement
- Run `AutoMatchBankLines` background job via Horizon

**Dependencies on Other Modules:** Core, Auth, GeneralLedger, CashManagement, Documents

**API Route Prefix:** `/api/v1/bank-reconciliation`

---

## 10. Fixed Assets

**Purpose:** Maintains the fixed asset register, calculates and posts depreciation, and manages asset disposals with gain/loss computation. It ensures that asset values on the balance sheet remain accurate over the asset lifecycle.

**Key Entities:**
- `AssetCategory` — grouping with default depreciation method, useful life, and linked GL accounts
- `Asset` — individual asset with cost, acquisition date, useful life, residual value, and depreciation method
- `DepreciationEntry` — monthly depreciation record per asset with amount and linked journal entry
- `AssetDisposal` — disposal record with sale proceeds, gain/loss amount, and disposal journal entry

**Key Operations:**
- Register new assets and assign to categories with default GL accounts
- Calculate depreciation using straight-line or declining-balance methods
- Run monthly depreciation for all assets in a period and post batch GL entry
- Record asset disposals with automatic gain/loss computation and GL posting
- Generate asset register report and depreciation schedule

**Dependencies on Other Modules:** Core, Auth, GeneralLedger, Documents, PeriodClosing

**API Route Prefix:** `/api/v1/fixed-assets`

---

## 11. Inventory

**Purpose:** Manages product/item master data, warehouse stock levels, and all stock movements (receipts, issues, adjustments, transfers). It maintains accurate stock valuation using configurable costing methods.

**Key Entities:**
- `Product` — item master with code, description, unit of measure, costing method (FIFO/WAC), and inventory GL account
- `Warehouse` — physical or logical storage location with a linked branch
- `StockBalance` — denormalized current quantity and value per product per warehouse
- `StockMovement` — individual stock transaction with type, quantity, unit cost, and linked source document

**Key Operations:**
- Create and maintain product master records and warehouse locations
- Record stock receipts from purchase orders (Purchasing integration)
- Record stock issues against sales orders (Sales integration)
- Perform stock adjustments with write-up/write-down GL entries
- Transfer stock between warehouses
- Calculate stock valuation using FIFO or weighted-average cost
- Generate stock-on-hand and stock movement reports

**Dependencies on Other Modules:** Core, Auth, GeneralLedger, Purchasing, Sales

**API Route Prefix:** `/api/v1/inventory`

---

## 12. Purchasing

**Purpose:** Manages the full procure-to-pay cycle excluding payment: requisitions, purchase orders, and goods receipt notes. It includes an approval workflow and three-way matching (PO vs. GRN vs. AP invoice).

**Key Entities:**
- `PurchaseOrder` — PO header with vendor, order date, expected delivery, status (`draft`/`approved`/`partially_received`/`closed`)
- `PurchaseOrderLine` — line item with product, quantity, unit price, tax code, and linked budget line
- `GoodsReceipt` — GRN header linked to a PO, recording actual quantities received on a date
- `GoodsReceiptLine` — per-product received quantity

**Key Operations:**
- Create purchase orders with multi-level approval routing based on amount thresholds
- Receive goods against a PO with partial receipt support
- Trigger inventory stock receipt on GRN confirmation
- Three-way match PO, GRN, and AP invoice before payment approval
- Budget availability check on PO approval (Budgeting integration)
- Generate outstanding PO and GRN reports

**Dependencies on Other Modules:** Core, Auth, GeneralLedger, AccountsPayable, Inventory, TaxManagement, Budgeting

**API Route Prefix:** `/api/v1/purchasing`

---

## 13. Sales

**Purpose:** Manages the order-to-cash cycle excluding invoicing and collection: sales orders, stock reservation, and delivery notes. It integrates with Inventory for fulfillment and with Accounts Receivable for invoice generation.

**Key Entities:**
- `SalesOrder` — order header with customer, date, status (`draft`/`confirmed`/`partially_fulfilled`/`fulfilled`)
- `SalesOrderLine` — line item with product, quantity, unit price, discount, and tax code
- `DeliveryNote` — fulfilment record linked to a sales order, recording items shipped on a specific date

**Key Operations:**
- Create and confirm sales orders with stock availability check
- Reserve stock on order confirmation
- Generate delivery notes against confirmed orders (partial delivery supported)
- Trigger inventory stock issue on delivery note confirmation
- Auto-generate AR customer invoice from a fulfilled delivery note
- Sales pipeline and order status reporting

**Dependencies on Other Modules:** Core, Auth, GeneralLedger, AccountsReceivable, Inventory, TaxManagement

**API Route Prefix:** `/api/v1/sales`

---

## 14. Tax Management (VAT/GST)

**Purpose:** Centrally manages tax rates, tax codes, and tax rule configurations, and provides tax calculation services consumed by all transactional modules. It also aggregates tax data for VAT/GST return preparation.

**Key Entities:**
- `TaxRate` — percentage rate with effective date range (supports rate changes over time)
- `TaxCode` — named code (e.g., `VAT_STD`, `VAT_ZERO`, `VAT_EXEMPT`) with linked tax rate and GL accounts for input/output tax
- `TaxReturn` — period-level VAT return aggregating total input tax, output tax, and net payable

**Key Operations:**
- Create and version-control tax rates with effective-date ranges
- Map tax codes to input tax and output tax GL accounts
- Calculate tax amount for any line item given a tax code and amount
- Handle exempt and zero-rated items correctly in tax calculations
- Aggregate input and output VAT for a return period
- Generate VAT return summary report

**Dependencies on Other Modules:** Core, Auth, GeneralLedger

**API Route Prefix:** `/api/v1/tax`

---

## 15. Budgeting

**Purpose:** Enables the creation, approval, and management of financial budgets at the account level per fiscal period. It provides variance analysis comparing actual GL activity against approved budgets.

**Key Entities:**
- `Budget` — named budget for a fiscal year with version number and status (`draft`/`approved`)
- `BudgetLine` — per-account, per-period target amount linked to a Budget and a GL Account

**Key Operations:**
- Create multiple budget versions for a fiscal year (scenario planning)
- Enter and edit budget amounts per GL account per fiscal period
- Submit budget for approval and lock approved version
- Compare actual GL balances against approved budget lines
- Compute variance (absolute and percentage) by account, department, or period
- Block purchase order approval when it would exceed budget (Purchasing integration)

**Dependencies on Other Modules:** Core, Auth, GeneralLedger, Company

**API Route Prefix:** `/api/v1/budgeting`

---

## 16. Recurring Transactions

**Purpose:** Manages templates for journal entries that should be automatically generated on a recurring schedule (daily, weekly, monthly, quarterly, annually). It eliminates manual re-entry of predictable periodic transactions such as rent, subscriptions, and depreciation provisions.

**Key Entities:**
- `RecurringTemplate` — template with a name, schedule (frequency + day-of-month), next-run date, status, and embedded journal entry lines
- `RecurringTemplateLog` — execution history recording when a template was run, what was generated, and any errors

**Key Operations:**
- Create recurring journal entry templates with cron-like schedule definitions
- Activate, pause, and deactivate templates
- Daily `ProcessRecurringTransactions` Horizon job identifies due templates and dispatches generation
- Generate a journal entry from a template and post it to the GL
- Prevent duplicate runs with idempotency checks using last-run date
- Log execution history per template

**Dependencies on Other Modules:** Core, Auth, GeneralLedger

**API Route Prefix:** `/api/v1/recurring`

---

## 17. Period Closing

**Purpose:** Orchestrates the formal month-end and year-end close process through a checklist-driven workflow. It enforces sign-off on close prerequisites before locking the fiscal period to prevent further posting.

**Key Entities:**
- `PeriodCloseChecklist` — ordered list of checklist items per fiscal period (e.g., "All bank accounts reconciled", "AP invoices approved"), each with a sign-off status and responsible user
- `FiscalPeriod` — (from Core) status transitions driven by this module: `open` → `closing` → `closed`

**Key Operations:**
- Initiate the close process for a fiscal period; generate the standard checklist
- Allow authorized users to sign off individual checklist items
- Validate that all checklist items are signed off before permitting period lock
- Lock the fiscal period (set `fiscal_periods.status = 'closed'`); fires `FiscalPeriodClosed` event
- Prevent any further journal entries to a locked period via `FiscalPeriodOpen` middleware
- Year-end close: transfer net income to retained earnings and open the new fiscal year periods

**Dependencies on Other Modules:** Core, Auth, GeneralLedger, AccountsPayable, AccountsReceivable, BankReconciliation, FixedAssets

**API Route Prefix:** `/api/v1/period-closing`

---

## 18. Reporting & KPI

**Purpose:** Generates the three core financial statements (P&L, Balance Sheet, Cash Flow Statement) and management KPI dashboards. It provides both real-time on-screen rendering and async export to PDF and Excel.

**Key Entities:**
- `SavedReport` — user-saved report configuration with parameters (date range, comparison period, filters)
- `ReportExport` — export job record with status (`queued`/`processing`/`done`/`failed`) and download URL

**Key Operations:**
- Generate Profit & Loss statement for any date range with comparative period support
- Generate Balance Sheet as at any date
- Generate Cash Flow Statement (indirect method) for any period
- Compute dashboard KPIs: revenue, expenses, net income, cash position, AR/AP outstanding, burn rate
- Export any report to PDF (DomPDF) or Excel (PhpSpreadsheet) asynchronously via `GenerateReportJob` on Horizon
- Cache KPI data in Redis with configurable TTL to reduce DB load
- Provide budget-vs-actual variance reports (Budgeting integration)

**Dependencies on Other Modules:** Core, Auth, GeneralLedger, AccountsPayable, AccountsReceivable, CashManagement, Budgeting, FixedAssets

**API Route Prefix:** `/api/v1/reporting`

---

## 19. Audit Trail

**Purpose:** Maintains an immutable, append-only log of every significant system event — including data creates, updates, deletions, status changes, and user logins. The audit trail is a compliance record and cannot be modified or deleted.

**Key Entities:**
- `AuditLog` — event record with: `event_name`, `auditable_type`, `auditable_id`, `user_id`, `company_id`, `branch_id`, `old_values` (JSONB), `new_values` (JSONB), `ip_address`, `user_agent`, `occurred_at`

**Key Operations:**
- Listen to all domain events via the universal `WriteAuditLog` listener subscribed in `EventServiceProvider`
- Dispatch `WriteAuditLogAsync` job to Horizon queue for non-blocking writes
- Store structured event payload with before/after values in JSONB columns
- Provide read-only, filterable audit log API (by user, entity, date range, event type)
- Export audit log to CSV for compliance reporting
- Retain logs indefinitely; never allow soft or hard delete via API

**Dependencies on Other Modules:** Core (all events from all modules are subscribed to here)

**API Route Prefix:** `/api/v1/audit`

---

## 20. Documents

**Purpose:** Provides a generic file attachment service that allows any entity in any module to have associated document files (PDFs, images, spreadsheets). Files are stored on S3-compatible object storage and accessed via signed URLs.

**Key Entities:**
- `Document` — polymorphic attachment with: `documentable_type`, `documentable_id`, `filename`, `mime_type`, `file_size`, `storage_path`, `description`, `uploaded_by`

**Key Operations:**
- Upload file attachments and store on S3-compatible disk via Laravel Storage
- Generate time-limited signed download URLs for secure client access
- Delete document records and underlying files
- List all documents attached to a specific entity (polymorphic query)
- Queue `ScanDocumentForMalware` job after upload for virus scanning hook
- Serve document metadata (filename, size, type, uploader) without exposing raw storage paths

**Dependencies on Other Modules:** Core, Auth

**API Route Prefix:** `/api/v1/documents`

---

## 21. Dashboard

**Purpose:** Aggregates KPIs and summary data from all financial modules into a single landing page experience, providing an executive-level view of the company's financial health in real time.

**Key Entities:** No owned entities — Dashboard is a read-only aggregation module that queries data from other modules via their services.

**Key Operations:**
- Retrieve cash position across all cash and bank accounts
- Retrieve total outstanding AR and AP balances
- Retrieve revenue and expense YTD vs. prior year
- Retrieve net income for the current and previous periods
- Retrieve top 5 customers by AR outstanding and top 5 vendors by AP outstanding
- Render KPI cards, trend charts, and period-comparison summaries
- All KPI queries are cached in Redis and refreshed on a configurable interval or on demand

**Dependencies on Other Modules:** Core, Auth, GeneralLedger, AccountsPayable, AccountsReceivable, CashManagement, BankReconciliation, Reporting

**API Route Prefix:** `/api/v1/dashboard`

---

## Module Dependency Map

```
Core ◄─────────────────────────────────────── All modules
Auth ◄─────────────────────────────────────── All modules
Company ◄──────────────────────────────────── All modules
Branch ◄───────────────────────────────────── All modules

GeneralLedger ◄────── AP, AR, Cash, Bank, FixedAssets, Inventory, Tax, Budgeting, Reporting
TaxManagement ◄────── AP, AR, Purchasing, Sales
Inventory ◄────────── Purchasing, Sales
Purchasing ──────────► AP (creates invoices), Inventory (creates GRNs), Budgeting (checks budget)
Sales ────────────────► AR (creates invoices), Inventory (issues stock)
Budgeting ◄────────── Purchasing, Reporting
PeriodClosing ───────► Core/FiscalPeriod (locks period), GL, AP, AR, BankReconciliation
AuditTrail ◄───────── All modules (via event bus)
Documents ◄────────── AP, AR, Purchasing, Sales, FixedAssets, BankReconciliation
Reporting ◄────────── GL, AP, AR, Cash, Budgeting, FixedAssets
Dashboard ◄────────── Reporting, GL, AP, AR, Cash
RecurringTransactions ─► GL (posts entries)
```
