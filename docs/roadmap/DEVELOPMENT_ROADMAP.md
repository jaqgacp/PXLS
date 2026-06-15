# Development Roadmap — Enterprise SMB ERP Accounting System

> Target: NetSuite-inspired simplicity for small and medium businesses
> Stack: Laravel 12 · React 18 + TypeScript · Vite · Tailwind CSS · PostgreSQL 16
> Team size: 2–3 developers
> Total timeline: 32 weeks (8 months)

---

## Technology Stack

| Layer | Technology | Notes |
|---|---|---|
| Backend framework | Laravel 12 | PHP 8.3+, API-first |
| Authentication | Laravel Sanctum | SPA token auth |
| Authorization | Spatie Laravel Permission | RBAC with company scoping |
| Database | PostgreSQL 16 | JSONB, partitioning, RLS |
| Queue / Jobs | Laravel Horizon + Redis | Recurring transactions, notifications |
| Cache | Redis 7 | Session, query cache |
| File storage | AWS S3 / compatible | Document attachments, exports |
| Frontend framework | React 18 | Functional components, hooks |
| Language | TypeScript 5 | Strict mode |
| Build tool | Vite 5 | Fast HMR, ESM |
| UI framework | Tailwind CSS 3 + shadcn/ui | Utility-first, headless components |
| State management | Zustand + React Query (TanStack) | Server state + client state |
| Forms | React Hook Form + Zod | Validated forms |
| Tables | TanStack Table v8 | Virtualized, sortable, filterable |
| Charts | Recharts | Dashboard KPI charts |
| PDF export | Laravel DomPDF / Browsershot | Server-side PDF generation |
| Excel export | Laravel Excel (PhpSpreadsheet) | Reports + imports |
| Testing (backend) | Pest PHP | Unit, feature, integration |
| Testing (frontend) | Vitest + Testing Library | Component + integration tests |
| CI/CD | GitHub Actions | Lint, test, deploy |

---

## Infrastructure Recommendations

### Hosting Options

| Option | Recommendation | Use Case |
|---|---|---|
| Laravel Forge + DigitalOcean | Primary recommendation | Cost-effective, full control, easy SSL |
| Laravel Vapor (AWS Lambda) | Scale-out option | Zero-downtime deploys, auto-scaling |
| AWS RDS PostgreSQL 16 | Database | Multi-AZ, automated backups, point-in-time restore |
| AWS ElastiCache Redis | Queue + cache | Cluster mode for HA |
| AWS S3 | File storage | Document attachments, exports, backups |
| AWS CloudFront | CDN | Static assets, React bundle |
| Cloudflare | DNS + WAF | DDoS protection, HTTPS termination |

### Minimum Production Architecture

```
[Cloudflare] → [Load Balancer] → [App Server x2 (Forge)]
                                        ↓
                              [RDS PostgreSQL (Multi-AZ)]
                              [ElastiCache Redis Cluster]
                              [S3 Bucket (private)]

[Horizon Worker Servers x1-2] → same Redis + RDS
```

### Environment Configuration

```
APP_ENV=production
APP_DEBUG=false
DB_CONNECTION=pgsql
DB_HOST=<rds-endpoint>
DB_PORT=5432
CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis
FILESYSTEM_DISK=s3
AWS_BUCKET=<bucket>
REDIS_HOST=<elasticache-endpoint>
```

---

## Non-Functional Requirements

| Requirement | Target | Implementation |
|---|---|---|
| Report response time | < 2 seconds (P95) | DB indexes, Redis query cache (TTL 60s), eager loading |
| API response time | < 300ms (P95) for CRUD | N+1 elimination, select only needed columns |
| Uptime | 99.9% (< 8.7h/year downtime) | Multi-AZ DB, 2 app servers, health checks |
| Data security | HTTPS everywhere | Cloudflare + ACM certs, HSTS headers |
| Row-level security | Company data isolation | PostgreSQL RLS + Laravel middleware |
| Soft deletes | Audit-friendly deletes | `deleted_at` on all master data tables |
| Audit trail | 100% write coverage | Laravel Observer + audit_logs table |
| Password policy | Min 10 chars, complexity | Laravel validation rules |
| Session security | Idle timeout 30 min | Sanctum token expiry + frontend timer |
| File uploads | Max 20MB, type whitelist | Laravel validation + S3 presigned URLs |
| Concurrent users | 50+ per company | Stateless API, Redis sessions |
| Backup | Daily automated + WAL | RDS automated backups, 30-day retention |
| GDPR/Data privacy | Data export + deletion | Artisan commands for data subject requests |

---

## Phase 1 — Foundation (Weeks 1–4)

**Goal:** Establish a working skeleton that developers can build modules on top of. At the end of this phase, a user can log in, switch between companies, manage users and roles, and view a basic Chart of Accounts.

**Duration:** 4 weeks

---

### Backend Tasks (Laravel)

#### Week 1: Project Setup

- Initialize Laravel 12 project with PostgreSQL driver (`pgsql`)
- Configure `.env` for local development (PostgreSQL, Redis)
- Install and configure packages:
  - `laravel/sanctum` — SPA authentication
  - `spatie/laravel-permission` — RBAC
  - `spatie/laravel-activitylog` — audit hooks
  - `laravel/horizon` — queue dashboard
  - `barryvdh/laravel-cors` — CORS for React SPA
- Create base migration structure and run initial migrations for `users`, `companies`, `branches`
- Set up GitHub Actions CI: lint (Pint), test (Pest), static analysis (PHPStan level 8)

#### Week 2: Multi-tenancy & Auth

