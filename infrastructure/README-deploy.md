# Email-Ops — bigboy deploy runbook (internal: `email-ops.magicunicorn.dev`)

Deploys the integrated stack — frontend (Next.js cockpit) + backend (NestJS) +
cleaner-engine (FastAPI) + Postgres — from the `feat/email-ops-integration`
branch. Only the frontend is exposed; it proxies `/api/v1/*` to the backend.

## 0. Prerequisites (do these first)
- **Keycloak `email-ops` client** in the `uchub` realm (confidential, audience
  `email-ops`, redirect for `email-ops.magicunicorn.dev`). Grab its client
  secret. Add the Google IdP `gmail.modify` scope if Gmail cleanup is wanted.
- **Garage bucket** `email-ops-archives` + an access/secret key pair scoped to it
  (this is where archive-and-purge backups live). Note the S3 endpoint.
- **DNS**: `email-ops.magicunicorn.dev` → bigboy (Cloudflare; the suite uses
  DNS-01 / Traefik LE).
- Secrets land in Vaultwarden as you generate them.

## 1. Get the code on bigboy
```bash
# in the Email-Ops checkout on bigboy
git fetch origin && git checkout feat/email-ops-integration && git pull
```

## 2. Fill the env
```bash
cp infrastructure/.env.bigboy.example infrastructure/.env.bigboy
# generate the secrets:
openssl rand -hex 24      # POSTGRES_PASSWORD
openssl rand -base64 64   # JWT_SECRET
openssl rand -base64 32   # CONNECTED_ACCOUNT_ENC_KEY  (decodes to 32 bytes)
openssl rand -hex 32      # CLEANER_ENGINE_TOKEN == ENGINE_SHARED_SECRET (same value!)
# then fill KEYCLOAK_CLIENT_SECRET + the GARAGE_S3_* block.
```
`CLEANER_ENGINE_URL` and `ADMIN_DATABASE_URL` are built by the compose file.
Leave `DATABASE_URL` unset until the RLS flip; while unset, compose uses the
admin/owner runtime URL.

## 3. Bring it up
```bash
docker compose -f infrastructure/docker-compose.bigboy.yml \
  --env-file infrastructure/.env.bigboy up -d --build
```
The backend container runs `prisma migrate deploy` on start, then boots. Postgres
init (`infrastructure/database/init/01-init.sql`) pre-creates the suite role
split on first run.

## 4. Verify
```bash
docker compose -f infrastructure/docker-compose.bigboy.yml ps   # all healthy
curl -fsS http://localhost:8085/                                # frontend
docker exec email-ops-backend curl -fsS http://localhost:3231/api/v1/health
docker exec email-ops-engine python -c "import urllib.request;print(urllib.request.urlopen('http://localhost:8000/health').read())"
```
Then point Cloudflare/Traefik at `:8085` and load `https://email-ops.magicunicorn.dev`.

## 5. Live-Garage smoke test (DO THIS before trusting archive-and-purge)
The archive path is unit-tested with mocks only. Validate it against the real
Garage instance + a real mailbox **on a throwaway/test message**:
1. Sign in, link a provider (Microsoft works today; Gmail needs the IdP scope).
2. Open **Cleanup**, build a plan on a narrow query that matches 1–2 disposable
   messages.
3. Run **Archive & purge** → confirm: a `.eml.zip` object appears in
   `email-ops-archives`, the **Download archive** link works, and the messages
   are gone from the mailbox.
4. Hit **Restore** (vault/while-live) → confirm the messages reappear.
- **If upload fails with a checksum/`x-amz-*` error**, that's the known
  Garage-vs-new-AWS-SDK gotcha (engine `boto3 1.43`, backend `@aws-sdk 3.10x`).
  Mitigate: set `AWS_REQUEST_CHECKSUM_CALCULATION=when_required` on the engine,
  or pin older SDKs, or use a Garage build that accepts the newer checksums.

## 6. RLS flip
Runtime tenancy enforcement is controlled by the runtime DB role plus
`EMAIL_OPS_TENANCY_ENABLED`.

Before flipping, run the acceptance proof as the NOBYPASSRLS app role:
```bash
cd backend
RLS_DATABASE_URL='postgresql://email_ops_app:APP_PASSWORD@HOST:PORT/emailops?schema=public' \
  npm run rls:acceptance
```

Flip:
```bash
# infrastructure/.env.bigboy
DATABASE_URL=postgresql://email_ops_app:APP_PASSWORD@email-ops-postgres:5432/emailops?schema=public
EMAIL_OPS_TENANCY_ENABLED=true
```

`ADMIN_DATABASE_URL` stays the admin/owner role and is supplied by compose. It is
used only for Prisma migrations and the three documented BYPASSRLS system
resolvers: Postmark provider-message resolve, Twilio inbound-line resolve, and
archive-retention scanning.

Rollback:
```bash
# infrastructure/.env.bigboy
# remove/comment DATABASE_URL so compose falls back to email_ops_admin
EMAIL_OPS_TENANCY_ENABLED=false
```
Then redeploy the backend.

## 7. Public tier split (`email-ops.unicorncommander.ai`)
The customer stack uses:
```bash
docker compose -f infrastructure/docker-compose.bigboy.prod.yml \
  --env-file infrastructure/.env.bigboy.prod up -d --build
```

Use `infrastructure/.env.bigboy.prod.example` as the template. Prod must use the
dedicated customer Keycloak realm, `UC_ENTITLEMENT_MODE=enforce`,
`EMAIL_OPS_TENANCY_ENABLED=true`, the `email_ops_app` runtime URL, and the
dedicated Garage bucket `email-ops-prod-archives`. Signup, billing, metering, and
quota UI remain off; access is invite/entitlement granted out of band.

Confirm Garage honors `ServerSideEncryption: AES256` (archives are encrypted at
rest) and set a bucket lifecycle as a backstop to the 7-day app sweep.
