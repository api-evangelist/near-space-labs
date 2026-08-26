---
generated: '2026-08-26'
method: generated
name: Authenticate with Near Space Labs
description: Exchange NSL client credentials for a 60-minute OAuth bearer token, or mint a one-year static API key, and use them correctly.
api: openapi/near-space-labs-oauth-service.json
operations: ['POST /oauth/token', 'POST /oauth/static_key']
source: >-
  Grounded in openapi/near-space-labs-oauth-service.json and
  https://docs.nearspacelabs.com/authentication. The provider's contracts declare no operationIds,
  so operations are identified by their verbatim method + path.
---

# Authenticate with Near Space Labs

Every Near Space Labs endpoint is authenticated. There is no anonymous access and no free tier.

## Prerequisites
- An `NSL_ID` (client id) and `NSL_SECRET` (client secret), issued by Near Space Labs sales onboarding at <https://www.nearspacelabs.com/contact>. There is no self-service signup.

## Steps — short-lived token (default)
1. **Request a token** — `POST https://api.nearspacelabs.net/oauth/token` with `Content-Type: application/json` and the body:
   ```json
   {"client_id":"<NSL_ID>","client_secret":"<NSL_SECRET>","audience":"https://api.nearspacelabs.com","grant_type":"client_credentials"}
   ```
   `audience` is a constant. It is an OAuth audience identifier only — `api.nearspacelabs.com` does not resolve in DNS and must never be used as a request host.
2. **Read the response** — `{"access_token":"eyJ...","expires_in":3600,"token_type":"Bearer"}`.
3. **Use it** — send `Authorization: Bearer <access_token>` on every downstream request.
4. **Refresh before expiry** — there is no refresh token. Re-POST `/oauth/token` when `current_time > token_issued_at + expires_in - 300`, i.e. at about 55 minutes, keeping a 5-minute buffer.

## Steps — one-year static key (only when refresh is impossible)
1. `POST https://api.nearspacelabs.net/oauth/static_key` with `{"client_id":"<NSL_ID>","client_secret":"<NSL_SECRET>"}`.
2. The response is `{"api_key":"eyJ...","expires_in":31536000,"token_type":"Bearer"}`.
3. Append it to any request as `?api_key=<api_key>`. No `Authorization` header is needed.

## When NOT to mint a static key
Use it only for embedding tiles in a web map or a shared demo — a client with no token-refresh logic. It is a bearer credential valid for a full year that travels in the URL, so it lands in server logs, proxy logs and browser history. **Near Space Labs publishes no revocation endpoint and does not state whether issuing a new key invalidates the old one.** Treat minting one as effectively irreversible for a year and never expose one to an end user or an untrusted agent. See `conventions/near-space-labs-conventions.yml` → `reversibility`.

## Errors
- `401` — token missing, expired, or invalid; on `/oauth/token` also `{"error":"incorrect NSL_ID or NSL_SECRET"}`.
- `403` — valid credential, insufficient permissions. Permissions are attached at issuance; there is no scope surface to request more, so this needs a conversation with Near Space Labs.

See `errors/near-space-labs-problem-types.yml` and `authentication/near-space-labs-authentication.yml`.
