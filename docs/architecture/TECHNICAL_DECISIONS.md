# Technical Decisions — ERP Accounting System

> This document records the key architectural and technology choices made for the system, the rationale behind each decision, and the implementation approach. It is intended as a long-lived reference for the engineering team.

---

## 1. API Architecture — REST + Versioning (`/api/v1/`)

### Decision
The backend exposes a pure REST API. There is no server-rendered HTML. All routes are under the `/api/v1/` prefix. A `v2` namespace is reserved for future breaking changes.

### Rationale
- **Decoupling:** The React SPA and the Laravel backend evolve independently. The backend can also serve mobile apps or third-party integrations without rework.
- **No Blade:** Removing Blade from the equation eliminates an entire class of CSRF complexity for non-SPA clients and keeps the backend focused on data, not presentation.
- **Versioning strategy:** The `v1` prefix allows non-breaking changes to be made freely within `v1`. When a breaking change is required, a `v2` namespace is introduced; both coexist during a transition window.

### Implementation
```php
// routes/api.php
Route::prefix('v1')->middleware(['auth:sanctum', 'set.tenant', 'force.json'])->group(function () {
    require __DIR__ . '/api/v1/auth.php';
    require __DIR__ . '/api/v1/general-ledger.php';
    // ... all module route files
});
```

### Conventions
- All responses use a consistent envelope: `{ "data": ..., "meta": ..., "errors": ... }`
- HTTP status codes are strictly observed: `200 OK`, `201 Created`, `204 No Content`, `422 Unprocessable Entity`, `403 Forbidden`, `404 Not Found`
- Pagination uses `?page=` and `?per_page=` query parameters; response includes `meta.pagination`
- Resource IDs are always UUIDs (never auto-increment integers exposed in the API)

---

## 2. Multi-Tenancy Strategy — Row-Level Isolation via `company_id` + `branch_id`

### Decision
All tenant data is stored in a **single shared database**. Every table that holds business data carries `company_id` (UUID) and, where applicable, `branch_id` (UUID) columns. Tenant isolation is enforced at the application layer via Eloquent global scopes, not at the database or schema level.

### Rationale
| Approach | Rejected Reason |
|---|---|
| Separate DB per tenant | Unmanageable at scale; migration and backup complexity grows linearly with tenant count |
| Separate schema per tenant (PostgreSQL schemas) | Still complex to migrate; Laravel ORM support is limited |
| Shared schema + row-level filtering | Simpler to operate; single migration path; PostgreSQL RLS can be added as a defence-in-depth layer later |

### Implementation

**BaseModel:**
```php
abstract class BaseModel extends Model
{
    use SoftDeletes, HasUuids;

    protected static function booted(): void
    {
        static::addGlobalScope(new TenantScope());
    }
}
```

**TenantScope (Eloquent Global Scope):**
```php
class TenantScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        $builder->where('company_id', TenantContext::companyId());
        if ($model->scopeByBranch ?? false) {
            $builder->where('branch_id', TenantContext::branchId());
        }
    }
}
```

**SetTenantContext Middleware:**
```php
class SetTenantContext
{
    public function handle(Request $request, Closure $next): Response
    {
        $companyId = $request->header('X-Company-ID') ?? $request->user()?->default_company_id;
        $branchId  = $request->header('X-Branch-ID')  ?? $request->user()?->default_branch_id;
        TenantContext::set($companyId, $branchId);
        return $next($request);
    }
}
```

### Defence in depth
- `company_id` and `branch_id` are declared `NOT NULL` with foreign key constraints at the DB level
- PostgreSQL Row Level Security (RLS) can be added in a later hardening phase as a secondary guard
- All `INSERT` statements go through repositories that automatically stamp `company_id`/`branch_id` from the context

---

## 3. Authentication — Laravel Sanctum + SPA Tokens + RBAC

### Decision
Authentication uses **Laravel Sanctum** in SPA token mode (Bearer tokens in the `Authorization` header, not cookie-based sessions). Role-based access control uses **`spatie/laravel-permission`** with per-user, per-company role assignments.

