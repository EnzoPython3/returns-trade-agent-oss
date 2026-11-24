# Returns Trade Agent

A returns and refunds system for an online shop. Customers sign in, see their orders, ask to return items and follow the courier. The database decides what can be returned, staff approve or reject, and everyone gets told by email and Slack. Messages sent after hours are passed to a person the next working morning.

## What is inside

| Folder | What it is |
|---|---|
| `frontend/` | Static website (HTML, CSS, plain JavaScript) using Supabase for login and data |
| `supabase/migrations/` | The whole database: tables, return rules, security rules, timers |
| `supabase/functions/` | Edge Functions for after-hours alerts, return notifications and login logging |
| `tests/` | Database and notification tests |
| `docs/` | Setup guide, database guide, AI prompts |

## Main features

- Email and password login, sign up, password reset, magic link, optional Google, Apple and Facebook
- Return rules enforced by the database: 14 days from delivery, no sale items, no food, one open return per item
- Row level security: customers see only their own data, staff see everything
- Every return status change is logged and sent to the customer, the owner and Slack
- After-hours support messages are flagged and forwarded with the next opening time (South African business hours and public holidays)
- Demo courier tracking you can swap for a real courier API

## Get started

Read [docs/setup.md](docs/setup.md). The short version for local use:

```bash
supabase start && supabase db reset
# paste the anon key from `supabase status` into frontend/supabase-public.example.js
cd frontend && python3 -m http.server 3000
```

## AI

This version makes refund decisions with fixed rules only, so it needs no AI key. The prompts from the earlier AI version are in [docs/ai-prompts.md](docs/ai-prompts.md) if you want to add a model of your own.

## Licence

MIT. See [LICENSE](LICENSE).
