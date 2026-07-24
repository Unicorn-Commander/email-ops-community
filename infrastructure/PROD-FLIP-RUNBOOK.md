# Email-Ops PROD flip runbook — email-ops.unicorncommander.ai

State as of 2026-07-09: **provisioned, one realm + one command from live.**

## Already done (safe prep)
- Garage prod bucket `email-ops-prod-archives` + scoped key `email-ops-prod-key`
  (GKd562ccd936c5c41c2c04f02d, RWO) — created.
- `infrastructure/.env.bigboy.prod` written (mode 600) — fresh secrets, shared
  James/LiteLLM/Postmark values carried from `.env.bigboy`. Two TODOs remain:
  `KEYCLOAK_REALM`, `KEYCLOAK_CLIENT_SECRET`.
- `infrastructure/docker-compose.bigboy.prod.yml` — de-collided names
  (`email-ops-prod-*`), own volume + internal net, Traefik labels for
  `email-ops.unicorncommander.ai`. Ready.

## Remaining steps (need Aaron's 2 decisions first: realm name + go/no-go)

### 1. Create the dedicated customer Keycloak realm + client (commander node)
On `commander`, in `uchub-keycloak`:
```
kcadm config credentials --server http://localhost:8080 --realm master \
  --user $KEYCLOAK_ADMIN --password $KEYCLOAK_ADMIN_PASSWORD
kcadm create realms -s realm=<CUSTOMER_REALM> -s enabled=true \
  -s ssoSessionMaxLifespan=172800 -s offlineSessionIdleTimeout=31536000 \
  -s offlineSessionMaxLifespanEnabled=false -s revokeRefreshToken=false
# client email-ops (confidential, standard flow + offline_access optional scope)
kcadm create clients -r <CUSTOMER_REALM> -s clientId=email-ops -s enabled=true \
  -s protocol=openid-connect -s publicClient=false -s standardFlowEnabled=true \
  -s 'redirectUris=["https://email-ops.unicorncommander.ai/*"]' \
  -s 'webOrigins=["https://email-ops.unicorncommander.ai"]' \
  -s 'defaultClientScopes=["openid","profile","email","roles","web-origins"]' \
  -s 'optionalClientScopes=["offline_access"]'
# read the generated secret -> KEYCLOAK_CLIENT_SECRET
kcadm get clients -r <CUSTOMER_REALM> -q clientId=email-ops --fields id,secret
```
Put realm + secret into `.env.bigboy.prod` (`KEYCLOAK_REALM`, `KEYCLOAK_CLIENT_SECRET`).

### 2. DNS (bigboy)
Cloudflare zone unicorncommander.ai (af7c768d018220f41ee3af797aa3b0ce), token in
[[reference_cf_dns_api_tokens]]. Proxied A record:
`email-ops` -> 69.153.31.100 (proxied). Traefik letsencrypt issues the cert.

### 3. Deploy prod stack (bigboy /home/muut/email-ops)
```
docker compose -p email-ops-prod \
  -f infrastructure/docker-compose.bigboy.prod.yml \
  --env-file infrastructure/.env.bigboy.prod up -d --build
```
Backend runs `prisma migrate deploy` on boot.

### 4. Rotate prod app role pw (RLS runtime role) inside prod PG
The init ships `email_ops_app` with a placeholder; set it to
`EMAIL_OPS_APP_PASSWORD` from the env, same as the internal flip:
```
docker exec email-ops-prod-postgres psql -U <admin> -d email_ops \
  -c "ALTER ROLE email_ops_app WITH PASSWORD '<EMAIL_OPS_APP_PASSWORD>';"
```
Confirm backend `DATABASE_URL` uses `email_ops_app`, `ADMIN_DATABASE_URL` uses admin.

### 5. Verify
- `curl -sk https://email-ops.unicorncommander.ai/api/v1/health` -> 200
- Frontend loads, SSO redirects to <CUSTOMER_REALM>.
- Run `backend/prisma/rls-acceptance.sql` as `email_ops_app` against prod (all
  tables fenced — add mail_contacts/mail_signatures/mail_vacation_settings first).

### 6. Grant the first customer (out of band — gated launch)
- Create the user in <CUSTOMER_REALM> (or broker their IdP).
- Set `EMAIL_OPS_ACCESS_LIST="customer@domain:OWNER"` in env + recreate backend,
  OR grant via the entitlement path (Lago SKU `email-ops`).

## Follow-ups (not blocking first customer on James-native webmail)
- Per-customer Postmark signed sender (currently carried `hq@unicorncommander.ai`).
- Fork per-tenant James service creds (currently shares internal JMAP admin).
- Register prod redirect URIs on Google/Microsoft OAuth apps before external
  Gmail/M365 connect works in prod (GOOGLE_/MICROSOFT_ client envs currently empty).
- Move prod secrets to Vaultwarden; add prod PG volume + prod Garage bucket to backups.