### Rationale
- Sanctum is the official Laravel package for SPA and mobile token auth; it integrates natively without the overhead of Passport/OAuth2
- `spatie/laravel-permission` is the de-facto standard for Laravel RBAC with a well-tested API and built-in caching
- Bearer tokens (rather than cookies) simplify cross-origin requests from the React SPA and future mobile clients
- Roles are scoped per company: a user can be `accountant` in Company A and `viewer` in Company B

### Roles (default seeded)
| Role | Description |
|---|---|
| `admin` | Full access; manages users, roles, company settings |
| `accountant` | Create and post transactions; run reports |
| `approver` | Approve invoices, POs, budgets |
| `viewer` | Read-only access to all financial data |
| `auditor` | Read-only access including audit trail; cannot modify data |

### Permission naming convention
Permissions follow the pattern `<module>.<action>`, e.g.:
- `journal_entry.create`, `journal_entry.post`, `journal_entry.reverse`
- `vendor_invoice.approve`, `vendor_invoice.pay`
- `fiscal_period.close`
- `audit_log.view`

### Token security
- Tokens expire after 8 hours of inactivity; configurable via `config/sanctum.php`
- Tokens are hashed in the database (`personal_access_tokens.token` stores SHA-256 hash)
- `RevokeExpiredTokens` runs nightly via the scheduler

---

## 4. Database — PostgreSQL (not MySQL)

### Decision
PostgreSQL 16 is the sole database engine. MySQL/MariaDB are not supported.

### Why PostgreSQL over MySQL for accounting

| Feature | PostgreSQL | MySQL 8 |
|---|---|---|
| **ACID guarantees** | Full serializable isolation | Weaker default isolation; gap-lock issues |
| **Recursive CTEs** | Native `WITH RECURSIVE` | Added in 8.0 but limited planner optimisation |
| **Window functions** | Full SQL:2003 window functions with frames | Available but less mature planner support |
| **JSONB columns** | Binary JSON with GIN indexing; used for audit log payloads | JSON type lacks binary storage and index efficiency |
| **Table partitioning** | Declarative range partitioning on `occurred_at` for audit_logs | Available but less flexible |
| **Check constraints** | Enforced and part of query planner | Parsed but historically not enforced by default |
| **Numeric precision** | `NUMERIC(20,6)` exact arithmetic | Same, but historical rounding edge cases |
| **Full-text search** | Built-in tsvector/tsquery | Requires MyISAM or InnoDB FTS with limitations |

### Key database design rules
- All monetary amounts are stored as `NUMERIC(20, 6)` — never `FLOAT` or `DOUBLE`
- All timestamps are stored as `TIMESTAMPTZ` (timezone-aware) in UTC
- All primary keys are `UUID` generated by PostgreSQL `gen_random_uuid()` or Laravel's `HasUuids` trait
- Foreign key constraints are declared on all relationship columns
- GIN indexes are placed on JSONB columns in `audit_logs`
- The `audit_logs` table is range-partitioned by `occurred_at` (monthly partitions) for query performance
- Recursive CTEs are used for chart-of-accounts hierarchy traversal in `AccountRepository`

---

## 5. Queue System — Laravel Horizon + Redis

### Decision
All background processing uses **Laravel Horizon** backed by **Redis**. Synchronous processing is avoided for any operation that is slow, non-critical to the HTTP response, or fan-out in nature.

### Rationale
- Horizon provides a real-time monitoring dashboard for queue depth, throughput, and failed jobs
- Redis delivers sub-millisecond queue push/pop; suitable for high-frequency audit log events
- Decoupling slow operations (report generation, email sending, bank auto-matching) improves API response times

### Queue pools configuration (`config/horizon.php`)
```php
'environments' => [
    'production' => [
        'supervisor-default' => [
            'connection' => 'redis',
            'queue' => ['default'],
            'balance' => 'auto',
            'processes' => 10,
            'tries' => 3,
        ],
        'supervisor-reports' => [
            'queue' => ['reports'],
            'processes' => 3,
            'timeout' => 300,  // Report generation can take up to 5 minutes
            'tries' => 2,
        ],
        'supervisor-audit' => [
            'queue' => ['audit'],
            'processes' => 5,
            'tries' => 5,      // Audit logs must not be lost; more retries
        ],
        'supervisor-notifications' => [
            'queue' => ['notifications'],
            'processes' => 3,
            'tries' => 3,
        ],
    ],
],
```