- Implement `CompanyMiddleware` — resolves current company from header (`X-Company-ID`) and stores in request context
- Implement `BranchMiddleware` — optional branch scoping
- Create `TenantScope` global scope injectable into Eloquent models (auto-apply `company_id` filter)
- Set PostgreSQL session variable `app.current_company_id` for RLS via `DB::statement`
- Auth endpoints: `POST /auth/login`, `POST /auth/logout`, `GET /auth/user`, `POST /auth/refresh`
- Company switching endpoint: `POST /auth/switch-company`
- Password reset flow (email + token)
- Seed: system roles (`super-admin`, `company-admin`, `accountant`, `viewer`)

#### Week 3: RBAC & User Management

- Company-scoped role assignment via `user_company_access`
- User CRUD: `GET /users`, `POST /users`, `PUT /users/{id}`, `DELETE /users/{id}`
- Role/permission management: `GET /roles`, `POST /roles`, assign permissions to roles
- Branch CRUD: `GET /branches`, `POST /branches`, `PUT /branches/{id}`
- Company settings endpoint: `GET /companies/{id}/settings`, `PATCH /companies/{id}/settings`
- `UserResource`, `CompanyResource`, `BranchResource` API resources (JSON:API-lite)
- Currencies & exchange rates: `GET /currencies`, `GET /exchange-rates`, `POST /exchange-rates`
- Fiscal periods CRUD: `GET /fiscal-periods`, `POST /fiscal-periods`, `PATCH /fiscal-periods/{id}/close`

#### Week 4: Chart of Accounts

- `AccountType` and `Account` models with hierarchical parent/child relationships
- Tree-flattening service: `ChartOfAccountsService::getTree(companyId)` — recursive CTE query
- Account CRUD: `GET /accounts`, `POST /accounts`, `PUT /accounts/{id}`, `DELETE /accounts/{id}`
- Account code auto-generation based on type prefix
- `CostCenter` CRUD
- Import stub: CSV upload endpoint (parsing only, no import logic yet)
- Validation: prevent deleting accounts with transactions, prevent posting to non-leaf accounts

---

### Frontend Tasks (React + TypeScript)

#### Week 1: Project Scaffold

- Initialize Vite + React 18 + TypeScript project
- Install dependencies: Tailwind CSS, shadcn/ui, React Router v6, TanStack Query, TanStack Table, Zustand, React Hook Form, Zod, Axios, Recharts
- Configure TypeScript strict mode and ESLint + Prettier
- Set up path aliases (`@/components`, `@/pages`, `@/hooks`, `@/store`, `@/api`)
- Configure Axios instance with base URL, auth token interceptor, and 401 redirect
- Create `AuthContext` and `useAuth` hook

#### Week 2: Layout Shell (NetSuite-inspired)

- **AppLayout** component:
  - Dark sidebar (bg-slate-900) with collapsible nav groups
  - Module navigation icons with tooltips (collapsed state)
  - Top bar: company switcher dropdown, user avatar, notification bell, breadcrumbs
  - Main content area with responsive padding
- **Sidebar navigation structure:**
  ```
  Dashboard
  ─ General Ledger
    ├ Journal Entries
    ├ Trial Balance
    └ Account Ledger
  ─ Accounts Payable
    ├ Vendors
    ├ Invoices
    └ Payments
  ─ Accounts Receivable
    ├ Customers
    ├ Invoices
    └ Payments
  ─ Cash & Bank
  ─ Purchasing
  ─ Sales
  ─ Inventory
  ─ Fixed Assets
  ─ Budgeting
  ─ Reports
  ─ Settings
  ```
- Breadcrumb component (auto-generated from route config)
- Loading skeleton components
- Error boundary with friendly error page

#### Week 3: Auth & Company UI

- Login page: email/password form, validation, error messages, "Remember me"
- Company switcher modal: searchable list of companies the user has access to
- Forgot password / reset password pages
- 2-column auth layout (left: branding, right: form)
- Protected route wrapper using `AuthContext`

#### Week 4: User & Settings UI

- **Users page:** paginated table with search, columns: name, email, role, status, last login, actions
- **User detail / edit modal:** name, email, role assignment per company, branch restriction
- **Roles page:** list roles, show permissions per role, assign permissions via checkbox grid grouped by module
- **Company settings page:** tabs for General, Currencies, Fiscal Periods, Branches
- **Chart of Accounts page:**
  - Expandable tree view of account hierarchy
  - Quick-add account inline
  - Full edit modal with all fields
  - Toggle active/inactive
  - Type and normal balance badges

---

### Deliverable / Demo Milestone

A working SPA where a user can:
1. Log in with email and password
2. Switch between multiple companies
3. View and manage users and role assignments
4. Configure company settings (currency, fiscal year, timezone)
5. Create and browse a hierarchical Chart of Accounts
6. Manage branches and cost centers

No financial transactions yet — this is the skeleton that all other modules sit on.

---

## Phase 2 — Core Accounting / General Ledger (Weeks 5–8)

**Goal:** The heart of the ERP — double-entry bookkeeping is enforced. Users can manually create journal entries, view a trial balance, and browse account ledgers. Fiscal period management with locking is operational.

**Duration:** 4 weeks

---

### Backend Tasks (Laravel)

#### Week 5: Journal Entry Engine

- `JournalEntry` and `JournalEntryLine` models
- `JournalEntryService::post()`:
  - Validate `SUM(debit_base) == SUM(credit_base)` — throw `ImbalancedEntryException` if not equal
  - Check fiscal period is `open`
  - Check all accounts `allow_direct_posting = TRUE`
  - Set status to `posted`, stamp `posted_at` and `posted_by`
  - Auto-assign `fiscal_period_id` based on `entry_date`
- `JournalEntryService::reverse()` — create a reversing entry with negated amounts, link via `reversal_of_id`
- Auto-numbering: `JE-{YYYY}-{NNNNNN}` with company-scoped sequence
- Journal entry CRUD: `GET /journal-entries`, `POST /journal-entries`, `PUT /journal-entries/{id}`, `POST /journal-entries/{id}/post`, `POST /journal-entries/{id}/reverse`

