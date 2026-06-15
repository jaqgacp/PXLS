# PXL ERP — Enterprise Architecture Blueprint
**Version 1.0**

---

## 1. Vision

PXL is a modern cloud-based ERP platform designed for Small and Medium Businesses (SMBs).

The objective is to provide a simplified alternative to NetSuite, SAP Business One, Dynamics 365 Business Central, and Odoo while maintaining enterprise-grade accounting accuracy, auditability, scalability, and operational control.

PXL will serve as the single source of truth for:

- Accounting
- Sales
- Purchasing
- Inventory
- Treasury
- Fixed Assets
- Payroll
- Reporting
- Compliance
- Business Operations

---

## 2. Product Philosophy

### Core Principles

**Accounting First**
Every business transaction must result in valid accounting entries.

```
Business Transaction
        ↓
Posting Engine
        ↓
Journal Entry
        ↓
General Ledger
        ↓
Financial Statements
```

**Single Source of Truth**
Never store balances. Store transactions. Generate balances.

```
Journal Entries
      ↓
GL Entries
      ↓
Trial Balance
      ↓
Balance Sheet
      ↓
Income Statement
```

**Complete Auditability**
Every action must be traceable. Track:
- Creation
- Modification
- Approval
- Reversal
- Deletion attempts
- User activity

**Modular ERP**
Each module must operate independently but integrate through the Accounting Engine.

---

## 3. Target Market

**Primary**
- Trading Companies
- Distribution Companies
- Retail Businesses
- Wholesale Businesses
- Service Businesses

**Secondary**
- Manufacturing
- Construction
- Logistics
- Multi-Branch Enterprises

---

## 4. System Architecture

```
┌───────────────────────────────────────────────┐
│                Presentation Layer             │
│                                               │
│  React + TypeScript                           │
│  Dashboards                                   │
│  Reports                                      │
│  Forms                                        │
│  Mobile Responsive UI                         │
└───────────────────────────────────────────────┘

                     ↓

┌───────────────────────────────────────────────┐
│                API Layer                      │
│                                               │
│  Laravel REST API                             │
│  Authentication                               │
│  Authorization                                │
│  Validation                                   │
│  API Gateway                                  │
└───────────────────────────────────────────────┘

                     ↓

┌───────────────────────────────────────────────┐
│             Application Services              │
│                                               │
│  Workflow Engine                              │
│  Approval Engine                              │
│  Notification Engine                          │
│  Document Engine                              │
│  Reporting Engine                             │
└───────────────────────────────────────────────┘

                     ↓

┌───────────────────────────────────────────────┐
│                 Domain Layer                  │
│                                               │
│  Accounting      Sales        Purchasing      │
│  Inventory       Treasury     Payroll         │
│  Assets          Projects                     │
└───────────────────────────────────────────────┘

                     ↓

┌───────────────────────────────────────────────┐
│             Financial Engine                  │
│                                               │
│  Posting Engine                               │
│  Tax Engine                                   │
│  Currency Engine                              │
│  Cost Allocation Engine                       │
│  Consolidation Engine                         │
└───────────────────────────────────────────────┘

                     ↓

┌───────────────────────────────────────────────┐
│                 PostgreSQL                    │
└───────────────────────────────────────────────┘
```

---

## 5. Navigation Architecture

### Dashboard
- Executive Dashboard
- Financial Dashboard
- Sales Dashboard
- Purchasing Dashboard
- Inventory Dashboard
- Treasury Dashboard

### Master Data

**Accounting**
- Chart of Accounts
- Account Groups
- Tax Codes / Tax Rates
- Fiscal Years / Fiscal Periods

**Parties**
- Customers
- Vendors
- Employees
- Contacts

**Inventory**
- Items / Categories
- Units of Measure
- Warehouses / Locations

**Organization**
- Companies / Branches
- Departments / Cost Centers / Profit Centers
- Projects

### Transactions

**Sales**
- Quotations → Sales Orders → Deliveries → Sales Invoices → Receipts → Credit Notes

**Purchasing**
- Purchase Requests → Purchase Orders → Goods Receipts → Vendor Bills → Vendor Payments → Debit Notes

**Inventory**
- Receipts / Issues / Transfers / Adjustments / Assemblies / Stock Counts

**Treasury**
- Collections / Disbursements / Bank Transfers / Petty Cash / Bank Reconciliation

**Payroll**
- Timesheets / Leave Requests / Payroll Runs / Payslips

**Fixed Assets**
- Acquisition / Depreciation / Revaluation / Disposal

**Financials**
- Journal Entries / Recurring Journals / Accruals / Allocations / Revaluations
- Deferred Revenue / Prepaid Expenses
- General Ledger / Trial Balance / Period Closing

