# How the Core Inventory Web App Works

This document explains the **architecture**, **workflow**, and **data flow** of the app.

---

## 1. High-Level Architecture

```
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│   BROWSER       │  HTTP   │   BACKEND        │   SQL   │   MYSQL         │
│   (React app    │ ──────► │   (Node/Express) │ ──────► │   (Database)    │
│   on port 3000) │ ◄──────  │   on port 4000   │ ◄────── │   core_inventory│
└─────────────────┘         └─────────────────┘         └─────────────────┘
        │                            │
        │  User clicks,               │  Validates JWT,
        │  fills forms,               │  runs business logic,
        │  sees data                  │  updates DB, returns JSON
```

- **Frontend (React):** Runs in the user’s browser. Shows pages, forms, and tables. Sends requests to the backend (e.g. `/api/products`, `/api/auth/login`).
- **Backend (Express):** Receives requests, checks auth (JWT), reads/writes the database, and returns JSON. All stock changes use **transactions**.
- **Database (MySQL):** Stores users, products, stock, documents (receipts, deliveries, transfers, adjustments), and the **stock ledger**.

When you run the app locally, the frontend is at `http://localhost:3000` and proxies `/api` to the backend at `http://localhost:4000`.

---

## 2. Authentication Workflow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Sign up    │     │   Login      │     │  Forgot pwd  │
│   (email,    │     │   (email,    │     │  (email →    │
│   password,  │     │   password)  │     │   OTP →      │
│   name)      │     │              │     │   new pwd)   │
└──────┬───────┘     └──────┬───────┘     └──────────────┘
       │                    │
       ▼                    ▼
   Backend creates      Backend checks
   user, hashes pwd,    credentials,
   returns JWT           returns JWT
       │                    │
       ▼                    ▼
   Frontend saves token in localStorage
   Sends "Authorization: Bearer <token>" on every API request
   Backend middleware checks token → allows or blocks request
```

- **Sign up:** Backend creates a user (hashed password), returns a **JWT**. Frontend stores the token and uses it for later requests.
- **Login:** Backend verifies email/password, returns JWT. Same storage and usage.
- **Forgot password:** User enters email → backend creates an **OTP** (saved in DB, in dev printed in console) → user enters OTP + new password → backend updates password and invalidates OTP.
- **Protected routes:** Dashboard, Products, Receipts, etc. only load if the backend accepts the JWT (middleware checks token and user).

---

## 3. Dashboard Workflow

- **On load:** Frontend calls `GET /api/dashboard/kpis` and `GET /api/dashboard/documents`.
- **Backend:** Runs SQL to get:
  - **KPIs:** total products, count of low-stock products, pending receipts, pending deliveries, scheduled transfers.
  - **Documents:** list of receipts, deliveries, transfers, adjustments (optionally filtered by type, status, warehouse, category).
- **Frontend:** Shows KPI cards and a documents table. Filters (dropdowns) change the query params and refetch documents.

So: **Dashboard = read-only summary + filters**. No stock changes here.

---

## 4. Product Management Workflow

- **List:** `GET /api/products` (optional: sku, categoryId, warehouseId, lowStockOnly). Backend returns products with total stock (and category/UoM names).
- **Create:** User fills form (SKU, name, category, UoM, low-stock threshold, optional initial stock + location). Frontend sends `POST /api/products`. Backend inserts product and, if initial stock is set, inserts/updates `product_stock` and writes an **adjustment** row in the **stock ledger**.
- **Edit:** Same as create but `PUT /api/products/:id`; only product fields, no stock change unless you add that feature.
- **Delete:** `DELETE /api/products/:id`; backend deletes the product (DB cascades to stock and ledger references).

Products are the **master data**; stock is stored per product per location in `product_stock`.

---

## 5. Receipts (Incoming Stock) Workflow

```
Create receipt (draft)  →  Add lines (product + location + qty)  →  Validate
       │                                                                  │
       │  No stock change yet                                            │  Stock INCREASES
       │                                                                  │  Ledger: movement_type = 'receipt'
       ▼                                                                  ▼
   receipts + receipt_items in DB                          product_stock updated,
   status = 'draft'                                         stock_ledger rows inserted
```

- **Create:** User picks warehouse, optional supplier, then adds lines (product, location, quantity). Frontend sends `POST /api/receipts` with `items[]`. Backend creates `receipts` row (status `draft`) and `receipt_items`. **Stock does not change yet.**
- **Validate:** User clicks Validate. Frontend calls `POST /api/receipts/:id/validate`. Backend:
  1. Starts a **DB transaction**.
  2. For each line: **increases** `product_stock` for that product + location, and **inserts** a row in `stock_ledger` (movement_type `receipt`, reference = receipt id).
  3. Sets receipt status to `validated`.
  4. Commits (or rolls back on error).

So: **Receipt = document first, stock only changes on Validate.**

---

## 6. Delivery Orders (Outgoing Stock) Workflow

```
Create delivery (draft)  →  Add lines  →  Pick  →  Pack  →  Validate
       │                                                      │
       │  No stock change                                      │  Stock DECREASES
       │                                                      │  Ledger: movement_type = 'delivery'
       ▼                                                      ▼
   delivery_orders + delivery_items                  product_stock decreased,
   status: draft → picked → packed → validated       stock_ledger rows inserted