#### Week 6: GL Reports

- **Trial Balance query:**
  ```sql
  SELECT a.code, a.name, at.name AS type,
         SUM(jel.debit_base) AS total_debit,
         SUM(jel.credit_base) AS total_credit
  FROM journal_entry_lines jel
  JOIN journal_entries je ON je.id = jel.journal_entry_id
  JOIN accounts a ON a.id = jel.account_id
  JOIN account_types at ON at.id = a.account_type_id
  WHERE je.company_id = ? AND je.status = 'posted'
    AND je.entry_date BETWEEN ? AND ?
  GROUP BY a.id, a.code, a.name, at.name
  ORDER BY a.code;
  ```
- `GET /reports/trial-balance?from=&to=&branch_id=`
- **Account Ledger:** transactions for a single account with running balance
- `GET /reports/account-ledger?account_id=&from=&to=`
- Fiscal period management: `POST /fiscal-periods/{id}/close`, `POST /fiscal-periods/{id}/lock`, `POST /fiscal-periods/{id}/reopen`
- Period posting guard middleware (applied on all journal entry creation)

#### Week 7: Currency & Exchange Rates

- Exchange rate lookup service: `ExchangeRateService::getRate(from, to, date, companyId)`
- Multi-currency journal entry: accept `currency_code` + `exchange_rate` per line, compute `debit_base`/`credit_base` in company base currency
- Unrealized FX gain/loss calculation stub (for Phase 6)
- `GET /exchange-rates?from=USD&to=EUR&date=2025-01-15` — returns nearest rate on or before date

#### Week 8: Recurring GL Templates

- `RecurringTemplate` model + seed template structure for GL type
- `ProcessRecurringTransactionsJob` — dispatched daily via scheduler
- Basic recurring log entries on success/failure

---

### Frontend Tasks (React + TypeScript)

#### Week 5: Journal Entry Form

- **Journal Entry List page:**
  - Filterable by date range, status, branch, source module
  - Status badge (draft = yellow, posted = green, reversed = red)
  - Columns: entry number, date, description, reference, status, debit total, credit total, actions
- **Journal Entry Form (create/edit):**
  - Header: date picker, description, reference, branch selector
  - Line item table:
    - Account selector (searchable dropdown with code + name)
    - Description per line
    - Currency + exchange rate (shows base amount preview)
    - Debit / Credit input (mutual exclusion enforced)
    - Cost center selector (optional)
    - Add/remove lines
  - Running totals footer: Total Debit vs Total Credit with balance indicator (green if balanced, red if not)
  - Draft save and Post buttons (Post triggers confirmation modal)

#### Week 6: GL Reports UI

- **Trial Balance page:**
  - Date range picker (from/to)
  - Branch filter
  - Table: Account Code | Account Name | Type | Debit | Credit | Net Balance
  - Subtotals per account type
  - Export to Excel button (calls `/reports/trial-balance?format=xlsx`)
- **Account Ledger page:**
  - Account selector at top
  - Date range picker
  - Table: Date | Entry# | Description | Reference | Debit | Credit | Running Balance
  - Opening balance row at top

#### Week 7: Fiscal Periods & Currencies UI

- **Fiscal Periods page** (under Settings):
  - List with status badges, dates
  - "Close period" button with confirmation (warns if draft entries exist)
  - "Lock period" (irreversible, requires admin role)
  - "Reopen" for closed (not locked) periods
- **Currencies & Exchange Rates page:**
  - Currency list (global, read-only codes)
  - Exchange rate table: sortable by date, filterable by currency pair
  - Add rate form: from currency, to currency, rate, effective date
  - "Use today's rate" button (future: integrate with open exchange rates API)

#### Week 8: GL Refinements

- Journal entry detail view (read-only) with print layout
- Reversal entry flow (confirm modal → creates reversal → links back)
- Keyboard shortcuts: `Ctrl+Enter` to post, `Tab` to move between line columns
- Journal entry line copy (duplicate a line for fast data entry)

---

### Deliverable / Demo Milestone

A working double-entry accounting engine where a user can:
1. Create multi-line journal entries (draft and post)
2. Create multi-currency journal entries with exchange rates
3. View a real-time Trial Balance for any date range
4. Browse an account ledger with running balance
5. Close and lock fiscal periods
6. View recurring template logs

---

## Phase 3 — Accounts Payable & Accounts Receivable (Weeks 9–14)

**Goal:** Complete the AP and AR cycles. Vendors, customers, invoices, payments, and payment allocation are all functional. AR and AP aging reports are available.

**Duration:** 6 weeks

---

### Backend Tasks (Laravel)

#### Week 9: Vendor & Customer Master

- `Vendor` and `Customer` models with JSONB contact/address serialization
- CRUD: `GET|POST /vendors`, `GET|PUT|DELETE /vendors/{id}`
- CRUD: `GET|POST /customers`, `GET|PUT|DELETE /customers/{id}`
- Searchable by name (full-text), code, tax number
- Auto-generate vendor/customer code: `VEN-000001`, `CUS-000001`
- `payment_terms` → `due_date` calculator: `InvoiceDueDateService::calculate(invoiceDate, paymentTermsDays)`
- Soft delete with check: block deletion if open invoices exist

#### Week 10: Vendor Invoices

- `VendorInvoice` + `VendorInvoiceLine` models
- `VendorInvoiceService::approve()` — creates GL journal entry:
  ```
  DR: Expense/Asset account per line
  DR: Tax Input account (if tax)
  CR: Accounts Payable control account
  ```
- Auto-numbering: `APINV-{YYYY}-{NNNNNN}`
- `VendorInvoiceService::cancel()` — only draft or approved with no payments
- API: `GET|POST /vendor-invoices`, `GET|PUT /vendor-invoices/{id}`, `POST /vendor-invoices/{id}/approve`, `POST /vendor-invoices/{id}/cancel`
- `paid_amount` computed from `vendor_payment_allocations` sum (triggered on allocation)

