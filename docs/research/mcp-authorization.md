# MCP authorization and AI-facing features: what they would ask of IdentityServer4Ever

Research for issue #5. Researched 2026-11-13 against the code at commit `5e40bd18` (`main`). Only `src/` was examined. Nothing here is a decision; it feeds one.

## Summary

"MCP support" can mean two unrelated things, and the owner needs to pick between them (or both):

1. **Being the login and token service for other people's MCP servers.** The Model Context Protocol (MCP) lets AI applications call tools hosted on remote servers. When such a server is protected, the AI application has to obtain an OAuth access token from an authorization server. IdentityServer4Ever could be that authorization server. This is protocol work inside the existing product.
2. **Being an MCP server itself.** IdentityServer4Ever could expose its own tools to AI agents, for example "list clients" or "show recent failed logins". This is a new product surface that does not exist in any form today.

Findings for the first meaning:

- The current MCP specification is revision **2026-07-28**. It is newer than the 2025-11-25 revision most write-ups describe.
- The code already has most of the hardened OAuth core the specification relies on: PKCE on by default, redirect URIs matched against a registered list, one-time refresh tokens by default, a discovery document that advertises PKCE support.
- Four things the specification asks of an authorization server are **absent**: reading the `resource` parameter (RFC 8707), returning the issuer in the authorization response (RFC 9207), letting a client identify itself with a URL (Client ID Metadata Documents), and dynamic client registration (RFC 7591, now deprecated in MCP).
- With a manually registered client, an MCP client can probably complete a login against the code as it stands. I did not run that test.
- The specification says "OAuth 2.1" in its text, but OAuth 2.1 is still an unpublished draft. What the specification needs in practice is OAuth 2.0 hardened per RFC 9700 plus the four additions above. The decided baseline is therefore sufficient as a foundation. No requirement depends on DPoP, PAR or FAPI.

Findings for the second meaning:

- Among the comparable products checked, Auth0, Okta and Microsoft Entra publish MCP servers that administer or query the identity system. Duende and Ory publish MCP servers that only search their documentation. No vendor-published administrative MCP server was found for Keycloak, Zitadel or OpenIddict.
- The vendors that expose administration to agents all limit it: Microsoft's is read-only, Auth0's starts with no permissions selected and has a read-only switch, Okta's only shows the tools the granted scopes allow.
- IdentityServer4Ever has no management API to put such tools on top of. The code can look up one client by its identifier and nothing more. An administrative MCP server therefore implies building a management layer first.

