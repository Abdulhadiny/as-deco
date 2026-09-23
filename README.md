# AS Deco (Decora) — Event Decoration & Rental Management System

An internal-first operations, inventory tracking, event lifecycle, and financial management platform engineered for **A&S Decorations**, an event decoration and equipment rental business based in Kano, Nigeria.

---

## 1. Problem: What the Business Was Doing Manually

Prior to Decora, A&S Decorations managed operations across multiple concurrent weddings, corporate events, and ceremonies through ad-hoc, manual mechanisms:

* **Fragmented Bookings & Customer Communications:** Bookings, inquiries, and reschedule requests were scattered across personal WhatsApp chat threads, voice calls, paper notebooks, and memory. Details like venue coordinates, setup dates, and teardown timelines were frequently misplaced.
* **Blind Inventory Availability:** Availability checks required searching physical paper logs or physically walking into the warehouse to check shelves. Staff could not reliably determine whether high-demand items (e.g., gold chiavari chairs, floral arches, canopies, chandeliers) reserved for a Saturday wedding were already committed to a Friday corporate event or returning in time. Double-booking and last-minute substitutions were constant operational risks.
* **Uncoordinated Damage & Loss Tracking:** When items returned broken, torn, or missing after teardown, reports relied on verbal updates or mental notes. Damaged inventory frequently sat in the warehouse untagged, while replacement or repair costs were rarely billed back to the client due to lost paper trails.
* **Scattered Financial Records & Quoting Bottlenecks:** Mubarak (the founder/CEO) was the operational bottleneck: he personally drafted quotes, tracked bank transfer screenshots, wrote paper invoices, and tallied customer balances across spreadsheets. Outstanding client balances were frequently miscalculated or forgotten.
* **Lack of Real-Time Business Visibility:** At the end of any business day, there was no single view of confirmed revenue, uncollected balances, operational expenses (logistics, repairs, labor), or true net profit.

---

## 2. Product: What AS Deco Does

Decora replaces disjointed spreadsheets, paper logs, and messaging threads with a centralized operations management hub:

* **Event Lifecycle Management:** Manages events end-to-end through distinct phases (`UPCOMING` → `IN_PROGRESS` → `COMPLETED` / `CANCELLED`), capturing event schedules, setup dates, return deadlines, venues, and assigned customers.
* **Location-Aware Inventory Tracking:** Tracks catalog items with unique tags, category classifications, rental pricing, condition notes, and media photos across physical locations/warehouses.
* **Atomic Item Allocation & Returns:** Allows warehouse and field teams to allocate inventory to events and check them back in post-event with condition categorization (`GOOD`, `DAMAGED`, `MISSING`).
* **Damage Reconciliation & Automated Write-offs:** In one consolidated flow, staff can flag damaged returns, generate an itemized `DAMAGE` quote for client compensation, and simultaneously record an expense write-off under the business's "Damage & Loss" ledger.
* **Quoting, Invoicing & PDF Generation:** Generates client quotes and invoices directly from allocated event items, calculates taxes and discounts, and renders branded, on-demand PDF documents via `@react-pdf/renderer`.
* **Payment Processing & Customer Ledgers:** Records customer payments against general accounts or links them to specific quotes, maintaining an auditable ledger and real-time payment statuses (`outstanding`, `partial`, `reconciled`).
* **Financial Analytics & P&L Reporting:** Delivers real-time Profit & Loss (P&L) statements, tracking gross rental income, categorized operating expenses, damage write-offs, and operational margins in Nigerian Naira (NGN).
* **Role-Based Access Control (RBAC):** Restricts administrative functions, financial data, and inventory adjustments to authorized roles (`admin`, `staff`) with fine-grained permission enforcement.
* **Push Notifications:** Integrates with OneSignal to dispatch notifications to staff when items leave the warehouse, inventory is checked back in, or customer payments are recorded.

---

## 3. Architecture

