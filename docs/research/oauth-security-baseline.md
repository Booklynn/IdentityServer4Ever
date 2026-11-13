# OAuth security baseline: RFC 9700, the OAuth 2.1 draft, and OpenID Connect flows

Research for issue #4. Sources read on 2026-11-13. Code read at commit `5e40bd18` (`main`), `src/` only.

## Summary

- **RFC 9700** (BCP 240, January 2025) is mostly already met by the code, because the 4.x defaults were tightened before the fork: PKCE is required by default, `plain` is off by default, refresh tokens rotate by default, access tokens in the front channel are off by default, and the password grant is inert until a host registers a validator.
- The code **violates** three RFC 9700 MUSTs outright: redirect URIs are compared case-insensitively, `http` redirect URIs are accepted for any client, and the resource owner password credentials grant is still implemented. A fourth (variable loopback port for native apps) is unmet by default and wrong in its opt-in form.
- Most of the rest is **left to configuration** with no guard: a host can switch off PKCE for a public client, allow `plain`, reuse refresh tokens for public clients, and enable every implicit and hybrid response type. Because the decision is "remove, do not disable", each of these options is a removal candidate, not a default to flip.
- **OAuth 2.1 is still an Internet-Draft**: `draft-ietf-oauth-v2-1-17`, dated 31 October 2026, an active OAuth WG document that has not entered IESG processing. It is not published.
- The **gap beyond OAuth 2.0 + RFC 9700 is small and concrete**: `iss` becomes mandatory in every authorization response, the server (not only the client) must enforce PKCE, `plain` is prohibited and `code_challenge_method` is required, `redirect_uri` disappears from the token request, `response_type=token` is undefined, browser-based apps get refresh-token lifetime rules, and JWT client authentication must use the issuer as its only audience (now RFC 10070). Today the code does not send `iss`, rejects a token request that omits `redirect_uri`, and still carries opt-outs for both PKCE rules.
- **OpenID Connect**: `id_token` and `code id_token` are compatible with both documents. The four response types that return an access token from the authorization endpoint (`token`, `id_token token`, `code token`, `code id_token token`) are what RFC 9700 discourages and OAuth 2.1 leaves undefined. OpenID Connect Core still makes `id_token token` mandatory for *Dynamic* OpenID Providers only; this code base has no dynamic registration, so nothing in Core forces it to keep them. FAPI 2.0, the OpenID Foundation's own high-security profile, allows `response_type=code` only.

## Sources and method

| Short name | Document | URL |
|---|---|---|
| RFC 9700 | Best Current Practice for OAuth 2.0 Security, BCP 240, January 2025 | https://www.rfc-editor.org/rfc/rfc9700.html |
| draft-17 | The OAuth 2.1 Authorization Framework, `draft-ietf-oauth-v2-1-17`, 31 October 2026 | https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-17 |
| Datatracker | Status page and API record for the draft | https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/ |
| RFC 10070 | Updates to OAuth 2.0 JWT Client Authentication and Assertion-Based Authorization Grants, November 2026 | https://www.rfc-editor.org/rfc/rfc10070.html |
| RFC 10017 | OAuth 2.0 for Browser-Based Applications, BCP, August 2026 | https://www.rfc-editor.org/rfc/rfc10017.html |
| RFC 8414 | OAuth 2.0 Authorization Server Metadata | https://www.rfc-editor.org/rfc/rfc8414.html |
| OIDC Core | OpenID Connect Core 1.0 incorporating errata set 2 | https://openid.net/specs/openid-connect-core-1_0.html |
| OIDC Discovery | OpenID Connect Discovery 1.0 incorporating errata set 2 | https://openid.net/specs/openid-connect-discovery-1_0.html |
| Multiple Response Types | OAuth 2.0 Multiple Response Type Encoding Practices | https://openid.net/specs/oauth-v2-multiple-response-types-1_0.html |
| FAPI 2.0 | FAPI 2.0 Security Profile, Final, 22 February 2025 | https://openid.net/specs/fapi-security-profile-2_0-final.html |
| FAPI 1.0 Advanced | Financial-grade API Security Profile 1.0 - Part 2: Advanced, Final | https://openid.net/specs/openid-financial-api-part-2-1_0.html |

The plain-text of each IETF document was downloaded and read directly; section numbers below are from those texts. Code paths are relative to the repository root.

Verdict vocabulary used in the tables:

- **Complies**: the code does this unconditionally.
- **Violates**: the code does the opposite, or lacks the required behaviour, whatever the configuration.
- **Config**: behaviour depends on a named option; the default is stated. "No guard" means nothing stops a host from choosing the non-compliant value.
- **Host**: the library has no code for this; the hosting application's login and consent pages decide.

## Part 1. RFC 9700 against the code

RFC 9700 section 2 is the normative summary; section 4 repeats and details it. Only items that bind an authorization server are listed. Items that bind clients or resource servers only are listed after the table.

### 1a. Requirements on the authorization server

| # | RFC 9700 | Level | Requirement | Current code | Verdict |
|---|---|---|---|---|---|
| 1 | §2.1, §4.1.3 | MUST | Compare redirect URIs by exact string matching (RFC 3986 §6.2.1 simple string comparison). | `StrictRedirectUriValidator` uses `StringComparer.OrdinalIgnoreCase` (`src/IdentityServer4/src/Validation/Default/StrictRedirectUriValidator.cs:30`). No wildcard or pattern matching exists. The same comparer is used for post-logout redirect URIs. | **Violates** (case-insensitive is not simple string comparison) |
| 2 | §2.1, §4.1.3 | MUST | Allow a variable port for native apps' localhost redirect URIs (RFC 8252 §7.3). | Default validator does not. Opt-in `AddAppAuthRedirectUriValidator()` (`.../BuilderExtensions/Additional.cs:56`) registers `StrictRedirectUriValidatorAppAuth`, which accepts any `http://127.0.0.1:{port}` followed by **any path**, only when the client registered the literal `http://127.0.0.1` and has `RequirePkce`. No `[::1]`. | **Config**: opt-in service, off by default; the opt-in ignores the path, so it is looser than "exact except port" |
| 3 | §2.1, §4.11.2 | MUST NOT / MUST / SHOULD | Do not expose open redirectors; do not redirect on an invalid `client_id`/`redirect_uri`; authenticate the user before redirecting (silent authentication excepted); only auto-redirect to trusted URIs. | Errors are shown on the host's error page unless the error is `access_denied`, `login_required`, `consent_required`, `interaction_required` or `account_selection_required` (`src/IdentityServer4/src/Endpoints/Results/AuthorizeResult.cs:76-93`). `invalid_scope` and similar never redirect to the client. | **Complies**. The host's handling of its own `returnUrl` was not audited |
| 4 | §2.1, §4.12 | MUST NOT / SHOULD | Do not use HTTP 307 when redirecting a request that may carry user credentials; SHOULD use 303. | Redirects use `HttpResponse.Redirect` (302) (`AuthorizeResult.cs:113`). No 307 anywhere in `src/IdentityServer4/src`. The credential POST is handled by the host's login page. | **Complies** with the MUST NOT; the SHOULD (303) is not followed |
| 5 | §2.1.1 | MUST | Support PKCE. | Supported for code and hybrid (`.../Validation/Default/AuthorizeRequestValidator.cs:481-537`). | **Complies** |
| 6 | §2.1.1 | MUST | If `code_challenge` was sent, enforce `code_verifier` at the token endpoint. | `TokenRequestValidator.cs:308-318`. | **Complies** |
| 7 | §2.1.1, §4.8.2 | MUST | Reject a token request that carries `code_verifier` when the authorization request had no `code_challenge` (PKCE downgrade). | `TokenRequestValidator.cs:320-327` returns `invalid_grant`. | **Complies** |
| 8 | §2.1.1 | MUST / RECOMMENDED | Provide a way to detect PKCE support; RECOMMENDED via `code_challenge_methods_supported`. | Published (`.../ResponseHandling/Default/DiscoveryResponseGenerator.cs:319`) as `["plain","S256"]`. | **Complies** (but advertises `plain`, see #10) |
| 9 | §2.1.1 | MUST (client) | Public clients MUST use PKCE; RECOMMENDED for confidential clients. | `Client.RequirePkce`, default `true` (`src/Storage/src/Models/Client.cs:98`). | **Config**, compliant default, no guard: `RequirePkce = false` is accepted for a client with `RequireClientSecret = false` |
| 10 | §2.1.1 | SHOULD (client) | Use a challenge method that does not expose the verifier (S256 only). | `Client.AllowPlainTextPkce`, default `false` (`Client.cs:103`). A missing `code_challenge_method` is treated as `plain` (`AuthorizeRequestValidator.cs:513-516`) and then rejected unless the option is on. | **Config**, compliant default |
| 11 | §2.1.2 | SHOULD NOT (client) | Do not use the implicit grant or any response type that returns an access token from the authorization endpoint. | Server implements `token`, `id_token token`, `code token`, `code id_token token` (`src/IdentityServer4/src/Constants.cs`, `SupportedResponseTypes`). A client needs the `implicit` or `hybrid` grant type plus `AllowAccessTokensViaBrowser`, default `false` (`Client.cs:117`; check at `AuthorizeRequestValidator.cs:471`). Discovery always lists all seven response types and the `implicit` grant type. | **Config**, compliant default |
| 12 | §2.2.1, §4.10 | SHOULD | Sender-constrain access tokens (mTLS or DPoP). | mTLS binding exists, `MutualTls.Enabled` default `false` (`.../Options/MtlsOptions.cs:15`). No DPoP code at all. | **Config**, off by default; DPoP missing |
| 13 | §2.2.2, §4.14.2 | MUST | Refresh tokens for public clients are sender-constrained or rotated, so replay is detected. | `Client.RefreshTokenUsage`, default `OneTimeOnly` (`Client.cs:204`). `ReUse` is accepted for public clients. With rotation, a consumed token is flagged and its reuse rejected (`.../Services/Default/DefaultRefreshTokenService.cs:116-123`), but the newer token in the chain is **not** revoked, which §4.14.2's description of rotation expects ("it will revoke the active refresh token"). | **Config**, compliant default, no guard; replay response is partial |
| 14 | §4.14.2 | MUST | Decide by risk assessment whether a client gets refresh tokens. | `Client.AllowOfflineAccess`, default `false` (`Client.cs:152`). | **Config**, compliant default |
| 15 | §4.14.2 | MUST | Bind refresh tokens to the consented scope and resource servers. | The refresh token stores the original access token; the refresh request has no scope processing that could widen it (`TokenRequestValidator.cs`, `ValidateRefreshTokenRequestAsync`). | **Complies** |
| 16 | §4.14.2 | SHOULD | Expire refresh tokens after inactivity. | `Client.RefreshTokenExpiration`, default `Absolute` with 30 days (`Client.cs:188`, `:219`). Inactivity expiry exists only as `Sliding` (15 days, `Client.cs:193`). `AbsoluteRefreshTokenLifetime = 0` with `Sliding` never expires (`DefaultRefreshTokenService.cs:245`). | **Config**; default has a hard limit but no inactivity expiry |
| 17 | §2.3, §4.10.2 | SHOULD | Audience-restrict access tokens. | `aud` is emitted only when a requested scope belongs to an `ApiResource`; `EmitStaticAudienceClaim` default `false` (`.../Options/IdentityServerOptions.cs:37`, `.../Services/Default/DefaultTokenService.cs:225-231`). No `resource` parameter (RFC 8707). | **Config**: depends on how the host models resources |
| 18 | §2.4 | MUST NOT | The resource owner password credentials grant is not used. | Implemented: `grant_type=password` is dispatched (`TokenRequestValidator.cs:156`, `:383`), `GrantTypes.ResourceOwnerPassword` presets exist (`src/IdentityServer4/src/Models/GrantTypes.cs`). It works only if a client lists the grant and the host registers an `IResourceOwnerPasswordValidator`; the default is `NotSupportedResourceOwnerPasswordValidator` (`.../BuilderExtensions/Core.cs:205`). | **Violates** as a capability; inert by default |
| 19 | §2.5 | SHOULD | Enforce client authentication where credentials can be issued and kept confidential. | `Client.RequireClientSecret`, default `true` (`Client.cs:50`). | **Config**, compliant default |
| 20 | §2.5 | RECOMMENDED | Asymmetric client authentication (mTLS, `private_key_jwt`). | Opt-in: `AddJwtBearerClientAuthentication()`, `AddMutualTlsSecretValidators()` (`Additional.cs:305`, `:333`). Default is shared secret via Basic and POST body (`Core.cs:245-246`). | **Config**, off by default |
| 21 | §2.6 | RECOMMENDED | Publish authorization server metadata (RFC 8414). | `/.well-known/openid-configuration`, on by default (`EndpointOptions.cs:47`). RFC 8414 §3 names that suffix as a permitted alternative. `/.well-known/oauth-authorization-server` is not served. | **Complies** in substance |
| 22 | §2.6, §4.15.1 | SHOULD NOT | Do not let clients influence `client_id` or claims that could be mistaken for a resource owner. | No dynamic registration. Client-credentials tokens carry `client_id` and no `sub`. Static client claims get the prefix `client_` (`Client.ClientClaimsPrefix`, `Client.cs:270`; `.../Services/Default/DefaultClaimsService.cs:137`). | **Complies** by default; **Config**: the prefix can be emptied |
| 23 | §2.6 | MUST NOT | Do not allow `http` redirect URIs, except loopback redirects of native clients. | No check at registration or at request time. `ValidationOptions.InvalidRedirectUriPrefixes` blocks `javascript:`, `file:`, `data:` and others but not `http:` (`.../Options/ValidationOptions.cs:17`; `.../Validation/Default/DefaultClientConfigurationValidator.cs:181`). | **Violates** |
| 24 | §2.6 | MUST NOT | No CORS support at the authorization endpoint. | Default CORS paths are discovery, JWKS, token, userinfo, revocation (`Constants.cs:233-240`). | **Complies** by default; **Config**: `Cors.CorsPaths` is settable |
| 25 | §2.6, §4.17 | MUST | Verify origins strictly if the authorization response is sent by `postMessage`. | No `web_message` response mode. The only `postMessage` is the check-session iframe, which replies to the explicit sender origin (`.../Endpoints/Results/CheckSessionResult.cs:370`). | Not applicable |
| 26 | §4.2.4 | MUST / SHOULD | Authorization codes are single use (from RFC 6749 §4.1.2); on a second redemption SHOULD revoke tokens already issued from that code. | Code is deleted on first redemption (`TokenRequestValidator.cs:250`). A replay just gets `invalid_grant`; nothing is revoked. | **Complies** with the MUST; SHOULD not implemented |
| 27 | §4.2.4 | SHOULD NOT | Authorization endpoint pages carry no third-party resources or external links; suppress `Referer`. | The library's own form-post page sets `Referrer-Policy: no-referrer` and a CSP (`AuthorizeResult.cs:128-136`). Login and consent pages belong to the host. | **Host** |
| 28 | §4.16 | MUST / SHOULD | Prevent clickjacking; SHOULD use CSP level 2 or later; SHOULD let administrators configure allowed framing origins per client. | The library renders no login or consent UI. `X-Frame-Options` appears only in the sample host (`src/IdentityServer4/host/Quickstart/SecurityHeadersAttribute.cs:24`). Library pages use `CspOptions.Level`, default `Two`. No per-client framing-origin setting. | **Host** |

### 1b. Items that bind clients or resource servers, for reference

These are not authorization server obligations, but the server decides whether a client *can* comply.

| RFC 9700 | Level | Requirement | What the server offers |
|---|---|---|---|
| §2.1, §4.7.1 | MUST | Clients prevent CSRF (PKCE, `nonce`, or `state`). | PKCE and `nonce` supported; `nonce` required for implicit and hybrid requests with `openid` scope (`AuthorizeRequestValidator.cs:648-670`). |
| §2.1, §4.4.2 | REQUIRED / SHOULD | Clients talking to several servers defend against mix-up, preferably with the `iss` response parameter (RFC 9207) or `iss` in the ID token. | No `iss` authorization response parameter. `iss` is in the ID token for OpenID Connect flows. |
| §4.3.2 | MUST NOT | Clients do not send access tokens in a URI query parameter. | The server's own protected endpoint reads the token from the header or form body only (`.../Validation/Default/BearerTokenUsageValidator.cs`). |
| §2.3, §4.9.3 | MUST | Resource servers refuse tokens not meant for them and treat tokens as secrets. | Out of scope (no resource server code in `src/`). |
| §4.13 | MUST | Reverse proxies sanitise security-relevant headers. | Deployment matter. |

### 1c. Not normative, but stated

RFC 9700 §2.1.1 "encourages" servers to detect constant PKCE challenges or nonces. The code does not.

## Part 2. OAuth 2.1 status and the remaining gap

### 2a. Status as of 2026-11-13

| Fact | Value | Source |
|---|---|---|
| Latest revision | `draft-ietf-oauth-v2-1-17` | Datatracker API record (`rev: "17"`), https://datatracker.ietf.org/api/v1/doc/document/draft-ietf-oauth-v2-1/ |
| Date | Document date 31 October 2026; posted 2026-10-31 04:16 UTC (the Datatracker page shows "Last updated 2026-10-30") | Draft header; Datatracker history |
| Draft state | Active Internet-Draft, expires 4 May 2027 | Datatracker API (`states`: Active) |
| Working group state | "WG Document" (OAuth WG); no working group last call recorded | Datatracker page |
| IESG state | "I-D Exists" (the IESG has not started processing it) | Datatracker page |
| Published as RFC | No (`rfc: null`) | Datatracker API |
| Intended status | Standards Track per the draft header; Datatracker field shows "(None)" | Draft header; Datatracker page |
| Effect when published | "Obsoletes: 6749, 6750 (if approved)" (added in -17) | Draft header and Appendix F |
| Recent cadence | -13 May 2025, -14 Oct 2025, -15 Mar 2026, -16 Sep 2026, -17 Oct 2026 | Datatracker history |

The draft says an authorization server "can support both OAuth 2.0 and OAuth 2.1 clients", with the rules chosen per client by configuration, and defines no signal for which version is in use (draft-17 §1.8).

### 2b. What draft-17 requires, compared with OAuth 2.0 + RFC 9700

"New" means not required by RFC 6749/6750 plus RFC 9700. "Stricter" means RFC 9700 says SHOULD, or binds only the client. "Same" rows are included because the code fails them or because they are easy to assume are new.

| # | draft-17 | Requirement | vs 2.0 + RFC 9700 | Current code | Gap? |
|---|---|---|---|---|---|
| G1 | §4.1.2, §4.1.2.1 | `iss` is REQUIRED in every authorization response, success and error. | **New** (RFC 9700 §2.1 only tells clients they SHOULD use it) | Not implemented | **Yes** |
| G2 | §4.1.1, §4.1.2.1, §7.5.1.1 | The server MUST reject authorization requests without `code_challenge` from public clients, and from confidential clients unless it has reasonable assurance the client uses the OpenID Connect `nonce` correctly. | **Stricter** (RFC 9700 §2.1.1 puts the duty on clients) | `RequirePkce` default `true`, can be switched off for any client | **Yes** (remove the opt-out for public clients; decide policy for confidential OpenID Connect clients) |
| G3 | §4.1.1, §7.5.2 | `plain` is "removed and explicitly prohibited"; `code_challenge_method` is REQUIRED. | **Stricter** (RFC 9700 §2.1.1: clients SHOULD use S256) | `plain` accepted behind `AllowPlainTextPkce`, advertised in discovery, and assumed when the method is omitted | **Yes** |
| G4 | §4.1.3, §10.2 | `redirect_uri` is no longer sent in the token request. A server that also serves OAuth 2.0 clients MUST still accept and enforce it for them. | **New** | Token request without `redirect_uri` is rejected (`TokenRequestValidator.cs:281-286`) | **Yes** (a 2.1-only client cannot redeem a code) |
| G5 | §10.1 | `response_type=token` is no longer defined. Other response types, "for example, `response_type=id_token`", are unaffected. | **Stricter** (RFC 9700 §2.1.2: SHOULD NOT) | See Part 1 #11 | **Yes** if implicit stays in any form |
| G6 | §10 | Resource owner password credentials grant omitted. | Same (RFC 9700 §2.4 MUST NOT) | See Part 1 #18 | Covered by RFC 9700 work |
| G7 | §4.1.1, §2.3.1 | Redirect URI comparison is "case-sensitive, character-by-character ... with no normalization". | Same, stated more explicitly | See Part 1 #1 | Covered by RFC 9700 work |
| G8 | §8.4.2, §2.3.1 | MUST allow any port for loopback IP redirect URIs; SHOULD reject private-use URI schemes without a period. | Loopback: same as RFC 9700 §4.1.3. Private-use scheme rule: **New** to the core (from RFC 8252) | Loopback: see Part 1 #2. No scheme-shape check | **Yes**, small |
| G9 | §1.5 | All OAuth protocol URLs MUST be `https`, loopback redirect URIs excepted. | Wider than RFC 9700 §2.6, which covers redirect URIs | Not enforced by the library | **Yes** for redirect URIs (Part 1 #23); server URLs are a deployment matter |
| G10 | §9.3.2 | For browser-based apps: refresh tokens MUST have a maximum lifetime or an inactivity expiry; rotation MUST NOT extend lifetime beyond the first token's; SHOULD end with the user's session. | **New** (from RFC 10017) | Rotation keeps the original creation time, so `Absolute` complies. `Sliding` with `AbsoluteRefreshTokenLifetime = 0` does not. No session link | **Yes**, small |
| G11 | §4.3.1 | On refresh token replay under rotation, revoke the active refresh token "as well as the access authorization grant associated with it". | Slightly stricter wording than RFC 9700 §4.14.2 | Reuse rejected, nothing revoked | **Yes** |
| G12 | §2, §8.1, §8.1.1, §9.3.1 | Registration MUST record the client type. Native and browser-based apps are public clients; the server MUST NOT rely on a shared secret from them. | **New** to the core (from RFC 8252 and RFC 10017) | No client-type field; `RequireClientSecret` is the only signal | **Yes** (modelling decision) |
| G13 | §2.4 | JWT client authentication MUST follow the updated audience rules of `draft-ietf-oauth-rfc7523bis`. That draft is now **RFC 10070** (November 2026): the `aud` MUST be the issuer identifier as its sole value, and the token endpoint URL MUST NOT be accepted (RFC 10070 §4). | **New**, and already in force for OAuth 2.0 because RFC 10070 updates RFC 7523 | Accepts **only** the token endpoint URL; the issuer is commented out (`.../Validation/Default/PrivateKeyJwtSecretValidator.cs:82-90`) | **Yes**: the code does the exact opposite |
| G14 | §2.4.1 | The server MUST support client credentials in the request body (`client_secret_post`); Basic is MAY. | **New** emphasis (RFC 6749 required Basic) | Both supported by default | No |
| G15 | §5.1 | Resource servers MUST ignore access tokens in a URI query parameter. | Stricter than RFC 6750 | Query is never read | No |
| G16 | §3.2, §9.3.4 | Token endpoint (and metadata, revocation, userinfo) MUST send CORS headers when browser-based apps are supported. | **New** as a MUST | Sent on token, userinfo, revocation, discovery, JWKS | No |
| G17 | §4.2 | Client credentials grant MUST only be used by confidential clients. | Same (RFC 6749 §4.4) | Not enforced: a client with `RequireClientSecret = false` and the `client_credentials` grant gets a token unauthenticated (`.../Validation/Default/ClientSecretValidator.cs:77`; no check in `TokenRequestValidator.cs:346`) | **Yes**, misconfiguration is not blocked |
| G18 | §4.1.3, §7.5.3 | Reused authorization code: MUST deny, SHOULD revoke tokens issued from it. | Same | See Part 1 #26 | SHOULD only |
| G19 | §7.3, §7.3.1 | SHOULD NOT process repeated authorization requests without user interaction unless the client's identity is assured. | Same (RFC 6749 §10.2) | `Client.RequireConsent` default `false` (`Client.cs:75`), for public clients too | SHOULD only; default is the wrong way round for public clients |
| G20 | §5.1.1 | The `Bearer` scheme name is case-insensitive. | Same (HTTP) | Case-sensitive prefix match (`BearerTokenUsageValidator.cs:69`) | Minor |
| G21 | §3.2.3 | `Cache-Control: no-store` on responses carrying tokens. | Same | Set on token and error responses | No |
| G22 | §4.1.2 | Authorization code lifetime: 10 minutes maximum RECOMMENDED. | Same | Default 300 seconds (`Client.cs:183`) | No |
| G23 | §7.7 | Guessing probability of generated tokens at most 2^-128. | Same | Handles are 32 random bytes (`.../Services/IHandleGenerationService.cs:19`) | No |

**The gap that is genuinely new relative to OAuth 2.0 + RFC 9700 is G1, G2, G3, G4, G5, G8 (scheme rule), G10, G11, G12 and G13.** G1 to G4 are protocol-visible and are the ones to keep ready; G13 is already binding through RFC 10070 regardless of OAuth 2.1.

draft-17 §10 is the draft's own non-normative list of changes; every bullet in it is covered by a row above.

## Part 3. OpenID Connect implicit and hybrid flows

### 3a. Response type by response type

Whether a response type conflicts depends on one thing: does the authorization endpoint return an access token?

| `response_type` | Flow (OIDC Core) | Access token in the front channel | RFC 9700 | OAuth 2.1 draft-17 | In this code |
|---|---|---|---|---|---|
| `code` | Authorization Code, §3.1 | No | Recommended (§2.1.2) | The only response type the draft defines (§4.1.1) | Supported |
| `id_token` | Implicit, §3.2 | No | Not discouraged: §2.1.2 targets response types "issuing access tokens in the authorization response" | Explicitly unaffected (§10.1) | Supported |
| `code id_token` | Hybrid, §3.3 | No | Named as an acceptable alternative (§2.1.2: "such as the `code id_token` response type") | Extension response type, allowed (§4.1.1); PKCE rules still apply (§7.5.1.1) | Supported; PKCE is enforced for hybrid as for code |
| `id_token token` | Implicit, §3.2 | **Yes** | SHOULD NOT (§2.1.2) | Not defined; the stated intent is "to no longer issue access tokens in the authorization response" (§10.1) | Supported behind `AllowAccessTokensViaBrowser` |
| `code token` | Hybrid, §3.3 | **Yes** | SHOULD NOT (§2.1.2) | As above | Supported behind the same option |
| `code id_token token` | Hybrid, §3.3 | **Yes** | SHOULD NOT (§2.1.2) | As above | Supported behind the same option |
| `token` | Not an OpenID Connect flow (no ID token) | **Yes** | SHOULD NOT (§2.1.2) | Removed (§10.1) | Supported behind the same option |

RFC 9700 §2.1.2 leaves a conditional exit: the SHOULD NOT applies "unless access token injection in the authorization response is prevented and the aforementioned token leakage vectors are mitigated". §4.6.1 notes that OpenID Connect's `at_hash` and `nonce` mitigate injection, and then adds that sender-constraining "is still required", which §2.1.2 says cannot be done for tokens issued from the authorization endpoint. In practice the exit does not open.

Two further points of contact:

- **`nonce` in place of PKCE.** RFC 9700 §2.1.1 lets confidential OpenID Connect clients use `nonce` instead of PKCE. draft-17 §7.5.1.1 keeps that exception but only where the server has "reasonable assurance" the client implements `nonce` properly, and still recommends PKCE. This is the one place a hybrid or code flow without PKCE survives into OAuth 2.1.
- **ID tokens in the front channel** are not an OAuth security conflict. They are a privacy concern raised by the OpenID Foundation itself (FAPI 2.0, below).

### 3b. What the OpenID Foundation's documents say

| Document | Section | Statement | Consequence here |
|---|---|---|---|
| OIDC Core | §15.1 (Mandatory to Implement Features for All OpenID Providers) | Lists RS256 signing, `prompt`, `display`, locales, `auth_time`, `max_age`, `acr_values`. No response type is listed. | A statically configured provider is not required by Core to implement implicit or hybrid. |
| OIDC Core | §15.2 (Mandatory to Implement Features for Dynamic OpenID Providers) | "These OpenID Providers MUST support the `id_token` Response Type and all that are not Self-Issued OPs MUST also support the `code` and `id_token token` Response Types." | Applies only to providers that support dynamic registration. This code has none. If dynamic registration is ever added, strict Core conformance and RFC 9700 pull in opposite directions on `id_token token`. |
| OIDC Discovery | §3 (`response_types_supported`, `grant_types_supported`) | "Dynamic OpenID Providers MUST support the `code`, `id_token`, and the `id_token token` Response Type values"; dynamic providers MUST support the `authorization_code` and `implicit` grant types. | Same scope: dynamic providers only. |
| OIDC Core | §3.2 | The implicit flow returns tokens directly to the client, "which may expose them to the End-User and applications that have access to the End-User's User Agent". | Core acknowledges the exposure and does not withdraw the flow. |
| OIDC Core | §17.4 | "When an Access Token is returned via the User Agent using the Implicit Flow or Hybrid Flow, there is a greater risk of it being exposed to an attacker"; offline access is to be ignored in that case. | Mitigation by restriction, not removal. |
| OIDC Core | §3.2.2.1, §3.2.2.10 | Implicit redirect URIs MUST NOT use `http` except for native loopback (§3.2.2.1); `nonce` is REQUIRED for the implicit flow (§3.2.2.10). | Already enforced for `nonce`; the `http` rule is not (Part 1 #23). |
| OIDC Core | §3.1.2.1 | Redirect URI matching uses RFC 3986 §6.2.1 simple string comparison. For the code flow the redirect URI "MAY use the `http` scheme, provided that the Client Type is `confidential`" and the provider allows it. | Core already agrees with RFC 9700 on exact matching. Its `http` allowance for confidential clients is **looser than RFC 9700 §2.6**; RFC 9700 states that it "introduces new requirements beyond those defined in ... OpenID Connect" (§1). |
| Multiple Response Types | §3, §5 | For `id_token`, `id_token token` and the `code`-combined types, the default response mode is fragment "and the query encoding MUST NOT be used". | The code already restricts implicit and hybrid to fragment or form post (`Constants.cs`, `AllowedResponseModesForGrantType`). |
| FAPI 2.0 Security Profile (Final, 22 Feb 2025) | §5.1.1, §5.3.2.1, §5.3.2.2 | Aims "to follow the recommendations in the OAuth Security BCP [RFC9700]". Authorization servers "shall reject requests using the resource owner password credentials grant", "shall require the value of `response_type` ... to be `code`", "shall require PKCE ... with `S256`", and "shall return an `iss` parameter in the authorization response according to [RFC9207]". | The Foundation's own current high-security profile forbids implicit and hybrid entirely and already mandates G1 and G3. |
| FAPI 2.0 Security Profile | §5.5 (table, row "response types `code id_token` or `code`" to "response type `code`") | Reason given: "no ID token in front-channel (privacy improvement); nonce/signature check can be skipped by clients, PKCE cannot (security improvement)". | The Foundation's stated rationale for dropping the hybrid flow it had required in FAPI 1.0. |
| FAPI 1.0 Advanced (Final) | §5.2.2 | Required `response_type` `code id_token`, or `code` with `response_mode=jwt` (JARM). | Historical reason hybrid `code id_token` exists in deployments; superseded by FAPI 2.0. |

No OpenID Foundation document was found that deprecates the implicit or hybrid flows in OpenID Connect Core itself. Core incorporating errata set 2 still specifies both.

## Questions that need a human decision

1. **Do `id_token` and `code id_token` stay?** Both are compatible with RFC 9700 and OAuth 2.1. Removing all implicit and hybrid support is simpler and matches FAPI 2.0; keeping these two preserves OpenID Connect clients that have no token endpoint call (`id_token`) or that verify the front-channel ID token before redeeming the code (`code id_token`). The four access-token-bearing response types have no such defence.
2. **Is the `nonce` exception to PKCE kept for confidential OpenID Connect clients (G2)?** OAuth 2.1 allows it; "remove, do not disable" suggests PKCE for every client with no opt-out.
3. **Does the product ever intend to be a "Dynamic OpenID Provider"?** If yes, OIDC Core §15.2 demands `id_token token`, which conflicts with question 1's likely answer. If no, say so in an ADR so the conflict is closed.
4. **Is `iss` in the authorization response (G1, RFC 9207) adopted now?** It is additive, already required by FAPI 2.0, and the largest protocol-visible OAuth 2.1 item.
5. **Token request `redirect_uri` (G4)**: accept-if-present-and-verify (works for 2.0 and 2.1 clients) or keep requiring it until OAuth 2.1 is published?
6. **RFC 10070 audience rule (G13)** breaks every existing `private_key_jwt` client that sends the token endpoint as `aud`. Given ADR 0001 (no compatibility), is a hard switch acceptable?
7. **Redirect URI case sensitivity (Part 1 #1)** will break clients whose registered and sent URIs differ only by case. Same compatibility question.
8. **Loopback redirect (Part 1 #2)**: build it into the default validator (exact scheme, host and path, any port, `127.0.0.1` and `[::1]`) and delete the AppAuth-specific one?
9. **Should the client model gain an explicit client type** (confidential or public; perhaps web, browser-based, native), so rules such as G2, G10, G12, G17 and Part 1 #13 can be enforced instead of inferred from `RequireClientSecret`?

## What could not be verified

- **Whether a working group last call for the OAuth 2.1 draft is planned or imminent.** The Datatracker shows state "WG Document" and no last-call event; mailing list and meeting minutes were not read.
- **RFC 9700 errata.** The errata page was fetched but its content could not be parsed reliably; no erratum is accounted for here.
- **RFC 10017** was confirmed to exist (header: August 2026, Best Current Practice) and is cited by draft-17, but its body was not read. Claims about browser-based apps come from draft-17 §9 only.
- **The host application's UI** (login, consent, error, logout pages) is outside the library. Clickjacking protection, third-party content, `returnUrl` handling and the credential POST redirect (Part 1 #3, #4, #27, #28) were judged from the library side only.
- **Duplicate request parameters** (draft-17 §3.1, §3.2, and RFC 6749 §3.1) and **brute-force protection for client secrets** (draft-17 §2.4.1) were not audited.
- **Entity Framework stores and ASP.NET Identity integration** (`src/EntityFramework*`, `src/AspNetIdentity`) were not read; findings assume they add no protocol behaviour.
- **OpenID Foundation certification profiles** (which response types each conformance profile tests) were not checked; only the specifications were read.
- No code was executed. Every verdict comes from reading the source, not from a test.
