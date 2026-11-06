# Restaurant Management System

React + Vite + Tailwind frontend, Supabase (PostgreSQL, Auth, RLS, Edge Functions) backend, deployed on Vercel.
Cash book, sales, purchases, suppliers, inventory, staff and restaurant payments, reports (Excel/PDF/print).

## Step 1 – Extract
Right-click `restaurant-management-system.zip` → Extract All. Open the folder; `package.json` must be directly inside.

## Step 2 – Supabase
1. Create a project at supabase.com.
2. **SQL Editor** → run `supabase/migrations/001_schema.sql`, then `002_functions_rls.sql` (in this order).
3. **Authentication → Providers**: keep Email enabled. For quick testing you may turn off "Confirm email".
4. **Authentication → URL Configuration**: set Site URL to your Vercel URL (and `http://localhost:5173` for testing).
5. **Database → Extensions**: enable `pg_cron` (for the weekly activity-log cleanup). If it was not enabled when you ran file 002, enable it and run:
   `select cron.schedule('cleanup-activity-logs','30 20 * * *','select public.cleanup_activity_logs()');`
6. **Edge Function** (used to add staff). Install the Supabase CLI, then in the project folder:
   ```
   supabase login
   supabase link --project-ref YOUR_PROJECT_REF
   supabase functions deploy admin-users
   ```
   The service-role key is provided to the function by Supabase itself. It is never in the React app.
7. **Project Settings → API**: copy the Project URL and the `anon` / publishable key.

## Step 3 – Environment
Copy `.env.example` to `.env.local`:
```
VITE_SUPABASE_URL=your_project_url
VITE_SUPABASE_ANON_KEY=your_anon_key
```

## Step 4 – Run locally
```
npm install
npm run dev
```
Open http://localhost:5173. On the sign-in page click **First-time setup** to create the restaurant and the Owner account (works only once).

## Step 5 – GitHub
Create a repository and upload the whole folder (not `node_modules`, not `.env.local`; `.gitignore` already excludes them).
`package.json`, `index.html`, `src`, `public`, `supabase`, `vercel.json` must be visible at the repo root.

## Step 6 – Vercel
Import the GitHub repo. Framework: Vite. Build command `npm run build`, output `dist`.
Add `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` under Environment Variables, then Deploy.
`vercel.json` already rewrites all routes to `index.html`.

## Step 7 – Verify
Owner login → add a Manager and an Employee (Employee Management) → employee signs in from the invitation email → check the employee sees only Dashboard/Inventory/Stock/Activity/Reports (inventory only) → add stock, use stock → open Inventory Activity Log → add a sale, purchase (try Credit/Udhaar), pay a supplier → Reports → download Excel/PDF.

## How the rules are enforced (in the database, not the browser)
- Money tables (sales, purchases, payments, cash ledger, reports) are readable only by Owner/Manager through Row Level Security.
- Stock quantity can only change through the `stock_move` function (atomic, locks the row, blocks negatives, is idempotent, records who/when/before/after). A trigger blocks direct quantity edits.
- `stock_transactions` (permanent) and `inventory_activity_logs` (7-day view) are separate. The nightly cleanup touches only the latter. Deletes on the permanent stock and cash ledgers are blocked.
- Credit purchases create a payable and do not touch cash. Supplier payments are separate ledger entries. Platform sales are not cash until a payout is recorded, and payouts are not counted as sales.
- Removing a sale/purchase/payment is a soft delete; its cash entries are marked reversed. Every change is written to Audit Logs.
- Users cannot change their own role or permissions; role changes go through server-side functions.

## Notes
- Employee permissions you can assign are inventory-related (view, add, use, activity, reports). Money pages are Owner/Manager only by design.
- Employees see their own inventory activity unless given "View all inventory activity".
