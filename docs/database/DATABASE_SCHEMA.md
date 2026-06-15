# Database Schema — Enterprise SMB ERP Accounting System

> Stack: Laravel 12 · PostgreSQL 16 · Multi-tenant (company-scoped rows)
> Convention: All primary keys are `BIGSERIAL`. All monetary values are `NUMERIC(20,4)`. Timestamps are `TIMESTAMPTZ`. Soft-deleted tables use `deleted_at TIMESTAMPTZ`.

---

## Table of Contents

1. [Core / Multi-tenancy](#1-core--multi-tenancy)
2. [Auth & RBAC](#2-auth--rbac)
3. [Chart of Accounts / General Ledger](#3-chart-of-accounts--general-ledger)
4. [Accounts Payable](#4-accounts-payable)
5. [Accounts Receivable](#5-accounts-receivable)
6. [Cash & Bank](#6-cash--bank)
7. [Fixed Assets](#7-fixed-assets)
8. [Inventory](#8-inventory)
9. [Purchasing](#9-purchasing)
10. [Sales](#10-sales)
11. [Tax](#11-tax)
12. [Budgeting](#12-budgeting)
13. [Recurring Transactions](#13-recurring-transactions)
14. [Audit Trail](#14-audit-trail)
15. [Documents](#15-documents)

---

## 1. Core / Multi-tenancy

### `companies`

```sql
CREATE TABLE companies (
    id               BIGSERIAL PRIMARY KEY,
    code             VARCHAR(20)   NOT NULL,
    name             VARCHAR(255)  NOT NULL,
    legal_name       VARCHAR(255),
    tax_number       VARCHAR(100),
    currency_code    CHAR(3)       NOT NULL DEFAULT 'USD',
    fiscal_year_start SMALLINT     NOT NULL DEFAULT 1
                         CHECK (fiscal_year_start BETWEEN 1 AND 12),
    timezone         VARCHAR(100)  NOT NULL DEFAULT 'UTC',
    logo_path        VARCHAR(500),
    is_active        BOOLEAN       NOT NULL DEFAULT TRUE,
    settings         JSONB         NOT NULL DEFAULT '{}',
    created_at       TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    deleted_at       TIMESTAMPTZ,

    CONSTRAINT uq_companies_code UNIQUE (code)
);

CREATE INDEX idx_companies_is_active   ON companies (is_active) WHERE deleted_at IS NULL;
CREATE INDEX idx_companies_deleted_at  ON companies (deleted_at);
CREATE INDEX idx_companies_settings    ON companies USING GIN (settings);
```

**Notes:** `settings` stores company-level feature flags and preferences (e.g. `{"enable_inventory": true, "enable_fixed_assets": false}`). `fiscal_year_start` is 1-based month number.

---

### `branches`

```sql
CREATE TABLE branches (
    id              BIGSERIAL PRIMARY KEY,
    company_id      BIGINT       NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    code            VARCHAR(20)  NOT NULL,
    name            VARCHAR(255) NOT NULL,
    address         JSONB        NOT NULL DEFAULT '{}',
    is_headquarter  BOOLEAN      NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_branches_company_code UNIQUE (company_id, code)
);

CREATE INDEX idx_branches_company_id  ON branches (company_id);
CREATE INDEX idx_branches_is_active   ON branches (company_id, is_active);
```

**Notes:** `address` stores structured address: `{"street": "", "city": "", "state": "", "postal_code": "", "country": ""}`. Only one branch per company should have `is_headquarter = TRUE` (enforced at application layer).

---

### `fiscal_periods`

```sql
CREATE TABLE fiscal_periods (
    id          BIGSERIAL PRIMARY KEY,
    company_id  BIGINT       NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    name        VARCHAR(100) NOT NULL,
    start_date  DATE         NOT NULL,
    end_date    DATE         NOT NULL,
    status      VARCHAR(20)  NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open', 'closed', 'locked')),
    closed_at   TIMESTAMPTZ,
    closed_by   BIGINT       REFERENCES users (id),

    CONSTRAINT uq_fiscal_periods_company_dates UNIQUE (company_id, start_date, end_date),
    CONSTRAINT chk_fiscal_period_dates CHECK (end_date >= start_date)
);

CREATE INDEX idx_fiscal_periods_company_id    ON fiscal_periods (company_id);
CREATE INDEX idx_fiscal_periods_company_status ON fiscal_periods (company_id, status);
CREATE INDEX idx_fiscal_periods_dates         ON fiscal_periods (company_id, start_date, end_date);
```

**Notes:** Posting to a `closed` or `locked` period is blocked at the application/service layer. `locked` is permanent; `closed` can be re-opened by admins.

---

### `currencies`

```sql
CREATE TABLE currencies (
    id             BIGSERIAL PRIMARY KEY,
    code           CHAR(3)      NOT NULL,
    name           VARCHAR(100) NOT NULL,
    symbol         VARCHAR(10)  NOT NULL,
    decimal_places SMALLINT     NOT NULL DEFAULT 2
                       CHECK (decimal_places BETWEEN 0 AND 8),

    CONSTRAINT uq_currencies_code UNIQUE (code)
);

-- Seed data examples
INSERT INTO currencies (code, name, symbol, decimal_places) VALUES
  ('USD', 'US Dollar',        '$',  2),
  ('EUR', 'Euro',             '€',  2),
  ('GBP', 'British Pound',    '£',  2),
  ('JPY', 'Japanese Yen',     '¥',  0),
  ('IDR', 'Indonesian Rupiah','Rp', 0),
  ('MYR', 'Malaysian Ringgit','RM', 2);
```

**Notes:** `currencies` is a global reference table, not company-scoped.

---

### `exchange_rates`

```sql
CREATE TABLE exchange_rates (
    id              BIGSERIAL PRIMARY KEY,
    from_currency   CHAR(3)      NOT NULL REFERENCES currencies (code),
    to_currency     CHAR(3)      NOT NULL REFERENCES currencies (code),
    rate            NUMERIC(20,8) NOT NULL CHECK (rate > 0),
    effective_date  DATE         NOT NULL,
    company_id      BIGINT       REFERENCES companies (id) ON DELETE CASCADE,

    CONSTRAINT uq_exchange_rates_pair_date_company
        UNIQUE (from_currency, to_currency, effective_date, company_id),
    CONSTRAINT chk_exchange_rates_different_currencies
        CHECK (from_currency <> to_currency)
);

CREATE INDEX idx_exchange_rates_company_id     ON exchange_rates (company_id);
CREATE INDEX idx_exchange_rates_pair_date      ON exchange_rates (from_currency, to_currency, effective_date DESC);
CREATE INDEX idx_exchange_rates_effective_date ON exchange_rates (effective_date DESC);
```

**Notes:** `company_id IS NULL` = system-wide rate. Company-specific rate takes precedence. Query pattern: `WHERE (company_id = :cid OR company_id IS NULL) ORDER BY company_id NULLS LAST, effective_date DESC LIMIT 1`.

---

## 2. Auth & RBAC

### `users`

```sql
CREATE TABLE users (
    id              BIGSERIAL PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    email           VARCHAR(255) NOT NULL,
    password        VARCHAR(255) NOT NULL,
    is_active       BOOLEAN      NOT NULL DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    remember_token  VARCHAR(100),
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ,

    CONSTRAINT uq_users_email UNIQUE (email)
);

CREATE INDEX idx_users_email      ON users (email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_is_active  ON users (is_active) WHERE deleted_at IS NULL;
```

---

### `user_company_access`

```sql
CREATE TABLE user_company_access (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT      NOT NULL REFERENCES users (id) ON DELETE CASCADE,
    company_id  BIGINT      NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    branch_id   BIGINT      REFERENCES branches (id) ON DELETE SET NULL,
    role        VARCHAR(100) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_user_company_access UNIQUE (user_id, company_id, branch_id)
);

CREATE INDEX idx_user_company_access_user_id    ON user_company_access (user_id);
CREATE INDEX idx_user_company_access_company_id ON user_company_access (company_id);
CREATE INDEX idx_user_company_access_branch_id  ON user_company_access (branch_id);
```

**Notes:** `branch_id IS NULL` = access to all branches within the company. Role values correspond to Spatie role names scoped to the company.

---

### `roles` (Spatie-compatible)

```sql
CREATE TABLE roles (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    guard_name  VARCHAR(255) NOT NULL DEFAULT 'web',
    company_id  BIGINT       REFERENCES companies (id) ON DELETE CASCADE,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_roles_name_guard_company UNIQUE (name, guard_name, company_id)
);

CREATE INDEX idx_roles_company_id ON roles (company_id);
```

**Notes:** `company_id IS NULL` = system-wide role (e.g. `super-admin`). Per-company roles allow custom permission sets.

---

### `permissions` (Spatie-compatible, extended)

```sql
CREATE TABLE permissions (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    guard_name  VARCHAR(255) NOT NULL DEFAULT 'web',
    module      VARCHAR(100) NOT NULL,   -- e.g. 'AP', 'AR', 'GL', 'inventory'
    action      VARCHAR(100) NOT NULL,   -- e.g. 'view', 'create', 'edit', 'delete', 'approve', 'post'
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_permissions_name_guard UNIQUE (name, guard_name)
);

CREATE INDEX idx_permissions_module ON permissions (module);
CREATE INDEX idx_permissions_action ON permissions (action);
```

**Sample permission names:** `ap.vendor_invoices.view`, `ap.vendor_invoices.create`, `ap.vendor_invoices.approve`, `gl.journal_entries.post`, `ar.customer_invoices.delete`.

---

### `role_has_permissions` (Spatie standard)

```sql
CREATE TABLE role_has_permissions (
    permission_id  BIGINT NOT NULL REFERENCES permissions (id) ON DELETE CASCADE,
    role_id        BIGINT NOT NULL REFERENCES roles (id) ON DELETE CASCADE,

    PRIMARY KEY (permission_id, role_id)
);

CREATE INDEX idx_rhp_role_id ON role_has_permissions (role_id);
```

---

### `model_has_roles` (Spatie standard)

```sql
CREATE TABLE model_has_roles (
    role_id     BIGINT       NOT NULL REFERENCES roles (id) ON DELETE CASCADE,
    model_type  VARCHAR(255) NOT NULL,
    model_id    BIGINT       NOT NULL,

    PRIMARY KEY (role_id, model_id, model_type)
);

CREATE INDEX idx_mhr_model ON model_has_roles (model_type, model_id);
```

---

### `model_has_permissions` (Spatie standard)

```sql
CREATE TABLE model_has_permissions (
    permission_id  BIGINT       NOT NULL REFERENCES permissions (id) ON DELETE CASCADE,
    model_type     VARCHAR(255) NOT NULL,
    model_id       BIGINT       NOT NULL,

    PRIMARY KEY (permission_id, model_id, model_type)
);

CREATE INDEX idx_mhp_model ON model_has_permissions (model_type, model_id);
```

---

## 3. Chart of Accounts / General Ledger

### `account_types`

```sql
CREATE TABLE account_types (
    id                  BIGSERIAL PRIMARY KEY,
    company_id          BIGINT       NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    code                VARCHAR(20)  NOT NULL,
    name                VARCHAR(100) NOT NULL,
    normal_balance      VARCHAR(10)  NOT NULL CHECK (normal_balance IN ('debit', 'credit')),
    financial_statement VARCHAR(10)  NOT NULL CHECK (financial_statement IN ('BS', 'IS', 'CF')),

    CONSTRAINT uq_account_types_company_code UNIQUE (company_id, code)
);

CREATE INDEX idx_account_types_company_id ON account_types (company_id);

-- Standard seed data per company:
-- Assets         (debit,  BS)
-- Liabilities    (credit, BS)
-- Equity         (credit, BS)
-- Revenue        (credit, IS)
-- Cost of Sales  (debit,  IS)
-- Expenses       (debit,  IS)
-- Other Income   (credit, IS)
-- Other Expenses (debit,  IS)
```

---

### `accounts`

```sql
CREATE TABLE accounts (
    id                  BIGSERIAL PRIMARY KEY,
    company_id          BIGINT       NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    code                VARCHAR(20)  NOT NULL,
    name                VARCHAR(255) NOT NULL,
    account_type_id     BIGINT       NOT NULL REFERENCES account_types (id),
    parent_id           BIGINT       REFERENCES accounts (id) ON DELETE SET NULL,
    currency_code       CHAR(3)      REFERENCES currencies (code),
    is_active           BOOLEAN      NOT NULL DEFAULT TRUE,
    allow_direct_posting BOOLEAN     NOT NULL DEFAULT TRUE,
    description         TEXT,
    created_at          TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,

    CONSTRAINT uq_accounts_company_code UNIQUE (company_id, code),
    CONSTRAINT chk_accounts_no_self_parent CHECK (parent_id <> id)
);

CREATE INDEX idx_accounts_company_id       ON accounts (company_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_accounts_account_type_id  ON accounts (account_type_id);
CREATE INDEX idx_accounts_parent_id        ON accounts (parent_id) WHERE parent_id IS NOT NULL;
CREATE INDEX idx_accounts_is_active        ON accounts (company_id, is_active) WHERE deleted_at IS NULL;
CREATE INDEX idx_accounts_code_search      ON accounts (company_id, code, name);
```

**Notes:** Hierarchical CoA supported via `parent_id`. Leaf accounts (those with `allow_direct_posting = TRUE`) are the only ones that can appear on journal entry lines.

---

### `cost_centers`

```sql
CREATE TABLE cost_centers (
    id          BIGSERIAL PRIMARY KEY,
    company_id  BIGINT       NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    code        VARCHAR(20)  NOT NULL,
    name        VARCHAR(255) NOT NULL,
    parent_id   BIGINT       REFERENCES cost_centers (id) ON DELETE SET NULL,
    is_active   BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_cost_centers_company_code UNIQUE (company_id, code),
    CONSTRAINT chk_cost_centers_no_self_parent CHECK (parent_id <> id)
);

CREATE INDEX idx_cost_centers_company_id ON cost_centers (company_id);
CREATE INDEX idx_cost_centers_parent_id  ON cost_centers (parent_id) WHERE parent_id IS NOT NULL;
```

---

### `journal_entries`

```sql
CREATE TABLE journal_entries (
    id             BIGSERIAL PRIMARY KEY,
    company_id     BIGINT       NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    branch_id      BIGINT       REFERENCES branches (id) ON DELETE SET NULL,
    fiscal_period_id BIGINT     REFERENCES fiscal_periods (id),
    entry_number   VARCHAR(50)  NOT NULL,
    entry_date     DATE         NOT NULL,
    description    TEXT,
    reference      VARCHAR(255),
    source_module  VARCHAR(50),   -- 'AP', 'AR', 'Bank', 'Asset', 'Inventory', 'Manual'
    source_id      BIGINT,        -- FK to the originating record (polymorphic)
    status         VARCHAR(20)  NOT NULL DEFAULT 'draft'
                       CHECK (status IN ('draft', 'posted', 'reversed')),
    is_recurring   BOOLEAN      NOT NULL DEFAULT FALSE,
    created_by     BIGINT       NOT NULL REFERENCES users (id),
    posted_at      TIMESTAMPTZ,
    posted_by      BIGINT       REFERENCES users (id),
    reversed_by    BIGINT       REFERENCES users (id),
    reversed_at    TIMESTAMPTZ,
    reversal_of_id BIGINT       REFERENCES journal_entries (id),
    created_at     TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at     TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_journal_entries_company_number UNIQUE (company_id, entry_number)
);

CREATE INDEX idx_je_company_id       ON journal_entries (company_id);
CREATE INDEX idx_je_branch_id        ON journal_entries (branch_id);
CREATE INDEX idx_je_entry_date       ON journal_entries (company_id, entry_date DESC);
CREATE INDEX idx_je_status           ON journal_entries (company_id, status);
CREATE INDEX idx_je_source           ON journal_entries (source_module, source_id) WHERE source_id IS NOT NULL;
CREATE INDEX idx_je_fiscal_period    ON journal_entries (fiscal_period_id);
CREATE INDEX idx_je_created_by       ON journal_entries (created_by);
```

---

### `journal_entry_lines`

```sql
CREATE TABLE journal_entry_lines (
    id                BIGSERIAL PRIMARY KEY,
    journal_entry_id  BIGINT        NOT NULL REFERENCES journal_entries (id) ON DELETE CASCADE,
    account_id        BIGINT        NOT NULL REFERENCES accounts (id),
    description       TEXT,
    debit             NUMERIC(20,4) NOT NULL DEFAULT 0 CHECK (debit >= 0),
    credit            NUMERIC(20,4) NOT NULL DEFAULT 0 CHECK (credit >= 0),
    currency_code     CHAR(3)       NOT NULL REFERENCES currencies (code),
    exchange_rate     NUMERIC(20,8) NOT NULL DEFAULT 1,
    debit_base        NUMERIC(20,4) NOT NULL DEFAULT 0,  -- debit * exchange_rate
    credit_base       NUMERIC(20,4) NOT NULL DEFAULT 0,  -- credit * exchange_rate
    cost_center_id    BIGINT        REFERENCES cost_centers (id) ON DELETE SET NULL,
    line_order        SMALLINT      NOT NULL DEFAULT 0,

    CONSTRAINT chk_jel_debit_or_credit
        CHECK (
            (debit > 0 AND credit = 0) OR
            (credit > 0 AND debit = 0) OR
            (debit = 0 AND credit = 0)  -- zero line allowed for templates
        )
);

CREATE INDEX idx_jel_journal_entry_id ON journal_entry_lines (journal_entry_id);
CREATE INDEX idx_jel_account_id       ON journal_entry_lines (account_id);
CREATE INDEX idx_jel_cost_center_id   ON journal_entry_lines (cost_center_id) WHERE cost_center_id IS NOT NULL;

-- Enforce double-entry balance at DB level via a deferred constraint trigger
-- (implemented as a PostgreSQL constraint trigger or enforced in service layer)
```

**Notes:** `debit_base` / `credit_base` are always in the company's base currency. The application enforces `SUM(debit_base) = SUM(credit_base)` per journal entry before posting.

---

## 4. Accounts Payable

### `vendors`

```sql
CREATE TABLE vendors (
    id              BIGSERIAL PRIMARY KEY,
    company_id      BIGINT        NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    code            VARCHAR(20)   NOT NULL,
    name            VARCHAR(255)  NOT NULL,
    tax_number      VARCHAR(100),
    currency_code   CHAR(3)       NOT NULL DEFAULT 'USD' REFERENCES currencies (code),
    payment_terms   SMALLINT      NOT NULL DEFAULT 30,   -- days
    credit_limit    NUMERIC(20,4) DEFAULT 0,
    contact         JSONB         NOT NULL DEFAULT '{}', -- {name, email, phone, fax}
    address         JSONB         NOT NULL DEFAULT '{}',
    bank_details    JSONB         NOT NULL DEFAULT '{}', -- {bank_name, account_name, account_number, routing, swift}
    is_active       BOOLEAN       NOT NULL DEFAULT TRUE,
    notes           TEXT,
    created_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ,

    CONSTRAINT uq_vendors_company_code UNIQUE (company_id, code)
);

CREATE INDEX idx_vendors_company_id  ON vendors (company_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_vendors_is_active   ON vendors (company_id, is_active) WHERE deleted_at IS NULL;
CREATE INDEX idx_vendors_name_search ON vendors USING GIN (to_tsvector('simple', name));
```

---

### `vendor_invoices`

```sql
CREATE TABLE vendor_invoices (
    id                    BIGSERIAL PRIMARY KEY,
    company_id            BIGINT        NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    branch_id             BIGINT        REFERENCES branches (id) ON DELETE SET NULL,
    vendor_id             BIGINT        NOT NULL REFERENCES vendors (id),
    fiscal_period_id      BIGINT        REFERENCES fiscal_periods (id),
    invoice_number        VARCHAR(100)  NOT NULL,
    vendor_invoice_number VARCHAR(100),
    invoice_date          DATE          NOT NULL,
    due_date              DATE          NOT NULL,
    currency_code         CHAR(3)       NOT NULL REFERENCES currencies (code),
    exchange_rate         NUMERIC(20,8) NOT NULL DEFAULT 1,
    subtotal              NUMERIC(20,4) NOT NULL DEFAULT 0,
    tax_amount            NUMERIC(20,4) NOT NULL DEFAULT 0,
    total_amount          NUMERIC(20,4) NOT NULL DEFAULT 0,
    paid_amount           NUMERIC(20,4) NOT NULL DEFAULT 0,
    status                VARCHAR(30)   NOT NULL DEFAULT 'draft'
                              CHECK (status IN ('draft','approved','partially_paid','paid','cancelled')),
    notes                 TEXT,
    journal_entry_id      BIGINT        REFERENCES journal_entries (id),
    approved_by           BIGINT        REFERENCES users (id),
    approved_at           TIMESTAMPTZ,
    created_by            BIGINT        NOT NULL REFERENCES users (id),
    created_at            TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at            TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    deleted_at            TIMESTAMPTZ,

    CONSTRAINT uq_vendor_invoices_company_number UNIQUE (company_id, invoice_number),
    CONSTRAINT chk_vendor_invoices_due_date CHECK (due_date >= invoice_date),
    CONSTRAINT chk_vendor_invoices_amounts CHECK (
        total_amount = subtotal + tax_amount AND
        paid_amount >= 0 AND
        paid_amount <= total_amount
    )
);

CREATE INDEX idx_vi_company_id    ON vendor_invoices (company_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_vi_vendor_id     ON vendor_invoices (vendor_id);
CREATE INDEX idx_vi_status        ON vendor_invoices (company_id, status);
CREATE INDEX idx_vi_due_date      ON vendor_invoices (company_id, due_date);
CREATE INDEX idx_vi_invoice_date  ON vendor_invoices (company_id, invoice_date DESC);
CREATE INDEX idx_vi_je            ON vendor_invoices (journal_entry_id) WHERE journal_entry_id IS NOT NULL;
```

---

### `vendor_invoice_lines`

```sql
CREATE TABLE vendor_invoice_lines (
    id                 BIGSERIAL PRIMARY KEY,
    vendor_invoice_id  BIGINT        NOT NULL REFERENCES vendor_invoices (id) ON DELETE CASCADE,
    account_id         BIGINT        NOT NULL REFERENCES accounts (id),
    description        TEXT,
    quantity           NUMERIC(20,4) NOT NULL DEFAULT 1,
    unit_price         NUMERIC(20,4) NOT NULL DEFAULT 0,
    amount             NUMERIC(20,4) NOT NULL DEFAULT 0,
    tax_code_id        BIGINT        REFERENCES tax_codes (id),
    tax_amount         NUMERIC(20,4) NOT NULL DEFAULT 0,
    cost_center_id     BIGINT        REFERENCES cost_centers (id),
    line_order         SMALLINT      NOT NULL DEFAULT 0,

    CONSTRAINT chk_vil_amount CHECK (amount >= 0)
);

CREATE INDEX idx_vil_invoice_id ON vendor_invoice_lines (vendor_invoice_id);
CREATE INDEX idx_vil_account_id ON vendor_invoice_lines (account_id);
```

---

### `vendor_payments`

```sql
CREATE TABLE vendor_payments (
    id               BIGSERIAL PRIMARY KEY,
    company_id       BIGINT        NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    branch_id        BIGINT        REFERENCES branches (id) ON DELETE SET NULL,
    vendor_id        BIGINT        NOT NULL REFERENCES vendors (id),
    fiscal_period_id BIGINT        REFERENCES fiscal_periods (id),
    payment_number   VARCHAR(100)  NOT NULL,
    payment_date     DATE          NOT NULL,
    amount           NUMERIC(20,4) NOT NULL CHECK (amount > 0),
    currency_code    CHAR(3)       NOT NULL REFERENCES currencies (code),
    exchange_rate    NUMERIC(20,8) NOT NULL DEFAULT 1,
    payment_method   VARCHAR(50)   NOT NULL DEFAULT 'bank_transfer'
                         CHECK (payment_method IN ('bank_transfer','cheque','cash','credit_card','other')),
    bank_account_id  BIGINT        REFERENCES bank_accounts (id),
    reference        VARCHAR(255),
    notes            TEXT,
    journal_entry_id BIGINT        REFERENCES journal_entries (id),
    status           VARCHAR(30)   NOT NULL DEFAULT 'draft'
                         CHECK (status IN ('draft','posted','voided')),
    created_by       BIGINT        NOT NULL REFERENCES users (id),
    created_at       TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ   NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_vendor_payments_company_number UNIQUE (company_id, payment_number)
);

CREATE INDEX idx_vp_company_id   ON vendor_payments (company_id);
CREATE INDEX idx_vp_vendor_id    ON vendor_payments (vendor_id);
CREATE INDEX idx_vp_payment_date ON vendor_payments (company_id, payment_date DESC);
CREATE INDEX idx_vp_status       ON vendor_payments (company_id, status);
```

---

### `vendor_payment_allocations`

```sql
CREATE TABLE vendor_payment_allocations (
    id                 BIGSERIAL PRIMARY KEY,
    vendor_payment_id  BIGINT        NOT NULL REFERENCES vendor_payments (id) ON DELETE CASCADE,
    vendor_invoice_id  BIGINT        NOT NULL REFERENCES vendor_invoices (id),
    allocated_amount   NUMERIC(20,4) NOT NULL CHECK (allocated_amount > 0),
    created_at         TIMESTAMPTZ   NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_vpa_payment_invoice UNIQUE (vendor_payment_id, vendor_invoice_id)
);

CREATE INDEX idx_vpa_payment_id ON vendor_payment_allocations (vendor_payment_id);
CREATE INDEX idx_vpa_invoice_id ON vendor_payment_allocations (vendor_invoice_id);
```

---

## 5. Accounts Receivable

### `customers`

```sql
CREATE TABLE customers (
    id              BIGSERIAL PRIMARY KEY,
    company_id      BIGINT        NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    code            VARCHAR(20)   NOT NULL,
    name            VARCHAR(255)  NOT NULL,
    tax_number      VARCHAR(100),
    currency_code   CHAR(3)       NOT NULL DEFAULT 'USD' REFERENCES currencies (code),
    payment_terms   SMALLINT      NOT NULL DEFAULT 30,
    credit_limit    NUMERIC(20,4) DEFAULT 0,
    contact         JSONB         NOT NULL DEFAULT '{}',
    address         JSONB         NOT NULL DEFAULT '{}',
    is_active       BOOLEAN       NOT NULL DEFAULT TRUE,
    notes           TEXT,
    created_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ,

    CONSTRAINT uq_customers_company_code UNIQUE (company_id, code)
);

CREATE INDEX idx_customers_company_id  ON customers (company_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_customers_is_active   ON customers (company_id, is_active) WHERE deleted_at IS NULL;
CREATE INDEX idx_customers_name_search ON customers USING GIN (to_tsvector('simple', name));
```

---

### `customer_invoices`

```sql
CREATE TABLE customer_invoices (
    id                BIGSERIAL PRIMARY KEY,
    company_id        BIGINT        NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    branch_id         BIGINT        REFERENCES branches (id) ON DELETE SET NULL,
    customer_id       BIGINT        NOT NULL REFERENCES customers (id),
    fiscal_period_id  BIGINT        REFERENCES fiscal_periods (id),
    sales_order_id    BIGINT        REFERENCES sales_orders (id),
    invoice_number    VARCHAR(100)  NOT NULL,
    invoice_date      DATE          NOT NULL,
    due_date          DATE          NOT NULL,
    currency_code     CHAR(3)       NOT NULL REFERENCES currencies (code),
    exchange_rate     NUMERIC(20,8) NOT NULL DEFAULT 1,
    subtotal          NUMERIC(20,4) NOT NULL DEFAULT 0,
    tax_amount        NUMERIC(20,4) NOT NULL DEFAULT 0,
    discount_amount   NUMERIC(20,4) NOT NULL DEFAULT 0,
    total_amount      NUMERIC(20,4) NOT NULL DEFAULT 0,
    paid_amount       NUMERIC(20,4) NOT NULL DEFAULT 0,
    status            VARCHAR(30)   NOT NULL DEFAULT 'draft'
                          CHECK (status IN ('draft','approved','sent','partially_paid','paid','overdue','cancelled')),
    notes             TEXT,
    journal_entry_id  BIGINT        REFERENCES journal_entries (id),
    approved_by       BIGINT        REFERENCES users (id),
    approved_at       TIMESTAMPTZ,
    created_by        BIGINT        NOT NULL REFERENCES users (id),
    created_at        TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    deleted_at        TIMESTAMPTZ,

    CONSTRAINT uq_customer_invoices_company_number UNIQUE (company_id, invoice_number),
    CONSTRAINT chk_customer_invoices_due_date CHECK (due_date >= invoice_date),
    CONSTRAINT chk_customer_invoices_amounts CHECK (
        paid_amount >= 0 AND paid_amount <= total_amount
    )
);

CREATE INDEX idx_ci_company_id    ON customer_invoices (company_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_ci_customer_id   ON customer_invoices (customer_id);
CREATE INDEX idx_ci_status        ON customer_invoices (company_id, status);
CREATE INDEX idx_ci_due_date      ON customer_invoices (company_id, due_date);
CREATE INDEX idx_ci_invoice_date  ON customer_invoices (company_id, invoice_date DESC);
CREATE INDEX idx_ci_sales_order   ON customer_invoices (sales_order_id) WHERE sales_order_id IS NOT NULL;
```

---

### `customer_invoice_lines`

```sql
CREATE TABLE customer_invoice_lines (
    id                   BIGSERIAL PRIMARY KEY,
    customer_invoice_id  BIGINT        NOT NULL REFERENCES customer_invoices (id) ON DELETE CASCADE,
    item_id              BIGINT        REFERENCES items (id),
    account_id           BIGINT        NOT NULL REFERENCES accounts (id),
    description          TEXT,
    quantity             NUMERIC(20,4) NOT NULL DEFAULT 1,
    unit_price           NUMERIC(20,4) NOT NULL DEFAULT 0,
    discount_pct         NUMERIC(5,2)  NOT NULL DEFAULT 0 CHECK (discount_pct BETWEEN 0 AND 100),
    amount               NUMERIC(20,4) NOT NULL DEFAULT 0,
    tax_code_id          BIGINT        REFERENCES tax_codes (id),
    tax_amount           NUMERIC(20,4) NOT NULL DEFAULT 0,
    cost_center_id       BIGINT        REFERENCES cost_centers (id),
    line_order           SMALLINT      NOT NULL DEFAULT 0
);

CREATE INDEX idx_cil_invoice_id ON customer_invoice_lines (customer_invoice_id);
CREATE INDEX idx_cil_item_id    ON customer_invoice_lines (item_id) WHERE item_id IS NOT NULL;
CREATE INDEX idx_cil_account_id ON customer_invoice_lines (account_id);
```

---

### `customer_payments`

```sql
CREATE TABLE customer_payments (
    id               BIGSERIAL PRIMARY KEY,
    company_id       BIGINT        NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    branch_id        BIGINT        REFERENCES branches (id) ON DELETE SET NULL,
    customer_id      BIGINT        NOT NULL REFERENCES customers (id),
    fiscal_period_id BIGINT        REFERENCES fiscal_periods (id),
    payment_number   VARCHAR(100)  NOT NULL,
    payment_date     DATE          NOT NULL,
    amount           NUMERIC(20,4) NOT NULL CHECK (amount > 0),
    currency_code    CHAR(3)       NOT NULL REFERENCES currencies (code),
    exchange_rate    NUMERIC(20,8) NOT NULL DEFAULT 1,
    payment_method   VARCHAR(50)   NOT NULL DEFAULT 'bank_transfer'
                         CHECK (payment_method IN ('bank_transfer','cheque','cash','credit_card','other')),
    bank_account_id  BIGINT        REFERENCES bank_accounts (id),
    reference        VARCHAR(255),
    notes            TEXT,
    journal_entry_id BIGINT        REFERENCES journal_entries (id),
    status           VARCHAR(30)   NOT NULL DEFAULT 'draft'
                         CHECK (status IN ('draft','posted','voided')),
    created_by       BIGINT        NOT NULL REFERENCES users (id),
    created_at       TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ   NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_customer_payments_company_number UNIQUE (company_id, payment_number)
);

CREATE INDEX idx_cp_company_id   ON customer_payments (company_id);
CREATE INDEX idx_cp_customer_id  ON customer_payments (customer_id);
CREATE INDEX idx_cp_payment_date ON customer_payments (company_id, payment_date DESC);
CREATE INDEX idx_cp_status       ON customer_payments (company_id, status);
```

---

### `customer_payment_allocations`

```sql
CREATE TABLE customer_payment_allocations (
    id                    BIGSERIAL PRIMARY KEY,
    customer_payment_id   BIGINT        NOT NULL REFERENCES customer_payments (id) ON DELETE CASCADE,
    customer_invoice_id   BIGINT        NOT NULL REFERENCES customer_invoices (id),
    allocated_amount      NUMERIC(20,4) NOT NULL CHECK (allocated_amount > 0),
    created_at            TIMESTAMPTZ   NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_cpa_payment_invoice UNIQUE (customer_payment_id, customer_invoice_id)
);

CREATE INDEX idx_cpa_payment_id ON customer_payment_allocations (customer_payment_id);
CREATE INDEX idx_cpa_invoice_id ON customer_payment_allocations (customer_invoice_id);
```

---

## 6. Cash & Bank

### `bank_accounts`

```sql
CREATE TABLE bank_accounts (
    id               BIGSERIAL PRIMARY KEY,
    company_id       BIGINT        NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    branch_id        BIGINT        REFERENCES branches (id) ON DELETE SET NULL,
    account_id       BIGINT        NOT NULL REFERENCES accounts (id),  -- GL account
    name             VARCHAR(255)  NOT NULL,
    bank_name        VARCHAR(255),
    account_number   VARCHAR(100),
    currency_code    CHAR(3)       NOT NULL REFERENCES currencies (code),
    opening_balance  NUMERIC(20,4) NOT NULL DEFAULT 0,
    opening_date     DATE,
    is_active        BOOLEAN       NOT NULL DEFAULT TRUE,
    created_at       TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ba_company_id ON bank_accounts (company_id);
CREATE INDEX idx_ba_account_id ON bank_accounts (account_id);
CREATE INDEX idx_ba_is_active  ON bank_accounts (company_id, is_active);
```

---

### `bank_transactions`

```sql
CREATE TABLE bank_transactions (
    id                  BIGSERIAL PRIMARY KEY,
    bank_account_id     BIGINT        NOT NULL REFERENCES bank_accounts (id) ON DELETE CASCADE,
    transaction_date    DATE          NOT NULL,
    description         TEXT          NOT NULL,
    amount              NUMERIC(20,4) NOT NULL,
    type                VARCHAR(10)   NOT NULL CHECK (type IN ('debit', 'credit')),
    reference           VARCHAR(255),
    reconciled          BOOLEAN       NOT NULL DEFAULT FALSE,
    reconciliation_id   BIGINT        REFERENCES bank_reconciliations (id) ON DELETE SET NULL,
    journal_entry_id    BIGINT        REFERENCES journal_entries (id),
    created_at          TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_bt_bank_account_id    ON bank_transactions (bank_account_id);
CREATE INDEX idx_bt_transaction_date   ON bank_transactions (bank_account_id, transaction_date DESC);
CREATE INDEX idx_bt_reconciled         ON bank_transactions (bank_account_id, reconciled);
CREATE INDEX idx_bt_reconciliation_id  ON bank_transactions (reconciliation_id) WHERE reconciliation_id IS NOT NULL;
```

---

### `bank_reconciliations`

```sql
CREATE TABLE bank_reconciliations (
    id                       BIGSERIAL PRIMARY KEY,
    bank_account_id          BIGINT        NOT NULL REFERENCES bank_accounts (id) ON DELETE CASCADE,
    statement_date           DATE          NOT NULL,
    statement_ending_balance NUMERIC(20,4) NOT NULL,
    reconciled_balance       NUMERIC(20,4) NOT NULL DEFAULT 0,
    status                   VARCHAR(20)   NOT NULL DEFAULT 'in_progress'
                                 CHECK (status IN ('in_progress', 'completed')),
    completed_at             TIMESTAMPTZ,
    completed_by             BIGINT        REFERENCES users (id),
    created_by               BIGINT        NOT NULL REFERENCES users (id),
    created_at               TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at               TIMESTAMPTZ   NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_bank_reconciliations_account_date UNIQUE (bank_account_id, statement_date)
);

CREATE INDEX idx_br_bank_account_id ON bank_reconciliations (bank_account_id);
CREATE INDEX idx_br_status          ON bank_reconciliations (bank_account_id, status);
```

---

### `cash_transfers`

```sql
CREATE TABLE cash_transfers (
    id                   BIGSERIAL PRIMARY KEY,
    company_id           BIGINT        NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    from_bank_account_id BIGINT        NOT NULL REFERENCES bank_accounts (id),
    to_bank_account_id   BIGINT        NOT NULL REFERENCES bank_accounts (id),
    amount               NUMERIC(20,4) NOT NULL CHECK (amount > 0),
    transfer_date        DATE          NOT NULL,
    reference            VARCHAR(255),
    notes                TEXT,
    journal_entry_id     BIGINT        REFERENCES journal_entries (id),
    status               VARCHAR(20)   NOT NULL DEFAULT 'draft'
                             CHECK (status IN ('draft','posted','voided')),
    created_by           BIGINT        NOT NULL REFERENCES users (id),
    created_at           TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at           TIMESTAMPTZ   NOT NULL DEFAULT NOW(),

    CONSTRAINT chk_cash_transfers_different_accounts
        CHECK (from_bank_account_id <> to_bank_account_id)
);

CREATE INDEX idx_ct_company_id          ON cash_transfers (company_id);
CREATE INDEX idx_ct_from_bank_account   ON cash_transfers (from_bank_account_id);
CREATE INDEX idx_ct_to_bank_account     ON cash_transfers (to_bank_account_id);
CREATE INDEX idx_ct_transfer_date       ON cash_transfers (company_id, transfer_date DESC);
```

---

## 7. Fixed Assets

### `asset_categories`

```sql
CREATE TABLE asset_categories (
    id                          BIGSERIAL PRIMARY KEY,
    company_id                  BIGINT        NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    name                        VARCHAR(255)  NOT NULL,
    asset_account_id            BIGINT        NOT NULL REFERENCES accounts (id),
    depreciation_account_id     BIGINT        NOT NULL REFERENCES accounts (id),
    accumulated_dep_account_id  BIGINT        NOT NULL REFERENCES accounts (id),
    depreciation_method         VARCHAR(50)   NOT NULL DEFAULT 'straight_line'
                                    CHECK (depreciation_method IN ('straight_line','declining_balance','sum_of_years','units_of_production')),
    useful_life_months          SMALLINT      NOT NULL CHECK (useful_life_months > 0),
    residual_value_pct          NUMERIC(5,2)  NOT NULL DEFAULT 0
                                    CHECK (residual_value_pct BETWEEN 0 AND 100),
    created_at                  TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at                  TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ac_company_id ON asset_categories (company_id);
```

---

### `fixed_assets`

```sql
CREATE TABLE fixed_assets (
    id                       BIGSERIAL PRIMARY KEY,
    company_id               BIGINT        NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    branch_id                BIGINT        REFERENCES branches (id) ON DELETE SET NULL,
    asset_category_id        BIGINT        NOT NULL REFERENCES asset_categories (id),
    code                     VARCHAR(50)   NOT NULL,
    name                     VARCHAR(255)  NOT NULL,
    description              TEXT,
    purchase_date            DATE          NOT NULL,
    in_service_date          DATE,
    purchase_cost            NUMERIC(20,4) NOT NULL CHECK (purchase_cost >= 0),
    residual_value           NUMERIC(20,4) NOT NULL DEFAULT 0 CHECK (residual_value >= 0),
    useful_life_months       SMALLINT      NOT NULL CHECK (useful_life_months > 0),
    depreciation_method      VARCHAR(50)   NOT NULL DEFAULT 'straight_line'
                                 CHECK (depreciation_method IN ('straight_line','declining_balance','sum_of_years','units_of_production')),
    accumulated_depreciation NUMERIC(20,4) NOT NULL DEFAULT 0,
    book_value               NUMERIC(20,4) NOT NULL DEFAULT 0,
    status                   VARCHAR(30)   NOT NULL DEFAULT 'active'
                                 CHECK (status IN ('active','disposed','fully_depreciated')),
    disposal_date            DATE,
    disposal_amount          NUMERIC(20,4),
    disposal_journal_entry_id BIGINT       REFERENCES journal_entries (id),
    vendor_id                BIGINT        REFERENCES vendors (id),
    serial_number            VARCHAR(100),
    location                 VARCHAR(255),
    purchase_invoice_id      BIGINT        REFERENCES vendor_invoices (id),
    created_by               BIGINT        NOT NULL REFERENCES users (id),
    created_at               TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at               TIMESTAMPTZ   NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_fixed_assets_company_code UNIQUE (company_id, code),
    CONSTRAINT chk_fixed_assets_residual CHECK (residual_value <= purchase_cost)
);

CREATE INDEX idx_fa_company_id         ON fixed_assets (company_id);
CREATE INDEX idx_fa_asset_category_id  ON fixed_assets (asset_category_id);
CREATE INDEX idx_fa_status             ON fixed_assets (company_id, status);
CREATE INDEX idx_fa_branch_id          ON fixed_assets (branch_id) WHERE branch_id IS NOT NULL;
```

---

### `asset_depreciation_schedules`

```sql
CREATE TABLE asset_depreciation_schedules (
    id                       BIGSERIAL PRIMARY KEY,
    fixed_asset_id           BIGINT        NOT NULL REFERENCES fixed_assets (id) ON DELETE CASCADE,
    period_date              DATE          NOT NULL,
    depreciation_amount      NUMERIC(20,4) NOT NULL CHECK (depreciation_amount >= 0),
    accumulated_depreciation NUMERIC(20,4) NOT NULL CHECK (accumulated_depreciation >= 0),
    book_value               NUMERIC(20,4) NOT NULL CHECK (book_value >= 0),
    journal_entry_id         BIGINT        REFERENCES journal_entries (id),
    posted                   BOOLEAN       NOT NULL DEFAULT FALSE,
    created_at               TIMESTAMPTZ   NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_ads_asset_period UNIQUE (fixed_asset_id, period_date)
);

CREATE INDEX idx_ads_fixed_asset_id ON asset_depreciation_schedules (fixed_asset_id);
CREATE INDEX idx_ads_period_date    ON asset_depreciation_schedules (period_date);
CREATE INDEX idx_ads_posted         ON asset_depreciation_schedules (fixed_asset_id, posted);
```

---

## 8. Inventory

### `warehouses`

```sql
CREATE TABLE warehouses (
    id          BIGSERIAL PRIMARY KEY,
    company_id  BIGINT       NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    branch_id   BIGINT       REFERENCES branches (id) ON DELETE SET NULL,
    code        VARCHAR(20)  NOT NULL,
    name        VARCHAR(255) NOT NULL,
    address     JSONB        NOT NULL DEFAULT '{}',
    is_active   BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_warehouses_company_code UNIQUE (company_id, code)
);

CREATE INDEX idx_warehouses_company_id ON warehouses (company_id);
CREATE INDEX idx_warehouses_branch_id  ON warehouses (branch_id) WHERE branch_id IS NOT NULL;
```

---

### `item_categories`

```sql
CREATE TABLE item_categories (
    id          BIGSERIAL PRIMARY KEY,
    company_id  BIGINT       NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    name        VARCHAR(255) NOT NULL,
    parent_id   BIGINT       REFERENCES item_categories (id) ON DELETE SET NULL,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    CONSTRAINT chk_item_categories_no_self_parent CHECK (parent_id <> id)
);

CREATE INDEX idx_item_categories_company_id ON item_categories (company_id);
CREATE INDEX idx_item_categories_parent_id  ON item_categories (parent_id) WHERE parent_id IS NOT NULL;
```

---

### `items`

```sql
CREATE TABLE items (
    id                   BIGSERIAL PRIMARY KEY,
    company_id           BIGINT        NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    code                 VARCHAR(50)   NOT NULL,
    name                 VARCHAR(255)  NOT NULL,
    description          TEXT,
    item_category_id     BIGINT        REFERENCES item_categories (id),
    unit_of_measure      VARCHAR(50)   NOT NULL DEFAULT 'unit',
    item_type            VARCHAR(20)   NOT NULL DEFAULT 'stock'
                             CHECK (item_type IN ('stock','service','expense')),
    purchase_account_id  BIGINT        REFERENCES accounts (id),
    sales_account_id     BIGINT        REFERENCES accounts (id),
    inventory_account_id BIGINT        REFERENCES accounts (id),
    cogs_account_id      BIGINT        REFERENCES accounts (id),
    standard_cost        NUMERIC(20,4) NOT NULL DEFAULT 0,
    sales_price          NUMERIC(20,4) NOT NULL DEFAULT 0,
    tax_code_id          BIGINT        REFERENCES tax_codes (id),
    is_active            BOOLEAN       NOT NULL DEFAULT TRUE,
    track_inventory      BOOLEAN       NOT NULL DEFAULT TRUE,
    barcode              VARCHAR(100),
    created_at           TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at           TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    deleted_at           TIMESTAMPTZ,

    CONSTRAINT uq_items_company_code UNIQUE (company_id, code)
);

CREATE INDEX idx_items_company_id      ON items (company_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_items_item_category   ON items (item_category_id);
CREATE INDEX idx_items_item_type       ON items (company_id, item_type);
CREATE INDEX idx_items_is_active       ON items (company_id, is_active) WHERE deleted_at IS NULL;
CREATE INDEX idx_items_name_search     ON items USING GIN (to_tsvector('simple', name));
```

---

### `item_stock`

```sql
CREATE TABLE item_stock (
    id                BIGSERIAL PRIMARY KEY,
    item_id           BIGINT        NOT NULL REFERENCES items (id) ON DELETE CASCADE,
    warehouse_id      BIGINT        NOT NULL REFERENCES warehouses (id) ON DELETE CASCADE,
    quantity_on_hand  NUMERIC(20,4) NOT NULL DEFAULT 0,
    quantity_on_order NUMERIC(20,4) NOT NULL DEFAULT 0,
    quantity_reserved NUMERIC(20,4) NOT NULL DEFAULT 0,
    average_cost      NUMERIC(20,8) NOT NULL DEFAULT 0,
    updated_at        TIMESTAMPTZ   NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_item_stock_item_warehouse UNIQUE (item_id, warehouse_id),
    CONSTRAINT chk_item_stock_quantities CHECK (
        quantity_on_hand >= 0 AND
        quantity_on_order >= 0 AND
        quantity_reserved >= 0
    )
);

CREATE INDEX idx_item_stock_item_id      ON item_stock (item_id);
CREATE INDEX idx_item_stock_warehouse_id ON item_stock (warehouse_id);
```

---

### `stock_movements`

```sql
CREATE TABLE stock_movements (
    id              BIGSERIAL PRIMARY KEY,
    company_id      BIGINT        NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    item_id         BIGINT        NOT NULL REFERENCES items (id),
    warehouse_id    BIGINT        NOT NULL REFERENCES warehouses (id),
    movement_type   VARCHAR(30)   NOT NULL
                        CHECK (movement_type IN ('purchase','sale','transfer_in','transfer_out','adjustment_in','adjustment_out','opening')),
    reference_type  VARCHAR(100),   -- 'GoodsReceipt', 'DeliveryOrder', 'StockAdjustment'
    reference_id    BIGINT,
    quantity        NUMERIC(20,4) NOT NULL,
    unit_cost       NUMERIC(20,8) NOT NULL DEFAULT 0,
    total_cost      NUMERIC(20,4) NOT NULL DEFAULT 0,
    movement_date   DATE          NOT NULL,
    journal_entry_id BIGINT       REFERENCES journal_entries (id),
    notes           TEXT,
    created_by      BIGINT        REFERENCES users (id),
    created_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sm_company_id      ON stock_movements (company_id);
CREATE INDEX idx_sm_item_id         ON stock_movements (item_id);
CREATE INDEX idx_sm_warehouse_id    ON stock_movements (warehouse_id);
CREATE INDEX idx_sm_movement_type   ON stock_movements (movement_type);
CREATE INDEX idx_sm_reference       ON stock_movements (reference_type, reference_id) WHERE reference_id IS NOT NULL;
CREATE INDEX idx_sm_movement_date   ON stock_movements (company_id, movement_date DESC);
```

---

## 9. Purchasing

### `purchase_orders`

```sql
CREATE TABLE purchase_orders (
    id                BIGSERIAL PRIMARY KEY,
    company_id        BIGINT        NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    branch_id         BIGINT        REFERENCES branches (id) ON DELETE SET NULL,
    vendor_id         BIGINT        NOT NULL REFERENCES vendors (id),
    po_number         VARCHAR(100)  NOT NULL,
    po_date           DATE          NOT NULL,
    expected_delivery DATE,
    currency_code     CHAR(3)       NOT NULL REFERENCES currencies (code),
    exchange_rate     NUMERIC(20,8) NOT NULL DEFAULT 1,
    subtotal          NUMERIC(20,4) NOT NULL DEFAULT 0,
    tax_amount        NUMERIC(20,4) NOT NULL DEFAULT 0,
    total_amount      NUMERIC(20,4) NOT NULL DEFAULT 0,
    status            VARCHAR(30)   NOT NULL DEFAULT 'draft'
                          CHECK (status IN ('draft','approved','partially_received','received','cancelled')),
    approved_by       BIGINT        REFERENCES users (id),
    approved_at       TIMESTAMPTZ,
    notes             TEXT,
    created_by        BIGINT        NOT NULL REFERENCES users (id),
    created_at        TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ   NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_purchase_orders_company_number UNIQUE (company_id, po_number)
);

CREATE INDEX idx_po_company_id    ON purchase_orders (company_id);
CREATE INDEX idx_po_vendor_id     ON purchase_orders (vendor_id);
CREATE INDEX idx_po_status        ON purchase_orders (company_id, status);
CREATE INDEX idx_po_po_date       ON purchase_orders (company_id, po_date DESC);
```

---

### `purchase_order_lines`

```sql
CREATE TABLE purchase_order_lines (
    id                 BIGSERIAL PRIMARY KEY,
    purchase_order_id  BIGINT        NOT NULL REFERENCES purchase_orders (id) ON DELETE CASCADE,
    item_id            BIGINT        REFERENCES items (id),
    account_id         BIGINT        REFERENCES accounts (id),
    description        TEXT          NOT NULL,
    quantity           NUMERIC(20,4) NOT NULL CHECK (quantity > 0),
    unit_price         NUMERIC(20,4) NOT NULL DEFAULT 0,
    tax_code_id        BIGINT        REFERENCES tax_codes (id),
    tax_amount         NUMERIC(20,4) NOT NULL DEFAULT 0,
    amount             NUMERIC(20,4) NOT NULL DEFAULT 0,
    quantity_received  NUMERIC(20,4) NOT NULL DEFAULT 0,
    line_order         SMALLINT      NOT NULL DEFAULT 0,

    CONSTRAINT chk_pol_quantity_received CHECK (quantity_received <= quantity)
);

CREATE INDEX idx_pol_purchase_order_id ON purchase_order_lines (purchase_order_id);
CREATE INDEX idx_pol_item_id           ON purchase_order_lines (item_id) WHERE item_id IS NOT NULL;
```

---

### `goods_receipts`

```sql
CREATE TABLE goods_receipts (
    id                BIGSERIAL PRIMARY KEY,
    company_id        BIGINT       NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    branch_id         BIGINT       REFERENCES branches (id) ON DELETE SET NULL,
    purchase_order_id BIGINT       NOT NULL REFERENCES purchase_orders (id),
    vendor_invoice_id BIGINT       REFERENCES vendor_invoices (id),
    receipt_number    VARCHAR(100) NOT NULL,
    receipt_date      DATE         NOT NULL,
    warehouse_id      BIGINT       NOT NULL REFERENCES warehouses (id),
    notes             TEXT,
    journal_entry_id  BIGINT       REFERENCES journal_entries (id),
    created_by        BIGINT       NOT NULL REFERENCES users (id),
    created_at        TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_goods_receipts_company_number UNIQUE (company_id, receipt_number)
);

CREATE INDEX idx_gr_company_id        ON goods_receipts (company_id);
CREATE INDEX idx_gr_purchase_order_id ON goods_receipts (purchase_order_id);
CREATE INDEX idx_gr_vendor_invoice_id ON goods_receipts (vendor_invoice_id) WHERE vendor_invoice_id IS NOT NULL;
CREATE INDEX idx_gr_receipt_date      ON goods_receipts (company_id, receipt_date DESC);
```

---

### `goods_receipt_lines`

```sql
CREATE TABLE goods_receipt_lines (
    id                    BIGSERIAL PRIMARY KEY,
    goods_receipt_id      BIGINT        NOT NULL REFERENCES goods_receipts (id) ON DELETE CASCADE,
    purchase_order_line_id BIGINT       NOT NULL REFERENCES purchase_order_lines (id),
    item_id               BIGINT        NOT NULL REFERENCES items (id),
    quantity_received     NUMERIC(20,4) NOT NULL CHECK (quantity_received > 0),
    unit_cost             NUMERIC(20,8) NOT NULL DEFAULT 0
);

CREATE INDEX idx_grl_goods_receipt_id      ON goods_receipt_lines (goods_receipt_id);
CREATE INDEX idx_grl_purchase_order_line_id ON goods_receipt_lines (purchase_order_line_id);
```

---

## 10. Sales

### `sales_orders`

```sql
CREATE TABLE sales_orders (
    id                BIGSERIAL PRIMARY KEY,
    company_id        BIGINT        NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    branch_id         BIGINT        REFERENCES branches (id) ON DELETE SET NULL,
    customer_id       BIGINT        NOT NULL REFERENCES customers (id),
    order_number      VARCHAR(100)  NOT NULL,
    order_date        DATE          NOT NULL,
    expected_delivery DATE,
    currency_code     CHAR(3)       NOT NULL REFERENCES currencies (code),
    exchange_rate     NUMERIC(20,8) NOT NULL DEFAULT 1,
    subtotal          NUMERIC(20,4) NOT NULL DEFAULT 0,
    tax_amount        NUMERIC(20,4) NOT NULL DEFAULT 0,
    discount_amount   NUMERIC(20,4) NOT NULL DEFAULT 0,
    total_amount      NUMERIC(20,4) NOT NULL DEFAULT 0,
    status            VARCHAR(30)   NOT NULL DEFAULT 'draft'
                          CHECK (status IN ('draft','confirmed','partially_fulfilled','fulfilled','cancelled')),
    notes             TEXT,
    created_by        BIGINT        NOT NULL REFERENCES users (id),
    created_at        TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ   NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_sales_orders_company_number UNIQUE (company_id, order_number)
);

CREATE INDEX idx_so_company_id   ON sales_orders (company_id);
CREATE INDEX idx_so_customer_id  ON sales_orders (customer_id);
CREATE INDEX idx_so_status       ON sales_orders (company_id, status);
CREATE INDEX idx_so_order_date   ON sales_orders (company_id, order_date DESC);
```

---

### `sales_order_lines`

```sql
CREATE TABLE sales_order_lines (
    id               BIGSERIAL PRIMARY KEY,
    sales_order_id   BIGINT        NOT NULL REFERENCES sales_orders (id) ON DELETE CASCADE,
    item_id          BIGINT        REFERENCES items (id),
    description      TEXT          NOT NULL,
    quantity         NUMERIC(20,4) NOT NULL CHECK (quantity > 0),
    unit_price       NUMERIC(20,4) NOT NULL DEFAULT 0,
    discount_pct     NUMERIC(5,2)  NOT NULL DEFAULT 0 CHECK (discount_pct BETWEEN 0 AND 100),
    tax_code_id      BIGINT        REFERENCES tax_codes (id),
    tax_amount       NUMERIC(20,4) NOT NULL DEFAULT 0,
    amount           NUMERIC(20,4) NOT NULL DEFAULT 0,
    quantity_fulfilled NUMERIC(20,4) NOT NULL DEFAULT 0,
    line_order       SMALLINT      NOT NULL DEFAULT 0,

    CONSTRAINT chk_sol_quantity_fulfilled CHECK (quantity_fulfilled <= quantity)
);

CREATE INDEX idx_sol_sales_order_id ON sales_order_lines (sales_order_id);
CREATE INDEX idx_sol_item_id        ON sales_order_lines (item_id) WHERE item_id IS NOT NULL;
```

---

### `delivery_orders`

```sql
CREATE TABLE delivery_orders (
    id               BIGSERIAL PRIMARY KEY,
    company_id       BIGINT       NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    branch_id        BIGINT       REFERENCES branches (id) ON DELETE SET NULL,
    sales_order_id   BIGINT       NOT NULL REFERENCES sales_orders (id),
    do_number        VARCHAR(100) NOT NULL,
    do_date          DATE         NOT NULL,
    warehouse_id     BIGINT       NOT NULL REFERENCES warehouses (id),
    status           VARCHAR(30)  NOT NULL DEFAULT 'draft'
                         CHECK (status IN ('draft','confirmed','shipped','delivered','cancelled')),
    journal_entry_id BIGINT       REFERENCES journal_entries (id),
    notes            TEXT,
    created_by       BIGINT       NOT NULL REFERENCES users (id),
    created_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_delivery_orders_company_number UNIQUE (company_id, do_number)
);

CREATE INDEX idx_do_company_id      ON delivery_orders (company_id);
CREATE INDEX idx_do_sales_order_id  ON delivery_orders (sales_order_id);
CREATE INDEX idx_do_status          ON delivery_orders (company_id, status);
CREATE INDEX idx_do_do_date         ON delivery_orders (company_id, do_date DESC);
```

---

### `delivery_order_lines`

```sql
CREATE TABLE delivery_order_lines (
    id                   BIGSERIAL PRIMARY KEY,
    delivery_order_id    BIGINT        NOT NULL REFERENCES delivery_orders (id) ON DELETE CASCADE,
    sales_order_line_id  BIGINT        NOT NULL REFERENCES sales_order_lines (id),
    item_id              BIGINT        NOT NULL REFERENCES items (id),
    quantity_delivered   NUMERIC(20,4) NOT NULL CHECK (quantity_delivered > 0)
);

CREATE INDEX idx_dol_delivery_order_id   ON delivery_order_lines (delivery_order_id);
CREATE INDEX idx_dol_sales_order_line_id ON delivery_order_lines (sales_order_line_id);
```

---

## 11. Tax

### `tax_codes`

```sql
CREATE TABLE tax_codes (
    id           BIGSERIAL PRIMARY KEY,
    company_id   BIGINT        NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    code         VARCHAR(20)   NOT NULL,
    name         VARCHAR(255)  NOT NULL,
    rate         NUMERIC(8,4)  NOT NULL CHECK (rate >= 0),
    tax_type     VARCHAR(20)   NOT NULL DEFAULT 'VAT'
                     CHECK (tax_type IN ('VAT','withholding','sales_tax','other')),
    account_id   BIGINT        NOT NULL REFERENCES accounts (id),
    is_inclusive BOOLEAN       NOT NULL DEFAULT FALSE,  -- TRUE = tax included in price
    is_active    BOOLEAN       NOT NULL DEFAULT TRUE,
    created_at   TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMPTZ   NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_tax_codes_company_code UNIQUE (company_id, code)
);

CREATE INDEX idx_tc_company_id ON tax_codes (company_id);
CREATE INDEX idx_tc_is_active  ON tax_codes (company_id, is_active);
```

---

### `tax_transactions`

```sql
CREATE TABLE tax_transactions (
    id                BIGSERIAL PRIMARY KEY,
    company_id        BIGINT        NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    tax_code_id       BIGINT        NOT NULL REFERENCES tax_codes (id),
    source_type       VARCHAR(100)  NOT NULL,  -- 'VendorInvoice','CustomerInvoice'
    source_id         BIGINT        NOT NULL,
    tax_base_amount   NUMERIC(20,4) NOT NULL,
    tax_amount        NUMERIC(20,4) NOT NULL,
    transaction_date  DATE          NOT NULL,
    direction         VARCHAR(10)   NOT NULL CHECK (direction IN ('input','output')),
    fiscal_period_id  BIGINT        REFERENCES fiscal_periods (id),
    created_at        TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tt_company_id      ON tax_transactions (company_id);
CREATE INDEX idx_tt_tax_code_id     ON tax_transactions (tax_code_id);
CREATE INDEX idx_tt_source          ON tax_transactions (source_type, source_id);
CREATE INDEX idx_tt_transaction_date ON tax_transactions (company_id, transaction_date DESC);
CREATE INDEX idx_tt_direction       ON tax_transactions (company_id, direction);
```

---

## 12. Budgeting

### `budgets`

```sql
CREATE TABLE budgets (
    id               BIGSERIAL PRIMARY KEY,
    company_id       BIGINT       NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    fiscal_period_id BIGINT       NOT NULL REFERENCES fiscal_periods (id),
    name             VARCHAR(255) NOT NULL,
    status           VARCHAR(20)  NOT NULL DEFAULT 'draft'
                         CHECK (status IN ('draft','approved','archived')),
    approved_by      BIGINT       REFERENCES users (id),
    approved_at      TIMESTAMPTZ,
    notes            TEXT,
    created_by       BIGINT       NOT NULL REFERENCES users (id),
    created_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_budgets_company_period_name UNIQUE (company_id, fiscal_period_id, name)
);

CREATE INDEX idx_budgets_company_id       ON budgets (company_id);
CREATE INDEX idx_budgets_fiscal_period_id ON budgets (fiscal_period_id);
```

---

### `budget_lines`

```sql
CREATE TABLE budget_lines (
    id           BIGSERIAL PRIMARY KEY,
    budget_id    BIGINT        NOT NULL REFERENCES budgets (id) ON DELETE CASCADE,
    account_id   BIGINT        NOT NULL REFERENCES accounts (id),
    branch_id    BIGINT        REFERENCES branches (id) ON DELETE SET NULL,
    cost_center_id BIGINT      REFERENCES cost_centers (id) ON DELETE SET NULL,
    jan          NUMERIC(20,4) NOT NULL DEFAULT 0,
    feb          NUMERIC(20,4) NOT NULL DEFAULT 0,
    mar          NUMERIC(20,4) NOT NULL DEFAULT 0,
    apr          NUMERIC(20,4) NOT NULL DEFAULT 0,
    may          NUMERIC(20,4) NOT NULL DEFAULT 0,
    jun          NUMERIC(20,4) NOT NULL DEFAULT 0,
    jul          NUMERIC(20,4) NOT NULL DEFAULT 0,
    aug          NUMERIC(20,4) NOT NULL DEFAULT 0,
    sep          NUMERIC(20,4) NOT NULL DEFAULT 0,
    oct          NUMERIC(20,4) NOT NULL DEFAULT 0,
    nov          NUMERIC(20,4) NOT NULL DEFAULT 0,
    dec          NUMERIC(20,4) NOT NULL DEFAULT 0,
    total_amount NUMERIC(20,4) GENERATED ALWAYS AS
                     (jan+feb+mar+apr+may+jun+jul+aug+sep+oct+nov+dec) STORED,

    CONSTRAINT uq_budget_lines_budget_account_branch
        UNIQUE (budget_id, account_id, branch_id, cost_center_id)
);

CREATE INDEX idx_bl_budget_id    ON budget_lines (budget_id);
CREATE INDEX idx_bl_account_id   ON budget_lines (account_id);
CREATE INDEX idx_bl_branch_id    ON budget_lines (branch_id) WHERE branch_id IS NOT NULL;
```

---

## 13. Recurring Transactions

### `recurring_templates`

```sql
CREATE TABLE recurring_templates (
    id             BIGSERIAL PRIMARY KEY,
    company_id     BIGINT       NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    name           VARCHAR(255) NOT NULL,
    module         VARCHAR(20)  NOT NULL CHECK (module IN ('AP','AR','GL')),
    frequency      VARCHAR(20)  NOT NULL
                       CHECK (frequency IN ('daily','weekly','monthly','quarterly','yearly')),
    next_run_date  DATE         NOT NULL,
    end_date       DATE,
    is_active      BOOLEAN      NOT NULL DEFAULT TRUE,
    template_data  JSONB        NOT NULL DEFAULT '{}',
    last_run_at    TIMESTAMPTZ,
    created_by     BIGINT       NOT NULL REFERENCES users (id),
    created_at     TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at     TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    CONSTRAINT chk_rt_end_date CHECK (end_date IS NULL OR end_date >= next_run_date)
);

CREATE INDEX idx_rt_company_id    ON recurring_templates (company_id);
CREATE INDEX idx_rt_next_run_date ON recurring_templates (next_run_date) WHERE is_active = TRUE;
CREATE INDEX idx_rt_module        ON recurring_templates (company_id, module);
```

**Notes:** `template_data` stores the JSON representation of the transaction to be created (journal entry lines, invoice details, etc.) and is deserialized by the `RecurringTransactionJob`.

---

### `recurring_logs`

```sql
CREATE TABLE recurring_logs (
    id                    BIGSERIAL PRIMARY KEY,
    recurring_template_id BIGINT       NOT NULL REFERENCES recurring_templates (id) ON DELETE CASCADE,
    run_date              DATE         NOT NULL,
    status                VARCHAR(20)  NOT NULL CHECK (status IN ('success','failed','skipped')),
    generated_id          BIGINT,       -- ID of the created record
    generated_type        VARCHAR(100), -- model class name
    error_message         TEXT,
    created_at            TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rl_template_id ON recurring_logs (recurring_template_id);
CREATE INDEX idx_rl_run_date    ON recurring_logs (run_date DESC);
CREATE INDEX idx_rl_status      ON recurring_logs (status);
```

---

## 14. Audit Trail

### `audit_logs`

```sql
CREATE TABLE audit_logs (
    id           BIGSERIAL PRIMARY KEY,
    company_id   BIGINT       REFERENCES companies (id) ON DELETE SET NULL,
    user_id      BIGINT       REFERENCES users (id) ON DELETE SET NULL,
    action       VARCHAR(50)  NOT NULL,   -- 'create','update','delete','login','logout','approve','post','void'
    module       VARCHAR(100) NOT NULL,
    entity_type  VARCHAR(255) NOT NULL,
    entity_id    BIGINT,
    old_values   JSONB,
    new_values   JSONB,
    ip_address   INET,
    user_agent   TEXT,
    session_id   VARCHAR(255),
    created_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Create quarterly partitions (example):
CREATE TABLE audit_logs_2025_q1 PARTITION OF audit_logs
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');
CREATE TABLE audit_logs_2025_q2 PARTITION OF audit_logs
    FOR VALUES FROM ('2025-04-01') TO ('2025-07-01');
-- (continue creating partitions per quarter)

CREATE INDEX idx_al_company_id   ON audit_logs (company_id, created_at DESC);
CREATE INDEX idx_al_user_id      ON audit_logs (user_id, created_at DESC);
CREATE INDEX idx_al_entity       ON audit_logs (entity_type, entity_id);
CREATE INDEX idx_al_action       ON audit_logs (action);
CREATE INDEX idx_al_created_at   ON audit_logs (created_at DESC);
```

**Notes:** Table is partitioned by `created_at` for performance. Old partitions can be archived or dropped. `old_values` and `new_values` store a snapshot diff of changed fields only.

---

## 15. Documents

### `document_attachments`

```sql
CREATE TABLE document_attachments (
    id           BIGSERIAL PRIMARY KEY,
    company_id   BIGINT       NOT NULL REFERENCES companies (id) ON DELETE CASCADE,
    entity_type  VARCHAR(255) NOT NULL,   -- 'VendorInvoice','CustomerInvoice','JournalEntry', etc.
    entity_id    BIGINT       NOT NULL,
    file_name    VARCHAR(500) NOT NULL,
    file_path    VARCHAR(1000) NOT NULL,  -- S3 key or local path
    file_size    INTEGER      NOT NULL,   -- bytes
    mime_type    VARCHAR(100) NOT NULL,
    uploaded_by  BIGINT       NOT NULL REFERENCES users (id),
    created_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_da_company_id ON document_attachments (company_id);
CREATE INDEX idx_da_entity     ON document_attachments (entity_type, entity_id);
CREATE INDEX idx_da_uploaded_by ON document_attachments (uploaded_by);
```

---

## Naming Conventions & Design Notes

| Convention | Rule |
|---|---|
| Primary keys | `BIGSERIAL`, always named `id` |
| Foreign keys | `{table_singular}_id` pattern |
| Monetary amounts | `NUMERIC(20,4)` |
| Exchange rates | `NUMERIC(20,8)` for precision |
| Timestamps | `TIMESTAMPTZ` (UTC stored, timezone-aware) |
| Soft deletes | `deleted_at TIMESTAMPTZ` (only on master data tables) |
| Company scoping | Every transaction table has `company_id` |
| Status enums | PostgreSQL `CHECK` constraints (not enum type, easier migrations) |
| JSONB columns | Used for flexible structured data (addresses, contacts, settings) |
| Full-text search | `GIN` index on `tsvector` for name/description search |
| Partitioning | `audit_logs` partitioned by quarter |

## Row-Level Security (RLS) Example

```sql
-- Enable RLS on all company-scoped tables
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;

CREATE POLICY accounts_company_isolation ON accounts
    USING (company_id = current_setting('app.current_company_id')::BIGINT);

-- Set at session start via Laravel middleware:
-- DB::statement("SET app.current_company_id = {$companyId}");
```

## Key Indexes for Report Performance

```sql
-- Trial Balance: sum debits/credits by account for a date range
CREATE INDEX idx_jel_account_je_date ON journal_entry_lines (account_id)
    INCLUDE (debit_base, credit_base);

-- AP Aging: unpaid invoices by due date
CREATE INDEX idx_vi_aging ON vendor_invoices (company_id, status, due_date)
    WHERE status IN ('approved','partially_paid');

-- AR Aging
CREATE INDEX idx_ci_aging ON customer_invoices (company_id, status, due_date)
    WHERE status IN ('approved','sent','partially_paid','overdue');

-- GL Account Ledger
CREATE INDEX idx_jel_account_date ON journal_entry_lines (account_id, journal_entry_id)
    INCLUDE (debit_base, credit_base, description);
```
