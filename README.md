<div align="center">

# Genesis — Quotation & Invoicing API

[![PHP](https://img.shields.io/badge/PHP-%3E%3D8.0-8892BF.svg)](https://php.net/)
[![Yii2](https://img.shields.io/badge/Yii2-Advanced%20Template-1E6887.svg)](https://www.yiiframework.com/)
[![API](https://img.shields.io/badge/API-RESTful%20JSON-orange.svg)]()
[![Auth](https://img.shields.io/badge/Auth-JWT%20%2B%20Refresh%20Tokens-yellow.svg)]()
[![License](https://img.shields.io/badge/License-BSD--3--Clause-blue.svg)](LICENSE.md)
[![Status](https://img.shields.io/badge/Status-Live%20%26%20Active-brightgreen.svg)]()

An **API-first, multi-tenant SaaS backend** built on Yii2. Genesis powers quotation, invoicing, and credit note management for businesses — with JWT authentication, role-based access control, tenant isolation, and secure public document sharing.

> **This repository contains the README only.** The source code is proprietary and actively running in production. Interested parties are welcome to [get in touch](#contact--demos) for a live walkthrough or code demo.

> **Frontend:** This is the backend API only. A React/Vite SPA consumes these endpoints. All authenticated requests require an `Authorization: Bearer <token>` header. Public document routes use cryptographically secure UUIDs.

</div>

---

## Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [User Roles & Permissions](#user-roles--permissions)
- [API Reference](#api-reference)
- [Document Workflows](#document-workflows)
- [Database Migrations](#database-migrations)
- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Security](#security)
- [Directory Structure](#directory-structure)
- [License](#license)
- [Contact & Demos](#contact--demos)

---

## Features

<details>
<summary><strong>Multi-Tenant SaaS</strong></summary>

- Workspace-based tenancy — every business operates in a fully isolated company context with all queries automatically scoped by `company_id`
- Tenant lifecycle management — soft-delete with POPIA-compliant grace archive periods
- Per-tenant logo uploads for personalised document branding
- JWT validation rejects tokens for archived or suspended tenants

</details>

<details>
<summary><strong>Quotation Management</strong></summary>

- Full CRUD with line items, tax calculations, and custom terms
- Status workflow: `Draft → Sent → Accepted / Declined / Expired`
- Secure UUID-based public links — clients view and respond without an account
- One-click quote-to-invoice conversion
- Quote duplication for reuse as templates
- Direct email delivery with configurable templates

</details>

<details>
<summary><strong>Invoice Management</strong></summary>

- Full invoice lifecycle: create, preview, email, track payment status
- Tax rate snapshots — rates are frozen at creation time for historical accuracy
- B2B invoicing columns
- Void invoices and issue credit notes against them
- Secure UUID-based public invoice portal for clients

</details>

<details>
<summary><strong>Credit Notes</strong></summary>

- Issue credits against existing invoices
- UUID-based public access for customer transparency
- Preview and email delivery

</details>

<details>
<summary><strong>Customer & Product Catalogs</strong></summary>

- Client records with contact info and billing preferences
- Product/service library with per-item activation toggles
- Scoped entirely per tenant

</details>

<details>
<summary><strong>Staff & Access Control</strong></summary>

- Three-tier RBAC: Owner, Admin, Staff (see [User Roles](#user-roles--permissions))
- Staff invitation, activation, and deactivation by Admins
- Self-service profile updates and password management

</details>

<details>
<summary><strong>Communications</strong></summary>

- Dynamic email templates with placeholder substitution (`{{client_name}}`, `{{quote_total}}`, etc.)
- Full email delivery log for audit purposes
- Auto-incrementing document sequence numbers per tenant

</details>

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│               Authenticated Clients (SPA)                │
│         Admin │ Staff │ Owner  (React / Vite)            │
└────────────────────────┬────────────────────────────────┘
                         │  JWT Bearer Token
                         ▼
              ┌──────────────────────┐
              │   CORS + Auth Layer  │
              │  BaseApiController   │
              └──────────┬───────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
    ┌─────────┐     ┌─────────┐    ┌──────────┐
    │  Quotes │     │Invoices │    │Customers │
    │  Staff  │     │ Credits │    │ Products │
    │  Tenant │     │Dashboard│    │ Auth     │
    └────┬────┘     └────┬────┘    └────┬─────┘
         └───────────────┴───────────────┘
                         │
              ┌──────────┴───────────┐
              │    Common Layer       │
              │  Models / Services    │
              └──────────┬───────────┘
                         │
              ┌──────────┴───────────┐
              │   MySQL (tenant-     │
              │   scoped via         │
              │   company_id)        │
              └──────────────────────┘

┌─────────────────────────────────────┐
│  Public Clients (No Auth Required)  │
│  /public/quote/<uuid>               │
│  /public/invoice/<uuid>             │
│  /public/credit-note/<uuid>         │
└─────────────────────────────────────┘
```

---

## Technology Stack

| Layer | Technology |
|---|---|
| Backend Framework | Yii2 Advanced App Template |
| Language | PHP >= 8.0 |
| API Format | RESTful JSON |
| Authentication | JWT + Refresh Token rotation |
| Rate Limiting | Yii2 `RateLimitInterface` + Redis/Cache |
| Database | MySQL / MariaDB |
| Caching | Redis (`yiisoft/yii2-redis`) |
| Email | Symfony Mailer / Google Mailer |
| Frontend Target | React/Vite SPA (via CORS) |

---

## User Roles & Permissions

| Role | Key | Description |
|---|---|---|
| **Owner** | `owner` | Full access — tenant settings, staff, all documents, billing. Cannot be deleted. |
| **Admin** | `admin` | Manage staff, all documents, templates, sequences, dashboard. |
| **Staff** | `staff` | Create/manage quotes, invoices, customers, products. Scoped to own tenant. |

### Permission Matrix

| Action | Owner | Admin | Staff |
|---|:---:|:---:|:---:|
| Manage tenant settings | ✅ | ❌ | ❌ |
| Invite / delete staff | ✅ | ✅ | ❌ |
| Toggle staff status | ✅ | ✅ | ❌ |
| Create quotes & invoices | ✅ | ✅ | ✅ |
| Void invoices | ✅ | ✅ | ❌ |
| Manage email templates | ✅ | ✅ | ❌ |
| View dashboard metrics | ✅ | ✅ | ✅ (scoped) |
| Manage products & customers | ✅ | ✅ | ✅ |

---

## API Reference

### Authentication

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/auth/login` | Authenticate — returns JWT + refresh token |
| `POST` | `/auth/refresh` | Rotate access token using refresh token |
| `POST` | `/auth/logout` | Revoke refresh tokens and end session |
| `POST` | `/auth/forgot-password` | Request password reset email |
| `POST` | `/auth/reset-password` | Reset password using token |

### Public Routes (No Auth)

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/public/quote/<uuid>` | Client view of a shared quote |
| `POST` | `/public/quote/<uuid>/respond` | Client accept / decline |
| `GET` | `/public/invoice/<uuid>` | Client view of a shared invoice |
| `GET` | `/public/credit-note/<uuid>` | Client view of a credit note |

### Core Resources

| Resource | Endpoints |
|---|---|
| **Dashboard** | `GET /dashboard/metrics` |
| **Profile** | `GET /profile` · `PUT /profile` · `POST /profile/change-password` |
| **Tenants** | `GET /tenants` · `POST /tenants` · `PUT /tenants/<id>` · `PUT /tenants/<id>/status` · `POST /tenants/<id>/logo` |
| **Staff** | `GET /staff` · `POST /staff` · `PATCH /staff/<id>/status` |
| **Customers** | Standard REST · `GET /customers/constants` |
| **Products** | Standard REST · `PATCH /products/<id>/activate` |
| **Quotes** | Standard REST · `PATCH /quotes/<id>/status` · `GET /quotes/<id>/preview` · `POST /quotes/<id>/email` · `POST /quotes/<id>/duplicate` · `POST /quotes/<id>/convert` |
| **Invoices** | Standard REST · `PATCH /invoices/<id>/status` · `GET /invoices/<id>/preview` · `POST /invoices/<id>/email` · `POST /invoices/<id>/void` |
| **Credit Notes** | `GET /credit-notes` · `GET /credit-notes/<id>` · `GET /credit-notes/<id>/preview` · `POST /credit-notes/<id>/email` |
| **Email Templates** | `GET /email-text` · `PUT /email-text/<id>` · `GET /email-text/placeholders` |

---

## Document Workflows

### Quote Lifecycle

```
┌───────┐     ┌──────┐     ┌──────────┐     ┌──────────┐
│ DRAFT │ ──► │ SENT │ ──► │ ACCEPTED │ ──► │ INVOICED │
└───────┘     └──────┘     └──────────┘     └──────────┘
                                │
                                ▼
                           ┌──────────┐
                           │ DECLINED │
                           └──────────┘
                                │
                                ▼
                           ┌─────────┐
                           │ EXPIRED │
                           └─────────┘
```

### Invoice Lifecycle

```
┌───────┐     ┌──────┐     ┌──────┐     ┌────────┐
│ DRAFT │ ──► │ SENT │ ──► │ PAID │ ──► │ CLOSED │
└───────┘     └──────┘     └──────┘     └────────┘
   │                          │
   ▼                          ▼
┌────────┐               ┌─────────┐
│ VOIDED │               │ OVERDUE │
└────────┘               └─────────┘
```

---

## Database Migrations

| Migration | Table | Purpose |
|---|---|---|
| `m240001_000001` | `tenants` | Workspace / company records |
| `m240001_000002` | `users` | Staff and owner accounts |
| `m240001_000003` | `refresh_tokens` | JWT refresh token storage |
| `m240001_000004` | `document_sequences` | Auto-incrementing document numbers per tenant |
| `m240001_000005` | `document_templates` | Email template definitions |
| `m240001_000006` | `customers` | Client records |
| `m240001_000007` | `products` | Product / service catalogue |
| `m240001_000008` | `quotes` | Quotation headers |
| `m240001_000009` | `quote_items` | Quotation line items |
| `m240001_000010` | `invoices` | Invoice headers |
| `m240001_000011` | `invoice_items` | Invoice line items |
| `m240001_000012` | `credit_notes` | Credit note records |
| `m240001_000013` | `email_log` | Outgoing email audit trail |
| `m260421_195313` | `invoices` | Add tax rate snapshot column |
| `m260422_070213` | `invoices` | Add B2B columns |

```bash
php yii migrate
```

---

## Requirements

- PHP >= 8.0 with PDO (pdo_mysql)
- MySQL / MariaDB >= 5.7
- Composer
- Apache or Nginx with `mod_rewrite` / equivalent
- Redis (optional — for rate limiting and caching)
- Valid SSL certificate (required for production)

---

## Installation

### 1. Clone and install dependencies

```bash
git clone <repository-url> genesis
cd genesis
composer install
```

### 2. Initialise the environment

```bash
php init        # Linux / macOS
init.bat        # Windows
```

Select `Development` or `Production` as appropriate.

### 3. Configure the database

Create a new database (e.g., `genesis_digital`), then update `common/config/main-local.php`:

```php
'db' => [
    'class'    => 'yii\db\Connection',
    'dsn'      => 'mysql:host=localhost;dbname=genesis_digital',
    'username' => 'root',
    'password' => 'your_password',
    'charset'  => 'utf8mb4',
],
```

Run migrations:

```bash
php yii migrate
```

### 4. Configure the API entry point

Point your web server to `api/v1/web/`. Example Nginx block:

```nginx
server {
    listen 80;
    server_name api.yourdomain.com;
    root /path/to/genesis/api/v1/web;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }
}
```

### 5. Set directory permissions

```bash
chmod -R 775 api/runtime api/v1/uploads
chmod -R 775 console/runtime frontend/runtime backend/runtime
```

> **Tip:** Use `775` with correct web server ownership in production — avoid `777`.

---

## Configuration

### JWT

In `api/config/params-local.php`:

```php
'jwt' => [
    'key' => 'your-256-bit-secret-key',  // min 32 chars, keep secret
],
```

### CORS

In `api/config/params.php`:

```php
'allowedOrigins' => [
    'http://localhost:5173',   // Vite dev
    'http://localhost:3000',   // Alt dev
    'https://yourdomain.com',  // Production
],
```

### Redis (Optional)

In `common/config/main-local.php`:

```php
'redis' => [
    'class'    => 'yii\redis\Connection',
    'hostname' => 'localhost',
    'port'     => 6379,
    'database' => 0,
],
```

### Email

In `common/config/main-local.php`:

```php
'mailer' => [
    'class'     => 'yii\symfonymailer\Mailer',
    'transport' => [
        'dsn' => 'smtp://user:pass@smtp.gmail.com:587',
    ],
],
```

---

## Security

| Measure | Detail |
|---|---|
| **JWT Bearer Auth** | Stateless, configurable expiry |
| **Refresh Token Rotation** | Database-backed revocation on logout or password change |
| **Auth Key Rotation** | All sessions invalidated on password change |
| **Rate Limiting** | 60 req / 60 s per user via token-bucket algorithm (Redis/cache) |
| **Tenant Isolation** | All queries automatically scoped by `company_id` |
| **Archived Tenant Rejection** | JWT validation rejects tokens for suspended tenants |
| **Soft Deletes** | `deleted_at` on all major entities — no hard data loss |
| **Password Policy** | Min 8 chars — uppercase, lowercase, number, special character |
| **Timing Attack Prevention** | `hash_equals()` used for all auth key comparisons |
| **Mass Assignment Protection** | `password_hash` and `auth_key` excluded from safe attributes |
| **Strict URL Parsing** | `enableStrictParsing` blocks undefined route access |
| **CORS Preflight** | Centralised handling in `BaseApiController` |

---

## Directory Structure

```
├── api/
│   ├── config/           # API config — CORS, URL rules, JWT params
│   ├── controllers/      # REST controllers — Auth, Quotes, Invoices, etc.
│   ├── models/           # API-specific models
│   ├── runtime/          # API logs and cache
│   └── v1/
│       ├── web/          # API entry point (index.php)
│       └── uploads/
│           └── logos/    # Tenant logo storage
│
├── common/
│   ├── components/       # AccessRule, MailService, AuthRateLimiter
│   ├── config/           # Shared configuration
│   ├── mail/             # Email view templates
│   └── models/           # Core models — User, Tenant, Quote, Invoice, etc.
│       └── queries/      # Custom ActiveQuery classes (TenantQuery, etc.)
│
├── console/
│   ├── migrations/       # All database migrations
│   └── runtime/
│
├── frontend/             # Legacy web UI (reserved)
├── backend/              # Legacy admin UI (reserved)
└── environments/         # Dev / prod environment overrides
```

---

## License

Licensed under the **BSD-3-Clause License**. See [LICENSE.md](LICENSE.md) for details.

---

## Contact & Demos

This repository is intentionally published as a README-only showcase — the full source code is proprietary and actively running in production.

If you're interested in any of the following, feel free to reach out:

- **Live demo or walkthrough** — a full presentation of the platform's features and workflows
- **Code review** — a private demo of the codebase for vetting or evaluation purposes
- **Similar build** — commissioning a comparable system for your business

📧 **[andrew@genesisdigital.co.za](mailto:andrew@genesisdigital.co.za)**
🌐 **[genesisdigital.co.za](https://genesisdigital.co.za)**

> Built by [Genesis Digital Solutions](https://genesisdigital.co.za)
