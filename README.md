# Poultry Management System

Next.js app with PostgreSQL for local development. Database access uses standard PostgreSQL via `pg`; migrations are plain SQL so the same schema can later be applied to Supabase PostgreSQL.

## Local setup

1. Copy `.env.example` to `.env.local` and set `DATABASE_URL` with the local PostgreSQL user and password.
2. Install packages with `npm install`.
3. Create the database with `npm run db:create` (requires a PostgreSQL role with database creation rights).
4. Apply SQL migrations with `npm run db:migrate`.
5. Start the app with `npm run dev` and open [http://localhost:3000](http://localhost:3000).

The database connection endpoint is `GET /api/health/db`. It reports the connected database and PostgreSQL version without exposing connection details.

Local migrations use `DATABASE_URL`. Supabase migrations use the separate `SUPABASE_DATABASE_URL` and the explicit `npm run db:migrate:supabase` command. On this IPv4-only Windows environment, use Supabase?s **Session pooler** connection string from the Dashboard Connect dialog. The command checks the project-specific pooler username and never falls back to the local URL. Migrations do not run automatically when the app starts.

## Database changes

Add ordered `.sql` files to `db/migrations`. The migration runner applies each file once and records it in `schema_migrations`. Migrations are transactionally applied.

The migrations create user profiles, farms, farm memberships, houses, batches, daily production records, and inventory. Supabase Auth user IDs map to profile IDs; the local development identity remains available only when running without Supabase configuration in development. Farm codes are generated from the farm name, with a numeric suffix when a code is already in use. Farm creation and batch label generation run transactionally. Houses receive immutable letter codes; batch labels include a house letter when the farm has multiple houses and monthly sequences are scoped by house. Daily bird counts carry forward from the prior saved day and recalculate later records when an earlier entry changes. Stock balances are calculated from immutable movements; receipts create lots, issues use FEFO, and count variances require manager or owner approval. The database enforces one owner per farm, one active batch per house, and one daily record per batch per date.

For Supabase Auth, set `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`, and `NEXT_PUBLIC_APP_URL`. Add the app URL and `/auth/callback` to the Supabase Auth redirect allowlist. Account creation, email verification, sign-in, sign-out, and password reset use cookie-based SSR sessions. Farm data queries run under the restricted `flockbook_app` role with a verified user ID, and migration `012_farm_row_level_security.sql` applies farm-scoped RLS policies. Configure the production database login so it can `SET ROLE flockbook_app`; keep migration-owner credentials separate and server-only.

See [docs/supabase-auth-setup.md](docs/supabase-auth-setup.md) for the project setup and local-development behavior.

Open **Daily records** from a farm workspace to enter mortality, culls, feed given and left over, water, temperatures, litter, illness signs, and notes. Net feed used is calculated automatically and deducted from available inventory using earliest-expiry lots first. Configure one active inventory item per feed type; when stock is short, the record still saves and shows the undeducted amount.

Open **Inventory** from a farm workspace to add stock items, receive deliveries as lots, issue stock by FEFO, record wastage, review expiry and low-stock alerts, and submit physical counts for approval. Daily feed entries are linked to inventory issues. Receipts are not yet posted to the cash book.

## Supabase production configuration

Supabase uses PostgreSQL, so the SQL migrations can be applied there. Production deployment still requires configuring the Supabase Auth redirect allowlist and a runtime database login permitted to `SET ROLE flockbook_app`, then confirming the RLS policies work with that connection. Keep migration-owner credentials separate and server-only. Never use service-role credentials in browser code.
