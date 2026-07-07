---
name: atproto-oauth
description: "Use this skill when implementing AT Protocol OAuth 2.1 authentication — PAR, DPoP-bound tokens, PKCE, client metadata documents, scope/permission-set design (including the rpc: aud audience field), identity verification after token exchange, session/refresh-token storage, or debugging OAuth errors like invalid_dpop_proof, invalid_grant, invalid_scope, or use_dpop_nonce. TypeScript-focused, using @atproto/oauth-client-node and @atproto/oauth-client-browser."
---

# AT Protocol OAuth

Guidance for implementing the AT Protocol OAuth 2.1 profile: PAR, DPoP-bound tokens, PKCE, client metadata documents, scopes/permission sets, identity verification, and session/security hardening in TypeScript.

## Protocol Essentials

AT Proto OAuth is OAuth 2.1 with **no opt-outs** on the following:

- **PKCE S256 is mandatory.** `code_challenge_method=S256` only — never `plain`. Verifier is 43–128 random chars from `[A-Z a-z 0-9 - . _ ~]`.
- **PAR (Pushed Authorization Request) is mandatory.** The browser never sees the real authorization parameters — they're POSTed server-to-server to `pushed_authorization_request_endpoint` first, returning a `request_uri` that's the only thing placed on the authorize redirect.
- **DPoP-bound tokens are mandatory.** Every request to the authorization server (AS) and resource server (PDS) carries a signed DPoP proof JWT (RFC 9449). `dpop_bound_access_tokens: true` is required in client metadata. There is no bearer-only mode.
- **`client_id` is a URL, not an opaque string.** It resolves to a JSON client metadata document that the AS fetches at the start of every flow — this is AT Proto's dynamic client registration mechanism, replacing static pre-registration. The exact same `client_id` string must appear byte-for-byte across metadata publication, PAR, authorize, and token/refresh calls — any drift invalidates the grant.
- **No `client_secret`, ever.** Confidential clients (anything with a server-side component) authenticate to the token endpoint with a `private_key_jwt` client assertion (JWT signed ES256, referencing a key published in the client's `jwks`/`jwks_uri`). Public clients (pure browser SPA, native apps with no backend) authenticate with DPoP proof possession alone — `token_endpoint_auth_method: none`.

Session lifetime differs by client type: confidential clients get refresh tokens valid up to 180 days (session itself unlimited, rotates keys periodically); public clients are capped at 14 days total, silently, until day 15 when refresh suddenly starts failing with `invalid_grant`.

## The End-to-End Flow

```
1. Resolve identity        handle/DID → DID document → PDS URL (via #atproto_pds service)
2. Discover PDS            GET {PDS}/.well-known/oauth-protected-resource
                              → authorization_servers[0]  (must be exactly one)
3. Discover AS              GET {AS}/.well-known/oauth-authorization-server
                              → issuer, authorization_endpoint, token_endpoint,
                                pushed_authorization_request_endpoint,
                                require_pushed_authorization_requests=true,
                                authorization_response_iss_parameter_supported=true,
                                client_id_metadata_document_supported=true,
                                dpop_signing_alg_values_supported ⊇ [ES256],
                                code_challenge_methods_supported ⊇ [S256]
4. PAR                      POST {par_endpoint} with client assertion (confidential) + DPoP proof
                              → 400 use_dpop_nonce on first try → retry with nonce → 201 request_uri
5. Redirect                 GET {authorize_endpoint}?client_id=...&request_uri=...  (nothing else)
6. User approves on AS
7. Callback                 GET {redirect_uri}?code=...&state=...&iss=...
                              verify state exists (delete row = single-use), verify iss == stored issuer
8. Token exchange           POST {token_endpoint}: code + code_verifier + client assertion + DPoP
                              → access_token, refresh_token, scope, sub (DID), token_type=DPoP
9. Identity verification    sub → DID doc → #atproto_pds → matches PDS from step 2 → AS matches step 3
10. Resource requests       Authorization: DPoP <access_token> + DPoP proof with ath=SHA-256(access_token)
```

In the BFF (confidential) pattern, the browser only ever talks to your backend; only the backend talks to the AS and PDS (PAR, token exchange, and all XRPC calls). That separation is what keeps DPoP private keys and access/refresh tokens off the browser entirely — the browser holds nothing but an `HttpOnly` session cookie.

## Client Metadata Document

`client_id` **is** the metadata URL — scheme MUST be `https://` (exception: `http://localhost` for local dev only, where the AS synthesizes virtual metadata from query params on the URL). The response body's own `client_id` field must exactly match the URL fetched.

Required fields:

```json
{
  "client_id": "https://example.app/oauth-client-metadata.json",
  "application_type": "web",
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "scope": "atproto repo:app.bsky.feed.post?action=create&action=delete rpc:app.bsky.feed.getTimeline?aud=did:web:api.bsky.app%23bsky_appview",
  "redirect_uris": ["https://example.app/oauth/callback"],
  "dpop_bound_access_tokens": true,
  "token_endpoint_auth_method": "private_key_jwt",
  "token_endpoint_auth_signing_alg": "ES256",
  "jwks_uri": "https://example.app/.well-known/jwks.json"
}
```

- Public clients omit `jwks`/`jwks_uri` and set `token_endpoint_auth_method: "none"`.
- The `scope` above is illustrative — request only the granular resources your app actually uses (see "Scopes and Permission Sets"). Do **not** default to `transition:generic` in new clients.
- `scope` in metadata is the **upper bound** — any authorize request's `scope` must be a subset of it, never a superset (`invalid_scope` otherwise).
- `redirect_uris` must contain the exact callback URI used; native clients use a reverse-DNS custom scheme (`com.example.app:/callback`) instead of HTTPS.
- `jwks`/`jwks_uri` publish only the **public** half of signing keys — never the `d` component. Leaking `d` means immediate key rotation + session revocation.
- Optional trust-building fields (`client_name`, `client_uri`, `logo_uri`, `tos_uri`, `policy_uri`) only render on the consent screen for AS-whitelisted "trusted" clients.

## DPoP (RFC 9449)

One DPoP keypair (P-256/ES256) per session, generated before the first PAR call and kept for the session's lifetime — losing the key ends the session; DPoP keys are never rotated mid-session.

Proof JWT shape:

```
header: { typ: "dpop+jwt", alg: "ES256", jwk: <public key JSON, no 'd'> }
claims: { jti, htm, htu, iat, nonce?, ath? }
```

- `htm` — uppercase HTTP method.
- `htu` — full target URL **with no query string and no fragment**. This normalization is caller-side unless the client library handles it — `@atproto/oauth-client-node`/`-browser` strip it automatically. Mismatched `htu` (e.g. forgetting to strip a query string) is one of the most common causes of `invalid_dpop_proof`.
- `jti` — fresh random value every proof; never reuse a proof across requests, even identical retries.
- `ath` — `base64url(SHA-256(access_token))`, required only when the request carries `Authorization: DPoP <token>` (i.e. resource requests, not PAR/token calls).
- `nonce` — server-issued, **mandatory after the first round-trip to each origin**. First request omits it; server replies `400`/`401` with body `{"error":"use_dpop_nonce"}` and header `DPoP-Nonce: <value>`; mint a new proof with that nonce and retry once. Track nonces **per origin** — the AS and the PDS have separate nonce spaces, and mixing them produces `invalid_dpop_proof`.

## Scopes and Permission Sets

`atproto` must always be the first scope requested and must be present in every token response — reject the session if it's missing. Scope grammar is richer than plain OAuth: `resource[:positional][?param=value&...]`, joined with spaces (`"atproto transition:generic repo:app.bsky.feed.post?action=create"`).

Granular resources (prefer these for any new app):

- `account:<attr>?action=read|manage` — account-level attributes (`email`, `repo`, `status`); `action` defaults to `read`, `manage` implies `read`.
- `identity:handle` (or `identity:*`) — handle-change permission.
- `repo:<nsid>?action=create&action=update&action=delete` or `repo:*` — record writes, scoped per collection, actions default to all three if omitted. No partial wildcards (`repo:app.bsky.*` is invalid — use `repo:*` or list exact NSIDs).
- `blob:<mime-pattern>` — media upload mime filters (`*/*`, `image/*`/`video/*` wildcard, or exact `type/subtype`; no `*/subtype`).
- `rpc:<lxm>?aud=<did>` — XRPC method call access; at least one of `lxm`/`aud` must be concrete (both wildcarded is forbidden). **See "XRPC audience" below — this is the scope most often gotten wrong.**
- `include:<nsid>?aud=<did>` — references an externally published permission-set lexicon (bundles multiple granular permissions under a user-facing label) that the AS dereferences, caches, and expands. A permission set can only reference resources in its own NSID group or deeper, never a sibling or parent namespace.

### Discourage transitional scopes

- `transition:generic`, `transition:chat.bsky`, `transition:email` exist **only** as a migration path off legacy App Passwords — each grants broad, coarse-grained access (`transition:generic` alone allows writing any record type, uploading any blob, and most XRPC calls except account management and `chat.bsky.*`). Requesting one is an implicit admission "I haven't scoped this app's permissions."
- **Do not default new implementations to `transition:generic`.** Treat it as a deprecated escape hatch, not a starting template — reach for it only when porting an app-password-era client that hasn't been re-scoped yet, and flag that as follow-up work.
- Build the client metadata's `scope` from the granular resources actually needed (`repo:`, `rpc:`, `blob:`, `account:`, `include:`) instead. It's more upfront design work, but it shrinks the blast radius of a stolen token and is what AS consent screens are increasingly optimized to explain to users — a client asking for "post, like, and read your profile" reads very differently from one asking for "everything".
- If a permission-set lexicon (`include:`) covering the app's use case already exists, prefer it over hand-assembling granular scopes **and** over falling back to `transition:*`.

### XRPC audience (`aud`) — why it matters more than the method name

An access token is bound to the AS/PDS pair you completed the OAuth flow with, but XRPC calls frequently target a *different* service than your own PDS — most `app.bsky.*` methods are actually served by the Bluesky AppView (a distinct service, its own DID, e.g. `did:web:api.bsky.app#bsky_appview`), not the user's PDS. The `rpc:` scope's `aud` parameter is what authorizes the token to be presented to *that specific service*, independent of which `lxm` (method NSID) is being called:

```
rpc:app.bsky.feed.searchPosts?aud=did:web:api.bsky.app%23bsky_appview   # one method, one service
rpc:*?aud=did:web:api.bsky.app%23bsky_appview                          # any method, but only against this one service
rpc:app.bsky.feed.searchPosts?aud=*                                    # this method, any service — rarely what you want
```

- **Only full wildcards are legal — no prefix wildcards.** `rpc:*` (all methods) and a concrete method NSID are valid; `rpc:app.bsky.*` is **not** a thing (partial NSID wildcards are explicitly rejected, same as `repo:app.bsky.*`). To narrow by service rather than method, keep `lxm` as `*` and pin `aud` to a concrete DID. `aud` and `lxm` may not **both** be `*`.
- `#` inside an `aud` service-DID reference **must** be percent-encoded as `%23` in the scope string — a raw `#` truncates the fragment and silently breaks the grant.
- Getting `lxm` right but `aud` wrong (or omitted) is the single most common scope bug for `app.bsky.*` integrations: the method name matches, but the token isn't authorized for the AppView's DID, so the call is rejected even though the scope "looks correct" at a glance. Always set `aud` to the actual service DID the request will hit — resolve it the same way you'd resolve any service endpoint (from the target's DID document), don't hardcode Bluesky's production AppView DID as a universal default if the app is meant to work against other AT Proto services.
- Calls to your **own** PDS (e.g. `com.atproto.repo.createRecord`) are the common case where `aud` is trivially "the PDS you authenticated against" — but the moment a lexicon is served by a third-party AppView, feed generator, or labeler, treat `aud` as a first-class, per-service value to get right, not an afterthought to the method name.