Decora is structured around a strict unidirectional flow:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           Client / Presentation Layer                           │
│               Next.js 16 (React 19) App Router + Tailwind CSS 4                │
│             Server Components (RSC) + Client Components (shadcn/ui)             │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │ Server Actions & API Routes
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            Application Logic Layer                              │
│                                                                                 │
│   NextAuth v5 Guard  ──►  Zod Validation Schemas  ──►  Granular Permissions     │
│   (checkPermission)       (src/lib/validators)         (module:action)          │
│                                                                                 │
│                   Domain Engines (src/lib/engines/):                            │
│     • InventoryEngine  (Credit, debit, adjust stock movements)                 │
│     • RentalEngine     (Atomic event item allocations & multi-condition returns)│
│     • FinanceEngine    (Payment recording, quote reconciliation, expenses)      │
│                                                                                 │
│                   Cross-Cutting Infrastructure:                                 │
│     • Audit Logger     (logAction with interactive transaction rollback)       │
│     • Notification Hub (OneSignal push triggers)                                │
│     • Currency Utility (NGN Decimal formatting)                                 │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │ Prisma Interactive Transactions ($transaction)
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                             Data Access Layer                                   │
│                                Prisma 7.8                                       │
│          Dynamic Runtime Adapter: @prisma/adapter-neon / @prisma/adapter-pg     │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │ WebSocket / TCP Pool
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              Database Layer                                     │
│                      PostgreSQL (Neon Serverless)                               │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Architectural Highlights
1. **Next.js 16 & Server Actions:** Mutations bypass traditional custom REST API endpoints in favor of type-safe Next.js Server Actions (`src/lib/actions/`). Every action performs authorization checks, payload validation via Zod, engine delegation, audit logging, and cache revalidation (`revalidatePath`).
2. **Domain Engine Separation:** Core business rules are isolated into pure domain engines (`src/lib/engines/`). Server actions act as controllers; engines execute the domain rules and accept optional interactive transaction clients (`TxClient`).
3. **Dynamic Database Adapter:** `src/lib/db.ts` dynamically inspects `DATABASE_URL` at runtime. When connecting to Neon (`neon.tech`), it uses `@prisma/adapter-neon` over WebSockets (preventing TCP connection limits in serverless environments). When running locally or on standard PostgreSQL, it falls back to `@prisma/adapter-pg`.

---

## 4. Core Domain Model

Decora models the event decoration business using a relational graph centered around the event lifecycle, inventory partitioning, and financial commitments:

```mermaid
erDiagram
    Customer ||--o{ Event : "books"
    Customer ||--o{ CustomerPayment : "makes"
    Location ||--o{ Event : "hosts/services"
    Location ||--o{ InventoryStock : "stores"

    Event ||--o{ EventItem : "allocates"
    Event ||--o{ Quote : "bills via"

    ItemCategory ||--o{ Item : "classifies"
    Item ||--o{ EventItem : "allocated in"
    Item ||--o{ InventoryStock : "held in"
    Item ||--o{ InventoryTransaction : "moved via"

    InventoryStore ||--o{ InventoryStock : "partitions"
    InventoryStore ||--o{ InventoryTransaction : "tracks"

    Quote ||--o{ QuoteLine : "contains"
    
    ExpenseCategory ||--o{ Expense : "categorizes"
    User ||--o{ AuditLog : "triggers"
```

### Domain Entity Walkthrough

```
Customer ──► Event ──► EventItems ──► InventoryStock ──► Transactions ──► Quotes / Payments
```

