# Setup guide

How to run the returns app on your own machine, then on your own hosted Supabase project. For how the database behaves once it is running, see [database.md](database.md).

## What you need

- [Supabase CLI](https://supabase.com/docs/guides/cli) and Docker (for the local database)
- Python 3 (to serve the site, and for the optional reply engine)
- Node 18 or later (only for the tests)
- A free [Supabase](https://supabase.com) account for the hosted version
- Optional: a Slack incoming webhook, and either a Gmail account with an app password or a Resend account, for the alerts

## 1. Run it locally

```bash
supabase start          # starts Postgres, Auth and the API in Docker
supabase db reset       # runs every migration, including the demo data
supabase status         # prints the local API URL and the anon key
```

Open `frontend/supabase-public.example.js` and set `SUPABASE_ANON_KEY` to the anon key from `supabase status`. Leave `SUPABASE_URL` as `http://127.0.0.1:54321`.

```bash
cd frontend
python3 -m http.server 3000
```

Open http://localhost:3000. Local email confirmation is off, so you can sign up and sign in straight away. Local Supabase catches outgoing mail in a local mail viewer (its URL is in `supabase status`) instead of sending it.

## 2. The migrations

They live in `supabase/migrations/` and run in date order. Never edit one that has already been pushed; add a new file with `supabase migration new <name>`.

| File | What it adds |
|---|---|
| `20261113120000_init_schema.sql` | Tables, sign-up trigger, login log, return rules, row level security, South African public holidays 2026 and 2027 |
| `20261113120100_seed_dummy_data.sql` | Demo staff and `DEMO-*` orders for `customer.one@example.com` and friends |
| `20261113140000_contact_forwarding.sql` | Business hours, after-hours flagging, `pg_net` and `pg_cron`, the one-minute timer that calls `forward-contact-requests` |
| `20261113150000_return_eligibility.sql` | `return_decision()` and the per-item eligibility checks the shop reads |
| `20261113160000_food_not_returnable.sql` | Food items can never be returned |
| `20261113170000_place_order.sql` | `place_order()`, the only way a customer can create an order |
| `20261113180000_return_notifications.sql` | Return status outbox and the one-minute timer that calls `notify-return-events` |
| `20261113190000_eligibility_text_casts.sql` | Type fix for the eligibility wording |
| `20261113200000_cancel_order_items.sql` | Customers can cancel items that have not shipped |

## 3. Create the hosted project

1. Create a project in the Supabase dashboard. Note its **project ref** (the `abcd...` part of `https://abcd....supabase.co`).
2. Link and push the schema:

   ```bash
   supabase login
   supabase link --project-ref <your-project-ref>
   supabase db push
   ```

3. Deploy the three Edge Functions. `supabase/config.toml` already turns off the user-login check for them (`verify_jwt = false`), because they are called by a timer or by signed-out visitors and check their own secret or key instead.

   ```bash
   supabase functions deploy forward-contact-requests
   supabase functions deploy notify-return-events
   supabase functions deploy log-auth-event
   ```

## 4. Function secrets

Set these with `supabase secrets set NAME=value`. They never go in the frontend or in Netlify. `SUPABASE_URL`, `SUPABASE_ANON_KEY` and `SUPABASE_SERVICE_ROLE_KEY` are provided to functions automatically.

| Secret | Used by | What it is |
|---|---|---|
| `FORWARD_SECRET` | forward-contact-requests | Any long random string. Must match Vault `forward_secret` |
| `SLACK_WEBHOOK_URL` | forward-contact-requests | Slack webhook for after-hours support messages |
| `NOTIFY_RETURN_SECRET` | notify-return-events | Any long random string. Must match Vault `notify_return_secret` |
| `SLACK_RETURNS_WEBHOOK` | notify-return-events | Slack webhook for return status changes |
| `OWNER_EMAIL` | notify-return-events | Where the shop owner's copy of each return update goes |
| `GMAIL_ADDRESS`, `GMAIL_APP_PASSWORD` | both | Gmail sender (port 465). Needs 2-step verification on the Google account to create an app password |
| `RESEND_API_KEY`, `EMAIL_FROM` | notify-return-events | Alternative to Gmail; needs a domain you own |
| `EMAIL_FROM_NAME`, `BRAND_NAME` | both, optional | Display name and brand in emails |
| `NOTIFY_MODE` | notify-return-events, optional | `mock` skips all sending. Local testing only |

Make a random secret with `openssl rand -hex 32`.

## 5. Vault entries (turns the timers on)

The database timers read the function URLs and shared secrets from Supabase Vault. Until all four exist, the timers run but do nothing. Run this in the SQL editor:

```sql
select vault.create_secret('https://<your-project-ref>.supabase.co/functions/v1/forward-contact-requests', 'forward_function_url');
select vault.create_secret('<same value as FORWARD_SECRET>', 'forward_secret');
select vault.create_secret('https://<your-project-ref>.supabase.co/functions/v1/notify-return-events', 'notify_return_function_url');
select vault.create_secret('<same value as NOTIFY_RETURN_SECRET>', 'notify_return_secret');
```

## 6. Auth settings (dashboard)

- **Authentication, URL Configuration**: set Site URL to your live site, and add it (plus `/index.html`) to Redirect URLs. Otherwise confirmation and password reset links point at localhost.
- **Authentication, SMTP**: add your own SMTP provider. The built-in mailer stops after a few emails with "email rate limit exceeded".
- **Email confirmation**: on by default for hosted projects. Leave it on for real use.
- **Google, Apple, Facebook buttons**: the login page shows them, but they only work after you enable each provider under Authentication, Providers.

## 7. Point the site at the hosted project

Edit `frontend/supabase-public.example.js`:

```js
window.SUPABASE_PUBLIC_CONFIG = {
  SUPABASE_URL: "https://<your-project-ref>.supabase.co",
  SUPABASE_ANON_KEY: "<your anon or publishable key>",
};
```

The anon key is meant to be public; row level security protects the data. Never put the service role key, a database password, or any email, Slack or AI key in this file.

## 8. Publish the site (Netlify)

`netlify.toml` publishes `frontend/` as it is. There is no build step, so Netlify environment variables are **not** put into the pages. The settings must be in the file from step 7.

## 9. Staff and demo logins

- **Demo customers**: the demo orders belong to emails like `customer.one@example.com`, but no login exists for them. Create the user in Authentication, Users with "Auto confirm" ticked. The sign-up trigger then links the matching orders.
- **Staff**: create a login for the staff member, then set `user_id` on their row in `employees`. Staff can see everything and change return status; customers never can.

## Gotchas

- `supabase db push` also loads the demo data, because it is a migration. Delete the `DEMO-*` rows afterwards if you go live with real orders.
- The pages load `supabase-public.example.js` directly. Editing a copy called `supabase-public.js` changes nothing.
- Business hours are Monday to Friday, 08:00 to 17:00 South Africa time. Holidays are only loaded for 2026 and 2027; add later years to `public_holidays` before they start.
- The 14-day return window is set in one place: `return_window_days()`.
- Failed alert or email sends are retried every minute, up to 5 times. Check for rows with `forward_attempts >= 5 and forwarded_at is null` (contact requests) or a filled `last_error` (return notifications).
- Gmail allows about 500 emails a day. Move to a sending service with your own domain before that matters.
- `frontend/reply_engine.py` is an optional standalone helper (port 8080). The pages work without it.
- There is no AI model in this version. See [ai-prompts.md](ai-prompts.md) to add one safely.

## Tests

```bash
node --test tests/notify_messages.test.mjs
psql "postgresql://postgres:postgres@127.0.0.1:54322/postgres" -v ON_ERROR_STOP=1 -f tests/place_order.sql
psql "postgresql://postgres:postgres@127.0.0.1:54322/postgres" -v ON_ERROR_STOP=1 -f tests/food_not_returnable.sql
psql "postgresql://postgres:postgres@127.0.0.1:54322/postgres" -v ON_ERROR_STOP=1 -f tests/return_notifications.sql
```

The SQL tests need the local database from step 1.