### Jobs dispatched to queues
| Job | Queue | Trigger |
|---|---|---|
| `GenerateReportJob` | `reports` | Export button clicked |
| `WriteAuditLogAsync` | `audit` | Any domain event fired |
| `AutoMatchBankLines` | `default` | Bank statement imported |
| `ProcessRecurringTransactions` | `default` | Daily scheduler |
| `RunMonthlyDepreciation` | `default` | Period-close checklist step |
| `SendInvoiceReminderEmail` | `notifications` | Scheduled daily check |
| `SendPaymentDueReminder` | `notifications` | Scheduled daily check |
| `ScanDocumentForMalware` | `default` | Document uploaded |
| `RevokeExpiredTokens` | `default` | Nightly scheduler |

---

## 6. Event Sourcing for Audit Trail — Laravel Events + `audit_logs` Table

### Decision
The audit trail is implemented using **Laravel's native event system** as a lightweight event bus. Every domain action fires a typed PHP event. A universal `WriteAuditLog` listener captures all events and appends an immutable record to the `audit_logs` table. This is not full event sourcing (state is not reconstructed from events), but the audit log is event-sourced in nature.

### Rationale
- Full event sourcing (à la EventSauce) would require significant infrastructure and a read-model projection layer; the benefit does not justify the complexity for an accounting ERP where state is well-modelled in relational tables
- Laravel's event system provides the fan-out capability needed (one event → multiple listeners) without external infrastructure
- Appending to a dedicated `audit_logs` table (never updating, never deleting) provides the immutability guarantee required for financial audit compliance

### Event structure
```php
// Example domain event
class JournalEntryPosted
{
    public function __construct(
        public readonly JournalEntry $journalEntry,
        public readonly User $postedBy,
        public readonly Carbon $occurredAt,
    ) {}
}
```

### AuditLog table schema
```sql
CREATE TABLE audit_logs (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id   UUID NOT NULL REFERENCES companies(id),
    branch_id    UUID REFERENCES branches(id),
    user_id      UUID REFERENCES users(id),
    event_name   VARCHAR(255) NOT NULL,          -- e.g. 'JournalEntryPosted'
    auditable_type VARCHAR(255),                 -- e.g. 'JournalEntry'
    auditable_id UUID,                           -- entity UUID
    old_values   JSONB,                          -- state before change
    new_values   JSONB,                          -- state after change
    ip_address   INET,
    user_agent   TEXT,
    occurred_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (occurred_at);

-- Monthly partitions created automatically
CREATE TABLE audit_logs_2025_01 PARTITION OF audit_logs
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE INDEX idx_audit_logs_company_occurred ON audit_logs (company_id, occurred_at DESC);
CREATE INDEX idx_audit_logs_auditable ON audit_logs (auditable_type, auditable_id);
CREATE INDEX idx_audit_logs_user ON audit_logs (user_id, occurred_at DESC);
CREATE INDEX idx_audit_logs_new_values ON audit_logs USING GIN (new_values);
```

### Universal listener registration
```php
// app/Providers/EventServiceProvider.php
protected $observers = [
    // All domain events → WriteAuditLog listener
];

public function boot(): void
{
    Event::listen('*', WriteAuditLog::class);
}
```

---

## 7. Double-Entry Enforcement — DB Constraint: sum(debits) = sum(credits)

### Decision
The double-entry balance rule is enforced at **multiple layers**: the `JournalEntryService`, a PostgreSQL `CHECK` constraint via a trigger, and the API request validator. A journal entry that does not balance cannot be posted.

### Rationale
- Application-layer validation alone is insufficient for a financial system; a developer mistake or direct DB access could bypass it
- A database-level trigger provides a last line of defence that cannot be bypassed without explicitly disabling it
- The check runs only on `posted` entries; `draft` entries are allowed to be temporarily unbalanced while being built

### Implementation

**Service layer:**
```php
class JournalEntryService
{
    public function post(JournalEntry $entry): void
    {
        $totalDebits  = $entry->lines->sum('debit_amount');
        $totalCredits = $entry->lines->sum('credit_amount');

        if (bccomp((string)$totalDebits, (string)$totalCredits, 6) !== 0) {
            throw new UnbalancedJournalEntryException(
                "Journal entry debits ({$totalDebits}) do not equal credits ({$totalCredits})"
            );
        }
        // ...proceed to post
    }
}
```