#### Week 11: Vendor Payments & Allocation

- `VendorPayment` model
- `VendorPaymentService::post()`:
  ```
  DR: Accounts Payable control account
  CR: Bank/Cash account
  ```
- Payment allocation: `POST /vendor-payments/{id}/allocate` body: `[{invoice_id, amount}]`
- Allocation validator: allocated amount ≤ invoice outstanding balance
- Invoice status auto-update after allocation: partially_paid / paid
- API: `GET|POST /vendor-payments`, `GET /vendor-payments/{id}`, `POST /vendor-payments/{id}/post`

#### Week 12: Customer Invoices & Payments

- Mirror of AP but for AR:
- `CustomerInvoice` GL posting:
  ```
  DR: Accounts Receivable control account
  CR: Revenue/Sales account per line
  CR: Tax Output account (if tax)
  ```
- `CustomerPayment` GL posting:
  ```
  DR: Bank/Cash account
  CR: Accounts Receivable control account
  ```
- API: full CRUD + approve + cancel + allocate
- Overdue status: scheduled job marks invoices past due date as `overdue`

#### Week 13: Bank Accounts & Cash Management

- `BankAccount` model with linked GL account
- `GET|POST /bank-accounts`, `GET|PUT /bank-accounts/{id}`
- Bank transaction auto-creation on vendor/customer payment posting
- `CashTransfer` service — transfers between bank accounts with GL posting:
  ```
  DR: Target bank account
  CR: Source bank account
  ```
- `GET /bank-accounts/{id}/balance` — returns current balance from GL

#### Week 14: AP & AR Reports

- **AP Aging report:**
  ```sql
  SELECT v.name, vi.invoice_number, vi.due_date,
         vi.total_amount - vi.paid_amount AS outstanding,
         CURRENT_DATE - vi.due_date AS days_overdue
  FROM vendor_invoices vi
  JOIN vendors v ON v.id = vi.vendor_id
  WHERE vi.company_id = ? AND vi.status IN ('approved','partially_paid')
  ORDER BY days_overdue DESC;
  ```
  Bucketed: Current, 1-30, 31-60, 61-90, 90+ days
- `GET /reports/ap-aging?as_of_date=&vendor_id=`
- `GET /reports/ar-aging?as_of_date=&customer_id=`
- `GET /reports/vendor-statement?vendor_id=&from=&to=` — all transactions for a vendor
- `GET /reports/customer-statement?customer_id=&from=&to=`

---

### Frontend Tasks (React + TypeScript)

#### Week 9: Vendor & Customer Master UI

- **Vendors list page:** searchable table, filter by status, quick view slide-over panel
- **Vendor form (create/edit):** tabs — General (code, name, tax number, currency, terms), Contact, Address, Bank Details, Notes
- **Customers list page:** same pattern as vendors
- **Customer form:** same tab structure (without bank details)
- Inline search component reused across AP/AR (async search with debounce)

#### Week 10: Vendor Invoice UI

- **Vendor Invoice list:** columns: invoice#, vendor, invoice date, due date, total, paid, outstanding, status badge
- Filter bar: vendor, date range, status, overdue only toggle
- **Vendor Invoice form:**
  - Header: vendor selector, invoice date, vendor's invoice number, due date (auto-calc from terms), currency
  - Line items table: account, description, qty, unit price, tax code, tax amount, line total
  - Totals sidebar: subtotal, tax, total, outstanding (after payments)
  - Action buttons: Save Draft, Approve (with confirmation)
- **Vendor Invoice detail view:** read-only with payment history at bottom

#### Week 11: Vendor Payment & Allocation UI

- **Vendor Payments list:** payment#, vendor, date, amount, method, status
- **Vendor Payment form:** vendor selector, payment date, amount, bank account, method, reference
- **Payment Allocation modal** (accessible from payment detail):
  - Table of open invoices for the vendor
  - Allocate amount input per invoice (auto-fills outstanding, allows partial)
  - Running total: payment amount vs. allocated — must balance before saving

#### Week 12: Customer Invoice & Payment UI

- Mirror of vendor invoice/payment UI:
- Customer invoice list + form (sales account instead of expense account)
- Customer payment list + form
- Customer payment allocation modal

#### Week 13: Bank Accounts UI

- **Bank Accounts page** (under Cash & Bank):
  - Card layout per account: name, bank, account number, currency, current balance
  - Add/edit account form
- **Cash Transfer form:** from account → to account, amount, date, reference
- **Bank Account Transactions view:** mini ledger with reconciled status indicator

#### Week 14: AP/AR Aging Reports UI

- **AP Aging page:**
  - As-of-date picker
  - Summary buckets at top (card widgets): Current, 1-30d, 31-60d, 61-90d, 90d+
  - Expandable table by vendor → invoices
  - Export to Excel / PDF
- **AR Aging page:** same structure
- **Vendor Statement page:** vendor selector + date range → printable statement
- **Customer Statement page:** same

---

### Deliverable / Demo Milestone

End-to-end AP and AR cycles:
1. Create a vendor and a customer
2. Enter a vendor invoice → approve → GL journal auto-created
3. Make a vendor payment → allocate to invoice → invoice marked as paid
4. Create a customer invoice → approve → receive customer payment → allocate
5. View AP and AR aging reports with aging buckets
6. View vendor and customer statements

---

## Phase 4 — Procurement, Sales & Inventory (Weeks 15–20)

**Goal:** Purchase-to-pay and order-to-cash workflows with 3-way match. Inventory is tracked with average cost method. Tax (VAT) is wired up.

**Duration:** 6 weeks

---

### Backend Tasks (Laravel)

#### Week 15: Items, Warehouses & Tax Codes

