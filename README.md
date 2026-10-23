# AI Soft Studio — production integration starter

This is a real full-stack Next.js application starter, not a fake-data demo. Public enquiries are inserted into Supabase. Admin users sign in through Supabase Auth and are allowlisted by `ADMIN_EMAILS`. The protected admin area reads live leads/invoices/payments and creates invoices. Customer checkout creates a Razorpay order from the invoice amount stored server-side. A signed Razorpay webhook confirms captured payments and updates invoice/payment status.

**It is not connected to your accounts or deployed for you.** Do not accept live payments until you finish the setup and complete a test transaction. This starter still needs production monitoring, rate limiting/CAPTCHA, a privacy policy, backup/restore testing and a review of your host's security settings.

## What you need
- A Node.js 20+ hosting provider that supports a persistent Next.js Node server or Docker (not static-only hosting).
- A Supabase project (database + Auth).
- Your Razorpay account with API keys enabled.
- A domain with HTTPS for production.

## 1. Create database
1. Create a project at https://supabase.com/dashboard.
2. Open SQL Editor and run `supabase/schema.sql` in full.
3. In Project Settings → API, copy Project URL, anon/public key and service-role key. The service-role key must only be stored in server environment variables.
4. In Supabase Authentication → Users, create your admin user with email/password. Turn off public sign-up if it is not needed.
5. Put that exact email in `ADMIN_EMAILS`. Multiple authorized addresses can be comma-separated. Never share admin credentials.

## 2. Configure environment variables
Set all variables from `.env.example` in your hosting provider's secret/environment-variable settings. Do not commit `.env.local` and do not paste secrets into chat. For live payments use the live `RAZORPAY_KEY_ID` and `RAZORPAY_KEY_SECRET`; for initial testing use Razorpay test keys first. `NEXT_PUBLIC_SITE_URL` must be your final HTTPS origin, without a trailing slash.

## 3. Run locally
- Install Node.js 20+.
- Copy `.env.example` to `.env.local` and fill values privately.
- Run `npm install`, then `npm run dev`.
- Public site: `http://localhost:3000`
- Admin: `http://localhost:3000/admin`
- Customer checkout: `http://localhost:3000/pay?invoice=INVOICE_UUID`
- Health check: `http://localhost:3000/api/health`

## 4. Deploy to non-Vercel Node/Docker hosting
Use a provider that supports Node 20 or Docker, such as a Node application service or a VPS with Docker. Import/upload this project. Set build command `npm run build` and start command `npm run start`, port `3000`, and configure every environment variable in `.env.example` in the provider dashboard. Or build/deploy the included Dockerfile. Do not deploy this as a static site: API routes require a Node server. Attach your domain and enable HTTPS. Exact UI steps depend on which hosting provider you use.

## 5. Configure Razorpay webhook
In Razorpay Dashboard → Webhooks, add:
`https://YOUR_DOMAIN/api/webhooks/razorpay`
Set a new webhook secret and copy that exact value into `RAZORPAY_WEBHOOK_SECRET`. Subscribe to `payment.captured` and `payment.failed`. Keep the webhook secret separate from the API key secret. Save, then redeploy/restart if environment values changed.

## 6. End-to-end verification
1. First use Razorpay test keys and test payment methods.
2. Sign in at `/admin` with the Supabase Auth user whose email is listed in `ADMIN_EMAILS`.
3. Create an invoice; share the generated payment link with the intended customer.
4. Pay using the matching customer email. The server uses the invoice's stored amount, not a browser-supplied amount.
5. Check Razorpay webhook delivery logs and the admin Payments tab. The invoice becomes `paid` only after a valid signed `payment.captured` webhook matches amount and currency.
6. Only after tests pass and your Razorpay live account is enabled, switch server variables to live keys and verify again with a small legitimate payment.

## Important production hardening
- Admin routes use Supabase Auth bearer tokens and an explicit `ADMIN_EMAILS` allowlist. Never make `SUPABASE_SERVICE_ROLE_KEY` public.
- RLS is enabled on data tables; server routes use the service role. Do not add public client-side table access without restrictive policies.
- Add rate limiting and CAPTCHA to the public lead endpoint, monitoring/alerts, database backups and tested restores, data retention policy, and privacy/terms pages before broad launch.
- Configure webhook retry monitoring. Review webhook event ledger and reconciliation if the provider returns repeated failures.
- A payment signature callback alone does not mark an invoice paid; webhook captured status is the source of truth.

## Routes
- `POST /api/leads` — validate and save public enquiry
- `GET /api/admin/overview` — protected live lead/invoice/payment records
- `POST /api/admin/invoices` — protected invoice creation
- `POST /api/payments/create-order` — create Razorpay order from DB invoice
- `POST /api/payments/verify` — verify Checkout signature (pending until webhook confirms capture)
- `POST /api/webhooks/razorpay` — validate webhook signature and reconcile payment
- `GET /api/health` — service health check

Official docs: https://razorpay.com/docs/payments/payment-gateway/web-integration/standard/ · https://razorpay.com/docs/api/orders/create/ · https://supabase.com/docs/guides/auth

## Customer-facing design update
- `/` is the animated public marketing site with responsive service cards, enquiry form, 3D-inspired CSS orb/orbits and reduced-motion support.
- `/portal` is the first client-workspace interface with navigation for projects, websites, automation, social media, WhatsApp, hosting, invoices, support and profile settings.
- The public enquiry form posts to `/api/leads`. The portal currently displays explicit empty states instead of fake customer records. Customer sign-in, per-user project ownership, support tickets and private customer invoice access must be implemented with authenticated server routes and restrictive database policies before treating `/portal` as production-ready.
