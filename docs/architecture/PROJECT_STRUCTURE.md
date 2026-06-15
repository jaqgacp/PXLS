# Project Structure — Laravel 12 + React + TypeScript + PostgreSQL ERP

> **Stack:** Laravel 12 (API-only) · React 18 + TypeScript · PostgreSQL 16 · Redis · Laravel Horizon

---

## Root Directory Tree

```
pxls-erp/
├── app/
│   ├── Console/                          # Artisan commands (scheduled jobs, maintenance)
│   │   └── Commands/
│   ├── Exceptions/                       # Global exception handler & custom exception classes
│   ├── Http/
│   │   ├── Kernel.php                    # HTTP kernel; registers global & route middleware stacks
│   │   └── Middleware/
│   │       ├── Authenticate.php          # Sanctum token guard
│   │       ├── CheckCompanyAccess.php    # Validates company_id scope on every request
│   │       ├── CheckBranchAccess.php     # Validates branch_id scope when branch context is required
│   │       ├── FiscalPeriodOpen.php      # Rejects writes when target fiscal period is closed
│   │       ├── SetTenantContext.php      # Injects company_id / branch_id into request context
│   │       └── ForceJsonResponse.php     # Ensures Accept: application/json on all API routes
│   └── Modules/                          # Domain modules — one subdirectory per bounded context
│       │
│       ├── Core/                         # Shared kernel: base classes, traits, value objects
│       │   ├── Controllers/
│       │   │   └── BaseApiController.php # Shared response helpers used by all module controllers
│       │   ├── Services/
│       │   │   └── TenantContextService.php # Resolves and caches company_id / branch_id for request
│       │   ├── Repositories/
│       │   │   └── BaseRepository.php    # Eloquent base with company_id / branch_id global scopes
│       │   ├── Models/
│       │   │   ├── BaseModel.php         # Adds SoftDeletes, UUIDs, and tenant scopes to all models
│       │   │   └── FiscalPeriod.php      # Fiscal period definition with open/closed status
│       │   ├── Requests/
│       │   │   └── PaginationRequest.php # Common pagination, sorting, filtering query params
│       │   ├── Resources/
│       │   │   └── BaseResource.php      # JSON:API-compatible resource wrapper
│       │   ├── Events/                   # Core-level domain events (e.g., TenantResolved)
│       │   ├── Listeners/                # Core-level event listeners
│       │   ├── Jobs/                     # Core infrastructure jobs (cleanup, archival)
│       │   └── DTOs/
│       │       └── PaginationDTO.php     # Immutable data transfer object for pagination params
│       │
│       ├── Auth/                         # Authentication, RBAC, sessions, API tokens
│       │   ├── Controllers/
│       │   │   ├── AuthController.php    # Login, logout, refresh token endpoints
│       │   │   ├── MeController.php      # Current-user profile and permissions
│       │   │   └── TokenController.php   # Personal access token management
│       │   ├── Services/
│       │   │   ├── AuthService.php       # Credentials validation and Sanctum token issuance
│       │   │   └── PermissionService.php # Wraps spatie/laravel-permission; role/permission sync
│       │   ├── Repositories/
│       │   │   └── UserRepository.php    # User queries with role and permission eager loading
│       │   ├── Models/
│       │   │   ├── User.php              # Authenticatable; HasRoles from spatie/laravel-permission
│       │   │   └── PersonalAccessToken.php # Extended Sanctum token with last_used_at tracking
│       │   ├── Requests/
│       │   │   ├── LoginRequest.php      # Validates email + password + optional 2FA code
│       │   │   └── AssignRoleRequest.php # Validates role assignment payload
│       │   ├── Resources/
│       │   │   ├── UserResource.php      # User JSON representation with roles & permissions
│       │   │   └── TokenResource.php     # Token response with expiry metadata
│       │   ├── Events/
│       │   │   └── UserLoggedIn.php      # Fires on successful login for audit trail
│       │   ├── Listeners/
│       │   │   └── LogLoginActivity.php  # Writes login event to audit_logs
│       │   ├── Jobs/
│       │   │   └── RevokeExpiredTokens.php # Scheduled job to clean up stale tokens
│       │   └── DTOs/
│       │       └── LoginDTO.php          # Typed transfer object for login credentials
│       │
│       ├── Company/                      # Top-level tenant entity (company master data)
│       │   ├── Controllers/
│       │   │   └── CompanyController.php # CRUD for company settings, logo, fiscal year config
│       │   ├── Services/
│       │   │   └── CompanyService.php    # Handles company provisioning and settings updates
│       │   ├── Repositories/
│       │   │   └── CompanyRepository.php # Company queries and settings retrieval
│       │   ├── Models/
│       │   │   └── Company.php           # Company entity with fiscal year, currency, timezone
│       │   ├── Requests/
│       │   │   └── UpdateCompanyRequest.php # Validates company settings update payload
│       │   ├── Resources/
│       │   │   └── CompanyResource.php   # Company JSON representation
│       │   ├── Events/
│       │   ├── Listeners/
│       │   ├── Jobs/
│       │   └── DTOs/
│       │       └── CompanySettingsDTO.php # Immutable company settings value object
│       │
│       ├── Branch/                       # Branch / cost-center management under a company
│       │   ├── Controllers/
│       │   │   └── BranchController.php  # CRUD for branches; activate/deactivate endpoints
│       │   ├── Services/
│       │   │   └── BranchService.php     # Branch creation with default chart-of-accounts seeding
│       │   ├── Repositories/
│       │   │   └── BranchRepository.php  # Branch queries filtered by company_id
│       │   ├── Models/
│       │   │   └── Branch.php            # Branch entity with address, contact, and currency fields
│       │   ├── Requests/
│       │   │   └── CreateBranchRequest.php
│       │   ├── Resources/
│       │   │   └── BranchResource.php
│       │   ├── Events/
│       │   ├── Listeners/
│       │   ├── Jobs/
│       │   └── DTOs/
│       │       └── BranchDTO.php
│       │
│       ├── GeneralLedger/                # Chart of accounts, journal entries, trial balance
│       │   ├── Controllers/
│       │   │   ├── AccountController.php       # Chart-of-accounts CRUD and account hierarchy
│       │   │   ├── JournalEntryController.php  # Create, post, reverse journal entries
│       │   │   └── TrialBalanceController.php  # Trial balance and account-balance reporting
│       │   ├── Services/
│       │   │   ├── AccountService.php          # Account creation with hierarchy validation
│       │   │   ├── JournalEntryService.php     # Double-entry validation and posting logic
│       │   │   └── TrialBalanceService.php     # Computes balances using window functions
│       │   ├── Repositories/
│       │   │   ├── AccountRepository.php       # Account queries with recursive CTE for hierarchy
│       │   │   └── JournalEntryRepository.php  # Journal queries with line-item aggregation
│       │   ├── Models/
│       │   │   ├── Account.php                 # Chart-of-accounts node (type, normal balance)
│       │   │   ├── JournalEntry.php            # Header record (date, reference, status, period)
│       │   │   └── JournalEntryLine.php        # Debit/credit line linked to an Account
│       │   ├── Requests/
│       │   │   ├── CreateAccountRequest.php
│       │   │   └── PostJournalEntryRequest.php
│       │   ├── Resources/
│       │   │   ├── AccountResource.php
│       │   │   └── JournalEntryResource.php
│       │   ├── Events/
│       │   │   ├── JournalEntryPosted.php      # Fires when a journal entry transitions to posted
│       │   │   └── JournalEntryReversed.php    # Fires when a reversal entry is created
│       │   ├── Listeners/
│       │   │   └── UpdateAccountBalance.php    # Maintains denormalized running balance cache
│       │   ├── Jobs/
│       │   │   └── RecalculateAccountBalances.php # Background job for balance reconciliation
│       │   └── DTOs/
│       │       ├── JournalEntryDTO.php
│       │       └── JournalEntryLineDTO.php
│       │
│       ├── AccountsPayable/              # Vendor invoices, credit notes, payment scheduling
│       │   ├── Controllers/
│       │   │   ├── VendorController.php        # Vendor master CRUD
│       │   │   ├── VendorInvoiceController.php # AP invoice lifecycle (draft→approved→paid)
│       │   │   └── PaymentController.php       # Record and allocate vendor payments
│       │   ├── Services/
│       │   │   ├── VendorService.php
│       │   │   ├── InvoiceService.php          # Invoice approval workflow and GL posting
│       │   │   └── PaymentAllocationService.php # Matches payments to outstanding invoices
│       │   ├── Repositories/
│       │   │   ├── VendorRepository.php
│       │   │   └── VendorInvoiceRepository.php
│       │   ├── Models/
│       │   │   ├── Vendor.php
│       │   │   ├── VendorInvoice.php
│       │   │   ├── VendorInvoiceLine.php
│       │   │   └── VendorPayment.php
│       │   ├── Requests/
│       │   ├── Resources/
│       │   ├── Events/
│       │   │   └── VendorInvoiceApproved.php
│       │   ├── Listeners/
│       │   │   └── PostAPJournalEntry.php      # Posts GL entry when invoice is approved
│       │   ├── Jobs/
│       │   │   └── SendPaymentDueReminder.php
│       │   └── DTOs/
│       │
│       ├── AccountsReceivable/           # Customer invoices, receipts, aging analysis
│       │   ├── Controllers/
│       │   │   ├── CustomerController.php
│       │   │   ├── CustomerInvoiceController.php
│       │   │   └── ReceiptController.php
│       │   ├── Services/
│       │   │   ├── CustomerService.php
│       │   │   ├── InvoiceService.php
│       │   │   └── AgingService.php            # Computes AR aging buckets (30/60/90/120+ days)
│       │   ├── Repositories/
│       │   ├── Models/
│       │   │   ├── Customer.php
│       │   │   ├── CustomerInvoice.php
│       │   │   ├── CustomerInvoiceLine.php
│       │   │   └── CustomerReceipt.php
│       │   ├── Requests/
│       │   ├── Resources/
│       │   ├── Events/
│       │   │   └── CustomerInvoicePosted.php
│       │   ├── Listeners/
│       │   │   └── PostARJournalEntry.php
│       │   ├── Jobs/
│       │   │   └── SendInvoiceReminderEmail.php
│       │   └── DTOs/
│       │
│       ├── CashManagement/               # Cash accounts, petty cash, internal transfers
│       │   ├── Controllers/
│       │   │   ├── CashAccountController.php
│       │   │   └── CashTransactionController.php
│       │   ├── Services/
│       │   │   └── CashTransactionService.php  # Validates sufficient balance before recording
│       │   ├── Repositories/
│       │   ├── Models/
│       │   │   ├── CashAccount.php
│       │   │   └── CashTransaction.php
│       │   ├── Requests/
│       │   ├── Resources/
│       │   ├── Events/
│       │   ├── Listeners/
│       │   ├── Jobs/
│       │   └── DTOs/
│       │
│       ├── BankReconciliation/           # Bank statement import, matching, reconciliation
│       │   ├── Controllers/
│       │   │   ├── BankAccountController.php
│       │   │   ├── BankStatementController.php # Import CSV/OFX bank statements
│       │   │   └── ReconciliationController.php # Match bank lines to GL entries
│       │   ├── Services/
│       │   │   ├── StatementImportService.php  # Parses and persists bank statement lines
│       │   │   └── ReconciliationService.php   # Auto-matching algorithm and manual match API
│       │   ├── Repositories/
│       │   ├── Models/
│       │   │   ├── BankAccount.php
│       │   │   ├── BankStatement.php
│       │   │   ├── BankStatementLine.php
│       │   │   └── BankReconciliation.php
│       │   ├── Requests/
│       │   ├── Resources/
│       │   ├── Events/
│       │   ├── Listeners/
│       │   ├── Jobs/
│       │   │   └── AutoMatchBankLines.php      # Background auto-matching job via Horizon
│       │   └── DTOs/
│       │
│       ├── FixedAssets/                  # Asset register, depreciation, disposal
│       │   ├── Controllers/
│       │   │   ├── AssetController.php
│       │   │   ├── DepreciationController.php  # Run/preview depreciation for a period
│       │   │   └── DisposalController.php
│       │   ├── Services/
│       │   │   ├── AssetService.php
│       │   │   ├── DepreciationService.php     # Straight-line and declining-balance methods
│       │   │   └── DisposalService.php         # Calculates gain/loss and posts GL entry
│       │   ├── Repositories/
│       │   ├── Models/
│       │   │   ├── Asset.php
│       │   │   ├── AssetCategory.php
│       │   │   ├── DepreciationEntry.php
│       │   │   └── AssetDisposal.php
│       │   ├── Requests/
│       │   ├── Resources/
│       │   ├── Events/
│       │   │   └── AssetDepreciationRun.php
│       │   ├── Listeners/
│       │   │   └── PostDepreciationJournalEntry.php
│       │   ├── Jobs/
│       │   │   └── RunMonthlyDepreciation.php
│       │   └── DTOs/
│       │
│       ├── Inventory/                    # Products, stock movements, valuation
│       │   ├── Controllers/
│       │   │   ├── ProductController.php
│       │   │   ├── WarehouseController.php
│       │   │   └── StockMovementController.php
│       │   ├── Services/
│       │   │   ├── ProductService.php
│       │   │   ├── StockMovementService.php    # Issues, receipts, adjustments with FIFO/WAC
│       │   │   └── ValuationService.php        # Stock valuation using chosen cost method
│       │   ├── Repositories/
│       │   ├── Models/
│       │   │   ├── Product.php
│       │   │   ├── Warehouse.php
│       │   │   ├── StockBalance.php            # Denormalized current stock per product/warehouse
│       │   │   └── StockMovement.php
│       │   ├── Requests/
│       │   ├── Resources/
│       │   ├── Events/
│       │   │   └── StockMovementRecorded.php
│       │   ├── Listeners/
│       │   │   └── UpdateStockBalance.php
│       │   ├── Jobs/
│       │   └── DTOs/
│       │
│       ├── Purchasing/                   # Purchase orders, GRN, three-way matching
│       │   ├── Controllers/
│       │   │   ├── PurchaseOrderController.php
│       │   │   └── GoodsReceiptController.php
│       │   ├── Services/
│       │   │   ├── PurchaseOrderService.php    # PO approval workflow and budget check
│       │   │   └── GoodsReceiptService.php     # GRN creation with inventory update
│       │   ├── Repositories/
│       │   ├── Models/
│       │   │   ├── PurchaseOrder.php
│       │   │   ├── PurchaseOrderLine.php
│       │   │   └── GoodsReceipt.php
│       │   ├── Requests/
│       │   ├── Resources/
│       │   ├── Events/
│       │   │   └── PurchaseOrderApproved.php
│       │   ├── Listeners/
│       │   ├── Jobs/
│       │   └── DTOs/
│       │
│       ├── Sales/                        # Sales orders, delivery notes, invoicing
│       │   ├── Controllers/
│       │   │   ├── SalesOrderController.php
│       │   │   └── DeliveryNoteController.php
│       │   ├── Services/
│       │   │   ├── SalesOrderService.php       # Order validation, stock reservation, pricing
│       │   │   └── DeliveryNoteService.php     # Fulfillment recording with inventory deduction
│       │   ├── Repositories/
│       │   ├── Models/
│       │   │   ├── SalesOrder.php
│       │   │   ├── SalesOrderLine.php
│       │   │   └── DeliveryNote.php
│       │   ├── Requests/
│       │   ├── Resources/
│       │   ├── Events/
│       │   │   └── SalesOrderFulfilled.php
│       │   ├── Listeners/
│       │   ├── Jobs/
│       │   └── DTOs/
│       │
│       ├── TaxManagement/                # VAT/GST rates, tax codes, return preparation
│       │   ├── Controllers/
│       │   │   ├── TaxRateController.php
│       │   │   ├── TaxCodeController.php
│       │   │   └── TaxReturnController.php     # Generate and submit VAT return summaries
│       │   ├── Services/
│       │   │   ├── TaxCalculationService.php   # Applies rates; handles exempt/zero-rated
│       │   │   └── TaxReturnService.php        # Aggregates input/output VAT for return period
│       │   ├── Repositories/
│       │   ├── Models/
│       │   │   ├── TaxRate.php
│       │   │   ├── TaxCode.php
│       │   │   └── TaxReturn.php
│       │   ├── Requests/
│       │   ├── Resources/
│       │   ├── Events/
│       │   ├── Listeners/
│       │   ├── Jobs/
│       │   └── DTOs/
│       │
│       ├── Budgeting/                    # Budget setup, line-item budgets, variance analysis
│       │   ├── Controllers/
│       │   │   ├── BudgetController.php
│       │   │   └── BudgetLineController.php
│       │   ├── Services/
│       │   │   ├── BudgetService.php           # Budget version management and approval
│       │   │   └── VarianceService.php         # Actual vs budget variance computation
│       │   ├── Repositories/
│       │   ├── Models/
│       │   │   ├── Budget.php
│       │   │   └── BudgetLine.php              # Per-account, per-period budget amount
│       │   ├── Requests/
│       │   ├── Resources/
│       │   ├── Events/
│       │   ├── Listeners/
│       │   ├── Jobs/
│       │   └── DTOs/
│       │
│       ├── RecurringTransactions/        # Templates for auto-generated periodic journals
│       │   ├── Controllers/
│       │   │   └── RecurringTransactionController.php
│       │   ├── Services/
│       │   │   └── RecurringTransactionService.php # Generates journal entries from templates
│       │   ├── Repositories/
│       │   ├── Models/
│       │   │   ├── RecurringTemplate.php       # Schedule, frequency, next-run date
│       │   │   └── RecurringTemplateLog.php    # Execution history per template
│       │   ├── Requests/
│       │   ├── Resources/
│       │   ├── Events/
│       │   │   └── RecurringEntryGenerated.php
│       │   ├── Listeners/
│       │   ├── Jobs/
│       │   │   └── ProcessRecurringTransactions.php # Daily Horizon job to fire due templates
│       │   └── DTOs/
│       │
│       ├── PeriodClosing/                # Month-end / year-end close workflow
│       │   ├── Controllers/
│       │   │   └── PeriodClosingController.php # Initiate, validate, and finalize period close
│       │   ├── Services/
│       │   │   ├── PeriodClosingService.php    # Orchestrates close checklist and status updates
│       │   │   └── YearEndService.php          # Retained-earnings transfer, new-year setup
│       │   ├── Repositories/
│       │   ├── Models/
│       │   │   └── PeriodCloseChecklist.php    # Tracks checklist items and sign-off status
│       │   ├── Requests/
│       │   ├── Resources/
│       │   ├── Events/
│       │   │   └── FiscalPeriodClosed.php      # Triggers lock on fiscal_periods row
│       │   ├── Listeners/
│       │   │   └── LockFiscalPeriod.php        # Sets fiscal_periods.status = 'closed'
│       │   ├── Jobs/
│       │   └── DTOs/
│       │
│       ├── Reporting/                    # Financial statements, KPIs, export engine
│       │   ├── Controllers/
│       │   │   ├── FinancialStatementController.php # P&L, Balance Sheet, Cash Flow
│       │   │   ├── KpiController.php                # Dashboard KPI endpoint
│       │   │   └── ExportController.php             # PDF and Excel export trigger
│       │   ├── Services/
│       │   │   ├── ProfitLossService.php
│       │   │   ├── BalanceSheetService.php
│       │   │   ├── CashFlowService.php
│       │   │   ├── KpiService.php               # Aggregates KPIs with caching
│       │   │   └── ExportService.php            # Delegates to DomPDF / PhpSpreadsheet
│       │   ├── Repositories/
│       │   ├── Models/                          # Report configuration/saved report models
│       │   ├── Requests/
│       │   ├── Resources/
│       │   ├── Events/
│       │   ├── Listeners/
│       │   ├── Jobs/
│       │   │   └── GenerateReportJob.php        # Async report generation dispatched to Horizon
│       │   └── DTOs/
│       │       └── ReportParametersDTO.php
│       │
│       ├── AuditTrail/                   # Immutable event log for all data changes
│       │   ├── Controllers/
│       │   │   └── AuditLogController.php       # Read-only audit log query and export
│       │   ├── Services/
│       │   │   └── AuditLogService.php          # Writes structured events to audit_logs table
│       │   ├── Repositories/
│       │   │   └── AuditLogRepository.php
│       │   ├── Models/
│       │   │   └── AuditLog.php                 # event_name, payload (JSONB), user, IP, timestamp
│       │   ├── Requests/
│       │   │   └── AuditLogFilterRequest.php
│       │   ├── Resources/
│       │   │   └── AuditLogResource.php
│       │   ├── Events/
│       │   ├── Listeners/
│       │   │   └── WriteAuditLog.php            # Universal listener subscribed to all domain events
│       │   ├── Jobs/
│       │   │   └── WriteAuditLogAsync.php       # Dispatched to Horizon queue for non-blocking write
│       │   └── DTOs/
│       │       └── AuditEventDTO.php
│       │
│       └── Documents/                    # File attachments linked to any entity
│           ├── Controllers/
│           │   └── DocumentController.php       # Upload, download, delete document attachments
│           ├── Services/
│           │   └── DocumentService.php          # Stores via Laravel Storage; generates signed URLs
│           ├── Repositories/
│           │   └── DocumentRepository.php
│           ├── Models/
│           │   └── Document.php                 # Polymorphic: documentable_type + documentable_id
│           ├── Requests/
│           │   └── UploadDocumentRequest.php
│           ├── Resources/
│           │   └── DocumentResource.php
│           ├── Events/
│           │   └── DocumentUploaded.php
│           ├── Listeners/
│           ├── Jobs/
│           │   └── ScanDocumentForMalware.php   # Queued post-upload virus scan hook
│           └── DTOs/
│               └── DocumentDTO.php
│
├── bootstrap/
│   ├── app.php                           # Application bootstrap; registers module service providers
│   └── providers.php                     # Auto-discovered and manually registered providers list
│
├── config/
│   ├── app.php                           # Application name, timezone (UTC), locale
│   ├── auth.php                          # Sanctum guard configuration
│   ├── cors.php                          # CORS allowed origins for React SPA
│   ├── database.php                      # PostgreSQL DSN and connection pool settings
│   ├── horizon.php                       # Horizon queue worker pools and retry configuration
│   ├── permission.php                    # spatie/laravel-permission cache and table settings
│   ├── filesystems.php                   # S3-compatible storage disk configuration
│   ├── erp.php                           # Custom ERP config: default currency, fiscal year start
│   └── sanctum.php                       # Stateful domains and token expiry settings
│
├── database/
│   ├── migrations/
│   │   ├── core/
│   │   │   ├── 0001_01_01_000000_create_companies_table.php
│   │   │   ├── 0001_01_01_000001_create_branches_table.php
│   │   │   └── 0001_01_01_000002_create_fiscal_periods_table.php
│   │   ├── auth/
│   │   │   ├── 0001_01_01_000010_create_users_table.php
│   │   │   └── 0001_01_01_000011_create_personal_access_tokens_table.php
│   │   ├── general_ledger/
│   │   │   ├── 0002_01_01_000000_create_accounts_table.php
│   │   │   ├── 0002_01_01_000001_create_journal_entries_table.php
│   │   │   └── 0002_01_01_000002_create_journal_entry_lines_table.php
│   │   ├── accounts_payable/
│   │   │   ├── 0003_01_01_000000_create_vendors_table.php
│   │   │   ├── 0003_01_01_000001_create_vendor_invoices_table.php
│   │   │   └── 0003_01_01_000002_create_vendor_payments_table.php
│   │   ├── accounts_receivable/
│   │   │   ├── 0004_01_01_000000_create_customers_table.php
│   │   │   ├── 0004_01_01_000001_create_customer_invoices_table.php
│   │   │   └── 0004_01_01_000002_create_customer_receipts_table.php
│   │   ├── cash_management/
│   │   │   └── 0005_01_01_000000_create_cash_accounts_table.php
│   │   ├── bank_reconciliation/
│   │   │   ├── 0006_01_01_000000_create_bank_accounts_table.php
│   │   │   └── 0006_01_01_000001_create_bank_statements_table.php
│   │   ├── fixed_assets/
│   │   │   ├── 0007_01_01_000000_create_asset_categories_table.php
│   │   │   └── 0007_01_01_000001_create_assets_table.php
│   │   ├── inventory/
│   │   │   ├── 0008_01_01_000000_create_products_table.php
│   │   │   ├── 0008_01_01_000001_create_warehouses_table.php
│   │   │   └── 0008_01_01_000002_create_stock_movements_table.php
│   │   ├── purchasing/
│   │   │   └── 0009_01_01_000000_create_purchase_orders_table.php
│   │   ├── sales/
│   │   │   └── 0010_01_01_000000_create_sales_orders_table.php
│   │   ├── tax_management/
│   │   │   └── 0011_01_01_000000_create_tax_rates_table.php
│   │   ├── budgeting/
│   │   │   └── 0012_01_01_000000_create_budgets_table.php
│   │   ├── recurring_transactions/
│   │   │   └── 0013_01_01_000000_create_recurring_templates_table.php
│   │   ├── period_closing/
│   │   │   └── 0014_01_01_000000_create_period_close_checklists_table.php
│   │   ├── audit_trail/
│   │   │   └── 0015_01_01_000000_create_audit_logs_table.php
│   │   └── documents/
│   │       └── 0016_01_01_000000_create_documents_table.php
│   ├── seeders/
│   │   ├── DatabaseSeeder.php            # Master seeder; calls module seeders in dependency order
│   │   ├── CompanySeeder.php
│   │   ├── RolesPermissionsSeeder.php    # Seeds default roles: admin, accountant, viewer, auditor
│   │   ├── ChartOfAccountsSeeder.php     # Standard chart of accounts for demo company
│   │   ├── FiscalPeriodsSeeder.php
│   │   └── TaxRatesSeeder.php
│   └── factories/                        # Model factories for testing and local seeding
│       ├── UserFactory.php
│       ├── JournalEntryFactory.php
│       └── ...
│
├── routes/
│   ├── api.php                           # Root API router — groups routes by module, applies middleware
│   ├── api/
│   │   ├── v1/
│   │   │   ├── auth.php                  # POST /login, POST /logout, GET /me
│   │   │   ├── companies.php             # /companies resource routes
│   │   │   ├── branches.php
│   │   │   ├── general-ledger.php        # /accounts, /journal-entries
│   │   │   ├── accounts-payable.php      # /vendors, /vendor-invoices, /payments
│   │   │   ├── accounts-receivable.php
│   │   │   ├── cash-management.php
│   │   │   ├── bank-reconciliation.php
│   │   │   ├── fixed-assets.php
│   │   │   ├── inventory.php
│   │   │   ├── purchasing.php
│   │   │   ├── sales.php
│   │   │   ├── tax-management.php
│   │   │   ├── budgeting.php
│   │   │   ├── recurring-transactions.php
│   │   │   ├── period-closing.php
│   │   │   ├── reporting.php
│   │   │   ├── audit-trail.php
│   │   │   └── documents.php
│   │   └── v2/                           # Placeholder for future API version
│   └── console.php                       # Artisan schedule definitions
│
├── resources/
│   └── js/                               # React + TypeScript SPA
│       ├── main.tsx                      # React DOM root; wraps app in providers
│       ├── App.tsx                       # Route tree definition (React Router v6)
│       ├── vite-env.d.ts
│       ├── api/                          # Axios instance, interceptors, typed API clients
│       │   ├── client.ts                 # Axios base instance with auth token injection
│       │   └── modules/
│       │       ├── auth.api.ts
│       │       ├── generalLedger.api.ts
│       │       ├── accountsPayable.api.ts
│       │       └── ...                   # One file per module
│       ├── components/                   # Shared/global UI components
│       │   ├── ui/                       # Primitive: Button, Input, Modal, Table, Badge
│       │   ├── layout/
│       │   │   ├── AppShell.tsx          # Sidebar + topbar shell
│       │   │   ├── Sidebar.tsx
│       │   │   └── Topbar.tsx
│       │   └── shared/
│       │       ├── DataTable.tsx         # Generic sortable/paginated table
│       │       ├── FormBuilder.tsx       # Schema-driven form renderer
│       │       ├── CurrencyInput.tsx     # Locale-aware currency field
│       │       ├── DateRangePicker.tsx
│       │       └── ConfirmDialog.tsx
│       ├── hooks/                        # Shared custom React hooks
│       │   ├── useAuth.ts                # Reads auth state from Zustand auth store
│       │   ├── useTenant.ts              # Provides company_id / branch_id context
│       │   ├── usePermission.ts          # Boolean permission check helper
│       │   └── usePagination.ts
│       ├── stores/                       # Zustand global state slices
│       │   ├── authStore.ts              # Current user, token, roles
│       │   ├── tenantStore.ts            # Active company and branch selection
│       │   └── uiStore.ts                # Sidebar collapse, theme, notification queue
│       ├── types/                        # Shared TypeScript interfaces and enums
│       │   ├── api.types.ts              # Generic ApiResponse<T>, PaginatedResponse<T>
│       │   ├── auth.types.ts
│       │   ├── accounting.types.ts       # JournalEntry, Account, etc.
│       │   └── index.ts                  # Re-exports all types
│       ├── utils/                        # Pure utility functions
│       │   ├── currency.ts               # Format/parse currency amounts
│       │   ├── date.ts                   # Format dates, fiscal period helpers
│       │   └── validators.ts             # Shared form validation rules
│       └── modules/                      # Feature modules — mirrors backend module structure
│           ├── Auth/
│           │   ├── pages/
│           │   │   └── LoginPage.tsx
│           │   └── components/
│           │       └── LoginForm.tsx
│           ├── Dashboard/
│           │   ├── pages/
│           │   │   └── DashboardPage.tsx
│           │   └── components/
│           │       ├── KpiCard.tsx
│           │       └── CashFlowChart.tsx
│           ├── GeneralLedger/
│           │   ├── pages/
│           │   │   ├── ChartOfAccountsPage.tsx
│           │   │   ├── JournalEntriesPage.tsx
│           │   │   └── TrialBalancePage.tsx
│           │   ├── components/
│           │   │   ├── AccountTree.tsx
│           │   │   └── JournalEntryForm.tsx
│           │   └── hooks/
│           │       └── useJournalEntries.ts    # React Query hooks for GL data
│           ├── AccountsPayable/
│           │   ├── pages/
│           │   │   ├── VendorsPage.tsx
│           │   │   ├── VendorInvoicesPage.tsx
│           │   │   └── PaymentsPage.tsx
│           │   ├── components/
│           │   └── hooks/
│           ├── AccountsReceivable/
│           │   ├── pages/
│           │   ├── components/
│           │   └── hooks/
│           ├── CashManagement/
│           │   ├── pages/
│           │   ├── components/
│           │   └── hooks/
│           ├── BankReconciliation/
│           │   ├── pages/
│           │   │   └── ReconciliationPage.tsx  # Split-pane: bank lines left, GL entries right
│           │   ├── components/
│           │   │   └── MatchingPanel.tsx
│           │   └── hooks/
│           ├── FixedAssets/
│           │   ├── pages/
│           │   ├── components/
│           │   └── hooks/
│           ├── Inventory/
│           │   ├── pages/
│           │   ├── components/
│           │   └── hooks/
│           ├── Purchasing/
│           │   ├── pages/
│           │   ├── components/
│           │   └── hooks/
│           ├── Sales/
│           │   ├── pages/
│           │   ├── components/
│           │   └── hooks/
│           ├── TaxManagement/
│           │   ├── pages/
│           │   ├── components/
│           │   └── hooks/
│           ├── Budgeting/
│           │   ├── pages/
│           │   ├── components/
│           │   └── hooks/
│           ├── RecurringTransactions/
│           │   ├── pages/
│           │   ├── components/
│           │   └── hooks/
│           ├── PeriodClosing/
│           │   ├── pages/
│           │   │   └── PeriodClosingPage.tsx   # Checklist-driven close wizard
│           │   ├── components/
│           │   └── hooks/
│           ├── Reporting/
│           │   ├── pages/
│           │   │   ├── ProfitLossPage.tsx
│           │   │   ├── BalanceSheetPage.tsx
│           │   │   └── CashFlowPage.tsx
│           │   ├── components/
│           │   │   └── ReportViewer.tsx        # Renders report data in printable layout
│           │   └── hooks/
│           ├── AuditTrail/
│           │   ├── pages/
│           │   │   └── AuditLogPage.tsx
│           │   ├── components/
│           │   └── hooks/
│           └── Documents/
│               ├── pages/
│               ├── components/
│               │   └── DocumentUploader.tsx    # Drag-and-drop uploader with progress
│               └── hooks/
│
├── storage/
│   ├── app/
│   │   └── documents/                    # Local fallback storage for document attachments
│   ├── framework/                        # Laravel cache, sessions, views (no Blade views used)
│   └── logs/
│
├── tests/
│   ├── Feature/                          # HTTP-level integration tests per module
│   │   ├── Auth/
│   │   │   └── AuthenticationTest.php
│   │   ├── GeneralLedger/
│   │   │   ├── JournalEntryTest.php      # Tests double-entry balance enforcement
│   │   │   └── TrialBalanceTest.php
│   │   ├── AccountsPayable/
│   │   ├── AccountsReceivable/
│   │   ├── PeriodClosing/
│   │   │   └── PeriodLockTest.php        # Tests that writes to closed periods are rejected
│   │   └── ...
│   ├── Unit/                             # Unit tests for services, DTOs, calculations
│   │   ├── GeneralLedger/
│   │   │   └── JournalEntryServiceTest.php
│   │   ├── FixedAssets/
│   │   │   └── DepreciationServiceTest.php
│   │   └── TaxManagement/
│   │       └── TaxCalculationServiceTest.php
│   └── Pest.php                          # Pest PHP test framework bootstrap
│
├── .env.example                          # Environment variable template
├── artisan                               # Laravel CLI entry point
├── composer.json                         # PHP dependencies
├── package.json                          # Node/NPM dependencies (Vite, React, TypeScript)
├── vite.config.ts                        # Vite build config for React SPA
├── tsconfig.json                         # TypeScript compiler options
├── phpstan.neon                          # Static analysis config (level 8)
└── pint.json                             # Laravel Pint code style configuration
```

---

## Key Structural Conventions

| Convention | Rule |
|---|---|
| Module isolation | No module imports directly from another module's namespace; cross-module communication via Events or injected Services |
| No Blade | `resources/views/` contains only a single `app.blade.php` that serves the React SPA `index.html` shell |
| UUID primary keys | All tables use `uuid` as primary key type via `$table->uuid('id')->primary()` |
| Tenant scoping | `BaseRepository` applies `WHERE company_id = ? AND branch_id = ?` as a global Eloquent scope |
| Immutable posted records | Posted journal entries, invoices, and payments have no update route; only reversal endpoints exist |
| API versioning | All routes live under `/api/v1/`; the version prefix is applied in `RouteServiceProvider` |
| DTO → Service → Repository | Controllers accept Requests, convert to DTOs, pass to Services; Services call Repositories for persistence |