**Database trigger (PostgreSQL):**
```sql
CREATE OR REPLACE FUNCTION enforce_journal_balance()
RETURNS TRIGGER AS $$
DECLARE
    v_debits  NUMERIC(20,6);
    v_credits NUMERIC(20,6);
BEGIN
    -- Only enforce on posted entries
    IF NEW.status = 'posted' THEN
        SELECT
            COALESCE(SUM(debit_amount), 0),
            COALESCE(SUM(credit_amount), 0)
        INTO v_debits, v_credits
        FROM journal_entry_lines
        WHERE journal_entry_id = NEW.id;

        IF v_debits <> v_credits THEN
            RAISE EXCEPTION 'Journal entry % is unbalanced: debits=% credits=%',
                NEW.id, v_debits, v_credits;
        END IF;

        IF v_debits = 0 THEN
            RAISE EXCEPTION 'Journal entry % has zero-value lines', NEW.id;
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER check_journal_balance
AFTER INSERT OR UPDATE ON journal_entries
FOR EACH ROW EXECUTE FUNCTION enforce_journal_balance();
```

### Additional GL integrity rules
- Every `journal_entry_line` must reference a valid, active `account_id` (`NOT NULL REFERENCES accounts(id)`)
- A line cannot have both `debit_amount > 0` and `credit_amount > 0` simultaneously (enforced by `CHECK (NOT (debit_amount > 0 AND credit_amount > 0))`)
- A posted journal entry must have at least 2 lines (enforced at the service layer)

---

## 8. Soft Deletes + Immutability — Posted Transactions Are Never Edited

### Decision
All models use **soft deletes** (`deleted_at` timestamp). Posted financial transactions (journal entries, invoices, payments) are **immutable** — no `PUT`/`PATCH` routes exist for posted records. Corrections are made via reversal entries.

### Rationale
- **Regulatory requirement:** Financial records must be auditable. Allowing edits to posted transactions would destroy the audit trail and is prohibited under most accounting standards (IFRS, GAAP)
- **Data integrity:** Immutable posted records allow balances to be reliably computed by summing a stable set of records
- **Soft deletes:** Records are never physically deleted. `deleted_at IS NOT NULL` rows are hidden by default Eloquent scopes but remain in the database for forensic and compliance purposes

### Immutability enforcement

**Middleware approach:**
```php
class PreventPostedTransactionMutation
{
    public function handle(Request $request, Closure $next, string $modelClass): Response
    {
        if (in_array($request->method(), ['PUT', 'PATCH', 'DELETE'])) {
            $record = $modelClass::findOrFail($request->route('id'));
            if ($record->status === 'posted') {
                throw new ImmutableRecordException(
                    'Posted transactions cannot be modified. Create a reversal entry instead.'
                );
            }
        }
        return $next($request);
    }
}
```

**Reversal workflow:**
- `POST /api/v1/gl/journal-entries/{id}/reverse` creates a mirror entry with all debits and credits swapped and status `posted`
- The original entry gains a `reversed_by` UUID pointing to the reversal entry
- Reversal entries are themselves immutable once posted

### Soft delete conventions
- Physical deletion is never performed in application code
- `VACUUM` and archival of very old soft-deleted records is handled by a scheduled maintenance command
- The `audit_logs` table is excluded from soft deletes — it is append-only with no delete capability at all

---

## 9. Frontend State Management — Zustand + React Query

### Decision
The React frontend uses **Zustand** for global client-side state and **React Query (TanStack Query)** for all server-state management (data fetching, caching, synchronisation, mutations).

### Rationale
| Concern | Tool | Why |
|---|---|---|
| Server data (API responses) | React Query | Declarative caching, background refetch, stale-while-revalidate, mutation with optimistic updates |
| Global client state (auth, tenant, UI) | Zustand | Minimal boilerplate; no Provider nesting hell; works outside React components |
| Form state | React Hook Form | Performant uncontrolled forms; integrates with Zod for schema validation |
| Redux | Rejected | Too much boilerplate for the problem; mixing server and client state in one store leads to cache invalidation complexity |