The authorize request's `scope` must be a subset of the client metadata's declared `scope`. The AS may grant fewer scopes than requested — always trust the token response's `scope` field as ground truth, not what you asked for.

## Identity Verification (do not skip this)

After token exchange, before trusting the session:

1. `sub` in the token response is a DID — resolve it to its DID document.
2. Extract the PDS service endpoint (`#atproto_pds`) from the DID document.
3. Fetch that PDS's `/.well-known/oauth-protected-resource` and confirm its `authorization_servers[0]` equals the AS `issuer` you just completed the flow with.
4. If the flow started from a handle, also verify the DID document's `alsoKnownAs` includes `at://{handle}`.

Skipping this check leaves a window where a malicious or compromised AS can mint a token for a DID whose real PDS is elsewhere — effectively a session-fixation/CSRF hole. Re-verify periodically (daily or on session renewal), since DID documents and handle bindings can change.

## Sessions and State Storage

Two distinct stores with different lifetimes:

- **Pre-flow state store** (keyed by `state`): `state`, PKCE verifier, DPoP private key, issuer, TTL ~10 minutes. Delete the row the moment token exchange begins — single-use, prevents replay.
- **Post-flow session store** (keyed by **DID, not handle**): `did`, `access_token`, `refresh_token`, `access_token_expires_at`, `dpop_private_key`, `issuer`, per-origin DPoP nonces, granted `scope`. Handles change; DIDs don't — never key sessions by handle.