```

- **Create:** Same idea as receipts: warehouse, lines (product, location, quantity). `POST /api/deliveries`. Status stays `draft`; no stock change.
- **Pick / Pack:** Frontend calls `PATCH /api/deliveries/:id/status` with `status: 'picked'` or `'packed'`. Backend only updates the status; **still no stock change**.
- **Validate:** User clicks Validate. Frontend calls `POST /api/deliveries/:id/validate`. Backend in a **transaction**:
  1. For each line: checks **sufficient** `product_stock`, then **decreases** it and **inserts** into `stock_ledger` (movement_type `delivery`).
  2. Sets delivery status to `validated`.
  3. Commits or rolls back.

So: **Delivery = draft → pick/pack (optional steps) → Validate = stock goes down.**

---

## 7. Internal Transfers Workflow

```
Create transfer (from warehouse + to warehouse + lines)  →  Validate
       │       Each line: product, FROM location, TO location, qty        │
       │                                                                  │
       │  No stock change                                                 │  Stock MOVES:
       │                                                                  │  FROM location decreases,
       ▼                                                                  │  TO location increases.
   internal_transfers + transfer_items                                   │  Ledger: 'transfer_out' + 'transfer_in'
   status: draft or scheduled                                             ▼
                                                              Total company stock unchanged
```

- **Create:** User selects from/to warehouse and for each line: product, from location, to location, quantity. `POST /api/transfers`. Backend creates header + lines; **no stock change**.
- **Validate:** `POST /api/transfers/:id/validate`. Backend in a **transaction**:
  1. For each line: **decrease** `product_stock` at **from** location, **increase** at **to** location.
  2. Writes **two** ledger rows per line: `transfer_out` (from) and `transfer_in` (to).
  3. Marks transfer as validated.

So: **Transfer = move stock between locations; total stock stays the same.**

---

## 8. Stock Adjustments Workflow

```
Create adjustment  →  Add lines (product, location, recorded qty, physical qty)  →  Validate
       │                                                                              │
       │  No stock change                                                             │  Stock updated to match
       │                                                                              │  physical count.
       ▼                                                                              ▼
   stock_adjustments + adjustment_items                                    difference = physical - recorded
   (recorded_qty = system, physical_qty = count)                          If + → increase stock; if - → decrease
                                                                          Ledger: movement_type = 'adjustment'
```

- **Create:** User picks warehouse, then for each line: product, location, **recorded qty** (what system says), **physical qty** (what they counted). `POST /api/adjustments`. No stock change yet.
- **Validate:** `POST /api/adjustments/:id/validate`. Backend in a **transaction**:
  1. For each line: difference = physical − recorded. If positive, **increase** stock; if negative, **decrease** (with sufficiency check).
  2. Inserts **stock_ledger** rows (movement_type `adjustment`).
  3. Marks adjustment as validated.

So: **Adjustment = correct system stock to match physical count.**

---

## 9. Stock Ledger (Move History)

- **Purpose:** One place where **every** stock change is logged: receipt, delivery, transfer in/out, adjustment (and initial product stock if you added that).
- **Frontend:** Calls `GET /api/ledger` with optional filters (productId, locationId, movementType, fromDate, toDate). Shows a table of movements.
- **Backend:** Reads from `stock_ledger` (joined with product, location, warehouse) and returns rows. **No writes here**; ledger rows are written only by the validate actions above.

So: **Ledger = read-only history of all stock movements.**

---

## 10. Summary: When Does Stock Change?

| Action                    | When stock changes | Ledger entry        |
|---------------------------|--------------------|---------------------|
| Product create (initial) | On create          | adjustment          |
| Receipt **Validate**      | On validate        | receipt             |
| Delivery **Validate**     | On validate        | delivery            |
| Transfer **Validate**     | On validate        | transfer_out + transfer_in |
| Adjustment **Validate**   | On validate        | adjustment          |

All of these run inside **database transactions** so either everything is saved or nothing is (no half-updated stock).

---

## 11. Quick Data Flow Summary

1. **User** uses the React app (browser).
2. **Frontend** sends HTTP requests to `/api/...` with JWT when logged in.
3. **Backend** checks JWT, runs the right route/controller, talks to **MySQL** (and for stock changes, uses transactions and the ledger).
4. **Backend** returns JSON; **frontend** updates the UI (tables, forms, KPIs).

That’s how the web app and its workflows work end to end.