### Zustand stores
```typescript
// stores/authStore.ts
interface AuthState {
  user: User | null;
  token: string | null;
  roles: string[];
  permissions: string[];
  login: (credentials: LoginCredentials) => Promise<void>;
  logout: () => void;
}

// stores/tenantStore.ts
interface TenantState {
  companyId: string | null;
  branchId: string | null;
  company: Company | null;
  branch: Branch | null;
  setCompany: (company: Company) => void;
  setBranch: (branch: Branch) => void;
}

// stores/uiStore.ts
interface UIState {
  sidebarCollapsed: boolean;
  theme: 'light' | 'dark';
  notifications: Notification[];
  toggleSidebar: () => void;
  addNotification: (n: Notification) => void;
}
```

### React Query conventions
```typescript
// hooks/useJournalEntries.ts
export const useJournalEntries = (params: JournalEntryParams) =>
  useQuery({
    queryKey: ['journalEntries', params],
    queryFn: () => generalLedgerApi.listJournalEntries(params),
    staleTime: 30_000,   // 30 seconds
  });

export const usePostJournalEntry = () =>
  useMutation({
    mutationFn: generalLedgerApi.postJournalEntry,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['journalEntries'] });
      queryClient.invalidateQueries({ queryKey: ['trialBalance'] });
    },
  });
```

### Zod schema validation (forms)
All form inputs are validated client-side using **Zod** schemas that mirror the backend `FormRequest` rules, providing immediate feedback before API submission.

---

## 10. File Storage — Laravel Storage + S3-Compatible

### Decision
Document attachments are stored using **Laravel's Storage abstraction** backed by an **S3-compatible object store** (AWS S3, MinIO, DigitalOcean Spaces, Cloudflare R2). Local disk storage is used for development only.

### Rationale
- S3-compatible storage provides unlimited capacity, geo-redundancy, and versioning without managing disk
- Laravel's storage abstraction allows the backend driver to be swapped without code changes
- Signed URLs keep the file content off the application server and deliver files directly from the object store to the client browser

### Implementation

**config/filesystems.php (production disk):**
```php
'documents' => [
    'driver' => 's3',
    'key'    => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION'),
    'bucket' => env('AWS_BUCKET'),
    'url'    => env('AWS_URL'),
    'endpoint' => env('AWS_ENDPOINT'),     // For S3-compatible stores (MinIO etc.)
    'use_path_style_endpoint' => env('AWS_USE_PATH_STYLE_ENDPOINT', false),
],
```

**DocumentService:**
```php
class DocumentService
{
    public function store(UploadedFile $file, string $documentableType, string $documentableId): Document
    {
        $path = Storage::disk('documents')->putFile(
            "{$documentableType}/{$documentableId}",
            $file
        );
        return Document::create([
            'documentable_type' => $documentableType,
            'documentable_id'   => $documentableId,
            'filename'          => $file->getClientOriginalName(),
            'mime_type'         => $file->getMimeType(),
            'file_size'         => $file->getSize(),
            'storage_path'      => $path,
            'uploaded_by'       => auth()->id(),
        ]);
    }

    public function signedUrl(Document $document): string
    {
        return Storage::disk('documents')
            ->temporaryUrl($document->storage_path, now()->addMinutes(15));
    }
}
```

### Security rules
- Storage paths are never returned directly in the API; only signed URLs are provided
- Signed URLs expire in 15 minutes
- File type validation is enforced on upload (allowlisted MIME types only)
- Files are stored under a path structure that includes `company_id` to prevent cross-tenant path collisions

---

## 11. Reporting Engine — DomPDF (PDF) + PhpSpreadsheet (Excel)

### Decision
Server-side report generation uses **DomPDF** (`barryvdh/laravel-dompdf`) for PDF output and **PhpSpreadsheet** (`phpoffice/phpspreadsheet`) for Excel output. Report generation is always **asynchronous** — dispatched as a `GenerateReportJob` to the `reports` Horizon queue.

### Rationale
- DomPDF renders HTML/CSS templates to PDF, allowing the same Blade/component template to be styled and previewed easily
- PhpSpreadsheet provides full Excel XLSX format support with cell formatting, formulas, and multi-sheet workbooks
- Asynchronous generation prevents large report jobs from timing out the HTTP request and allows the user to continue working while the report is generated