- `Item`, `ItemCategory`, `Warehouse`, `ItemStock` models
- Item CRUD with type differentiation (stock/service/expense account routing)
- `ItemStockService::getAvailableQty(itemId, warehouseId)`
- `TaxCode` model + `TaxCalculationService::calculate(amount, taxCodeId, isInclusive)`
- Tax code assignment to items
- `GET|POST /items`, `GET|PUT /items/{id}`
- `GET|POST /warehouses`
- `GET|POST /tax-codes`

#### Week 16: Purchase Orders

- `PurchaseOrder` + `PurchaseOrderLine` models
- `PurchaseOrderService::approve()` — updates `item_stock.quantity_on_order` for stock items
- `PurchaseOrderService::cancel()` — reverses quantity_on_order
- Auto-number: `PO-{YYYY}-{NNNNNN}`
- API: full CRUD + approve + cancel + `GET /purchase-orders/{id}/lines`
- Receiving status auto-update from goods receipt totals

#### Week 17: Goods Receipts & 3-Way Match

- `GoodsReceipt` + `GoodsReceiptLine` models
- `GoodsReceiptService::receive()`:
  - Validate quantity received ≤ remaining on PO line
  - Update `item_stock.quantity_on_hand` via `StockMovementService`
  - Update `item_stock.quantity_on_order` (reduce)
  - Update `item_stock.average_cost` using weighted average:
    `new_avg = (current_qty * current_avg + received_qty * unit_cost) / (current_qty + received_qty)`
  - Create stock movement record
  - GL posting (inventory account debit, GR/IR clearing credit)
- 3-Way match on vendor invoice approval:
  - Validate: GR exists for PO, quantities match
  - Apply GR/IR clearing: DR Accounts Payable, CR GR/IR Clearing (net to zero)
- API: `GET|POST /goods-receipts`, `POST /goods-receipts/{id}/receive`

#### Week 18: Sales Orders & Delivery

- `SalesOrder` + `SalesOrderLine` models
- `SalesOrderService::confirm()` — reserves inventory: `quantity_reserved += quantity`
- `DeliveryOrder` + `DeliveryOrderLine` models
- `DeliveryOrderService::ship()`:
  - Validate quantity_delivered ≤ reserved
  - Reduce `quantity_on_hand` and `quantity_reserved`
  - Create stock movement (sale type)
  - GL posting: DR COGS, CR Inventory (at average cost)
  - Update sales order line `quantity_fulfilled`
  - Update sales order status
- `SalesOrderService::createInvoice()` — generates `CustomerInvoice` from confirmed sales order
- API: full CRUD + confirm + cancel + create-invoice for SO; full CRUD + ship for DO

#### Week 19: Stock Adjustments & Movements

- Stock adjustment endpoint: `POST /stock-adjustments` — adjust quantity_on_hand with reason
- GL posting for adjustments: DR/CR Inventory Adjustment account
- `GET /items/{id}/stock` — stock levels per warehouse
- `GET /items/{id}/movements` — full movement history
- Low-stock notification: threshold-based (stored in item, checked on each movement)
- Reorder report: `GET /reports/reorder-report` — items below reorder point

#### Week 20: VAT / Tax Engine

- Tax transaction auto-creation on invoice posting
- Input vs output tax tracking
- `GET /reports/tax-summary?period=&from=&to=` — VAT report: output tax, input tax, net payable
- Withholding tax support (flag on vendor)
- Tax code on PO/SO lines flows through to invoice

---

### Frontend Tasks (React + TypeScript)

#### Week 15: Item & Warehouse Management UI

- **Items list:** code, name, type badge, category, UOM, cost, price, stock status
- **Item form:** tabs — General, Accounts (purchase/sales/inventory/COGS), Pricing, Tax, Notes
- **Item Categories:** tree view with drag-and-drop reorder (react-dnd or dnd-kit)
- **Warehouses page:** card list with address, branch, active toggle
- **Tax Codes page:** table with rate, type, GL account, inclusive flag

#### Week 16: Purchase Order UI

- **PO list:** columns: PO#, vendor, date, delivery, total, status
- Filter: vendor, status, date range
- **PO form:**
  - Header: vendor, PO date, expected delivery, currency, branch
  - Line table: item selector (with auto-fill description, price from item master), qty, unit price, tax code, amount
  - Totals: subtotal, tax, total
  - Actions: Save Draft, Submit for Approval, Approve (role-gated)
- **PO detail:** shows PO lines with quantity received per line, receiving status

#### Week 17: Goods Receipt UI

- **Goods Receipt form:**
  - Link to PO (autocomplete)
  - Warehouse selector
  - Line table auto-populated from PO remaining quantities
  - Editable `quantity_received` per line (≤ outstanding)
  - Unit cost (editable for cost adjustment)
  - Notes
- **PO detail → Goods Receipts tab:** list of all receipts against the PO
- **3-Way match indicator** on vendor invoice: PO reference + GR reference chips, match status badge

#### Week 18: Sales Order UI

- **SO list:** columns: SO#, customer, date, delivery, total, status, fulfillment %
- **SO form:** same structure as PO form but with discount % per line
- **Delivery Order form:** linked to SO, warehouse selector, lines auto-populated, quantity_delivered input
- **"Create Invoice" button** on confirmed SO → pre-fills customer invoice with SO data

#### Week 19: Inventory Reports UI

- **Stock Position page:** warehouse matrix — items × warehouses with on-hand, on-order, reserved
- **Stock Movement History page:** filterable by item, warehouse, movement type, date range
- **Reorder Report:** items below threshold with suggested reorder quantity

#### Week 20: Tax UI

- **Tax Summary Report page:** period selector, breakdown of output vs input tax, net liability
- Tax code applied inline in PO/SO/invoice line tables (dropdown with rate preview)

---

### Deliverable / Demo Milestone