The options and their scope cost are in [Options](#options-for-what-mcp-support-could-mean).

## Part 1: what MCP authorization requires of an authorization server

### 1.1 Which specification

The current revision is **2026-07-28**. The versioning page states: "The **current** protocol version is **2026-07-28**" ([versioning](https://modelcontextprotocol.io/specification/versioning)). The published revisions are 2024-11-05, 2025-03-26, 2025-06-18, 2025-11-25 and 2026-07-28, plus a `draft` tree ([index](https://modelcontextprotocol.io/llms.txt)).

In 2026-07-28 the authorization section is split across four pages:

- [Authorization](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization)
- [Authorization Server Discovery](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/authorization-server-discovery)
- [Client Registration](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/client-registration)
- [Security Considerations](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/security-considerations)

Authorization changes since 2025-11-25, from the [changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog):

- Authorization servers SHOULD return the `iss` parameter (RFC 9207), and clients MUST validate it when present (SEP-2468).
- Dynamic Client Registration is deprecated in favour of Client ID Metadata Documents. The [deprecation registry](https://modelcontextprotocol.io/specification/2026-07-28/deprecated) gives its earliest removal as the "First revision released on or after 2027-07-28".
- Clients MUST send an `application_type` when they do use Dynamic Client Registration (SEP-837).
- Clients MUST bind stored client credentials to the issuer that granted them (SEP-2352).

The rest of the 2026-07-28 revision removed sessions and the initialization handshake from the protocol. That affects anyone building an MCP server (Part 2), not the authorization server.

Authorization is optional in MCP and applies to HTTP transports. For local (stdio) servers the specification says implementations "SHOULD NOT follow this specification, and instead retrieve credentials from the environment" ([Authorization, Protocol Requirements](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization)).

### 1.2 Who has to do what

The specification defines three roles ([Authorization, Roles](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization)):

- The **MCP client** is the OAuth client.
- The **MCP server** is the OAuth resource server. It publishes Protected Resource Metadata (RFC 9728), sends `WWW-Authenticate` challenges, and validates token audience.
- The **authorization server** issues the tokens. The specification says its implementation details "are beyond the scope of this specification".

Most of the MUSTs in the specification fall on the client and the MCP server. RFC 9728 is an MCP-server duty, not an authorization-server duty. Keycloak's guide makes the same reading: "The standard is for an MCP server and not for an authorization server like Keycloak" ([Keycloak MCP guide](https://www.keycloak.org/securing-apps/mcp-authz-server)). The table below lists only what falls on the authorization server, directly or because a compliant client or MCP server depends on it.

### 1.3 Requirements and code status

Paths are relative to the repository root. "Core" means `src/IdentityServer4/src`.

| # | Requirement on the authorization server | Level in MCP 2026-07-28 | Underlying document | Status in `src/` |
| - | - | - | - | - |
| 1 | Implement OAuth 2.1 "with appropriate security measures for both confidential and public clients" | MUST | draft-ietf-oauth-v2-1-13 | **Partial** (see 1.4) |
| 2 | Publish metadata by RFC 8414 or OpenID Connect Discovery | MUST (one of the two) | RFC 8414, OIDC Discovery 1.0 | **Present** (OIDC Discovery only) |
| 3 | Include `code_challenge_methods_supported` in OIDC discovery metadata | MUST | MCP security considerations | **Present** |
| 4 | Support PKCE; accept `S256` | MUST (through OAuth 2.1) | OAuth 2.1 §4.1.1, §7.5.2; RFC 7636 | **Present** |
| 5 | Validate redirect URIs by exact match against registered values | MUST | OAuth 2.1; RFC 9700 §2.1 | **Partial** (case-insensitive compare) |
| 6 | Serve all authorization server endpoints over HTTPS | MUST | OAuth 2.1 §1.5 | **Not enforced** by the library; left to hosting |
| 7 | Redirect URIs are `localhost` or HTTPS | MUST | OAuth 2.1 §1.5 | **Not enforced** |
| 8 | Rotate refresh tokens for public clients | MUST | OAuth 2.1 §4.3.1; RFC 9700 §2.2.2 | **Partial** (default yes, can be configured off) |
| 9 | Issue short-lived access tokens | SHOULD | MCP security considerations | **Present** (configurable, default one hour) |
| 10 | Return `iss` in authorization responses, including errors; if so, advertise `authorization_response_iss_parameter_supported` | SHOULD, announced to become MUST | RFC 9207 | **Missing** |
| 11 | Honour the `resource` parameter and bind the token audience to it | No explicit MUST on the server; clients MUST send it and MCP servers MUST reject tokens not issued for them | RFC 8707 | **Missing**; audience binding by scope is available as a substitute |
| 12 | Support Client ID Metadata Documents | SHOULD | draft-ietf-oauth-client-id-metadata-document-00 | **Missing** |
| 13 | Accept pre-registered clients | Implicit (clients SHOULD support it) | RFC 6749 | **Present** |
| 14 | Support Dynamic Client Registration | MAY, deprecated | RFC 7591, OIDC Registration 1.0 | **Missing** |
| 15 | Avoid redirecting to untrusted URIs | MUST | OAuth 2.1 §7.12.2 | **Present** (redirects only to registered URIs) |
| 16 | Client credentials grant, with secret or `private_key_jwt` | Optional extension | RFC 6749 §4.4, RFC 7523 | **Present** (`private_key_jwt` is opt-in) |
| 17 | Accept an identity assertion grant (ID-JAG) | Optional extension | draft-ietf-oauth-identity-assertion-authz-grant | **Missing** |

Sources for the levels: rows 1, 2, 12, 14 from the [Authorization overview](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization); rows 3 to 9 and 15 from [Security Considerations](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/security-considerations); row 10 from [Authorization Response Validation](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization); row 11 from [Resource Parameter Implementation](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization) and [Token Audience Binding](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/security-considerations); rows 12 to 14 from [Client Registration](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/client-registration); rows 16 and 17 from the [authorization extensions](https://modelcontextprotocol.io/extensions/auth/overview).

### 1.4 Detail per requirement

#### Metadata discovery (rows 2 and 3)

The specification: "MCP authorization servers **MUST** provide at least one of the following discovery mechanisms: OAuth 2.0 Authorization Server Metadata (RFC8414), OpenID Connect Discovery 1.0", and clients MUST support both. For an issuer without a path, clients try `/.well-known/oauth-authorization-server` and then `/.well-known/openid-configuration`. Clients MUST reject a document whose `issuer` differs from the issuer they used to build the URL ([Authorization Server Discovery](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/authorization-server-discovery)).

For OIDC providers there is one extra rule: "Authorization servers providing OpenID Connect Discovery 1.0 **MUST** include `code_challenge_methods_supported` in their metadata to ensure MCP compatibility", because clients MUST refuse to proceed when it is absent ([Security Considerations](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/security-considerations)).

In the code:

- The discovery document is served at `.well-known/openid-configuration` (`src/IdentityServer4/src/Constants.cs:216`, registered in `Configuration/DependencyInjection/BuilderExtensions/Core.cs:86`). That satisfies the "one of the two" rule.
- There is no `/.well-known/oauth-authorization-server` route. A search of `src/` for `oauth-authorization-server` returns nothing. It is not required. One consumer that does expect it: Microsoft 365 Copilot's dynamic registration needs the server to publish metadata "at the `.well-known/oauth-authorization-server` endpoint" ([Microsoft Learn](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/plugin-authentication-dynamic-client-registration)).
- `code_challenge_methods_supported` is always emitted, with both `plain` and `S256` (`ResponseHandling/Default/DiscoveryResponseGenerator.cs:319`). It is not behind any of the `Show*` discovery switches.
- The document does not contain `authorization_response_iss_parameter_supported`, `client_id_metadata_document_supported` or `registration_endpoint`, because none of those features exist. Extra entries can be added through `Options.Discovery.CustomEntries` (`DiscoveryResponseGenerator.cs:337`), but advertising a feature that is not implemented would mislead clients.
- The issuer comes from the request or from `IssuerUri`, and is lower-cased by default (`Configuration/DependencyInjection/Options/IdentityServerOptions.cs:19-24`). Clients compare the issuer as an exact string, so the configured issuer and the URL given to MCP servers must match character for character.

#### OAuth 2.1 core (rows 1, 4 to 9, 15)

What OAuth 2.1 changes relative to OAuth 2.0, from the draft's own list ([draft-ietf-oauth-v2-1-17 §10](https://www.ietf.org/archive/id/draft-ietf-oauth-v2-1-17.txt)), against the code:

| OAuth 2.1 change | Code |
| - | - |
| Authorization code grant requires PKCE | `Client.RequirePkce` defaults to `true` (`src/Storage/src/Models/Client.cs:98`). Enforced in `Validation/Default/AuthorizeRequestValidator.cs:481-540`. A client can still be configured with `RequirePkce = false`. |
| PKCE `plain` method removed | `plain` is still implemented and advertised. It is rejected unless the client sets `AllowPlainTextPkce`, which defaults to `false` (`Client.cs:103`, `AuthorizeRequestValidator.cs:527-535`). |
| PKCE downgrade protection (RFC 9700 §2.1.1) | Present. A `code_verifier` without a stored challenge is rejected (`Validation/Default/TokenRequestValidator.cs:308-326`). |
| Redirect URIs compared by exact string matching | `StrictRedirectUriValidator` compares with `StringComparer.OrdinalIgnoreCase` (`Validation/Default/StrictRedirectUriValidator.cs:30`). That is a list match with no wildcards, but it ignores case, which is looser than exact string matching. |
| Implicit grant removed | Still implemented; allowed per client through `AllowedGrantTypes`. The discovery document always lists `implicit` (`DiscoveryResponseGenerator.cs:258-264`). |
| Resource owner password grant removed | Still implemented; available when a password validator is registered (`DiscoveryResponseGenerator.cs:266-269`). |
| No bearer tokens in the query string | The server's own endpoints accept a bearer token from the `Authorization` header or the form body only (`Validation/Default/BearerTokenUsageValidator.cs:37-46`). |
| Refresh tokens for public clients are sender-constrained or one-time-use | `RefreshTokenUsage` defaults to `OneTimeOnly` (`Client.cs:204`). A consumed token is rejected (`Services/Default/DefaultRefreshTokenService.cs:113-123`, `154-160`). `ReUse` can still be configured for a public client; nothing ties the setting to the client type. |
| Token request no longer carries `redirect_uri` | The code requires it and fails the request when it is missing (`TokenRequestValidator.cs:281-286`). |
| Server must accept client credentials in the request body | Present (`Validation/Default/PostBodySecretParser.cs`). |

Other points:

- Authorization codes are single-use. The code is deleted on first redemption (`TokenRequestValidator.cs:250`). Tokens already issued from a replayed code are not revoked, which RFC 6749 §4.1.2 lists as a SHOULD.
- Nothing in the library refuses to run over plain HTTP, and `DefaultClientConfigurationValidator` blocks only a list of dangerous schemes such as `javascript:` (`Configuration/DependencyInjection/Options/ValidationOptions.cs:17`). It does not require redirect URIs to be HTTPS or loopback.
- Access token lifetime defaults to 3600 seconds per client (`Client.cs:178`).
- The optional loopback validator, `StrictRedirectUriValidatorAppAuth`, accepts any port on `http://127.0.0.1` but not `http://localhost` (`Validation/Default/StrictRedirectUriValidatorAppAuth.cs:43`, `93-94`). The specification's own example client uses both forms.

Summary for row 1: the secure behaviour is the default, but the insecure OAuth 2.0 behaviour remains reachable by configuration. That is exactly the "hardened per RFC 9700" position rather than an "OAuth 2.1 only" position.

#### Issuer in the authorization response (row 10)

The specification: "MCP authorization servers **SHOULD** include the `iss` parameter in authorization responses, including error responses, as defined in RFC9207 Section 2. Authorization servers that include the `iss` parameter **MUST** advertise this by setting `authorization_response_iss_parameter_supported` to `true` in their metadata." It adds: "A future revision of this specification is expected to upgrade authorization server inclusion of `iss` from **SHOULD** to **MUST**" ([Authorization Response Validation](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization)).

In the code: missing. The authorization response is built in `Extensions/AuthorizeResponseExtensions.cs` and carries `code`, `id_token`, `access_token`, `scope`, `state`, `session_state`, and `error` fields. There is no `iss`.

A client talking to this server today falls into the specification's fourth table row (metadata flag absent, `iss` absent: "Proceed"), so the gap does not block a login. It leaves the mix-up protection off, and the MCP security guidance notes the mitigation "provides no protection against an honest server that does not" emit `iss` ([Security Best Practices, Mix-Up Attacks](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices)).

This is a small change and is also recommended by RFC 9700 §2.1 for any client that talks to more than one authorization server ([RFC 9700](https://www.rfc-editor.org/rfc/rfc9700.txt)). It may already belong in the RFC 9700 baseline regardless of MCP.

#### Resource indicators and token audience (row 11)

The specification puts the MUSTs on the other two parties:

- "MCP clients **MUST** implement Resource Indicators for OAuth 2.0 as defined in RFC 8707". The `resource` parameter "**MUST** be included in both authorization requests and token requests" and identifies the MCP server by its canonical URI. "MCP clients **MUST** send this parameter regardless of whether authorization servers support it."
- "MCP servers **MUST** validate that access tokens were issued specifically for them as the intended audience" and "**MUST** reject tokens that do not include them in the audience claim or otherwise verify that they are the intended recipient of the token."

For the authorization server the specification says only that resource indicators bind tokens to audiences "**when the Authorization Server supports the capability**" ([Security Considerations](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/security-considerations)). RFC 8707 §2 itself says the server "SHOULD audience-restrict issued access tokens to the resource(s) indicated by the `resource` parameter" and may "map from that value to a more general URI or abstract identifier" ([RFC 8707](https://www.rfc-editor.org/rfc/rfc8707.txt)). Keycloak's compliance table nonetheless lists RFC 8707 as MUST for the authorization server ([Keycloak MCP guide](https://www.keycloak.org/securing-apps/mcp-authz-server)). The practical position: an MCP server can only meet its own MUST if the authorization server puts something in the token that identifies that MCP server.

In the code:

- The `resource` parameter is not read anywhere. The only trace is a commented-out property, `// public IEnumerable<string> ResourceIndicators { get; set; }` (`Validation/Models/ResourceValidationRequest.cs:29`). Unknown parameters are ignored, so an MCP client's `resource` value is silently dropped rather than rejected.
- The `aud` claim of an access token is the set of `ApiResource` names that own the granted scopes (`Services/Default/DefaultTokenService.cs:222-226`).

So audience binding exists, but it is driven by scope, not by `resource`. An operator can register each MCP server as an `ApiResource` whose name is the MCP server's canonical URI and give it its own scopes; tokens for those scopes then carry that URI as `aud`. The EF storage layer limits the name to 200 characters (`src/EntityFramework.Storage/src/Extensions/ModelBuilderExtensions.cs:219`). What this substitute cannot do:

- Narrow a token when one scope belongs to several API resources. The token then lists every one of them as audience.
- Reject a request whose `resource` names a server the client may not use (the `invalid_target` error of RFC 8707).
- Give different audiences to tokens obtained from one refresh token.

Other products made the same journey. Keycloak without its experimental feature "does not recognize the `resource` parameter" and documents an audience-mapper substitute. Zitadel's `resource` parameter is "accepted on the authorization code flow but ignored, so it does not narrow `aud`" ([Zitadel](https://zitadel.com/docs/guides/integrate/dynamic-client-registration)). Ory Hydra added RFC 8707 support in v26.4.7 specifically for "access tokens targeted to Model Context Protocol (MCP) servers" ([Ory changelog](https://www.ory.com/docs/self-hosted/oel/oauth2/changelog/v26.4.7)).

#### Client registration (rows 12 to 14)

The specification gives clients this priority order: pre-registered information if they have it; Client ID Metadata Documents if the server advertises `client_id_metadata_document_supported`; Dynamic Client Registration if the server has a `registration_endpoint`; otherwise ask the user ([Client Registration](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/client-registration)).

**Pre-registration (present).** A public client is a `Client` with `RequireClientSecret = false` (`Client.cs:50`), the authorization code grant, PKCE and its redirect URIs. This works today. Its cost falls on users: someone has to register the client and type the client ID into the AI application. That is why the other two mechanisms exist; the specification calls the no-prior-relationship case the "most common" one.

**Client ID Metadata Documents (missing).** The client's `client_id` is an HTTPS URL. The authorization server fetches a JSON document from that URL and treats it as the client registration. Requirements on the server, from the specification:

- "**SHOULD** fetch metadata documents when encountering URL-formatted client_ids"
- "**MUST** validate that the fetched document's `client_id` matches the URL exactly"
- "**SHOULD** cache metadata respecting HTTP cache headers"
- "**MUST** validate redirect URIs presented in an authorization request against those in the metadata document"
- "**MUST** validate the document structure is valid JSON and contains required fields"
- Advertise support with `"client_id_metadata_document_supported": true`
- "**SHOULD** consider" server-side request forgery risks when fetching
- "**SHOULD** display additional warnings for `localhost`-only redirect URIs" and "**MUST** clearly display the redirect URI hostname during authorization"
- "**MAY** implement domain-based trust policies"

The underlying draft adds: client ID URLs must be HTTPS with a path and no fragment or credentials; shared-secret authentication methods are forbidden in the document; error responses and invalid documents must not be cached; and the response size should be limited, with 5 kilobytes recommended ([draft -00](https://www.ietf.org/archive/id/draft-ietf-oauth-client-id-metadata-document-00.txt)).

Where this meets the code:

- Clients are resolved only through `IClientStore.FindClientByIdAsync` (`src/Storage/src/Stores/IClientStore.cs:20`, called at `AuthorizeRequestValidator.cs:209`). A URL-shaped client ID is simply an unknown client.
- `client_id` is limited to 100 characters on input (`Configuration/DependencyInjection/Options/InputLengthRestrictions.cs:17`) and 200 in the grant tables (`ModelBuilderExtensions.cs:139`, `161`). URLs can exceed both.
- There is one precedent for the server fetching a client-supplied URL: the `request_uri` fetcher (`Services/Default/DefaultJwtRequestUriHttpClient.cs:39`). A metadata fetcher would be a second one with stricter rules.
- The consent screen is not part of `src/IdentityServer4/src`; the host application supplies it. The core does hand the redirect URI and client to the UI (`Models/Messages/AuthorizationRequest.cs:20`, `36`), so the "display the hostname" requirement would be met by guidance or a reference UI rather than by core code.

One caution on stability: MCP pins revision **-00** of the draft, while the IETF draft was at revision **-03** on 2026-11-02 ([datatracker](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/)). I did not compare the two revisions.

**Dynamic Client Registration (missing, and deprecated).** "MCP clients and authorization servers **MAY** support" RFC 7591; the page carries the warning "Dynamic Client Registration is deprecated. New implementations should use Client ID Metadata Documents instead." There is no registration endpoint in `src/` and the client store is read-only, so this would need a writable store, a new endpoint and abuse controls. It still matters in the field: earlier MCP revisions used it (Keycloak's table shows it as SHOULD for 2025-03-26 and 2025-06-18), Microsoft 365 Copilot offers it as an authentication type for MCP plugins, and Zitadel states open registration is "Required for the MCP flow". Duende's guidance for its current version is "use pre-registration or DCR" and "Do not expose an unrestricted DCR endpoint" ([Duende](https://docs.duendesoftware.com/identityserver/ai/)).

#### Scopes and refresh tokens

These are mostly client and MCP-server rules, with two touch points on the authorization server:

- A client omits `scope` when the MCP server publishes no `scopes_supported` and sends no scope in its challenge ([Scope Selection Strategy](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization)). This server rejects an authorization request without `scope` (`AuthorizeRequestValidator.cs:547-552`). Operators avoid the failure by having the MCP server publish its scopes.
- A client "**MAY** add `offline_access`" when the authorization server lists it in `scopes_supported`, and "**MUST NOT** assume refresh tokens will be issued". This server lists `offline_access` whenever API scopes are shown (`DiscoveryResponseGenerator.cs:233`) and issues refresh tokens only when the client has `AllowOfflineAccess`. That matches.

#### Optional extensions (rows 16 and 17)

Two official extensions live outside the core specification ([overview](https://modelcontextprotocol.io/extensions/auth/overview)). Both are opt-in.

- **OAuth Client Credentials**, for machine-to-machine use, with a client secret or a signed JWT ([extension](https://modelcontextprotocol.io/extensions/auth/oauth-client-credentials)). The client credentials grant is built in, and `private_key_jwt` client authentication is available through `AddJwtBearerClientAuthentication` (`Configuration/DependencyInjection/BuilderExtensions/Additional.cs:308-309`). The extension's diagram also shows `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer`, which this code does not have built in; it would be an `IExtensionGrantValidator`. I read the overview page, not the normative extension text.
- **Enterprise-Managed Authorization**, where a corporate identity provider issues an "ID-JAG" assertion that the MCP server's authorization server exchanges for an access token ([extension](https://modelcontextprotocol.io/extensions/auth/enterprise-managed-authorization)). It rests on an IETF draft at revision -06 ([datatracker](https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/)). Nothing in `src/` implements either side. The extension page's own support table marks Keycloak's acceptance of ID-JAGs as "Experimental" and Auth0's as "Early access".

### 1.5 OAuth 2.1 or OAuth 2.0 hardened per RFC 9700?

**The text says OAuth 2.1; in practice RFC 9700-hardened OAuth 2.0 is enough.**

The text: "Authorization servers **MUST** implement OAuth 2.1 with appropriate security measures for both confidential and public clients", citing draft-ietf-oauth-v2-1-13.

Why the hardened baseline is enough:

1. **OAuth 2.1 does not exist as a standard yet.** The draft is at revision 17, dated 31 October 2026. Its datatracker state is "I-D Exists" and "WG Document", meaning the IESG has not started processing it ([datatracker](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/)). MCP cites revision 13, and one MCP paragraph cites revision 14. Nobody can conform to a moving draft in a strict sense.
2. **OAuth 2.1 is defined as OAuth 2.0 plus current best practice.** The draft says it "consolidates the functionality in OAuth 2.0 [RFC6749], OAuth 2.0 for Native Apps [RFC8252], Proof Key for Code Exchange [RFC7636], OAuth 2.0 for Browser-Based Applications [RFC10017], OAuth Security Best Current Practice [RFC9700], and Bearer Token Usage [RFC6750]" and that "OAuth 2.1 is compatible with OAuth 2.0 with the extensions and restrictions from known best current practices applied" ([draft -17 §1.8, §10](https://www.ietf.org/archive/id/draft-ietf-oauth-v2-1-17.txt)). Nearly every item in its list of differences cites a section of RFC 9700.
3. **A server may serve both.** "An authorization server can support both OAuth 2.0 and OAuth 2.1 clients. Which rules apply to a given client is determined by the authorization server's configuration for that client" (same draft, §1.8). Per-client settings such as `RequirePkce` are that mechanism.
4. **The items MCP actually spells out are RFC 9700 items**: PKCE with `S256`, exact redirect URI matching, refresh token rotation for public clients, HTTPS, no tokens in query strings.

Where OAuth 2.1 goes beyond RFC 9700, and whether it bites:

| Difference | RFC 9700 | OAuth 2.1 draft | Effect for MCP |
| - | - | - | - |
| PKCE for confidential clients | RECOMMENDED (§2.1.1) | Required | None. MCP clients MUST use PKCE anyway. |
| PKCE `plain` | Clients SHOULD use `S256` | `plain` removed | None. MCP clients MUST use `S256`. |
| Implicit grant | SHOULD NOT be used (§2.1.2) | Removed | None. MCP uses the code flow. |
| `redirect_uri` in the token request | Not addressed | Removed; a server that supports both generations "MUST allow clients to send the redirect_uri parameter in the token request" | **Possible interoperability issue.** A client that follows only OAuth 2.1 omits the parameter, and this code rejects such a request. The official TypeScript SDK client still sends it ([source](https://github.com/modelcontextprotocol/typescript-sdk/blob/main/packages/client/src/client/auth.ts), `prepareAuthorizationCodeRequest`, read 2026-11-13). Other clients were not checked. |

So the decided baseline (OAuth 2.0 as tightened by RFC 9700, OAuth 2.1 specifics deferred) covers the security substance. The one OAuth 2.1 specific worth pulling forward for MCP is tolerating a token request without `redirect_uri` for clients that used PKCE. The real gaps for MCP are not OAuth 2.1 at all. They are the separate documents: RFC 8707, RFC 9207, Client ID Metadata Documents and, for older clients, RFC 7591.

A caveat on claims: Keycloak states it "supports" the OAuth 2.1 draft in its MCP table. If IdentityServer4Ever wants to print "conforms to MCP authorization 2026-07-28", it will have to decide what it says about the OAuth 2.1 line. That is a wording decision, not an engineering one.

### 1.6 What comparable products do as an MCP authorization server

Only what the vendor's own documentation states.

| Product | As authorization server for MCP | Source |
| - | - | - |
| Keycloak | Has an MCP guide. Rates itself "Supported" for MCP 2025-03-26 and "Experimental" for 2025-06-18, 2025-11-25 and 2026-07-28. RFC 8414, RFC 9207 and RFC 7591 supported; RFC 8707 and Client ID Metadata Documents experimental behind feature flags. | [keycloak.org](https://www.keycloak.org/securing-apps/mcp-authz-server) |
| Duende IdentityServer | Has an MCP page for v8 that links the 2026-07-28 specification. Describes resource indicators and DCR. Says Client ID Metadata Documents are "preferred by the current MCP specification" but "For IdentityServer v8 deployments, use pre-registration or DCR." | [docs.duendesoftware.com](https://docs.duendesoftware.com/identityserver/ai/) |
| Auth0 | Documents tenant settings "Resource Parameter Compatibility Profile" and "Include Issuer in Authorization Responses" for MCP. The page read did not mention DCR or Client ID Metadata Documents. | [auth0.com](https://auth0.com/ai/docs/mcp/get-started/authorization-for-your-mcp-server) |
| Microsoft Entra ID | "Microsoft Entra ID doesn't publish an RFC 7591 registration endpoint"; clients must be registered statically. Client ID Metadata Documents and RFC 8707 were not checked. | [Microsoft Learn](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/plugin-authentication-dynamic-client-registration), [App Service](https://learn.microsoft.com/en-us/azure/app-service/configure-authentication-mcp) |
| Okta | The MCP extension page lists Okta as issuing and accepting ID-JAGs under the name Cross App Access. Okta's own pages on this, and its position on RFC 8707, DCR and Client ID Metadata Documents, were not read. | [MCP extension page](https://modelcontextprotocol.io/extensions/auth/enterprise-managed-authorization) |
| Ory Hydra | RFC 8707 added in v26.4.7 for "access tokens targeted to Model Context Protocol (MCP) servers"; MCP clients "register through dynamic client registration (DCR)". The changelog does not mention Client ID Metadata Documents. Ory also publishes MCP guides, which were not read. | [ory.com changelog](https://www.ory.com/docs/self-hosted/oel/oauth2/changelog/v26.4.7) |
| Zitadel | RFC 7591 registration, off by default, with a token-protected mode and an open mode "Required for the MCP flow". `resource` is accepted but ignored. No mention of Client ID Metadata Documents on the page. | [zitadel.com](https://zitadel.com/docs/guides/integrate/dynamic-client-registration) |
| OpenIddict | Nothing found in its documentation about MCP, DCR, resource indicators or Client ID Metadata Documents. Its issue tracker lists an open request, "Implement Dynamic Client Registration (RFC 7591/7592) support" (#2404). | [documentation.openiddict.com](https://documentation.openiddict.com/), [issue search](https://github.com/openiddict/openiddict-core/issues?q=MCP) |

The pattern: every product that claims MCP readiness added resource indicators, and most relied on DCR first. Client ID Metadata Documents are experimental at best in the products checked, even though the specification now prefers them.

## Part 2: an identity server's own MCP server or AI feature

### 2.1 What comparable products offer

| Product | Offering | What it does | Limits the vendor documents | Source |
| - | - | - | - | - |
| Auth0 | Auth0 MCP Server, run locally with `npx`. The repository README says "currently in beta". | Tenant management: list, read, create and update applications, APIs (resource servers), Actions and Forms; deploy Actions; create application grants; read logs. | Signs in with the device authorization flow; credentials kept in the system keychain. "By default, no scopes are selected for maximum security." A `--read-only` flag and a `--tools` filter. Guidance to "review them carefully before confirming" changes. | [docs](https://auth0.com/docs/get-started/auth0-mcp-server), [tools reference](https://auth0.com/docs/get-started/auth0-mcp-server/auth0-mcp-tools-reference), [README](https://github.com/auth0/auth0-mcp-server) |
| Okta | Two servers: an Open Source MCP Server (self-hosted, stdio) and an Okta Managed MCP Server (Okta-hosted, HTTPS). Release status not stated on the pages read. | Users, groups, applications, policies, system logs, brands and customization, including create, update, deactivate and delete. | Tools are registered "based on the granted API scopes of the Okta admin app", then checked again "immediately before executing a command". Activity goes to the Okta System Log. | [concepts](https://developer.okta.com/docs/concepts/mcp-server/), [overview](https://developer.okta.com/docs/guides/okta-open-source-mcp-server/main/) |
| Microsoft Entra | Microsoft MCP Server for Enterprise, hosted by Microsoft, "currently in PREVIEW". | Three tools: suggest Microsoft Graph queries, run "read-only Microsoft Graph API calls", list entity properties. | Read-only. Enforces "the user's privileges and the scopes granted to the MCP client". 100 calls per minute per user. Every request appears in the Microsoft Graph activity logs. | [Microsoft Learn](https://learn.microsoft.com/en-us/graph/mcp-server/overview) |
| Duende | Documentation MCP server (`Duende.Documentation.Mcp`), run locally, plus agent "skills". | Searches documentation, blog posts and samples. Does not administer or configure a running IdentityServer. | Not applicable; it touches no live system. "No Duende license is required". | [docs](https://docs.duendesoftware.com/general/ai-agent-tools/) |
| Ory | Ory Docs MCP server, hosted. | Lets assistants "search and retrieve Ory documentation in real time". No administration tools described. | Not applicable. | [ory.com](https://www.ory.com/docs/ecosystem/mcp) |
| Keycloak | **Nothing found** published by the Keycloak project. | Community-built administrative MCP servers exist on third-party directories; they are not vendor offerings and were not evaluated. | | Search of keycloak.org, 2026-11-13 |
| Zitadel | **Nothing found** published by Zitadel. | A community server exists; not evaluated. | | Search of zitadel.com, 2026-11-13 |
| OpenIddict | **Nothing found.** | | | [documentation.openiddict.com](https://documentation.openiddict.com/) |

Three kinds of offering appear:

1. **Documentation search** (Duende, Ory). It helps developers who use an AI coding assistant. It never touches a running identity server.
2. **Read-only query over live data** (Microsoft). It answers questions such as "which users didn't sign in last month?".
3. **Administrative tools with write access** (Auth0, Okta), always with opt-in permissions.

### 2.2 What IdentityServer4Ever could usefully expose

These are candidates derived from what the code holds, not recommendations.

**Read-only configuration and diagnostics**

- List and describe clients, API resources, API scopes and identity resources.
- Explain a client's effective settings and flag ones that depart from RFC 9700 (PKCE off, implicit grant on, reusable refresh tokens, non-HTTPS redirect URIs). The rules are the same ones `DefaultClientConfigurationValidator` and this document's table 1.4 apply.
- Show the discovery document and signing key status.
- Query recent events. The core raises typed events for token issuance, login, consent, client authentication and errors (`src/IdentityServer4/src/Events/`), delivered to an `IEventSink` (`Services/IEventSink.cs`).
- List a user's grants, which the grant store can already filter by subject, session, client and type (`src/Storage/src/Stores/PersistedGrantFilter.cs:16-31`).

**Administrative, with write access**

- Create or change clients, scopes and resources; rotate client secrets.
- Revoke a user's or a client's grants.

**Developer assistance, outside the running server**

- A documentation or samples search server in the style of Duende's and Ory's. It lives outside `src/`.

**What the code lacks for any of the live-system options**

- There is **no management API**. Configuration is read through `IClientStore` and `IResourceStore`. `IClientStore` has a single method, `FindClientByIdAsync` (`src/Storage/src/Stores/IClientStore.cs:20`); it cannot list clients and nothing can write them. Listing or editing would go through the Entity Framework context directly or through new store interfaces.
- Events are pushed to a sink and not stored, so "recent failed logins" needs an event store that does not exist.
- An MCP server over HTTP has to follow the 2026-07-28 protocol (no sessions, a `server/discover` method, new required headers, per the [changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog)). It also has to be an OAuth resource server itself: publish RFC 9728 metadata and validate token audience. It would therefore be the first consumer of the Part 1 work.
- The core targets `netcoreapp3.1` (`src/IdentityServer4/src/IdentityServer4.csproj:5`). Whether an MCP server library is usable from the eventual target framework depends on the modernization decisions. I did not check the .NET MCP SDK's requirements.

### 2.3 Security considerations

An identity server is the system that decides who can access everything else. Giving an AI agent tools on it deserves more caution than for an ordinary application.

**From the MCP specification**

- A human should stay in the loop. "there **SHOULD** always be a human in the loop with the ability to deny tool invocations", and clients SHOULD "Prompt for user confirmation on sensitive operations" ([Tools](https://modelcontextprotocol.io/specification/2026-07-28/server/tools)). These are duties of the AI application, which the identity server does not control. The server cannot assume the confirmation happened.
- The server's own duties: "Servers **MUST**: Validate all tool inputs; Implement proper access controls; Rate limit tool invocations; Sanitize tool outputs" (same page).
- Tool annotations such as "read-only" are advisory. Clients "**MUST** consider tool annotations to be untrusted unless they come from trusted servers". A read-only promise has to be enforced by the server, not declared.
- The tool list "**MAY** vary by the authorization presented on the request — for example, returning only the tools the caller's granted scopes permit". This is the mechanism Okta uses.
- Token rules: the MCP server "**MUST NOT** accept any tokens that were not explicitly issued for the MCP server", and must not forward the caller's token to another API ([Security Best Practices](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices)). An administrative MCP server built into the identity server would need its own audience, distinct from any other API.
- Scope design: start with a "Minimal initial scope set" of "low-risk discovery/read operations" and raise privileges through step-up challenges; avoid "wildcard or omnibus scopes" (same document, Scope Minimization).
- Local servers: use stdio or require a token, because a local HTTP server can be reached by other processes (same document, Local MCP Server Compromise).

**Specific to an identity server** (my analysis, not sourced)

- **Write tools are privilege-escalation tools.** Creating a client with a redirect URI, adding a scope to a client, or changing a secret each amount to minting access. An agent that can be steered by text it reads can be steered into doing these.
- **Read tools return attacker-influenced text.** Client names, redirect URIs, user-supplied claims and log entries are written by outsiders. With Dynamic Client Registration or Client ID Metadata Documents enabled, anyone on the internet can choose a client name that the agent will later read. That is a prompt-injection path into an agent that may also hold write tools. Read-only is not the same as harmless.
- **Secrets must never leave.** Client secrets, signing keys, refresh tokens, authorization codes and reference tokens should not be returnable by any tool, including in error messages.
- **Personal data.** Grant and login listings contain user identifiers. Tool results are sent to a model provider chosen by the operator's AI application, which may be a data-protection matter for the operator.
- **Audit.** Every tool call should be attributable to a human and a client, and recorded as an event. Okta and Microsoft both make this point in their documentation.
- **Blast radius of the feature itself.** A network-reachable administrative surface on the identity server is a new attack surface even when no AI is involved. Shipping it off by default, as a separate package, limits that.

**What the vendors' choices suggest**

- Microsoft: read-only, hosted, existing permissions apply, rate limited, logged.
- Auth0: local process, nothing permitted until the operator selects scopes, a read-only switch, a tool allow-list.
- Okta: tools filtered by granted scopes and re-checked on execution, logged.

None of the three ships unrestricted write access as a default.

### 2.4 Client ID Metadata Documents carry their own risks

These apply to Part 1 but belong with the security discussion:

- **Server-side request forgery.** The authorization server fetches a URL chosen by an unauthenticated party. "A malicious client could use this to trigger the authorization server to make requests to arbitrary URLs, such as requests to private administration endpoints the authorization server has access to" ([Security Best Practices](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices)).
- **Localhost impersonation.** The document proves control of a domain but "cannot prove which local process is listening on a `localhost` redirect URI". Any local program can present a well-known client's URL and receive the code. The countermeasures are warnings on the consent screen.
- **Open by design.** Accepting any HTTPS client ID means anyone can become a client. The specification allows domain allow-lists as policy.

Dynamic Client Registration has the mirror-image problem: an open write endpoint that "anyone can call without credentials" ([Duende blog](https://duendesoftware.com/blog/cimd-vs-dcr-a-better-client-registration-model-for-mcp-authorization)).

## Options for what "MCP support" could mean

The options are not exclusive. A to D concern Part 1; E to G concern Part 2.

| | Option | What it adds to scope | New protocol features | Size |
| - | - | - | - | - |
| A | **Documented compatibility, pre-registered clients only.** State how to register an MCP client and model an MCP server as an API resource so tokens carry the right audience. Add `iss` to authorization responses (RFC 9207). Accept a token request without `redirect_uri` when PKCE was used. | Two small protocol changes, a discovery entry, documentation, an end-to-end test with a real MCP client. | RFC 9207 only | Small |
| B | **A plus resource indicators.** Read `resource` at the authorize and token endpoints, validate it, bind `aud` to it, return `invalid_target`. | Changes to request validation, the authorization code and refresh token records, token creation, and the API resource model. | RFC 8707 | Medium |
| C | **B plus Client ID Metadata Documents.** Lets AI applications connect with no manual registration, the way the current specification prefers. | A second kind of client resolution, a hardened outbound fetcher, caching, trust policy, longer client identifiers in validation and storage, consent-screen guidance, a discovery entry. Built against an IETF draft that has already moved from -00 to -03. | Draft CIMD | Medium to large |
| D | **Dynamic Client Registration**, alone or with C, for clients built to earlier MCP revisions. | A registration endpoint, a writable client store, abuse and clean-up controls. Deprecated in MCP with removal possible from July 2027. | RFC 7591 | Medium |
| E | **Agent and enterprise extensions.** Identity assertion grant (ID-JAG) acceptance, token exchange. | New grant types on draft specifications. Of the products compared here, the MCP extension page lists Okta as available, Auth0 as early access and Keycloak as experimental. | Draft ID-JAG, RFC 8693 | Large |
| F | **Documentation MCP server** in the style of Duende and Ory. | A separate tool outside `src/`. No change to the product. Needs product documentation to exist first. | None | Small, separate |
| G | **Read-only diagnostics MCP server** as an optional package. | A management read layer over configuration and grants, an event store if log queries are wanted, an MCP server host on protocol 2026-07-28, its own resource-server protection (which needs at least A). | None in the identity protocol | Medium to large |
| H | **Administrative MCP server with write tools.** | Everything in G plus a write layer, an authorization model for administrators, audit, and safeguards for destructive actions. Highest security exposure. | None in the identity protocol | Large |
| I | **None of the above.** Leave MCP out and revisit after modernization. | Nothing. The server remains usable with pre-registered clients, undocumented. | None | None |

Notes on fit with decisions already made:

- No option needs DPoP, PAR or FAPI.
- Options B, C and D introduce protocol features beyond the RFC 9700 baseline. They are the items the "out of scope unless MCP support needs them" rule would have to admit.
- Option A's `iss` parameter is arguably part of the RFC 9700 baseline already (RFC 9700 §2.1).
- G and H depend on a management API. Whether the product should have one is a larger question than MCP.

## Questions that need a human decision

1. Which meaning of "MCP support" is wanted: authorization server for MCP servers (Part 1), an MCP server of its own (Part 2), or both?
2. Is the product willing to implement IETF drafts (Client ID Metadata Documents, and the pieces of OAuth 2.1 MCP leans on) before they are published? The earlier decision defers OAuth 2.1; MCP's text names it.
3. If Part 1: is zero-touch client onboarding a goal? If not, option A or B suffices and C and D can be skipped.
4. If zero-touch onboarding is a goal: build the deprecated mechanism most deployed clients and products use today (DCR), the preferred one that is still experimental elsewhere (CIMD), or both?
5. Which MCP revisions should be claimed? Keycloak rates itself per revision, which avoids an all-or-nothing claim.
6. If Part 2: should the product have a management API at all? An MCP server would be a thin layer over one.
7. Does the product ship a reference consent UI? Client ID Metadata Documents put MUST-level requirements on what the consent screen shows, and `src/` has no UI.

## What I could not verify

- **No end-to-end test.** I did not run an MCP client against this code. "A pre-registered client can probably log in today" is an inference from reading the validators.
- **Client behaviour beyond one SDK.** I read the official TypeScript SDK client source for two points (it sends `redirect_uri` in the token request; it falls back to no client authentication when `none` is not advertised). Other MCP clients were not examined.
- **Normative extension texts.** For the two authorization extensions I read the overview pages on modelcontextprotocol.io, not the specification files in the `ext-auth` repository.
- **Draft revisions.** I did not compare Client ID Metadata Documents -00 with -03, or read OAuth 2.1 -13 section by section against -17. Section numbers quoted from MCP refer to -13.
- **Vendor pages were read through a summarising fetch tool**, except the Microsoft Learn pages and the MCP and IETF documents, which I read as raw text. Vendor quotations are as returned by that tool. Release-status labels on the Auth0 and Okta documentation pages did not render; Auth0's "beta" comes from its GitHub README.
- **"Nothing found" is not proof of absence.** For Keycloak, Zitadel and OpenIddict I searched the vendor's site and found no vendor-published administrative MCP server. For OpenIddict I read only the documentation home page and an issue-search listing.
- **Microsoft Entra, Okta and Auth0 as authorization servers**: I confirmed only the points in table 1.6. Their support for RFC 8707, RFC 9207 and Client ID Metadata Documents is otherwise unchecked.
- **The .NET MCP SDK** and its framework requirements were not checked.
- **`samples/`, `docs/` and the host application** under `src/IdentityServer4/host` were not examined, as instructed. Any consent UI there is not counted as product code.

## Sources

MCP specification, revision 2026-07-28:

- https://modelcontextprotocol.io/specification/versioning
- https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization
- https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/authorization-server-discovery
- https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/client-registration
- https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/security-considerations
- https://modelcontextprotocol.io/specification/2026-07-28/changelog
- https://modelcontextprotocol.io/specification/2026-07-28/deprecated
- https://modelcontextprotocol.io/specification/2026-07-28/server/tools
- https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices
- https://modelcontextprotocol.io/extensions/auth/overview
- https://modelcontextprotocol.io/extensions/auth/oauth-client-credentials
- https://modelcontextprotocol.io/extensions/auth/enterprise-managed-authorization
- https://github.com/modelcontextprotocol/typescript-sdk/blob/main/packages/client/src/client/auth.ts

IETF and OpenID documents:

- OAuth 2.1 draft: https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/ (revision 17: https://www.ietf.org/archive/id/draft-ietf-oauth-v2-1-17.txt; revision 13, cited by MCP: https://www.ietf.org/archive/id/draft-ietf-oauth-v2-1-13.txt)
- Client ID Metadata Documents: https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/ (revision 00, cited by MCP: https://www.ietf.org/archive/id/draft-ietf-oauth-client-id-metadata-document-00.txt)
- Identity Assertion Authorization Grant: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/
- RFC 9700, OAuth 2.0 Security Best Current Practice: https://www.rfc-editor.org/rfc/rfc9700.txt
- RFC 8707, Resource Indicators: https://www.rfc-editor.org/rfc/rfc8707.txt
- RFC 9207, Authorization Server Issuer Identification: https://www.rfc-editor.org/rfc/rfc9207.txt
- RFC 8414, Authorization Server Metadata: https://www.rfc-editor.org/rfc/rfc8414.txt
- RFC 9728, Protected Resource Metadata: https://www.rfc-editor.org/rfc/rfc9728.txt
- RFC 7591, Dynamic Client Registration: https://www.rfc-editor.org/rfc/rfc7591.txt
- RFC 6749, OAuth 2.0: https://www.rfc-editor.org/rfc/rfc6749.txt
- OpenID Connect Discovery 1.0: https://openid.net/specs/openid-connect-discovery-1_0.html

Vendor documentation:

- Keycloak: https://www.keycloak.org/securing-apps/mcp-authz-server
- Duende: https://docs.duendesoftware.com/identityserver/ai/ , https://docs.duendesoftware.com/general/ai-agent-tools/ , https://duendesoftware.com/blog/cimd-vs-dcr-a-better-client-registration-model-for-mcp-authorization
- Auth0: https://auth0.com/docs/get-started/auth0-mcp-server , https://auth0.com/docs/get-started/auth0-mcp-server/auth0-mcp-tools-reference , https://auth0.com/docs/get-started/auth0-mcp-server/getting-started-with-auth0-mcp-server , https://github.com/auth0/auth0-mcp-server , https://auth0.com/ai/docs/mcp/get-started/authorization-for-your-mcp-server
- Okta: https://developer.okta.com/docs/concepts/mcp-server/ , https://developer.okta.com/docs/guides/okta-open-source-mcp-server/main/
- Microsoft: https://learn.microsoft.com/en-us/graph/mcp-server/overview , https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/plugin-authentication-dynamic-client-registration , https://learn.microsoft.com/en-us/azure/app-service/configure-authentication-mcp
- Ory: https://www.ory.com/docs/ecosystem/mcp , https://www.ory.com/docs/self-hosted/oel/oauth2/changelog/v26.4.7
- Zitadel: https://zitadel.com/docs/guides/integrate/dynamic-client-registration
- OpenIddict: https://documentation.openiddict.com/ , https://github.com/openiddict/openiddict-core/issues?q=MCP
