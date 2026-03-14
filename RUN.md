# How to Run the Core Inventory Web App

Follow these steps in order. You need **Node.js** (v18+) and **MySQL** installed and running.

---

## Step 1: Start MySQL

- Make sure **MySQL Server** is running on your machine.
- Note your MySQL **username** and **password** (often `root` with no password for local dev).

---

## Step 2: Create the Database and Tables

**Option A – Using the Node script (recommended)**

1. Open a terminal in the project folder.
2. Go to the backend and copy the env file:

   ```powershell
   cd c:\Users\Asus\OneDrive\Desktop\CoreInventory\backend
   copy .env.example .env
   ```

3. Edit **`.env`** and set your MySQL password (and user/host if different):

   ```
   DB_HOST=localhost
   DB_PORT=3306
   DB_USER=root
   DB_PASSWORD=YOUR_MYSQL_PASSWORD
   DB_NAME=core_inventory
   JWT_SECRET=any-long-secret-string
   ```

4. Run the migration to create the database and tables:

   ```powershell
   npm install
   node src/db/migrate.js
   ```

   You should see: `Database migration completed.`

**Option B – Using MySQL command line**

```powershell
mysql -u root -p < backend\src\db\schema.sql
```

Enter your MySQL password when asked.

---

## Step 3: Run the Backend

1. In the same terminal (or a new one), from the **backend** folder:

   ```powershell
   cd c:\Users\Asus\OneDrive\Desktop\CoreInventory\backend
   npm run dev
   ```

2. When it’s ready you’ll see something like:
   - `IMS Backend running at http://localhost:4000`
   - `Swagger UI: http://localhost:4000/api-docs`

3. Leave this terminal **open**; the backend must keep running.

---

## Step 4: Run the Frontend

1. Open a **second** terminal.
2. Go to the frontend folder and start the app:

   ```powershell
   cd c:\Users\Asus\OneDrive\Desktop\CoreInventory\frontend
   npm install
   npm run dev
   ```

3. When it’s ready, Vite will show something like:
   - `Local: http://localhost:3000`

4. Leave this terminal **open** as well.

---

## Step 5: Open the Web App

1. In your browser go to: **http://localhost:3000**
2. You should see the **login** page.
3. Click **“Sign up”** and create an account (e.g. email + password).
4. After signup you’re logged in and see the **Dashboard**.

---

## Quick Reference

| What              | URL / Command                          |
|-------------------|----------------------------------------|
| **Web app**       | http://localhost:3000                  |
| **API (backend)** | http://localhost:4000                  |
| **API docs**      | http://localhost:4000/api-docs         |
| **Backend**       | `cd backend` → `npm run dev`           |
| **Frontend**      | `cd frontend` → `npm run dev`          |

---

## Optional: Send OTP by email (password reset)

By default the backend does not send email; the OTP is only in the API response or console. To **send the OTP to the user’s email**:

1. Add mail settings to **`backend/.env`**. Example for **Gmail** (use an [App Password](https://support.google.com/accounts/answer/185833)):
   ```
   MAIL_HOST=smtp.gmail.com
   MAIL_PORT=587
   MAIL_SECURE=false
   MAIL_USER=your@gmail.com
   MAIL_PASSWORD=your-app-password
   MAIL_FROM=your@gmail.com
   ```
2. See **`backend/EMAIL_SETUP.md`** for Gmail, Mailtrap, or other SMTP.

---

## Troubleshooting

- **“Cannot connect to database”**  
  Check that MySQL is running and that `DB_USER`, `DB_PASSWORD`, and `DB_HOST` in `backend/.env` are correct.

- **“Port 4000 already in use”**  
  Another app is using port 4000. Stop it or change `PORT=4000` in `backend/.env` to another port (e.g. `4001`). If you change it, the frontend proxy in `frontend/vite.config.ts` must target that port.

- **“Port 3000 already in use”**  
  Change the frontend port in `frontend/vite.config.ts` (e.g. `server: { port: 3001 }`).

- **Blank page or API errors**  
  Make sure the **backend** is running on port 4000 when you use the app on port 3000.
