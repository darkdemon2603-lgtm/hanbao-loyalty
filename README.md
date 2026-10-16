# HANBAO Loyalty Backend — GitHub-ready starter

A self-hostable loyalty backend for HANBAO, built around Tevalis sales exports and SevenRooms guest data. Includes FastAPI, PostgreSQL/SQLite support, staff login, points ledger, rewards/redemptions, CSV imports, idempotency checks, a browser staff console, and optional Google Wallet save-link generation.

**Read this first:** this is a functional MVP/staging starter, not a production-certified loyalty system. Google Wallet requires issuer setup and real testing. Apple Wallet signing and live pass updates are not implemented yet. Staff camera scanning is also not included in this first version; staff can enter the card token manually. These gaps are clearly separated so the system does not pretend wallet features are live when they are not.

## Recommended data flow

1. Export Tevalis sales CSV; use its unique receipt/transaction ID, timestamp and eligible spend.
2. Import SevenRooms guest CSV with stable guest ID, email and/or phone.
3. Match sales to a customer only when a stable guest ID, email or phone matches.
4. Award points only to linked loyalty members, based on the configured points-per-£ rule.
5. Staff use the web console to scan the QR code on a customer's web card, look up the member and redeem rewards.
6. Customer's web card shows their current points. Add Apple/Google Wallet after the official issuer/signing setup is completed.

## Local setup

Requires Python 3.11+.

```bash
git clone https://github.com/YOUR-USERNAME/hanbao-loyalty.git
cd hanbao-loyalty
python -m venv .venv
source .venv/bin/activate       # Windows: .venv\\Scripts\\activate
pip install -r requirements.txt
cp .env.example .env
```

Edit `.env` and set `APP_SECRET` to a random secret (at least 32 characters), and `BOOTSTRAP_ADMIN_PASSWORD` to a unique password of at least 12 characters.

```bash
uvicorn backend.main:app --reload
```

Open `http://127.0.0.1:8000` for the staff console and `http://127.0.0.1:8000/docs` for API docs. First startup creates the bootstrap admin user. Do not commit `.env`.

## Configure a reward

1. Sign in via `POST /auth/token` in `/docs`.
2. Click Authorize and paste the token as `Bearer YOUR_TOKEN`.
3. Use `POST /admin/rewards`, for example:

```json
{"name":"Free side","description":"One eligible side","points_cost":250}
```

Set `POINTS_PER_GBP=1` for one point per £1. Confirm the exact earning rule before launch: VAT, tips, discounts, refunds, delivery aggregators and staff meals must be explicitly included/excluded.

## Tevalis CSV import

Use `POST /admin/import/tevalis` in `/docs`:
- `file`: CSV export
- `mapping_json`: maps canonical field names to exact CSV column headers
- `auto_award`: leave `false` for first import; use `true` only after checking mappings and test data

Required canonical fields: `transaction_id`, `transaction_at`, `total`.
Optional customer-matching fields: `sevenrooms_guest_id`, `email`, `phone`, `customer_name`.
Optional `items` field can hold item summaries.

Example mapping — replace each value with the exact column name in your export:

```json
{
  "transaction_id":"Receipt Number",
  "transaction_at":"Closed At",
  "total":"Net Total",
  "email":"Guest Email",
  "phone":"Guest Phone",
  "customer_name":"Guest Name",
  "sevenrooms_guest_id":"Guest ID",
  "items":"Items"
}
```

If Tevalis exports one row per item rather than one row per receipt, aggregate line items to a single receipt total before import; otherwise spend could be counted more than once. Run a small test export first. CSV imports are deduplicated by `(source_system, source_transaction_id)`, but production use still needs refunds/reversals and reconciliation.

## SevenRooms guest import

Use `POST /admin/import/sevenrooms-guests` with `file` and `mapping_json`. The importer maps stable SevenRooms guest ID, email, phone and name. Matching precedence is guest ID, then email, then phone. Review duplicate profiles and identity merges manually before any large import.

## Staff workflow