1. **`Customer`:** The party requesting services. Stores contact details, addresses, and full booking and payment history.
2. **`Event`:** The central operational entity. Holds dates (`eventDate`, `setupDate`, `returnDate`), event type (`WEDDING`, `NAMING`, `BIRTHDAY`, `GRADUATION`, `OTHER`), venue, operational status (`UPCOMING`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED`), and location.
3. **`EventItem`:** Join entity between `Event` and `Item`. Tracks item quantity, allocation time (`allocatedAt`), return time (`returnedAt`), return condition (`GOOD`, `DAMAGED`, `MISSING`), and damage notes/photos.
4. **`Item` & `InventoryStore`:** The physical equipment catalog (with tags, rental price, photos, and status). Physical inventory is partitioned into 3 virtual stores:
   * `MAIN`: Available rentable equipment.
   * `DAMAGED`: Damaged items requiring repair, cleaning, or write-off.
   * `LOST`: Missing items awaiting recovery or replacement.
5. **`InventoryStock`:** Composite unique quantity tracking record identified by `[storeId, itemId, locationId]`.
6. **`InventoryTransaction`:** Immutable double-entry audit record for every stock movement, noting `credit` or `debit`, quantity, `referenceType` (`EVENT_ALLOCATION`, `EVENT_RETURN`, `INITIAL_STOCK`, `ADJUSTMENT`), and timestamp.
7. **`Quote` & `QuoteLine`:** Itemized financial obligations linked to an event. Quotes can be `RENTAL` (generated from allocated items) or `DAMAGE` (generated from damaged teardown returns).
8. **`CustomerPayment`:** Financial receipts credited toward a customer and optionally linked to a specific `Quote`. Reconciles quotes automatically and updates customer ledger balances.
9. **`Expense` & `ExpenseCategory`:** Operating expenditures and damage write-offs feeding directly into P&L calculations.

---

## 5. Important Engineering Decisions

### 1. How Stock Is Maintained
* **Store Partitioning Over Single Counters:** Rather than tracking a single scalar quantity on an item, stock is partitioned across `InventoryStore` records (`MAIN`, `DAMAGED`, `LOST`) and scoped to physical `Location`s. An item damaged during an event does not simply "disappear"; its stock decrements from `MAIN` and increments in `DAMAGED`.
* **Immutable Transactional Ledger:** Stock balances in `InventoryStock` are never modified in isolation. Any modification must generate an `InventoryTransaction` row recording the actor (`createdBy`), reference entity (`referenceType` + `referenceId`), and movement type (`credit` / `debit`).
* **Location-Aware Availability:** Stock availability is computed against specific warehouse locations (`src/lib/availability.ts`). Multi-location stock is never treated as a single pooled global counter, preventing impossible reservations across branches.

### 2. How Allocation & Return Works
* **Atomic Allocation (`RentalEngine.allocateItems`):** When items are assigned to an event:
  1. The engine checks current availability in the `MAIN` store at the event's location. If quantity is insufficient, it throws an error and aborts the entire transaction.
  2. It executes an atomic decrement on `InventoryStock` (`currentQty: { decrement: qty }`).
  3. It records an `InventoryTransaction` of type `debit` with reference `EVENT_ALLOCATION`.
  4. It creates or increments the `EventItem` record with `allocatedAt`.
* **Multi-Condition Returns (`RentalEngine.returnItems`):** When teardown teams return items, each item is inspected:
  * Condition `GOOD` credits `MAIN`.
  * Condition `DAMAGED` credits `DAMAGED`.
  * Condition `MISSING` credits `LOST`.
  * The `EventItem` record is stamped with `returnedAt`, `returnCondition`, and optional damage notes.
* **Automated Damage Reconciliation (`reconcileDamages`):** When damaged or missing items are finalized:
  1. A `DAMAGE` quote is generated with itemized replacement/repair costs for the customer.
  2. Simultaneously, an `Expense` write-off entry is created in the database under category "Damage & Loss", immediately reflecting the balance sheet cost in the P&L ledger.

### 3. How Payments Are Represented
* **High-Precision Decimals:** All financial calculations (item prices, quote totals, discounts, taxes, payments, and expenses) use PostgreSQL `Decimal(18, 2)` to eliminate floating-point arithmetic errors.
* **Flexible Customer & Quote Reconciliation:** Payments are captured as `CustomerPayment` records linked to a customer. When a `quoteId` is specified, `FinanceEngine.recordPayment` updates `Quote.amountPaid` and automatically updates `paymentStatus`:
  * If `amountPaid == 0` → `outstanding`
  * If `0 < amountPaid < total` → `partial`
  * If `amountPaid >= total` → `reconciled`
* **Defensive Ledger Calculations:** The customer ledger excludes `DRAFT` and `DECLINED` quotes when computing outstanding balances and total billed amounts, preventing unrealized quotes from corrupting customer balances.

### 4. How Auditability Works
* **Centralized `AuditLog` Table:** Tracks actor (`userId`), action (`create`, `update`, `delete`, `read`), target table (`recordTable`), record ID (`recordId`), `oldValues` (JSON), `newValues` (JSON), and IP address.
* **Transactional Coupling (`tx` parameter):** `logAction` accepts an optional Prisma transaction client. Critical state transitions pass the active transaction to `logAction`, guaranteeing that if the operation fails, the audit record rolls back with it.
* **Non-Blocking Fault Isolation:** If an audit logging write fails due to an unexpected serialization or JSON error, `logAction` catches the exception silently, logging error details to the server console without rolling back legitimate business transactions.
* **Audit Exploration UI:** Accessible under `/settings/audit-log` for administrative inspection of all mutations across the system.

### 5. How Permissions Work
* **Role-Based Access Control (RBAC):** Users are linked to a `Role`, which maps to granular `Permission` records via `RolePermission` using the format `module:action` (e.g., `inventory:manage`, `events:manage`, `finance:manage`, `users:manage`, `settings:manage`).
* **Server-Side Enforcement (`checkPermission`):** Every mutating Server Action calls `checkPermission("module:action")` before performing any database write. Admins (`role === "admin"` or `"super_admin"`) bypass permission checks; unauthorized users receive an immediate rejection.
* **Client-Side Capabilities (`usePermissions`):** A custom React hook inspects JWT claims and exposes `hasPermission()`, `isAdmin`, and `isReadOnly`, enabling conditional UI rendering (hiding/disabling action triggers for unauthorized staff).
* **Edge & Node Middleware (`src/middleware.ts`):** Protects all routes under `/(dashboard)/*`, redirecting unauthenticated requests to `/login`.

---

## 6. Testing: What Was Tested and Why

The platform was verified through targeted integration tests, transaction stress tests, and automated type validation:

| Test Area | What Was Tested | Why It Was Critical |
| :--- | :--- | :--- |
| **Atomic Stock Operations** | Interactive `$transaction` rollbacks on insufficient stock during multi-item allocations | Prevents race conditions and negative inventory counters when multiple staff allocate items simultaneously. |
| **Multi-Store Return Routing** | Event returns split across `GOOD`, `DAMAGED`, and `MISSING` conditions | Ensures stock accurately moves to `DAMAGED` or `LOST` stores rather than returning to rentable `MAIN` stock. |
| **Damage Reconciliation Flow** | Atomic creation of `DAMAGE` quotes alongside corresponding P&L expense write-offs | Prevents accounting discrepancies where damages are billed to customers without write-offs appearing on company financial statements. |
| **Customer Ledger Integrity** | Exclusion of `DRAFT` and `DECLINED` quotes from customer `totalBilled` and balance tallies | Ensures customer account statements reflect only agreed-upon, enforceable financial obligations. |
| **Server Action Permissions** | Direct invocation of Server Actions with non-privileged staff accounts | Verifies that UI button disabling cannot be bypassed by sending raw Server Action requests. |
| **Multi-Location Availability** | Stock queries with overlapping event dates across different warehouse location IDs | Guarantees that inventory allocated in one warehouse does not artificially deplete stock in another. |
| **Zod Input Boundary Validation** | Edge-case payloads (negative rental amounts, duplicate item tags, missing dates) | Ensures human-readable form errors propagate cleanly to the UI without triggering Next.js 500 runtime crashes. |
| **TypeScript & Linting Strictness** | Full type-checking via `tsc --noEmit` and ESLint with generated Prisma client filtering | Ensures strict type safety across Next.js 16 (React 19) without unhandled promises or `any` type violations. |

---

## 7. Deployment: How It Is Deployed

Decora is deployed on modern cloud infrastructure optimized for serverless Next.js applications:

* **Compute & Hosting:** **Vercel**
  * Deploys Next.js App Router with the `nodejs` runtime for Server Components, Server Actions, and on-demand `@react-pdf/renderer` PDF generation.
* **Database:** **Neon Serverless PostgreSQL**
  * Serverless Postgres database connected through `@prisma/adapter-neon` via WebSockets, eliminating connection exhaustion on serverless cold starts.
* **Asset Storage:** **Cloudinary**
  * Handles image uploads for inventory catalog items and teardown damage proof photos via `src/app/api/upload/route.ts`.
* **Push Notifications:** **OneSignal**
  * Real-time web push notifications dispatching operational updates to staff devices.
* **Build & Migration Pipeline:**
  * `package.json` includes `"postinstall": "prisma generate"` ensuring the Prisma Client (`src/generated/prisma`) is compiled upon deployment.
  * Schema updates are managed via `npx prisma migrate deploy` or `npx prisma db push`.
  * **Firewalled Network Escape Hatch (`scripts/apply-schema.ts`):** In environments where corporate or ISP firewalls block standard PostgreSQL TCP port 5432, migrations can be applied over WebSockets via Neon's serverless connection pool:
    ```bash
    # 1. Generate diff script without live DB connection
    npx prisma migrate diff --from-empty --to-schema prisma/schema.prisma --script -o prisma/current-schema.sql
    # 2. Execute SQL over WebSocket (HTTP/WS)
    npx tsx scripts/apply-schema.ts
    ```

---

## 8. Environment Variables

Create a `.env` file in the project root based on `.env.example`:

```env
# Database (Neon Serverless PostgreSQL or local PostgreSQL)
DATABASE_URL="postgresql://user:password@ep-xyz.neon.tech/as-deco?sslmode=require"

# NextAuth v5 Secret
AUTH_SECRET="generate-with-openssl-rand-base64-32"
NEXTAUTH_URL="http://localhost:3000"

# Cloudinary (Item and damage photos)
CLOUDINARY_CLOUD_NAME="your-cloud-name"
CLOUDINARY_API_KEY="your-api-key"
CLOUDINARY_API_SECRET="your-api-secret"

# OneSignal (Web Push Notifications)
NEXT_PUBLIC_ONESIGNAL_APP_ID="your-onesignal-app-id"
ONESIGNAL_REST_API_KEY="your-onesignal-rest-key"
```

---

## 9. Local Development & Setup

### Prerequisites
* Node.js 20+
* npm or pnpm
* PostgreSQL database (local or Neon)

### Setup Steps

1. **Clone the repository and install dependencies:**
   ```bash
   git clone https://github.com/Abdulhadiny/a-and-s-deco.git
   cd a-and-s-deco
   npm install
   ```

2. **Configure environment variables:**
   ```bash
   cp .env.example .env
   # Populate DATABASE_URL and AUTH_SECRET in .env
   ```

3. **Generate Prisma client and push schema:**
   ```bash
   npx prisma generate
   npx prisma db push
   ```

4. **Seed initial database records:**
   Seeds the default location (`main-warehouse`), stores (`MAIN`, `DAMAGED`, `LOST`), system roles (`admin`, `staff`), permissions, default expense categories, and an initial administrator account:
   ```bash
   npx tsx prisma/seed.ts
   ```

5. **Start the development server:**
   ```bash
   npm run dev
   ```
   Open [http://localhost:3000](http://localhost:3000) to access Decora.

---

## 10. Available Scripts

* `npm run dev` — Starts the local Next.js development server.
* `npm run build` — Runs production build with Next.js compiler.
* `npm run start` — Starts the production Next.js server.
* `npm run lint` — Runs ESLint across the codebase.
* `npx prisma studio` — Opens the interactive Prisma database GUI.
* `npx tsx prisma/seed.ts` — Seeds the database with default configuration and roles.