Complete purchase-to-pay and order-to-cash with inventory:
1. Create purchase order → approve → receive goods → match to vendor invoice → post payment
2. Create sales order → confirm → deliver goods → generate customer invoice → receive payment
3. Average cost inventory updated automatically on each movement
4. VAT report showing input/output tax for the period
5. Reorder report for low stock items

---

## Phase 5 — Advanced Modules (Weeks 21–26)

**Goal:** Fixed assets, bank reconciliation, budgeting, recurring transactions, audit trail viewer, and document attachments. These are "completing the picture" features that differentiate from basic accounting software.

**Duration:** 6 weeks

---

### Backend Tasks (Laravel)

#### Week 21: Fixed Assets

- `AssetCategory`, `FixedAsset`, `AssetDepreciationSchedule` models
- `DepreciationScheduleService::generate(fixedAssetId)` — creates full depreciation schedule on asset creation:
  - Straight-line: `(cost - residual) / useful_life_months` per month
  - Declining balance: `book_value * (2 / useful_life_months)` per month
- `DepreciationPostingJob` — monthly scheduled job:
  - Find schedules with `period_date <= current_month` and `posted = FALSE`
  - Create GL journal entry:
    ```
    DR: Depreciation Expense account
    CR: Accumulated Depreciation account
    ```
  - Mark schedule row as posted
- Disposal: `FixedAssetService::dispose(assetId, disposalDate, disposalAmount)`:
  - GL posting: remove asset cost, remove accumulated depreciation, recognize gain/loss
- API: full CRUD, `POST /fixed-assets/{id}/dispose`, `GET /fixed-assets/{id}/schedule`

#### Week 22: Bank Reconciliation

- `BankReconciliation` model
- CSV import: `POST /bank-accounts/{id}/import-statement` — parse CSV, create `bank_transactions` (unmatched)
- Matching service: `BankReconciliationService::autoMatch(reconciliationId)` — match by amount + date proximity
- Manual match: `POST /bank-reconciliations/{id}/match` body: `{bank_transaction_id, journal_entry_line_id}`
- Unmatch: `DELETE /bank-reconciliations/{id}/match/{id}`
- Complete reconciliation: validate `statement_ending_balance == reconciled_balance`, set status to `completed`
- API: `GET|POST /bank-reconciliations`, `POST /bank-reconciliations/{id}/complete`

#### Week 23: Budgeting

- `Budget` + `BudgetLine` models
- Budget vs actual: query compares `budget_lines` monthly columns vs actual GL movements for the period
- `GET /reports/budget-vs-actual?budget_id=&period=` — returns month-by-month comparison with variance
- Budget approval workflow: draft → approved
- API: full CRUD for budgets and budget lines

#### Week 24: Recurring Transactions

- Extend `RecurringTemplate` for AP/AR types (template_data stores invoice draft)
- `ProcessRecurringTransactionsJob` — runs via `Schedule::job()->daily()`:
  - Find templates where `next_run_date <= today` and `is_active = TRUE`
  - Deserialize `template_data`, create draft invoice/journal entry
  - Advance `next_run_date` based on frequency
  - Log result to `recurring_logs`
- Dead-letter queue for failed recurring jobs with Horizon retry
- API: full CRUD + manual trigger: `POST /recurring-templates/{id}/run-now`

#### Week 25: Audit Trail

- Laravel Observer on all major models: captures `created`, `updated`, `deleted`, `restored` events
- Writes to `audit_logs` asynchronously (queued job) to avoid performance hit on write path
- Login/logout events logged via Sanctum event listener
- `GET /audit-logs?entity_type=&entity_id=&user_id=&from=&to=&action=`
- Paginated response with filtering

#### Week 26: Document Attachments

- S3 upload: `POST /documents/upload` — returns presigned PUT URL, client uploads directly to S3
- `POST /documents/attach` body: `{entity_type, entity_id, file_name, file_path, file_size, mime_type}`
- `GET /documents?entity_type=&entity_id=` — list attachments
- `DELETE /documents/{id}` — removes S3 object and DB record
- `GET /documents/{id}/download` — returns presigned GET URL (expiry 15 minutes)
- File type whitelist: PDF, PNG, JPG, XLSX, DOCX (configurable in company settings)

---

### Frontend Tasks (React + TypeScript)

#### Week 21: Fixed Assets UI

- **Fixed Assets list:** code, name, category, purchase date, cost, book value, status badge
- **Asset form:** category (auto-fills depreciation method, useful life), purchase details, vendor link
- **Asset detail page:** tabs — Details, Depreciation Schedule, Attachments
- **Depreciation Schedule tab:** table of all periods — period, depreciation amount, accumulated, book value, journal entry link, posted badge
- **Disposal modal:** date, disposal amount, calculate and preview gain/loss

#### Week 22: Bank Reconciliation UI

- **Bank Reconciliation page:**
  - Start new reconciliation: bank account selector, statement date, statement ending balance
  - CSV import dropzone with column mapping UI
  - Two-panel layout:
    - Left: unmatched bank statement lines
    - Right: unmatched GL transactions (from journal entries)
  - Drag-to-match or click-to-select both sides and "Match Selected" button
  - Matched pairs shown at bottom with "Unmatch" option
  - Difference indicator at top: statement balance vs book balance
  - "Complete Reconciliation" button (only enabled when difference = 0)

#### Week 23: Budgeting UI

- **Budgets list:** name, fiscal period, status, total budget amount
- **Budget form:** header (name, fiscal period), line item table:
  - Account selector
  - Branch (optional)
  - 12 monthly amount columns (jan–dec)
  - Total column (computed)
  - Bulk fill: "Spread total evenly across months"
- **Budget vs Actual report:**
  - Budget and period selector
  - Table: Account | Budget | Actual | Variance | Variance %
  - Color-coded variance (green = favorable, red = unfavorable)
  - Chart: grouped bar chart per month

