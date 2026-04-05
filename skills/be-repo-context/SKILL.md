---
name: be-repo-context
description: "Architecture knowledge base for sparkiq-gh/sparkiq-erp-be. Used during epic planning to understand the backend codebase without cloning."
user-invocable: false
last-updated: "2026-04-05"
---

# Backend Repo Context — sparkiq-gh/sparkiq-erp-be

## Tech Stack

- **Framework:** NestJS 11 on Fastify
- **Language:** TypeScript 5
- **ORM:** Drizzle ORM 0.45 (PostgreSQL)
- **Database:** PostgreSQL via Supabase
- **CQRS:** @nestjs/cqrs 11 (commands + queries with handlers)
- **Queue:** BullMQ via @nestjs/bullmq (background jobs)
- **Auth:** Supabase Auth (JWT, RLS policies)
- **Validation:** class-validator + class-transformer + Zod
- **API docs:** @nestjs/swagger (OpenAPI)
- **Testing:** Jest 30 + Supertest 7
- **Monitoring:** Sentry

## Directory Structure

```
src/
├── app/                          # App bootstrap (module, controller)
├── config/                       # Environment config
├── core/
│   ├── application/services/     # Cross-cutting services (accounting-period, document-sequence, legal-entity, org, party, property, space)
│   ├── cqrs/
│   │   ├── commands/handlers/    # Command handlers (write operations)
│   │   └── queries/handlers/     # Query handlers (read operations)
│   ├── database/
│   │   ├── schema/               # Drizzle schema (schema.ts, relations.ts, index.ts)
│   │   └── repositories/         # Base repository patterns
│   ├── events/                   # Domain event handlers
│   ├── interfaces/repositories/  # Repository interfaces
│   ├── queue/processors/         # BullMQ job processors
│   └── shared/services/          # Shared utilities
├── modules/                      # Feature modules (see below)
├── test/
│   ├── e2e/documents/            # End-to-end tests
│   └── integration/              # Integration tests
drizzle/
├── migrations/                   # SQL migration files
└── meta/                         # Migration metadata
```

## Database Schema (key tables)

| Table | Purpose | Key Fields |
|---|---|---|
| `orgs` | Multi-tenant root | id, name |
| `orgMembers` | User ↔ org membership | org_id, user_id |
| `legalEntities` | Legal entity hierarchy | org_id, name, entity_type, parent_entity_id, is_management |
| `properties` | Real estate properties | org_id, name, status |
| `spaces` | Units within properties | org_id, property_id, name, space_type, status |
| `parties` | Tenants, vendors, owners | org_id, name, type |
| `partyRoles` | Multi-role party links | party_id, role |
| `accounts` | Chart of accounts | org_id, name, code, account_type, detail_type, parent_account_id |
| `documents` | All financial documents (CQRS) | org_id, document_type, status, total_amount, posted_at |
| `documentLines` | Line items on documents | document_id, account_id, amount, description |
| `journalEntries` | Immutable GL entries | org_id, document_id, entry_date, status |
| `journalLines` | GL line items (debit/credit) | journal_entry_id, account_id, debit, credit |
| `contracts` | Lease agreements | org_id, property_id, tenant_id, start_date, end_date |
| `contractTerms` | Rent terms (base, escalation, CAM) | contract_id, term_type, amount |
| `financialEvents` | Billing schedule events | contract_id, event_date, amount |
| `bankAccounts` | Bank accounts linked to GL | org_id, account_id, name |
| `bankTransactions` | Bank transaction imports | bank_account_id, amount, date |
| `paymentApplications` | Payment ↔ invoice matching | payment_document_id, invoice_document_id, amount |
| `items` | Revenue/expense catalog | org_id, name, code, item_type |
| `accountingPeriods` | Monthly open/close/lock | org_id, year, month, status |
| `documentSequences` | Auto-numbering per doc type | org_id, document_type, year, last_value |
| `eventLog` | Audit trail | entity_type, entity_id, action, before, after |
| `dimensions` | Reporting dimensions | org_id, name |
| `icSettlementApplications` | Intercompany settlements | settlement_id, transaction_id, amount |

### Enums
- `partyRole`: tenant
- `partyType`: individual, company, financial, internal, etc.

### Key Invariants
- **GL is immutable** — no destructive edits to journalEntries/journalLines. Changes = reversal + new entry.
- **Documents ≠ Journals** — documents are mutable business intent; journals are immutable accounting truth.
- **All queries are org-scoped** — multi-tenancy via org_id + RLS policies.
- **All financial mutations need audit trails** — eventLog table.

## API Modules

| Module | Controller(s) | Endpoints |
|---|---|---|
| `accounting` | accounts, gl-activity, journals | COA CRUD, GL activity reports, journal entries |
| `banking` | bank-accounts, banking | Bank accounts, transaction import/match |
| `billing-schedules` | billing-schedules | Contract billing schedule management |
| `contracts` | contracts | Lease/management agreement CRUD |
| `credit-memos` | credit-memos | Credit memo CRUD + lifecycle |
| `documents` | documents, credit-applications, payment-applications | Document CRUD, posting, credit/payment application |
| `intercompany` | intercompany | IC transactions, settlements |
| `items` | items | Revenue/expense item catalog |
| `legal-entities` | legal-entities | Entity hierarchy CRUD |
| `org` | org | Organization management |
| `parties` | parties | Tenant/vendor/owner CRUD |
| `properties` | properties | Property + space CRUD |
| `tax-rates` | tax-rates | Tax rate management |
| `vendor-credits` | vendor-credits | Vendor credit CRUD + lifecycle |
| `health` | health | Health check endpoint |

## CQRS Pattern

The documents module uses full CQRS:
```
src/modules/documents/
├── application/
│   ├── commands/          # CreateDocumentCommand, PostDocumentCommand, etc.
│   ├── events/handlers/   # Domain event handlers
│   ├── handlers/          # Command handlers
│   ├── queries/handlers/  # Query handlers
│   └── services/          # Application services
├── domain/
│   ├── entities/          # Domain entities
│   ├── services/          # Domain services
│   └── value-objects/     # Value objects
└── infrastructure/
    ├── dto/               # Data transfer objects
    └── repositories/      # Drizzle repository implementations
```

### How to add a new document type
1. Add type to `documentSequences` check constraint
2. Create module in `src/modules/[type]/` with controller, DTOs, module file
3. If it needs CQRS (complex lifecycle), add commands/queries/handlers
4. Add Drizzle schema changes in `src/core/database/schema/schema.ts`
5. Generate migration: `npx drizzle-kit generate`
6. Register module in `src/app/app.module.ts`

### How to add a new CRUD endpoint
1. Create/extend controller in `src/modules/[module]/`
2. Create DTOs in `infrastructure/dto/`
3. Create/extend service in `application/services/`
4. Create/extend repository in `infrastructure/repositories/`
5. Register in the module file

## Conventions

- **Naming:** snake_case for DB columns, camelCase for TS properties. Drizzle handles mapping.
- **Validation:** class-validator decorators on DTOs for request validation.
- **Error handling:** NestJS exception filters. Business rule violations → BadRequestException with descriptive message.
- **Testing:** Unit tests co-located in `__tests__/` dirs. E2E tests in `test/e2e/`. Integration tests in `test/integration/`.
- **Migrations:** Drizzle Kit generates SQL migrations. Never edit generated files.
