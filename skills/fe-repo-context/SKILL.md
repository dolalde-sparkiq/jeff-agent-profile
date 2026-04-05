---
name: fe-repo-context
description: "Architecture knowledge base for sparkiq-gh/sparkiq-erp-fe. Used during epic planning to understand the frontend codebase without cloning."
user-invocable: false
last-updated: "2026-04-05"
---

# Frontend Repo Context — sparkiq-gh/sparkiq-erp-fe

## Tech Stack

- **Framework:** Next.js 16 (App Router)
- **Language:** TypeScript 5
- **UI Library:** shadcn/ui (Radix primitives)
- **Styling:** Tailwind CSS 4
- **Tables:** TanStack React Table 8
- **Auth:** Supabase SSR (@supabase/ssr)
- **State:** React Server Components + client hooks (no Redux/Zustand)
- **Icons:** Lucide React
- **Toasts:** Sonner
- **Date Picking:** react-day-picker
- **Monitoring:** Sentry + Vercel Analytics
- **Package Manager:** pnpm

## Directory Structure

```
app/                              # Next.js App Router (pages + layouts)
├── auth/                         # Sign-in, callback
├── accounting/
│   ├── coa/                      # Chart of accounts page
│   ├── gl-activity/              # General ledger activity
│   └── journals/                 # Journal entries
├── banking/
│   ├── accounts/                 # Bank accounts
│   └── transactions/             # Bank transactions
├── billing-schedules/contracts/  # Billing schedule management
├── contracts/                    # Lease agreements
├── documents/
│   ├── bills/                    # AP bills
│   ├── credit-memos/             # Credit memos
│   ├── intercompany/             # IC transactions
│   ├── invoices/                 # AR invoices
│   ├── journal-entries/          # Manual journal entries
│   └── vendor-credits/           # Vendor credits
├── items/                        # Item catalog
├── organization/legal-entities/  # Entity management
├── parties/
│   ├── tenants/                  # Tenant management
│   └── vendors/                  # Vendor management
├── properties/                   # Property + space management
└── reports/pl/                   # P&L report

components/                       # React components
├── accounting/                   # Accounting-specific components
├── banking/                      # Banking components
├── common/                       # Shared components (PageHeader, etc.)
├── documents/                    # Document components (bills, invoices, credit-memos, vendor-credits, intercompany, journal-entries, shared)
├── items/                        # Item components
├── nav/                          # Navigation (sidebar, etc.)
├── properties/                   # Property components
├── states/                       # Empty/loading states
├── table/                        # Data table components (generic)
└── ui/                           # shadcn/ui primitives (see below)

hooks/                            # Custom React hooks
├── accounting/                   # accounts, gl-activity, journals
├── banking/                      # Bank hooks
├── common/                       # Shared hooks (useUrlSortState, etc.)
├── contracts/                    # Contract hooks
├── documents/                    # bills, credit-memos, invoices, vendor-credits, intercompany, journal-entries
├── items/                        # Item hooks
├── legal-entities/               # Entity hooks
├── parties/                      # Party hooks
└── properties/                   # Property hooks

lib/                              # Data fetching + business logic
├── api/                          # API client (client.ts)
├── accounting/                   # accounts, gl-activity, journals
├── banking/                      # Bank data
├── billing-schedules/            # Billing schedule data
├── contracts/                    # Contract data
├── documents/                    # bills, credit-memos, invoices, vendor-credits, intercompany (each with infinite/ for pagination)
├── intercompany/                 # IC data
├── items/                        # Item data (with infinite/)
├── legal-entities/               # Entity data
├── org/                          # Org data
├── parties/                      # Party data
├── properties/                   # Property data
├── reports/pl/                   # P&L report data
├── supabase/                     # Supabase client setup
└── tax-rates/                    # Tax rate data

types/                            # TypeScript type definitions
utils/                            # Utility functions
```

## shadcn/ui Components Available

alert, avatar, badge, button, calendar, card, collapsible, content, datePicker, dialog, document-status-badge, drawer, dropdownMenu, field, input, label, popover, searchSuggestionInput, select-native, select, separator, sheet, sidebar, skeleton, table, textarea, tooltip

## Architecture Patterns

### API Client
`lib/api/client.ts` — single API client that wraps fetch calls to the BE. All data fetching goes through this.

### Data Fetching Pattern
```
lib/[module]/             # Server-side data fetching functions
  ├── index.ts            # fetch functions (getDocuments, getDocument, etc.)
  └── infinite/           # Infinite scroll / pagination variants

hooks/[module]/           # Client-side React hooks wrapping lib/ functions
  └── useDocuments.ts     # Hook that calls lib/ and manages state
```

### Page Pattern
```
app/[module]/page.tsx     # Server component, fetches initial data
  → components/[module]/  # Client components for UI
    → hooks/[module]/     # Hooks for mutations and state
      → lib/[module]/     # Data fetching to BE API
```

### Document Pages
All document types follow the same pattern:
- List page: `app/documents/[type]/page.tsx` → data table with filters
- New page: `app/documents/[type]/new/page.tsx` → creation form
- Components: `components/documents/[type]/` → form, table, detail views
- Shared: `components/documents/shared/` — reusable document components (line items, status badges, etc.)

### How to add a new page/module
1. Create `app/[module]/page.tsx` (server component)
2. Create `components/[module]/` for UI components
3. Create `hooks/[module]/` for client-side hooks
4. Create `lib/[module]/` for data fetching functions
5. Add types to `types/`
6. Add navigation entry in `components/nav/`

### How to add a new document type page
1. Create `app/documents/[type]/page.tsx` and `app/documents/[type]/new/page.tsx`
2. Create `components/documents/[type]/` — reuse `components/documents/shared/` for common patterns (line items, posting, status)
3. Create `hooks/documents/[type]/` for mutations
4. Create `lib/documents/[type]/` for data fetching (+ `infinite/` for list pagination)
5. Add route to sidebar in `components/nav/`

## Conventions

- **Server vs Client:** Pages are server components by default. Interactive components use `"use client"`.
- **Styling:** Tailwind utility classes. No CSS modules. Use `cn()` (from `clsx` + `tailwind-merge`) for conditional classes.
- **Forms:** Controlled components with local state. No form library (react-hook-form not installed).
- **Tables:** TanStack React Table with custom `components/table/` wrappers. Column sorting via `useUrlSortState` hook (URL-persisted).
- **Toasts:** Sonner for notifications.
- **Testing:** Co-located `__tests__/` directories. Tests exist for some hooks, components, and lib functions.