#### Week 24: Recurring Transactions UI

- **Recurring Templates list:** name, module, frequency, next run, status
- **Recurring Template form:** name, module (GL/AP/AR), frequency, start/end dates, active toggle
  - GL type: embedded journal entry line editor (same component as Phase 2)
  - AP type: embedded invoice line editor
- **Recurring Logs tab:** run history table — date, status, link to generated record, error message
- "Run Now" button for manual trigger

#### Week 25: Audit Trail UI

- **Audit Log page:** filterable by entity type, entity ID, user, action, date range
- Expandable row shows old_values vs new_values diff (JSON diff viewer — color-coded)
- User search for filtering
- Export to CSV

#### Week 26: Document Attachments UI

- **Attachment panel component** (reusable, embedded in every detail view):
  - Drag-and-drop upload zone
  - File list: name, size, upload date, uploaded by
  - Download button (opens presigned URL in new tab)
  - Delete (with confirmation)
- Show attachment count badge on list views (e.g., vendor invoice rows)

---

### Deliverable / Demo Milestone

1. Fixed assets: create, generate depreciation schedule, post monthly depreciation, dispose asset
2. Bank reconciliation: import CSV statement, match transactions, complete reconciliation
3. Budget vs actual report showing variance per account per month
4. Recurring invoice auto-generated by scheduler (demo via "Run Now")
5. Audit log showing who changed what, with before/after diff
6. File attachments on vendor invoices and journal entries

---

## Phase 6 — Reporting, Polish & Launch (Weeks 27–32)

**Goal:** Production-ready financial statements, KPI dashboard, import tools, performance hardening, security review, and user acceptance testing.

**Duration:** 6 weeks

---

### Backend Tasks (Laravel)

#### Week 27: Financial Statements

- **Balance Sheet:**
  - Query: sum of all asset, liability, equity accounts as of a given date
  - Comparative columns (current period vs prior period)
  - `GET /reports/balance-sheet?as_of=&comparative=true`
- **Income Statement (P&L):**
  - Revenue - COGS = Gross Profit
  - Gross Profit - Operating Expenses = Operating Income
  - Operating Income +/- Other Income/Expenses = Net Income
  - Date range with comparative period
  - `GET /reports/income-statement?from=&to=&compare_from=&compare_to=`
- **Cash Flow Statement:**
  - Indirect method: start from net income, adjust for non-cash items
  - `GET /reports/cash-flow?from=&to=`
- All financial statements accept `branch_id` for branch-level P&L
- All statements cached in Redis (TTL 5 minutes, invalidated on new journal entry posting)

#### Week 28: KPI Dashboard & Advanced Reports

- Dashboard API: `GET /dashboard/summary` returns:
  ```json
  {
    "cash_position": 125000.00,
    "ar_total": 45000.00,
    "ap_total": 32000.00,
    "revenue_mtd": 89000.00,
    "expenses_mtd": 67000.00,
    "net_income_mtd": 22000.00,
    "overdue_invoices_count": 5,
    "overdue_amount": 12000.00
  }
  ```
- Revenue trend: `GET /dashboard/revenue-trend?months=12` — monthly breakdown
- AP/AR aging summary: `GET /dashboard/aging-summary`
- Cash flow chart: `GET /dashboard/cash-flow-chart?months=6`

#### Week 29: Import Tools & Period Closing

- **Chart of Accounts CSV import:**
  - Parse and validate CSV: code, name, type, parent code, currency, posting flag
  - Preview mode: show rows with validation errors before committing
  - `POST /import/chart-of-accounts` (multipart/form-data)
- **Opening balances import:** upload trial balance CSV to seed initial journal entry
- **Period closing workflow:**
  - Checklist: unposted journals, outstanding allocations, depreciation run, tax filed
  - `POST /fiscal-periods/{id}/run-closing-checklist` — returns checklist status
  - Close period (auto-creates retained earnings entry if IS accounts need rolling to equity)
- **Exchange rate import:** CSV upload for bulk historical rates

#### Week 30: PDF & Excel Export

- Install `barryvdh/laravel-dompdf` and `maatwebsite/laravel-excel`
- PDF templates (Blade views):
  - Customer invoice / vendor invoice printable layout
  - Vendor/customer statement
  - Balance Sheet, Income Statement (formatted)
  - Delivery order / purchase order
- `GET /vendor-invoices/{id}/pdf` — streamed PDF response
- `GET /reports/balance-sheet?format=pdf`
- Excel export for all reports + list views (paginated → full export)
- Background export for large datasets: `POST /exports` → job queued → email user download link

#### Week 31: Performance Tuning

- Query profiling: identify slow queries using Laravel Debugbar (dev) + pg_stat_statements (prod)
- Add missing composite indexes discovered during load testing
- Redis query caching for:
  - Trial balance (5-minute TTL)
  - Account tree (10-minute TTL, busted on CoA changes)
  - Dashboard KPIs (2-minute TTL)
- N+1 elimination audit across all API endpoints (use `barryvdh/laravel-debugbar` + Clockwork)
- Eager load relationships in all list endpoints
- Pagination: enforce max 100 per page, use cursor pagination for large datasets
- PostgreSQL `EXPLAIN ANALYZE` on all report queries, optimize with partial indexes

#### Week 32: Security Hardening & UAT

- Security checklist:
  - Mass assignment protection (`$fillable` on all models)
  - Authorization policies for every endpoint (no route without policy check)
  - SQL injection: ensure all queries use bindings (audit for raw queries)
  - CSRF protection (Sanctum SPA default)
  - Rate limiting: login (10/min), API (300/min per user)
  - File upload validation: MIME sniffing, size limits, no executable extensions
  - S3 bucket policy: private + server-side encryption
  - Secrets rotation procedure documented
