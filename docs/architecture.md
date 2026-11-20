# Stellog technical architecture

Stellog is a multi-tenant web and API analytics service for Stellar projects. It is a fork of Umami 3.5.0. PostgreSQL is the store. Stellog does not hold funds and does not hold a tenant's Stellar secret key.

The public demo runs at `https://stellog-eta.vercel.app`. Self-hosting steps are in [hosting.md](hosting.md).

## 1. System overview

```text
tenant site  -->  /s/1.js  -->  POST /s/e  -->  PostgreSQL  -->  tenant dashboard
tenant API   -->  OTLP     -->  POST /v1/* -->  PostgreSQL  -->  operator ranking
Stellar app  -->  event    -->  POST /api/send --> ledger row --> Stellar Expert
```

| Component | Role | Status |
| --- | --- | --- |
| Tracker `public/s/1.js` | Pageviews, referrers, and a per-tab session. No cookies. | Live. sha384 is pinned in `src/stellog/tracker-pin.json`. |
| Web collector `POST /s/e` | Origin check, bot rejection, rate limit. City and region are not stored on this path. | Live. A wrong Origin returns 403. A known bot is accepted and stored as nothing. |
| OTel collector `POST /v1/metrics`, `/v1/traces`, `/v1/logs` | JSON OTLP in. Counters and latency histograms are kept. Log bodies are not stored. | Live for JSON. Protobuf traces were accepted in the 15 Nov 2026 smoke. Gzip was not exercised. |
| PostgreSQL | Web events and API rollups in one database. | Live. |
| Dashboards | A tenant sees only their projects. An SCF operator ranks every project. | Live. |
| Anomaly detection | Not in this repository. | Not built. |

## 2. Stellar integration

Sign-in uses a SEP-10 style challenge on the public Stellar network (`WebAuth.buildChallengeTx` in `src/lib/starlog-accounts.ts`). The challenge lasts five minutes. Freighter and xBull are the wallets offered by `GET /api/auth/wallet/config`. The signed address must already be linked to an email account under Settings → Wallets. An unknown address is rejected. Wallet-only signup is not built. The server key is `STELLAR_AUTH_SECRET`. When that variable is absent in production, wallet sign-in is off.

The ledger is a set of named events, not a chain watcher. A project reports `wallet_created`, `trustline_added`, `tx_sent`, and `tx_received`. Stellog stores the event name, network (`mainnet` or `testnet`), asset, success flag, and transaction hash. The hash links to Stellar Expert. The collector does not store the account address, the secret, the memo, the amount, or the phone number. Amount display, when present, is a separate field the sender chose to include.

API route names are normalized before they are stored (`src/stellog/normalize-route.ts`). A Stellar G-address becomes `{account}`, a C-address `{contract}`, an M-address `{muxed}`, a UUID `{uuid}`, a hex hash `{hash}`, and a numeric id `{id}`. Query strings are dropped. The smoke on 15 Nov 2026 stored `/accounts/{account}` and `/tx/{hash}` from those inputs.

Obba on mainnet is the reference reporter. After a confirmed USDC send, trustline, or new account, Obba posts the event to `POST /api/send` on the Stellog host for the Obba website. Stellog does not submit the Stellar transaction.

Automatic capture of wallet-kit events is out of scope. If the project does not send the event, the ledger stays empty.

## 3. Data flow and privacy

Web collection uses a short-lived cache token, checks the Origin, rejects bots, and rate-limits by IP. The Stellog collect path truncates the IP before lookup and does not write city or region. The session salt rotates daily. The tracker sets no cookie.

API tenants send OpenTelemetry to `/v1/metrics`, `/v1/traces`, and `/v1/logs` with a team ingest key (`stk_`). The key is stored as a hash. A payload that names another tenant's website is rejected. Trace and log bodies are not kept. The retained API data is request counts, error counts, and latency histograms, rolled into hour and day tables.

Operator reads of the ranking go through an application check. An operator key is a bearer token, stored as a hash, revocable, and limited to the ranking route. It cannot call the tenant website APIs.

## 4. Multi-tenancy and isolation

A project is an Umami team. A website or API property is an Umami website on that team. Queries for a tenant are scoped to the teams and websites that user belongs to. The operator role can list every project that is not deleted.

Row-level security is not enabled. `docs/rls-decision.md` records why: the app uses one pooled database role, and a session variable on that pool would leak across requests. Isolation is the application query and the ingest-key check. Two-tenant tests cover a foreign website id and a foreign ingest key. Migration 31 creates a `stellog_operator` Postgres role as `NOLOGIN` with `SELECT` on the API rollup tables. The application does not connect as that role.

## 5. Supply chain

`public/s/1.js` is built from this repository. The snippet sets `integrity` to the sha384 in `src/stellog/tracker-pin.json` and `crossorigin="anonymous"`. The script does not fetch further code at runtime. A tenant can recompute the hash:

```sh
openssl dgst -sha384 -binary public/s/1.js | openssl base64 -A
```

`pnpm-lock.yaml` locks the install. That is not a third-party reproducible-build audit.

## 6. Deployment and operations

The shared service is the Next.js app on Vercel, with Postgres reached through `DATABASE_URL`. `GET /api/heartbeat` returns `{ "ok": true }` when the process is up and does not check Postgres. `GET /api/health` runs `SELECT 1` and returns 503 when the database does not answer.

A self-hosted install is the same Node process and Postgres. [hosting.md](hosting.md) covers a Linux VPS and Fly.io. [runbooks.md](runbooks.md) covers backup (`scripts/backup-db.sh`), restore, and the ops check. The prune timer in `infra/systemd/` is in the tree and is not installed by the Vercel deploy.

## 7. Threats this repository already addresses

| Threat | What the code does |
| --- | --- |
| Forged pageviews | Origin check, bot rejection, rate limit, daily salt. |
| Metric inflation or cross-tenant ingest | Ingest keys are hashed and bound to one team. A foreign website id is rejected. |
| Stolen operator key | The key is a hash. Revoking it makes the ranking route return 401. |
| Cross-tenant reads | Tenant APIs require a session on that team. A database superuser can still read every row, because row-level security is off. |
| Secret keys on Stellar | Stellog never receives a tenant secret. Wallet sign-in verifies a signature and stores the public address only. |