### Report generation flow
```
1. User clicks "Export PDF" → POST /api/v1/reporting/exports
2. API creates ReportExport record (status: 'queued') and returns export ID immediately
3. GenerateReportJob dispatched to 'reports' queue with report parameters
4. Frontend polls GET /api/v1/reporting/exports/{id} (or uses SSE/WebSocket for push)
5. Horizon worker runs the job:
   a. Fetches data via reporting service
   b. Renders Blade template (PDF) or builds worksheet (Excel)
   c. Stores output file on S3 via DocumentService
   d. Updates ReportExport.status = 'done' and stores download URL
6. Frontend detects 'done' status; user clicks download → signed URL served
```

### PDF template example
```php
// resources/views/reports/profit_loss.blade.php (used only for PDF generation)
// Styled with inline CSS for DomPDF compatibility
```

### Excel generation conventions
- Financial figures use `NUMERIC_FORMAT = '#,##0.00'` cell format
- Summary rows use bold font and grey fill
- Each fiscal period becomes a column; accounts become rows
- Multi-sheet workbooks: Sheet 1 = Summary, Sheet 2 = Detail, Sheet 3 = Notes

---

## 12. Period Locking — Fiscal Periods with Open/Closed Status

### Decision
The system maintains a `fiscal_periods` table with a `status` column (`open` / `closing` / `closed`). A middleware (`FiscalPeriodOpen`) intercepts any write request that targets a closed period and rejects it with `HTTP 422`.

### Rationale
- Period locking is a fundamental accounting control: once a period is closed and financial statements are produced, no further entries should modify that period's balances
- Middleware enforcement means the lock is applied uniformly across all modules without each module needing to implement its own check
- The `closing` status allows the period-close checklist workflow to proceed while signalling to users that the period is being finalised

### `fiscal_periods` table
```sql
CREATE TABLE fiscal_periods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id      UUID NOT NULL REFERENCES companies(id),
    name            VARCHAR(50) NOT NULL,          -- e.g. 'January 2025'
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    fiscal_year     SMALLINT NOT NULL,             -- e.g. 2025
    period_number   SMALLINT NOT NULL,             -- 1–12 (or 1–4 for quarterly)
    status          VARCHAR(20) NOT NULL DEFAULT 'open'
                        CHECK (status IN ('open', 'closing', 'closed')),
    closed_by       UUID REFERENCES users(id),
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (company_id, fiscal_year, period_number)
);
```

### `FiscalPeriodOpen` middleware
```php
class FiscalPeriodOpen
{
    public function handle(Request $request, Closure $next): Response
    {
        // Only check on write operations
        if (!in_array($request->method(), ['POST', 'PUT', 'PATCH', 'DELETE'])) {
            return $next($request);
        }

        $transactionDate = $request->input('date') ?? $request->input('transaction_date');
        if (!$transactionDate) {
            return $next($request);
        }

        $period = FiscalPeriod::where('company_id', TenantContext::companyId())
            ->whereDate('start_date', '<=', $transactionDate)
            ->whereDate('end_date', '>=', $transactionDate)
            ->first();

        if ($period && $period->status !== 'open') {
            throw new ClosedFiscalPeriodException(
                "The fiscal period '{$period->name}' is {$period->status}. " .
                "No further entries can be posted to this period."
            );
        }

        return $next($request);
    }
}
```

### Period-close workflow integration
1. Accountant initiates close: `POST /api/v1/period-closing/{periodId}/initiate` → status becomes `closing`
2. Period-close checklist items are signed off one by one
3. When all checklist items are `signed_off`, accountant submits: `POST /api/v1/period-closing/{periodId}/finalise`
4. `PeriodClosingService` fires `FiscalPeriodClosed` event
5. `LockFiscalPeriod` listener updates `fiscal_periods.status = 'closed'` and sets `closed_by` / `closed_at`
6. All subsequent write attempts to this period are rejected by `FiscalPeriodOpen` middleware with `HTTP 422`

### Year-end close additions
- `YearEndService` runs the retained-earnings transfer: net income for the year is moved to the retained-earnings account via a zero-dated GL entry on the last day of the fiscal year
- New fiscal period rows are created for the next year
- The chart of accounts is carried forward; income/expense account balances reset to zero for the new year via the retained-earnings transfer
