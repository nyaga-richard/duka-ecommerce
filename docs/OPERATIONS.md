# Operations runbook

## Topology

```
browser ──HTTPS──> web (Next, :3100) ──server-side proxy──> api (Express, :4100) ──> PostgreSQL
                                                              │
                                                              └──> worker (holds, ERP sync) ──> ERP HTTP API (only if enabled)
```

- Only `web` (and optionally `api`, if the web app is not used) is published.
- PostgreSQL is never published.
- The ERP is never a datastore for this service; a compromise here cannot write
  to the ERP database.
- The API is the only process that holds ERP credentials and payment keys.

## First deploy

1. Copy `docker/env.example` to `docker/.env`, set `POSTGRES_PASSWORD`,
   `ALLOWED_ORIGINS`, and `COOKIE_SECURE=true` when behind TLS.
2. `docker compose --env-file docker/.env -f docker/docker-compose.yml up -d --build`.

3. Confirm: `curl -sf http://localhost:4100/health/ready | jq`.
4. Create the first administrator (once), then unset the bootstrap variables:
   ```bash
   docker compose --env-file docker/.env -f docker/docker-compose.yml run --rm \
     -e ADMIN_BOOTSTRAP_EMAIL=owner@example.com \
     -e ADMIN_BOOTSTRAP_PASSWORD='a-long-unique-password' \
     migrate node dist/seed.mjs
   ```
   The seed only writes store settings and the administrator; it never invents
   catalogue or order data.
5. Sign in at `https://<host>/admin`.

## Separate ERP integration

This repository deploys the e-commerce services independently. Configure
`ERP_ENABLED`, `ERP_BASE_URL`, and the related `ERP_*` values in `docker/.env`
only when the ERP API is reachable from the shop's API and worker containers.
The ERP remains a separate service with its own database and deployment; the
shop uses its HTTP API for supported catalogue and inventory operations.

Use the `docker compose` commands above for this stack. Back up the shop
PostgreSQL volume separately from the ERP, and never restore a shop database
dump into the ERP database.

## Health

| Endpoint            | Meaning                                                                                                |
| ------------------- | ------------------------------------------------------------------------------------------------------ |
| `GET /health/live`  | Process is up. No dependency checks.                                                                   |
| `GET /health/ready` | Database reachable, migrations current, worker heartbeat age, payments/ERP mode. `503` when not ready. |

`checks.worker.required` is `false` outside production: a missing worker degrades
hold expiry and ERP retries but does not make the API unready. In production,
alert when `checks.worker.stale` is `true` for more than five minutes.

## Migrations

Forward-only and checksummed.

```bash
npm run db:status      # applied / pending / checksum mismatches
npm run db:migrate     # apply pending migrations
```

Rules:

- Never edit an applied migration; add a new one. The runner refuses to start on
  a checksum mismatch (protects every environment from drift).
- Migrations take a PostgreSQL advisory lock, so concurrent deploys serialise.
- Back up before migrating. There is no down-migration by design.

## Enabling the ERP integration

Follow [`ERP_CONNECTION_GUIDE.md`](ERP_CONNECTION_GUIDE.md). It is the ordered,
copy-pasteable procedure — ERP role and identity, the origin rule, the six
`ERP_*` values in `docker/.env`, the probe, the first
catalogue and stock pulls, one customer and one order document, plus a
symptom→cause table. Read [`ERP_INTEGRATION_CONTRACT.md`](ERP_INTEGRATION_CONTRACT.md)
before promising anything beyond it: those additions (G1–G9) are proposals, and
the guide only uses capabilities the ERP already implements.

In short: create a dedicated ERP identity (never an owner account, never with
MFA enrolled) holding `catalog.read`, `inventory.read`, `customers.read`,
`customers.write` and `documents.write`; set `ERP_ENABLED=true`,
`ERP_BASE_URL=https://<erp public host>`, `ERP_INTEGRATION_EMAIL`,
`ERP_INTEGRATION_PASSWORD`, `ERP_DEFAULT_BRANCH_ID` and
`ERP_DEFAULT_WAREHOUSE_ID`; recreate the `api` and `worker` containers (a
restart does not re-read the file); then
`POST /admin/erp/probe`, `POST /admin/sync/pull-catalog`,
`POST /admin/sync/pull-inventory`, and watch `GET /admin/sync/overview`.

Leave `ERP_RESERVE_ORDERS_IN_ERP=false` until the ERP implements a reservation
contract (gap G3): approving a document is a POS-shaped action with no release
path for abandoned carts. Orders are pushed as sales-order documents in `draft`,
which does **not** move ERP stock.

## Payments

- Phase 1 default: `PAYMENTS_MODE=manual`, capture disabled. Staff record
  confirmed cash/bank payments; refunds are recorded and settled outside the
  system.
- `PAYMENTS_MODE=fake` exists for tests only and is rejected in production.
- M-Pesa: set `MPESA_MODE=sandbox` and the Daraja credentials plus
  `MPESA_CALLBACK_SECRET` (32+ chars). The adapter is written but **unverified
  against live Daraja**; validate STK push, callback signature, idempotent
  replay and the refund path (the current code refuses M-Pesa refunds with
  `MPESA_REFUND_UNSUPPORTED`).
- The webhook URL is `POST /api/v1/webhooks/payments/mpesa`. It is exempt from
  CSRF/Origin and authenticated by HMAC; every event is stored before it is
  acted on, duplicates are ignored, and invalid signatures are recorded as
  `ignored` and answered with `401`.

## Routine checks

- `erp_sync_job` rows in `dead` with `last_error` — a contract gap or a data
  problem; nothing retries automatically.
- `payment_webhook` rows with `signature_valid=false` — someone is posting to
  the callback URL with the wrong secret.
- `security_event` for repeated `customer.login_failed`.
- Reservation volume vs. abandoned carts: 15-minute TTL by default
  (`CART_RESERVATION_MINUTES`).
- Backup/restore drills: PostgreSQL logical dump plus the ERP's own backups.

## Incident: ERP down

- Catalogue and inventory reads keep serving the last synced values.
- Checkout still works locally; ERP pushes queue with backoff and then `retry`.
- Do not invent stock or prices. If the queue grows, fix the ERP or stop selling
  at the edge rather than mutating data by hand.

## Incident: payment provider down

- Orders stay `placed` with `payment_status=pending`.
- Webhooks can arrive out of order; the ingest path is idempotent and re-checks
  the transaction state inside a lock, so replaying the same event is safe.
- Manual capture remains available as a documented fallback.