- Staff signs in.
- Staff scans the QR code on the customer card using the browser camera (HTTPS required), or enters the token manually.
- Staff enters eligible spend and a unique POS transaction reference.
- Backend awards points and records the transaction reference so a repeated submission does not award twice.
- Staff selects a reward and confirms redemption.

The web card is a fallback, not an Apple Wallet pass. For real restaurant launch, add camera QR scanning over HTTPS, secure staff sessions, concurrent-redemption protection and an audit review. A customer's card token must never authorize staff actions on its own.

## Free-tier-first deployment

### Database
Create a private Supabase project and put its PostgreSQL connection string in `DATABASE_URL`. The free plan is useful for development, but quotas, project pausing and backup limits can change; verify current terms and establish tested backups before live use. See https://supabase.com/docs/guides/platform/billing-on-supabase

### API hosting
Deploy the FastAPI service to a host supporting HTTPS Python web services and environment variables. Free hosting tiers change, often sleep when idle, and may not guarantee production uptime. GitHub is for code/versioning, not for running this API. Do not use GitHub Pages as the backend.

### Required secrets
`APP_SECRET`, `BOOTSTRAP_ADMIN_USERNAME`, `BOOTSTRAP_ADMIN_PASSWORD`, `DATABASE_URL`, `PUBLIC_BASE_URL`; wallet secrets only after the respective platform is configured.

## Wallet integrations

### Google Wallet
See `wallet/google/README.md`. Google Wallet requires an issuer account and a configured class; new issuer accounts begin in demo mode until publishing access is approved. The save URL generator is starter code and must be tested against your actual class/object schema. Balance updates need a server-side update after every points transaction.

Official guides:
- https://developers.google.com/wallet/retail/loyalty-cards
- https://developers.google.com/wallet/retail/loyalty-cards/getting-started/issuer-onboarding

### Apple Wallet
See `wallet/apple/README.md`. A valid Apple pass needs signing certificates and a real PassKit implementation; a ZIP renamed to `.pkpass` will not work. Updating balances on an existing pass requires Apple's PassKit web service over HTTPS.

Official guide:
- https://developer.apple.com/documentation/walletpasses/distributing-and-updating-a-pass

## GitHub setup

```bash
git init
git add .
git commit -m "Initial HANBAO loyalty backend"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/hanbao-loyalty.git
git push -u origin main
```

Never commit `.env`, customer CSVs, database dumps, Apple certificates or Google service-account JSON. Keep the repo private while developing.

## Endpoints

- `GET /health`
- `POST /auth/token`
- `POST /customers/register`
- `GET /card/{card_token}`
- `GET /wallet/google/{card_token}`
- `GET /wallet/apple/status`
- `GET /rewards`
- `GET /staff/customer/{card_token}`
- `POST /staff/award`
- `POST /staff/redeem`
- `GET /staff/ledger/{card_token}`
- `POST /admin/staff`
- `POST /admin/rewards`
- `POST /admin/customers/{customer_id}/adjust`
- `POST /admin/import/tevalis`
- `POST /admin/import/sevenrooms-guests`
- `GET /admin/import/fields`
- `GET /admin/summary`

## Production launch checklist

- [ ] Confirm actual Tevalis CSV headings and transaction granularity.
- [ ] Confirm SevenRooms guest ID is included and stable.
- [ ] Test identity matching on a small sample; manually review merges.
- [ ] Confirm point rate and spend exclusions with the business.
- [ ] Add refunds/voids/reversals and daily reconciliation.
- [ ] Add migrations (e.g. Alembic), backup/restore tests, monitoring and rate limits.
- [ ] Add camera QR scanning and test on the actual staff device.
- [ ] Implement and test signed Apple Wallet passes and update service.
- [ ] Test Google Wallet object creation, updates and publishing access.
- [ ] Add email verification, data access/deletion flows, privacy notice, consent and retention controls for UK GDPR.
- [ ] Use database transactions/locking to prevent simultaneous redemptions.
- [ ] Security review and pilot before issuing cards to all customers.
