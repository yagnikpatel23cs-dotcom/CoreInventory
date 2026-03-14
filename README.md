# Core Inventory Management System (IMS)

Modular Inventory Management System with React + Tailwind frontend, Node.js + Express backend, and MySQL database.

## Features

- **Authentication**: Sign-up, Login, OTP-based password reset
- **Dashboard**: KPIs (Total Products, Low Stock, Pending Receipts, Pending Deliveries, Scheduled Transfers) and dynamic filters (document type, status, warehouse, category)
- **Product Management**: CRUD with Name, SKU, Category, UoM, Initial stock, low stock threshold
- **Receipts (Incoming)**: Create → Add products/qty → Validate (stock increases automatically)
- **Delivery Orders (Outgoing)**: Pick → Pack → Validate (stock decreases automatically)
- **Internal Transfers**: Move stock between warehouses/locations (total stock unchanged)
- **Stock Adjustments**: Correct recorded vs physical count (system updates and logs difference)
- **Stock Ledger**: Central log of every movement with filters
- **Multi-warehouse / Multi-location** support
- **Low stock alerts** (dashboard + product list filter)

## Tech Stack

- **Frontend**: React 18, TypeScript, Vite, Tailwind CSS, Lucide icons, React Router
- **Backend**: Node.js, Express, MySQL2, JWT, bcryptjs, express-validator
- **Docs**: Swagger/OpenAPI at `/api-docs`

## Setup

### 1. MySQL

Create the database and tables:

```bash
# Ensure MySQL is running, then from project root:
cd backend
# Set DB credentials in .env (copy from .env.example)
node src/db/migrate.js
```

Or run the SQL file manually:

```bash
mysql -u root -p < backend/src/db/schema.sql
```

### 2. Backend

```bash
cd backend
cp .env 
# Edit .env: DB_PASSWORD, JWT_SECRET, etc.
npm install
npm run dev
```

Backend runs at **http://localhost:4000**. Swagger UI: **http://localhost:4000/api-docs**.

### 3. Frontend

```bash
cd frontend
npm install
npm run dev
```

Frontend runs at **http://localhost:3000** and proxies `/api` to the backend.

## Project Structure

```
CoreInventory/
├── backend/
│   ├── src/
│   │   ├── config/db.js
│   │   ├── db/schema.sql, migrate.js
│   │   ├── middleware/auth.js, errorHandler.js
│   │   ├── routes/ (auth, dashboard, products, warehouses, receipts, deliveries, transfers, adjustments, stockLedger)
│   │   ├── controllers/
│   │   ├── services/ (stock.js, ledger.js)  # Transactions + ledger logging
│   │   ├── app.js, server.js, swagger.js
│   └── package.json
├── frontend/
│   ├── src/
│   │   ├── components/ (Layout, ProductForm, ReceiptForm, ...)
│   │   ├── contexts/AuthContext.tsx
│   │   ├── lib/api.ts, utils.ts
│   │   ├── pages/ (Dashboard, Products, Receipts, ...)
│   │   ├── App.tsx, main.tsx, index.css
│   └── package.json
└── README.md
```

## Stock Logic (transactions)

- All stock-changing operations use **database transactions** (getConnection → beginTransaction → ... → commit / rollback).
- **Receipt validate**: increases `product_stock` per line and inserts into `stock_ledger` (movement_type `receipt`).
- **Delivery validate**: decreases `product_stock` (checks sufficient qty) and inserts into `stock_ledger` (`delivery`).
- **Transfer validate**: decreases from source location, increases at destination, logs `transfer_out` and `transfer_in`.
- **Adjustment validate**: for each line, applies difference (physical − recorded) as increase or decrease and logs as `adjustment`.

## OTP (password reset)

In development, the OTP is logged to the server console. In production, wire `requestPasswordReset` to your email provider (e.g. Nodemailer, SendGrid).
