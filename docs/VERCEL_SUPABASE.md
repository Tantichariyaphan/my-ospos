# Deploy OSPOS on Vercel with Supabase

## Status and constraints

This repository now includes a Vercel PHP function entrypoint and production support for a `DATABASE_URL` environment variable. The URL may use the `postgres://` or `postgresql://` scheme supplied by Supabase; `app/Config/Database.php` converts it to CodeIgniter's `Postgre://` DSN format.

This does not make OSPOS PostgreSQL-compatible. OSPOS is a MySQL application: its supplied schema and migrations use MySQL constructs, and application queries use functions such as `GROUP_CONCAT`, `DATE_FORMAT`, `IFNULL`, `ENGINE=MEMORY`, and MySQL temporary-table syntax. A PostgreSQL migration must replace those schema definitions and queries, then run the application and report test suites against PostgreSQL before production data is used.

Vercel Functions also have a read-only deployed filesystem. OSPOS writes item images and company logos under `public/uploads`, so production image uploads require an object-storage implementation (for example, Supabase Storage) before this is suitable for normal POS use. Do not rely on `/tmp` for durable files.

## 1. Prepare the repository

1. Create a fork or repository on GitHub.
2. Commit the Vercel files and the configuration change. Do not commit `.env`, `.env.local`, database URLs, encryption keys, or throttle keys.
3. Install PHP dependencies locally with `composer install` and run `composer test`.
4. Push the branch that Vercel will deploy.

The Vercel PHP runtime runs `composer install`. `.vercelignore` excludes local dependency folders so Vercel builds from `composer.lock` rather than uploading local packages.

## 2. Port OSPOS to PostgreSQL before importing live data

1. Inventory every MySQL-specific migration and query.
2. Create PostgreSQL schema migrations for all OSPOS tables, indexes, sequences, default values, and the `sessions` table.
3. Replace MySQL-only SQL with PostgreSQL equivalents and test every report, sale, receiving, import, and image-upload flow.
4. Run the ported migrations against an empty Supabase project and load only a tested export.

Keep the table prefix as `ospos_` unless the ported schema intentionally changes it. Do not run the existing MySQL SQL scripts directly in Supabase.

## 3. Create the Supabase connection

1. Create a Supabase project in a region close to the Vercel function region.
2. In Supabase, select **Connect** and choose **Transaction pooler**. This is the connection mode Supabase recommends for serverless functions.
3. Copy the complete connection string. Its shared-pooler form uses port `6543` and a username like `postgres.[PROJECT-REF]`.
4. If the database password contains reserved URL characters such as `&`, `#`, `?`, or spaces, percent-encode it in the URL.
5. This repository pins `vercel-php@0.9.0`, whose documented PHP 8.5 runtime includes `pgsql` and `pdo_pgsql`. Confirm its PHP patch version satisfies the PHP constraint in `composer.json` in the first build log.

Use the direct connection only for one-off migrations, backups, or database tools, not for the serverless application.

## 4. Configure Vercel

1. In Vercel, choose **Add New → Project** and import the GitHub repository.
2. Leave the project at the repository root. Vercel reads `vercel.json`, creates `api/index.php` as the PHP function, serves files with an extension from `public/`, and routes remaining application requests to CodeIgniter's existing `public/index.php`.
3. Open **Settings → Environment Variables** and add these values for Production. Add separate Preview values only if that deployment has its own database.

   | Name | Value |
   | --- | --- |
   | `CI_ENVIRONMENT` | `production` |
   | `DATABASE_URL` | Supabase Transaction pooler URL, including `?sslmode=require` |
   | `ALLOWED_HOSTNAMES` | The Vercel domain and any custom domains, comma-separated, without `https://` |
   | `ENCRYPTION_KEY` | A new random persistent secret |
   | `THROTTLE_KEY` | A separate new random persistent secret |

4. Store `DATABASE_URL`, `ENCRYPTION_KEY`, and `THROTTLE_KEY` as Vercel Secrets. Do not place their values in `vercel.json`.
5. Redeploy after changing environment variables; Vercel only injects changes into new deployments.

## 5. Deploy and verify

1. Click **Deploy**. Every push to the configured production branch will then create a production deployment; other branches create previews.
2. Open the deployment URL and verify the host is listed in `ALLOWED_HOSTNAMES`.
3. Check the Vercel function logs for database-driver or missing-extension errors without logging secrets.
4. Smoke-test login, sales, reports, sessions, static assets, and uploads. Keep the deployment out of production until the PostgreSQL port and durable upload storage both pass these checks.

## Local configuration

For a local PostgreSQL-port test, set `CI_ENVIRONMENT=production` and `DATABASE_URL` in an untracked `.env` file. The application only selects the PostgreSQL DSN from `DATABASE_URL` in the production environment, preventing the test and development groups from accidentally pointing to the deployed database.