Refresh tokens are single-use; every refresh call returns a new access token **and** a new refresh token, which must be persisted atomically. The classic bug: two concurrent requests both see a near-expiry token and both refresh — the second invalidates the first's new refresh token, silently killing one session copy. Mitigate with a per-DID lock (mutex, single-flight, or DB row-level lock) around every refresh — never let two refreshes for the same DID run concurrently.

Refresh-lifetime caps: **public clients 14 days**, **confidential clients 180 days per token** (with unlimited overall session lifetime via periodic key rotation).

## Security Requirements

- **Session cookies: `SameSite=Lax`, never `Strict`.** The AS→callback redirect is a cross-origin top-level navigation; `Strict` cookies are dropped on that hop, so the callback handler can't find its pre-flow state and fails with an "unknown state" error. Also set `HttpOnly` and `Secure`.
- **Encrypt tokens at rest.** Access/refresh tokens and DPoP private keys are credentials — never in a client-readable cookie, never logged (not even truncated).
- **SSRF-harden every fetch to a user-derived URL** (DID documents, `/.well-known/atproto-did`, PDS metadata, AS metadata, permission-set lexicons): block private/loopback/link-local ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`, `169.254.0.0/16`, `fc00::/7`, `fe80::/10`, `::1`), cap body size and total time, limit redirects, never allow a scheme downgrade.
- `state` and PKCE verifier must be random, ≥16/43 chars respectively, and single-use; reject duplicates.

## Troubleshooting Cheatsheet

| Symptom | Likely cause |
|---|---|
| `use_dpop_nonce` (400/401) | Expected on first request to a new origin — extract `DPoP-Nonce` header, retry once with a fresh proof carrying that nonce. Twice in a row is a bug (clock skew, wrong `htu`, or nonce copied from the wrong origin). |
| `invalid_dpop_proof` | Missing `ath` on a resource request, wrong `htm`/`htu` (often a stray query string), stale/wrong-origin nonce, clock skew, wrong `typ` (must be exactly `dpop+jwt`), or a reused proof. |
| `invalid_grant` | Authorization code already used or expired; or refresh token already used/session revoked. No retry is possible — re-authenticate. |
| `invalid_client` | Client assertion's `kid` not in the currently published `jwks`, assertion expired, `aud` mismatch, or metadata document not fetchable. |
| Callback handler can't find stored state | `SameSite=Strict` on the session cookie dropped it during the cross-origin redirect — switch to `Lax`. |
| `invalid_scope` | Requested scope isn't a subset of client metadata's declared `scope`, malformed scope syntax, or missing `atproto`. |
| Sporadic 401s despite recent refreshes | Refresh race — two concurrent refreshes for the same DID. Add a per-DID lock. |

## Library Guidance

Always prefer the official reference implementation over hand-rolling PAR/DPoP/PKCE — the protocol is unforgiving at the byte level:

- `@atproto/oauth-client-node` — confidential/BFF and native clients. Ships a built-in refresh lock (`NodeRequestLock`).
- `@atproto/oauth-client-browser` — public SPA clients, persists session state to IndexedDB.

Both handle DPoP proof generation, `htu` normalization, and nonce retry internally — never hand-roll PAR, DPoP proof minting, or PKCE when these are available.


## Guidelines

- **Default to the confidential BFF pattern** for any app with a backend — it keeps DPoP private keys and tokens off the browser entirely and is the most robust to XSS/token theft.
- **Host the client metadata document at a stable HTTPS URL** that exactly matches `client_id` byte-for-byte across every OAuth call.
- **Persist sessions by DID, never by handle.**
- **Serialize refresh calls per DID** — this is the single highest-value correctness fix in this whole protocol.
- **Verify identity after every token exchange** (`sub` → DID doc → PDS → AS match) — don't treat token exchange alone as proof of account ownership.
- **Request the narrowest scopes you need**; treat the token response's `scope` as ground truth, not what you asked for.
- **Never log tokens, DPoP private keys, or PKCE verifiers**, even redacted.
- **Test against `http://localhost` first**, then a real HTTPS `client_id` on staging, before production.
- **SameSite=Lax, HttpOnly, Secure on every session cookie.** Never Strict.

## Verifying against current docs

Endpoint signatures, scope grammar, and token formats can change. If an MCP client with AT Protocol documentation access is available, use it to check current details rather than relying solely on this skill. The AT Protocol docs MCP server is available at `https://atproto.mcp.kapa.ai` if one isn't already connected.