### Reports

**Financial**
- Trial Balance
- Balance Sheet
- Income Statement
- Cash Flow Statement
- GL Report

**Sales**
- Sales Analysis / Customer Aging / Customer Ledger

**Purchasing**
- Vendor Aging / Vendor Ledger

**Inventory**
- Stock Ledger / Inventory Valuation / Inventory Aging

**Treasury**
- Bank Register / Cash Position

**Compliance**
- Tax Reports / Audit Reports

### Compliance
- Audit Trail / Approvals / Tax Filing / Compliance Reports

### Tools
- Data Import / Data Export / Workflow Designer / API Monitor / Job Queue / System Logs

### Setup
- Company Settings / Branch Settings / Number Series
- Approval Matrix / Roles / Permissions / Integrations

---

## 6. Core ERP Modules

### Module 1: Organization Management
Manage organizational structure.

```
companies
branches
departments
cost_centers
profit_centers
projects
```

### Module 2: User & Security
Identity and access control.

```
users
roles
permissions
user_roles
role_permissions
```

Capabilities: MFA · Password Policies · Session Control · IP Restrictions

### Module 3: Accounting Core
Financial foundation.

**Chart of Accounts**
```
accounts
account_groups
account_types
```

**Journal Management**
```
journal_entries
journal_lines
```

**General Ledger**
```
gl_entries
```

**Fiscal Management**
```
fiscal_years
fiscal_periods
period_locks
```

### Module 4: Accounts Receivable
```
customers
sales_orders
sales_invoices
customer_payments
credit_notes
```

### Module 5: Accounts Payable
```
vendors
purchase_orders
vendor_bills
vendor_payments
debit_notes
```

### Module 6: Inventory Management
```
items
warehouses
locations
inventory_transactions
inventory_ledger
```

Costing Methods: FIFO · Weighted Average · Standard Cost

### Module 7: Treasury
```
bank_accounts
bank_transactions
bank_reconciliations
cash_receipts
cash_disbursements
```

### Module 8: Fixed Assets
```
assets
asset_categories
asset_books
depreciation_entries
```

### Module 9: Payroll
```
employees
timesheets
payroll_runs
payslips
```

### Module 10: Tax Engine
```
tax_codes
tax_rates
tax_transactions
```

Support: VAT · Withholding Tax · Sales Tax

### Module 11: Workflow Engine
```
workflow_definitions
workflow_instances
workflow_steps
approvals
```

### Module 12: Reporting Engine
Supports: Operational Reports · Financial Reports · Dashboards · BI Analytics

---

## 7. Posting Engine Architecture

> The most important ERP component. No module writes directly to General Ledger.

| Transaction | Debit | Credit |
|---|---|---|
| Sales Invoice | AR | Sales Revenue, VAT Payable |
| Customer Payment | Cash | AR |
| Vendor Bill | Expense, VAT Input | AP |
| Vendor Payment | AP | Cash |
| Inventory Receipt | Inventory | GRNI |
| Inventory Sale | COGS | Inventory |

---

## 8. Multi-Tenant Architecture

Every table contains:

```sql
tenant_id
company_id
branch_id
```

---

## 9. Audit Architecture

Every transaction stores:

```sql
created_by      created_at
updated_by      updated_at
approved_by     approved_at
posted_by       posted_at
reversed_by     reversed_at
```

---

## 10. Integration Architecture

| Phase | Integrations |
|---|---|
| Phase 1 | Email, PDF Export |
| Phase 2 | Banking APIs, Payment Gateways |
| Phase 3 | Shopify, WooCommerce, POS |
| Phase 4 | EDI, Government Tax APIs |

---

## 11. Technical Stack

| Layer | Technology |
|---|---|
| Frontend | React, TypeScript, TanStack Query, Zustand, Tailwind CSS |
| Backend | Laravel 12, PHP 8.4+ |
| Database | PostgreSQL |
| Infrastructure | Docker, Redis, Queues, Object Storage |
| Auth | Laravel Sanctum, MFA, RBAC |

---

## 12. Development Roadmap

| Phase | Scope |
|---|---|
| Phase 1 | Users, Companies, Branches, Chart of Accounts, Journal Entries, GL, Financial Statements |
| Phase 2 | Customers, Vendors, Invoices, Bills, Payments (AR/AP) |
| Phase 3 | Inventory Ledger, Warehouses, Purchasing, Sales Orders |
| Phase 4 | Banking, Reconciliation, Cash Management (Treasury) |
| Phase 5 | Fixed Assets, Payroll |
| Phase 6 | Workflow Engine, Approval Engine, Consolidation, Multi-Tenant SaaS |