- Penetration test scope: auth bypass, IDOR, privilege escalation, SSRF
- UAT with sample company data (seeder with realistic transactions)
- Fix UAT-discovered bugs
- Write deployment runbook and rollback procedure

---

### Frontend Tasks (React + TypeScript)

#### Week 27: Financial Statements UI

- **Balance Sheet page:**
  - As-of date picker + comparison date (optional)
  - Hierarchical display: Assets → Current Assets → individual accounts
  - Subtotals per section (Current Assets, Non-Current Assets, etc.)
  - Comparative columns with variance % column
  - Expand/collapse sections
- **Income Statement page:**
  - Date range picker + comparison range
  - Sections: Revenue, COGS, Gross Profit, Operating Expenses, Operating Income, Other, Net Income
  - Subtotals and totals highlighted
- **Cash Flow Statement page:**
  - Operating / Investing / Financing activities sections
  - Net change in cash + ending cash reconciliation

#### Week 28: KPI Dashboard

- **Dashboard page** (default landing after login):
  - Top row: 4 KPI cards — Cash Position, Total AR, Total AP, Net Income MTD
  - Revenue vs Expenses: dual-line chart (last 12 months, Recharts)
  - AR Aging donut chart: current / 1-30 / 31-60 / 60+ buckets
  - AP Aging donut chart: same
  - Recent transactions table: last 10 journal entries with links
  - Overdue invoices widget: count + total amount, link to filtered AR list
  - All data refreshes on page focus (TanStack Query `refetchOnWindowFocus`)

#### Week 29: Import Tools & Period Closing UI

- **CSV Import page** (under Settings → Import):
  - Drag-and-drop file upload
  - Column mapper: preview CSV columns, map to system fields
  - Validation preview: rows with errors highlighted in red, valid rows in green
  - Import summary: X created, Y skipped, Z errors
- **Period Closing Wizard:**
  - Step 1: Review checklist (green checkmarks for each prerequisite)
  - Step 2: Run depreciation (button + confirmation)
  - Step 3: Post closing entries (retained earnings roll-up)
  - Step 4: Lock period (final confirmation with typed "CLOSE" input)

#### Week 30: Export & Print UI

- Export dropdown on every list view and report: "Export to Excel", "Export to PDF"
- Loading state during PDF generation (progress indicator)
- Print stylesheet for invoice/statement views (`@media print`)
- Background export notification: toast → "Your export is ready, click to download"
- Invoice print layout: company logo, invoice details, line items, totals, payment terms footer

#### Week 31: Performance & UX Polish

- Virtual scrolling for large tables (TanStack Virtual for 1000+ row lists)
- Skeleton loading states on all pages (no blank flashes)
- Optimistic updates on toggle actions (active/inactive) for snappier UX
- Global search: `Ctrl+K` opens command palette — search accounts, vendors, customers, invoices
- Keyboard navigation throughout (focus trapping in modals, `Esc` to close)
- Form auto-save (draft) every 30 seconds to prevent data loss
- Unsaved changes warning on route navigation

#### Week 32: UAT Fixes & Launch Prep

- UAT feedback incorporation
- Empty state illustrations for all list views (no data yet)
- Onboarding wizard for new companies: 5-step setup (currency → CoA template → opening balances → first vendor → first customer)
- Help tooltip system (question mark icons on complex fields)
- Error message review: all API errors have user-friendly messages (no raw stack traces)
- Browser compatibility test: Chrome, Firefox, Safari, Edge
- Mobile responsiveness audit (tablet minimum — list views should work on iPad)

---

### Deliverable / Demo Milestone

Full production-ready system:
1. Balance Sheet, Income Statement, Cash Flow Statement with comparatives
2. KPI dashboard with live charts and AR/AP aging donut charts
3. CSV import for Chart of Accounts and opening balances
4. PDF export for all reports and source documents
5. Period closing wizard with checklist
6. Global search command palette (`Ctrl+K`)
7. System passes security review and UAT sign-off
8. Deployment runbook complete

---

## Milestone Summary

| Phase | Weeks | Key Deliverable |
|---|---|---|
| 1 — Foundation | 1–4 | Login, multi-company, CoA, user management |
| 2 — Core Accounting | 5–8 | Double-entry journal entries, trial balance, fiscal periods |
| 3 — AP & AR | 9–14 | Full vendor/customer invoice and payment cycle, aging reports |
| 4 — Procurement & Sales | 15–20 | PO → GR → Invoice 3-way match, SO → DO → Invoice, inventory |
| 5 — Advanced Modules | 21–26 | Fixed assets, bank reconciliation, budgets, recurring, audit trail |
| 6 — Reporting & Polish | 27–32 | Financial statements, KPI dashboard, PDF/Excel export, UAT |

---

## Team Allocation Guidance

| Developer | Primary Focus | Secondary |
|---|---|---|
| Dev 1 (Backend Lead) | Laravel services, GL engine, AP/AR, reports | DB schema, queue jobs |
| Dev 2 (Frontend Lead) | React components, layout, forms, reports UI | API integration layer |
| Dev 3 (Full-stack) | Shared modules (inventory, assets, reconciliation) | Testing, DevOps, CI |

**Note for 2-developer teams:** Phases 3–5 may each extend by 1–2 weeks. Prioritize AP/AR (Phase 3) over inventory (Phase 4) if scope must be cut for an MVP.

---

## Definition of Done (Per Feature)

- [ ] Backend: service class with unit tests (Pest), API endpoint with feature test covering happy path and validation errors
- [ ] Backend: authorization policy applied, tested for unauthorized access
- [ ] Frontend: component renders with mock data (Vitest + Testing Library)
- [ ] Frontend: loading and error states handled
- [ ] GL impact: journal entry auto-posted and verified against expected debit/credit accounts
- [ ] No N+1 queries (verified via Clockwork in dev)
- [ ] Code reviewed by at least one other developer
- [ ] Feature demoed and signed off in weekly demo session
