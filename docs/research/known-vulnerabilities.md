# Known vulnerabilities in the IdentityServer4 4.1.2 code and its pinned dependencies

Research for issue #3. Sources read on 2026-11-13. Code read at commit `5e40bd18` (`main`), `src/` only.

## Summary

- **One published advisory is confirmed present in the code:** the open redirect CVE-2024-39694 (Moderate, CVSS 4.7). Every method the advisory names exists unchanged, and all of them rest on one hand-written check, `StringsExtensions.IsLocalUrl`. The advisory lists no fixed IdentityServer4 version.
- **One CVE could not be confirmed:** CVE-2026-4349 (Medium, CVSS 5.6 from the reporting CNA, not analysed by NVD) claims improper authentication through `id_token_hint` at `/connect/authorize`. The code never reads `id_token_hint` on that endpoint. The only source with technical detail (VulDB) refused automated access, so the claim itself is unverified.
- **Six other advisories were checked and do not apply:** four are about features this code does not have (pushed authorization requests, DPoP on local APIs, two for a separate BFF product), one was fixed in 2018 and the fix is in the history, and one is disputed and concerns a file that is not in `src/`. The 4.1.2 release itself was an HTML-encoding fix with no advisory; it is present.
- **Dependencies of the five libraries: four advisories match the resolved versions** (1 Critical, 2 High, 1 Moderate):
  - Critical: CVE-2021-26701 in `System.Text.Encodings.Web` 4.7.0, pulled in by `IdentityModel` 4.4.0 into all five libraries.
  - High: CVE-2024-21907 in `Newtonsoft.Json` 12.0.2 and 11.0.2.
  - High: CVE-2026-32933 in `AutoMapper` 10.0.0. **The only patched versions are under AutoMapper's post-MIT licence.**
  - Moderate: CVE-2024-21319 in `System.IdentityModel.Tokens.Jwt` 5.6.0 and `Microsoft.IdentityModel.JsonWebTokens` 5.6.0.
- **Host, test and migration projects only: five more** (4 High, 1 Moderate), in `Microsoft.Data.SqlClient`, the bundled SQLite native library, `System.Net.Http` 4.3.0 and `System.Text.RegularExpressions` 4.3.0.
- **The .NET Core 3.1 shared framework is the largest exposure by count.** It has been out of support since 13 December 2022. The Advisory Database lists 19 advisories with an explicit 3.1 range (13 High, 6 Moderate), all fixed by runtime 3.1.29 or earlier, and 2 more High advisories whose range includes the final 3.1.32 with no 3.1 fix. Roughly a hundred later runtime advisories name only supported release lines, so their effect on 3.1 is simply not assessed.
- **No advisory exists** for `IdentityModel` 4.4.0, `Microsoft.IdentityModel.Protocols.OpenIdConnect` 5.6.0, `Microsoft.IdentityModel.Tokens` 5.6.0, `System.Security.Principal.Windows` 4.7.0, or any `Microsoft.EntityFrameworkCore.*`, `Microsoft.Extensions.*` or `Microsoft.AspNetCore.*` 3.1.0 package at the resolved version.

"Confirmed" in this note means two different things, kept apart below. For IdentityServer4's own code it means the affected methods were read in `src/`. For a dependency it means the resolved version is inside the advisory's range; whether the code in `src/` can actually reach the flaw is stated separately for each one, and in most cases was not proven either way.

## Sources and method

| Short name | What it is | URL |
|---|---|---|
| GHSA | GitHub Advisory Database, queried per NuGet package through the GraphQL `securityVulnerabilities` API and per advisory through the REST `advisories` API | https://github.com/advisories |
| NuGet audit | `dotnet list package --vulnerable --include-transitive` (SDK 10.0.403) against nuget.org, which serves GHSA data | https://learn.microsoft.com/nuget/concepts/auditing-packages |
| NVD | NVD CVE API 2.0, keyword searches `IdentityServer`, `IdentityServer4`, `Duende` | https://nvd.nist.gov/ |
| CVE.org | CVE record API, for the CNA's own record | https://www.cve.org/ |
| OSV | OSV query API, as a cross-check on GHSA | https://osv.dev/ |
| IdentityServer4 advisories | Security advisories published on the archived IdentityServer4 repository (1 advisory) | https://github.com/DuendeArchive/IdentityServer4/security/advisories |
| Duende advisories | Security advisories published on the Duende repository (5 advisories). Advisory text only | https://github.com/DuendeSoftware/products/security/advisories |
| IdentityServer4 releases | Release notes of the archived project | https://github.com/DuendeArchive/IdentityServer4/releases |

Method, in the order it was done:

1. **Advisories for the code.** GHSA was queried for the five `IdentityServer4*` package ids, the five `Duende.IdentityServer*` ids, `IdentityModel` and `Duende.IdentityModel`. NVD was searched by keyword. The advisory lists on both repositories were read. Each advisory's affected methods or feature were then looked up in `src/`.
2. **Advisories for dependencies.** `git archive HEAD src` was extracted to a scratch directory outside the repository, without `global.json` and with nuget.org as the only package source, and all five solutions were restored. That produced 17 `project.assets.json` files, 240 package/version pairs and 202 distinct package ids. NuGet audit was run on every solution, and GHSA was independently queried for all 202 ids. The two agree: every advisory whose range contains a resolved version is listed in Part 2, and nothing else matched.
3. **Shared framework.** The .NET Core 3.1 runtime is not a NuGet dependency of these projects, so NuGet audit does not see it. GHSA was queried for the runtime pack ids `Microsoft.AspNetCore.App.Runtime.win-x64` and `Microsoft.NETCore.App.Runtime.win-x64` (the `linux-x64` packs give the same 3.1 list) and the ranges were tested against 3.1.0 and 3.1.32.

Constraint observed: Duende IdentityServer is commercially licensed. Nothing below comes from Duende source code or fix commits. Advisories link to fix commits; those links were not opened. Where an advisory states its own workaround, that statement is quoted as advisory text.

One structural fact matters for reading Part 2. The sibling libraries reference each other as **packages**, not projects: `IdentityServer4.EntityFramework` and `IdentityServer4.AspNetIdentity` have `<PackageReference Include="IdentityServer4" />` at version `4.1.2-*` (`src/Directory.Build.targets`). The repository's `NuGet.config` adds a local `./nuget` folder as a source, which the build scripts fill. In a clean restore without that folder, these two libraries compile against the published `IdentityServer4` 4.1.2 package from nuget.org, and NuGet audit flags it. A fix made in `src/IdentityServer4` does not reach them until that reference changes.

## Part 1. Advisories against the IdentityServer4 code

### 1.1 Confirmed present in the code

#### CVE-2024-39694: open redirect through return URL validation

| | |
|---|---|
| Ids | CVE-2024-39694, GHSA-ff4q-64jc-gx98 (Duende repository), GHSA-55p7-v223-x366 (the same advisory republished on the IdentityServer4 repository, no CVE of its own) |
| Severity | Moderate, CVSS 3.1 4.7 `AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:N/A:N` |
| Weakness | CWE-601, URL redirection to untrusted site |
| Affected | `IdentityServer4` `<= 4.1.2`, no patched version. `Duende.IdentityServer` 6.0 to 7.0 lines, patched in 6.0.5, 6.1.8, 6.2.5, 6.3.10, 7.0.6 |
| Sources | https://github.com/advisories/GHSA-ff4q-64jc-gx98 , https://github.com/advisories/GHSA-55p7-v223-x366 , https://nvd.nist.gov/vuln/detail/CVE-2024-39694 |

**What the advisory says.** "It is possible for an attacker to craft malicious Urls that certain functions in IdentityServer will incorrectly treat as local and trusted. If such a Url is returned as a redirect, some browsers will follow it to a third-party, untrusted site." It adds that the flaw alone does not expose credentials, codes or tokens, but can support phishing. It names the affected methods:

- `DefaultIdentityServerInteractionService.GetAuthorizationContextAsync` (may return non-null) and `IsValidReturnUrl` (may return true). The advisory calls UI code that trusts these two "the most commonly used code path", including the default Login, Challenge and Consent pages.
- `ServerUrlExtensions.GetIdentityServerRelativeUrl`, `ReturnUrlParser.ParseAsync`, `OidcReturnUrlParser.ParseAsync`, `ReturnUrlParser.IsValidReturnUrl`, `OidcReturnUrlParser.IsValidReturnUrl`.

The IdentityServer4 copy of the advisory says "IdentityServer4 is no longer supported and will not be receiving updates." The only workaround the advisory states is: "use `IUrlHelper.IsLocalUrl` from ASP.NET Core 5.0 or later to validate return Urls in user interface code in the IdentityServer host."

**Where it lives in `src/`.** All named methods exist and all funnel into one check.

| Advisory name | In this code |
|---|---|
| the local-URL test itself | `src/IdentityServer4/src/Extensions/StringsExtensions.cs:147`, `IsLocalUrl(this string url)` |
| `OidcReturnUrlParser.IsValidReturnUrl`, `ParseAsync` | `src/IdentityServer4/src/Services/Default/OidcReturnUrlParser.cs:60` and `:35`; `IsValidReturnUrl` calls `returnUrl.IsLocalUrl()` at `:62` |
| `ReturnUrlParser.ParseAsync`, `IsValidReturnUrl` | `src/IdentityServer4/src/Services/Default/ReturnUrlParser.cs:32` and `:53`; delegates to the registered parsers |
| `DefaultIdentityServerInteractionService.GetAuthorizationContextAsync`, `IsValidReturnUrl` | `src/IdentityServer4/src/Services/Default/DefaultIdentityServerInteractionService.cs:52` and `:147`; delegates to `ReturnUrlParser` |
| `GetIdentityServerRelativeUrl` | `src/IdentityServer4/src/Extensions/HttpContextExtensions.cs:117` (the advisory calls the class `ServerUrlExtensions`; in 4.1.2 the method is on `HttpContextExtensions`) |

`IsLocalUrl` decides by inspecting only the first two or three characters of the string (a leading `/` not followed by `/` or `\`, or the `~/` equivalent). It does not examine the rest of the value. That is the whole of the validation the named methods rely on before treating a return URL as safe.

Other callers of the same check inside the library: `ConfigureInternalCookieOptions.cs:59`, `HttpResponseExtensions.cs:83` (`RedirectToAbsoluteUrl`), `EndSessionResult.cs:85`, `LoginPageResult.cs:78`, `ConsentPageResult.cs:78`, `CustomRedirectResult.cs:72`, `DeviceAuthorizationResponseGenerator.cs:111`. In those the input is a configured URL (`UserInteraction.LoginUrl` and similar), not a request value, so they are a different risk class, but they share the weak check.

The three host projects under `src/*/host/Quickstart/Account/` are the UI code path the advisory describes. `AccountController` redirects to `model.ReturnUrl` whenever `GetAuthorizationContextAsync` returned non-null, with the comment "we can trust model.ReturnUrl since GetAuthorizationContextAsync returned non-null", and otherwise falls back to `Url.IsLocalUrl`. Because the hosts target `netcoreapp3.1`, that `Url.IsLocalUrl` is the ASP.NET Core 3.1 implementation, which is older than the 5.0 one the advisory recommends.

**Verdict: present.** Moving to `net10.0` changes the framework's `IUrlHelper.IsLocalUrl` but does not touch the library's own `IsLocalUrl` copy. That copy has to be replaced on purpose.

### 1.2 Advisory exists, could not confirm it applies

#### CVE-2026-4349: improper authentication via `id_token_hint` at `/connect/authorize`

| | |
|---|---|
| Ids | CVE-2026-4349, GHSA-46qm-5mvh-mmxr (unreviewed, mapped to no package), VulDB VDB-351380 |
| Severity | Medium. CVSS 3.1 5.6 `AV:N/AC:H/PR:N/UI:N/S:U/C:L/I:L/A:L` and CVSS 4.0 6.3, both assigned by the CNA (VulDB). NVD status is "Deferred": it has published no analysis or score of its own |
| Weakness | CWE-287, improper authentication |
| Affected | "Duende IdentityServer4 up to 4.1.2"; the CVE.org record lists 4.1.0, 4.1.1 and 4.1.2. Tagged `unsupported-when-assigned` |
| Sources | https://nvd.nist.gov/vuln/detail/CVE-2026-4349 , https://www.cve.org/CVERecord?id=CVE-2026-4349 , https://github.com/advisories/GHSA-46qm-5mvh-mmxr |

**What the record says.** "The affected element is an unknown function of the file /connect/authorize of the component Token Renewal Endpoint. This manipulation of the argument id_token_hint causes improper authentication. It is possible to initiate the attack remotely. The attack is considered to have high complexity. The exploitability is described as difficult." The GHSA copy adds that the vendor "was contacted early about this disclosure but did not respond in any way." The record's third reference is titled "DuendeSoftware Identity Server 4 Authentication Bypass Issues". No fix, workaround or affected function is given.

**What the code does.** The authorize endpoint does not process `id_token_hint` at all.

- The only occurrence of the authorize-request `id_token_hint` constant in `src/IdentityServer4/src` is `Configuration/DependencyInjection/Options/LoggingOptions.cs:34`, where it is listed so the value is scrubbed from logs.
- `Validation/Default/AuthorizeRequestValidator.cs` reads `login_hint` (`:756`) but never `id_token_hint`. `ResponseHandling/Default/AuthorizeInteractionResponseGenerator.cs` does not read it either.
- The parameter is validated in one place only, the end session endpoint: `Validation/Default/EndSessionRequestValidator.cs:115-131` validates the token's signature (lifetime deliberately not checked) and compares its `sub` with the signed-in user.

So at `/connect/authorize` the user is identified by the session cookie alone and the hint is ignored. OpenID Connect Core 3.1.2.1 says that if the End-User identified by the hint is not logged in, the server "MUST return an error, such as login_required" (https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest). Ignoring the hint therefore falls short of the specification: a `prompt=none` request carrying a hint for one user succeeds for whichever user holds the session. Whether that is what the CVE describes, and whether it amounts to an authentication bypass rather than a conformance gap, **could not be determined**. This paragraph is an inference from the code and the specification, not from the CVE.

**Why it is unverified.** The CVE text names no function and no mechanism. The VulDB entry and the submission behind it are the only places with detail, and both returned HTTP 403 to automated requests (`https://vuldb.com/?id.351380`, `https://vuldb.com/?submit.772071`). GitHub has not reviewed the advisory and NVD has not analysed it. No IdentityServer4 or Duende advisory corresponds to it.

**Verdict: not confirmed.** Because the GHSA entry maps to no package, NuGet audit will never flag it. Someone should read the VulDB entry in a browser before the claim is accepted or dismissed.

### 1.3 Advisory exists, does not apply

| Id | Severity | What it is | Why it does not apply | Source |
|---|---|---|---|---|
| GHSA-mxv6-xwqj-ww2p (no CVE yet) | High, 8.2 | "Insufficient validation of pushed authorization requests in IdentityServer": pushed requests not bound to the authenticated client, plus two weaknesses that need `AllowUnregisteredPushedRedirectUris`. Affects `Duende.IdentityServer` 7.0.0 and later | No pushed authorization request endpoint exists in `src/` (no match for `PushedAuthorization`). Published 28 September 2026 on the Duende repository only; it is not yet in the global Advisory Database or OSV | https://github.com/DuendeSoftware/products/security/advisories/GHSA-mxv6-xwqj-ww2p |
| CVE-2024-49755, GHSA-v9xq-2mvm-x8xc | Low, 3.1 | Insufficient validation of the DPoP `cnf` claim in the local API authentication handler. Affects `Duende.IdentityServer` 7.0.0 to 7.0.7 | The advisory states "Version 6.3 and below are unaffected, as they do not support DPoP in Local APIs". `LocalApiAuthenticationHandler` exists (`src/IdentityServer4/src/Hosting/LocalApiAuthentication/`) but there is no DPoP code anywhere in `src/` | https://github.com/advisories/GHSA-v9xq-2mvm-x8xc |
| GHSA-vvg7-p7jw-8qr3, GHSA-4j63-5v5f-xcc4 (no CVEs yet) | Moderate, 6.8 and 6.5 | Session cookie forwarding and a missing anti-forgery check in `Duende.BFF.Yarp` | Different product. Nothing in `src/` descends from or resembles it | https://github.com/DuendeSoftware/products/security/advisories |
| CVE-2018-8899 | Medium, 6.1 | Redirect URI not encoded on the authorization response page (XSS). Affects IdentityServer4 1.x before 1.5.3 and 2.x before 2.1.3 | Fixed before 4.x. The fix commit `21d0da22` ("encode redirect uri in authorize response", 2018-03-20) is an ancestor of `HEAD`, and `Endpoints/Results/AuthorizeResult.cs:169` encodes the URL | https://nvd.nist.gov/vuln/detail/CVE-2018-8899 |
| CVE-2019-12250 | Medium, 6.1, **disputed** | Stored XSS via `host/Extensions/RequestLoggerMiddleware.cs`, IdentityServer4 through 2.4. The maintainer disputed it because the logger "is not part of IdentityServer but only our development test host" | No `RequestLoggerMiddleware` file exists anywhere under `src/` | https://nvd.nist.gov/vuln/detail/CVE-2019-12250 |
| IdentityServer4 4.1.2 release (no advisory) | none assigned | The only code change in 4.1.2 was "HtmlEncode iframe URLs in EndSessionCallbackResult" (#5184). The release notes say only "minor bug fixes" | Already fixed: tag `4.1.2` is that commit (`997a6cdd`), and `Endpoints/Results/EndSessionCallbackResult.cs:83` encodes each iframe URL | https://github.com/DuendeArchive/IdentityServer4/releases |

Also returned by the NVD keyword searches and unrelated to this code: CVE-2017-12677 (IdentityServer3), CVE-2021-28290 (Skoruba IdentityServer4.Admin), CVE-2024-7211 (a third-party product reporting the same open redirect as CVE-2024-39694), CVE-2026-31016 (Squidex), CVE-2024-51987 and CVE-2025-26620 (`Duende.AccessTokenManagement`).

The IdentityServer4 release notes from 4.0 to 4.1.2 contain no item labelled as a security fix. The repository's `SECURITY.MD` only gives a reporting address.

## Part 2. Dependencies

### 2.1 Confirmed: vulnerable version resolved by the five libraries

"Libraries" lists which of the five shipping projects resolve the package. "Reach" is the assessment of whether code in `src/` can get to the flaw.

#### CVE-2021-26701: `System.Text.Encodings.Web` 4.7.0 (Critical, 9.8)

| | |
|---|---|
| Ids | CVE-2021-26701, GHSA-ghhp-997w-qr28 |
| Range | `>= 4.6.0, < 4.7.2`, patched in 4.7.2 (also 4.0.0 to 4.5.0 and 5.0.0) |
| Libraries | All five. Not pinned in `src/Directory.Build.targets`; it arrives transitively because `IdentityModel` 4.4.0 depends on exactly 4.7.0 |
| Sources | https://github.com/advisories/GHSA-ghhp-997w-qr28 , https://github.com/dotnet/announcements/issues/178 |

**Weakness.** "A remote code execution vulnerability exists in .NET 5 and .NET Core due to how text encoding is performed." Microsoft "has not identified any mitigating factors". Affected: any application "that uses the System.Text.Encodings.Web package with a vulnerable version". The announcement gives runtime 3.1.13 as the fixed .NET Core 3.1 runtime.

**Reach.** The library calls the encoders from this assembly on request-derived values: `HtmlEncoder.Default.Encode` in `Endpoints/Results/AuthorizeResult.cs:169` and `EndSessionCallbackResult.cs:83`, and `System.Text.Json` serialisation elsewhere. Which copy of the assembly actually loads (the 4.7.0 package asset or the one inside the installed shared framework) depends on the deployment and was **not verified**. Treat as exposed until the package is at 4.7.2 or later, or gone.

#### CVE-2024-21907: `Newtonsoft.Json` 12.0.2 and 11.0.2 (High, 7.5)

| | |
|---|---|
| Ids | CVE-2024-21907, GHSA-5crp-9r3c-p9vr (GHSA-8rfx-6mr3-5jh3 is a withdrawn duplicate) |
| Range | `< 13.0.1`, patched in 13.0.1 |
| Libraries | 12.0.2 in IdentityServer4 (direct reference), EntityFramework and AspNetIdentity. 11.0.2 in Storage and EntityFramework.Storage, through `IdentityModel` 4.4.0 |
| Source | https://github.com/advisories/GHSA-5crp-9r3c-p9vr |

**Weakness.** CWE-755. "Improper handling of expressions with high nesting level that lead to StackOverFlow exception or high CPU and RAM usage", resulting in denial of service. The advisory distinguishes deserialising (CPU and memory burn, needs very large input) from serialising a deeply nested value (stack overflow, which ends the process). Its stated mitigations are updating to 13.0.1 or setting `MaxDepth` in `JsonSerializerSettings`.

**Reach.** Paths exist; the triggering condition was not tested.

- JWTs received from the network (client assertions, request objects, `id_token_hint` at end session, bearer tokens at userinfo and introspection) are parsed by `System.IdentityModel.Tokens.Jwt` 5.6.0, which uses Newtonsoft.Json for the payload.
- `Validation/Default/JwtRequestValidator.cs:193-216` (`ProcessPayloadAsync`) turns nested JSON values from a request object back into strings with `JObject.ToString` and `JArray.ToString`, which is a serialising path. It runs after the request object's signature has been validated.
- Input size is capped by `InputLengthRestrictions.Jwt`, default 51200 characters (`Configuration/DependencyInjection/Options/InputLengthRestrictions.cs:107`), checked in `AuthorizeRequestValidator.cs:180`, `JwtBearerClientAssertionSecretParser.cs:73` and `TokenValidator.cs:67` and `:134`. Whether that cap keeps nesting below the depth the advisory describes was **not measured**.
- `Storage/src/Stores/Serialization/PersistentGrantSerializer.cs:46` deserialises persisted grants with no `MaxDepth`. That data is written by the server itself, so it is a lower-risk path.
- No `MaxDepth` is set anywhere in the five libraries.

#### CVE-2026-32933: `AutoMapper` 10.0.0 (High, 7.5)

| | |
|---|---|
| Ids | CVE-2026-32933, GHSA-rvv3-g6hj-g44x |
| Range | `< 15.1.1` and `>= 16.0.0, < 16.1.1`; patched in 15.1.1 and 16.1.1 |
| Libraries | EntityFramework.Storage (direct, pinned `[10.0.0,11.0)`), and EntityFramework through it |
| Source | https://github.com/advisories/GHSA-rvv3-g6hj-g44x |

**Weakness.** CWE-674, uncontrolled recursion. "When mapping deeply nested object graphs, the library uses recursive method calls without enforcing a default maximum depth limit", so a sufficiently deep graph exhausts the stack and terminates the process.

**Reach.** Low on the evidence in `src/`, not proven absent. AutoMapper is used only in `src/EntityFramework.Storage/src/Mappers/` to convert between EF entities and the configuration and grant models (`Client`, `ApiResource`, `ApiScope`, `IdentityResource`, `PersistedGrant`, and their child collections mapped to strings, secrets and key/value pairs). None of the destination types declared in the five profiles nests inside itself, and the data comes from the configuration and operational stores, not from a request body. No profile sets `MaxDepth`. The entity types do carry back-references to their parent (for example a client scope row to its client); how AutoMapper 10 walks those was not traced.

**Licence consequence, verified separately.** There is no patched release in the MIT-licensed line. nuget.org shows `AutoMapper` 14.0.0 as MIT and flagged with this advisory (https://www.nuget.org/packages/AutoMapper/14.0.0). The 15.0.0 release notes list "Requiring license key" and "Moving from MIT license to dual commercial/OSS license" (https://github.com/LuckyPennySoftware/AutoMapper/releases/tag/v15.0.0), and the licence page for 15.1.3 is the Reciprocal Public License 1.5 with a commercial alternative (https://www.nuget.org/packages/AutoMapper/15.1.3/License). nuget.org also marks 15.1.1 itself as deprecated in favour of 15.1.3 or later. Clearing this advisory by upgrading therefore means accepting that licence; the other route is to stop using AutoMapper. See the open questions.

#### CVE-2024-21319: `System.IdentityModel.Tokens.Jwt` 5.6.0 and `Microsoft.IdentityModel.JsonWebTokens` 5.6.0 (Moderate, 6.8)

| | |
|---|---|
| Ids | CVE-2024-21319, GHSA-59j7-ghrg-fj52 (GHSA-8g9c-28fc-mcx2 is the library's own advisory, withdrawn from the global database as a duplicate) |
| Range | `< 5.7.0`, patched in 5.7.0 (also `>= 6.5.0, < 6.34.0` and `>= 7.0.0-preview, < 7.1.2`) |
| Libraries | IdentityServer4, EntityFramework, AspNetIdentity. Pulled in by `Microsoft.IdentityModel.Protocols.OpenIdConnect` 5.6.0; `System.IdentityModel.Tokens.Jwt` is also pinned `[5.6.0,6.0)` for the integration tests |
| Sources | https://github.com/advisories/GHSA-59j7-ghrg-fj52 , https://github.com/AzureAD/azure-activedirectory-identitymodel-extensions-for-dotnet/security/advisories/GHSA-8g9c-28fc-mcx2 |

**Weakness.** CWE-400. The GHSA text describes "an unauthenticated client" able "to consume arbitrarily large amounts of server memory". The library's own advisory is more specific: "a malicious JSON Web Encryption (JWE) token with a high compression ratio" causes "excessive memory allocation and processing time during decompression", and the attacker "must have access to the public encrypt key registered with the IDP". It states no workaround: "users must upgrade".

**Reach.** Not confirmed. The code hands network-supplied JWTs to `JwtSecurityTokenHandler` in `TokenValidator`, `JwtRequestValidator` and `PrivateKeyJwtSecretValidator`, but no `TokenDecryptionKey` is configured anywhere in `src/IdentityServer4/src`, so the library has no key with which to decrypt a JWE. Whether 5.6.0 can reach decompression without a decryption key was **not verified**, since that would mean reading the Microsoft library's source at that version.

### 2.2 Confirmed for host, test and migration projects only

None of these is resolved by any of the five libraries. They matter for running the tests and sample hosts, and for what a consumer who adds a SQL Server or SQLite provider inherits.

| Id | Package and resolved version | Severity | Weakness, from the advisory | Comes from | Source |
|---|---|---|---|---|---|
| CVE-2024-0056, GHSA-98g6-xh36-x2p7 | `Microsoft.Data.SqlClient` 1.0.19269.1 (range `< 2.1.7`) | High, 8.7 | CWE-319. "SQL Data Provider Security Feature Bypass" | `Microsoft.EntityFrameworkCore.SqlServer` 3.1.0 | https://github.com/advisories/GHSA-98g6-xh36-x2p7 |
| CVE-2022-41064, GHSA-8g2p-5pqh-5jmc | `Microsoft.Data.SqlClient` 1.0.19269.1 (range `<= 1.1.3`) | Moderate, 5.8 | "a timeout occurring under high load can cause incorrect data to be returned as the result of an asynchronously executed query" | same | https://github.com/advisories/GHSA-8g2p-5pqh-5jmc |
| CVE-2025-6965, GHSA-2m69-gcr7-jv3q | `SQLitePCLRaw.lib.e_sqlite3` 2.0.2 (range `<= 2.1.11`, **no patched version**) | High (GHSA label), CVSS 9.8 | CWE-197. Bundled SQLite before 3.50.2: "the number of aggregate terms could exceed the number of columns available", leading to memory corruption | `Microsoft.EntityFrameworkCore.Sqlite` 3.1.0 via `SQLitePCLRaw.bundle_e_sqlite3` | https://github.com/advisories/GHSA-2m69-gcr7-jv3q |
| CVE-2018-8292, GHSA-7jgj-8wvc-jh57 | `System.Net.Http` 4.3.0 (range `< 4.3.4`) | High, 7.5 | CWE-200. "authentication information is inadvertently exposed in a redirect" | `NETStandard.Library` 1.6.1, wanted by xunit 2.4.1 and `Microsoft.TestPlatform.ObjectModel` 16.2.0. Test projects only | https://github.com/advisories/GHSA-7jgj-8wvc-jh57 |
| CVE-2019-0820, GHSA-cmhx-cq75-c4mj | `System.Text.RegularExpressions` 4.3.0 (range `>= 4.3.0, < 4.3.1`) | High, 7.5 | CWE-400, CWE-1333. Regular expression denial of service | `NETStandard.Library` 1.6.1 in tests; `Microsoft.Data.SqlClient` → `Microsoft.Identity.Client` 3.0.8 in hosts | https://github.com/advisories/GHSA-cmhx-cq75-c4mj |

The host, test and migration projects also resolve the `IdentityServer4` 4.1.2 package and with it everything in 2.1.

### 2.3 The .NET Core 3.1 shared framework

The three ASP.NET Core libraries target `netcoreapp3.1` with `<FrameworkReference Include="Microsoft.AspNetCore.App" />`, and `global.json` pins SDK 3.1.402. `FrameworkVersion`, `ExtensionsVersion` and `EntityFrameworkVersion` in `src/Directory.Build.targets` are all `3.1.0`. The runtime that actually executes is whichever 3.1.x is installed; the last one is 3.1.32.

- .NET Core 3.1 was released on 3 December 2019 and went out of support on 13 December 2022. Microsoft's policy page: "End of life refers to the date when Microsoft no longer provides fixes, updates, or online technical assistance." (https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core)
- **Explicit 3.1 ranges, all fixed within the 3.1 line** (the highest fix is 3.1.29). A deployment on 3.1.32 has these; one on 3.1.0 has none of them.

  | Runtime pack | Advisories | CVEs |
  |---|---|---|
  | ASP.NET Core (`Microsoft.AspNetCore.App.Runtime.*`) | 12 (10 High, 2 Moderate) | CVE-2020-0602, CVE-2020-0603, CVE-2020-1045, CVE-2020-1161, CVE-2020-1597, CVE-2021-1723, CVE-2022-23267, CVE-2022-24464, CVE-2022-29117, CVE-2022-29145, CVE-2022-34716, CVE-2022-38013 |
  | .NET runtime (`Microsoft.NETCore.App.Runtime.*`) | 7 (3 High, 4 Moderate) | CVE-2020-1108, CVE-2020-1147, CVE-2020-8927, CVE-2021-1721, CVE-2021-26423, CVE-2021-34485, CVE-2022-24512 |

- **Ranges with no lower bound, which include 3.1.32 and have no 3.1 fix:**

  | Id | Severity | Weakness, from the advisory | Relevance |
  |---|---|---|---|
  | CVE-2023-33170, GHSA-25c8-p796-jg6r (range `< 6.0.20`) | High, 8.1 | CWE-362. "account lockout maximum failed attempts may not be immediately updated, allowing an attacker to try more passwords". The advisory says it affects "ASP.NET Core 2.1 and above" | ASP.NET Core Identity sign-in, which `IdentityServer4.AspNetIdentity` builds on. https://github.com/advisories/GHSA-25c8-p796-jg6r |
  | CVE-2024-21386, GHSA-g74q-5xw3-j7q9 (range `<= 6.0.26`) | High, 7.5 | CWE-400. Denial of service in "ASP.NET applications using SignalR" | The advisory text names only 6.0, 7.0 and 8.0, and nothing in `src/` uses SignalR. Listed because the range matches. https://github.com/advisories/GHSA-g74q-5xw3-j7q9 |

- **Everything later.** The two win-x64 runtime packs carry 117 and 130 advisory entries in total, the newest dated 11 November 2026. Apart from the rows above, each names only release lines that were supported when it was published (6.0 and later). Their absence from the 3.1 list reflects that 3.1 was no longer assessed, not that 3.1 is unaffected. This is an inference from the support policy; the database does not say it.

Source for the lists: GHSA `securityVulnerabilities` for `Microsoft.AspNetCore.App.Runtime.win-x64` and `Microsoft.NETCore.App.Runtime.win-x64`, e.g. https://github.com/advisories?query=Microsoft.AspNetCore.App.Runtime.win-x64 . Only these two pack ids (and the `linux-x64` equivalents, which gave the same 3.1 rows) were queried, so a 3.1 advisory filed only against another pack id would be missed.

Moving to `net10.0` retires this whole section, which is the planned direction anyway.

### 2.4 Pinned packages with no matching advisory

Checked in GHSA across all published ranges; nothing matches the resolved version.

| Package (resolved) | Result |
|---|---|
| `IdentityModel` 4.4.0 | No advisory for the package id in GHSA or OSV. It is the source of the `System.Text.Encodings.Web` 4.7.0 and `Newtonsoft.Json` 11.0.2 dependencies in 2.1 |
| `Microsoft.IdentityModel.Protocols.OpenIdConnect` 5.6.0, `Microsoft.IdentityModel.Protocols` 5.6.0, `Microsoft.IdentityModel.Tokens` 5.6.0, `Microsoft.IdentityModel.Logging` 5.6.0 | No advisory for these ids. The 5.6.0 family is affected through the two sibling packages in CVE-2024-21319 |
| `System.Security.Principal.Windows` 4.7.0 | No advisory |
| `Microsoft.AspNetCore.Authentication.OpenIdConnect` 3.1.0, `Microsoft.AspNetCore.Authentication.Certificate` 3.1.0, `Microsoft.AspNetCore.TestHost` 3.1.0, `Microsoft.AspNetCore.Identity.EntityFrameworkCore` 3.1.0 | No advisory at these versions |
| `Microsoft.EntityFrameworkCore` 3.1.0 and `.Relational`, `.Sqlite`, `.SqlServer`, `.InMemory`, `.Design` | No advisory on the EF Core packages. The SQL Server and SQLite providers bring the native and client libraries in 2.2 |
| `Microsoft.Extensions.*` 3.1.0 | No match. `Microsoft.Extensions.Caching.Memory` has CVE-2024-43483 (High) for the 6.0, 8.0 and 9.0 lines only; 3.1 is not listed, with the same caveat as 2.3 |
| `System.Text.Json` 4.7.0, `System.IO.Pipelines` 4.7.0, `System.Security.Cryptography.X509Certificates` 4.3.0, `Microsoft.Identity.Client` 3.0.8, `Microsoft.Build.Tasks.Git` 1.0.0 | Have advisories for other versions; the resolved version is outside every range |
| Build and test tooling: `MinVer` 2.3.0, `SimpleExec` 6.2.0, `Bullseye` 3.3.0, `Microsoft.SourceLink.GitHub` 1.0.0, `FluentAssertions` 5.10.2, `Microsoft.NET.Test.Sdk` 16.2.0, `xunit` 2.4.1, Serilog packages | No advisory |

"No advisory" is a statement about the database on the day it was read. These versions are five to seven years old and unsupported, so it is weak evidence of safety.

## Questions this raised that need a decision

1. **AutoMapper.** The CVE-2026-32933 fix exists only in releases under the Reciprocal Public License 1.5 or a commercial licence. Choose between adopting that licence, replacing AutoMapper with hand-written mapping in `EntityFramework.Storage`, or staying on an MIT release and accepting a permanently flagged dependency. The mapping surface is small (five profile files).
2. **CVE-2026-4349.** Someone needs to read the VulDB entry by hand and decide whether it describes a real flaw. Independently of that, decide whether the authorize endpoint should honour `id_token_hint` as OpenID Connect Core requires. Today it ignores it.
3. **How to treat the two unassessed 3.1-era runtime advisories and the unassessed later ones.** If any 3.1 build is ever shipped before the move to `net10.0`, it ships with known, unpatched framework flaws. If nothing ships until `net10.0`, this needs no action.
4. **Scanner identity after the fix.** The open redirect advisory is recorded against the package id `IdentityServer4` at `<= 4.1.2` with no patched version, and the libraries currently reference that published package by id. Under the decision that this product owes IdentityServer4 no compatibility, a new package id means scanners will neither flag nor clear the product on that advisory. Decide whether the fix should be recorded somewhere public (a repository security advisory, release notes) so the status is discoverable.

## What could not be verified

- **CVE-2026-4349's actual claim.** VulDB returned HTTP 403 to automated requests. Only the one-paragraph CVE description was read.
- **Reachability of the dependency flaws.** For CVE-2024-21907 the nesting depth reachable under the 51200-character JWT cap was not measured. For CVE-2024-21319 the behaviour of version 5.6.0 without a decryption key was not checked. For CVE-2021-26701 which copy of the assembly loads at run time was not checked. Nothing was executed against the code; these are readings of `src/` only.
- **Completeness of the Duende advisory list.** The three advisories published in September and October 2026 are on the repository page but not yet in the global Advisory Database or OSV, and have no CVE ids. Their text was read through a page summariser rather than the API, so the details in 1.3 for GHSA-mxv6-xwqj-ww2p should be re-read at the source before being relied on. Advisories Duende may have communicated only to customers are not visible at all.
- **Flaws fixed in Duende IdentityServer without an advisory.** Out of reach by design: release notes and fix commits were not consulted, so any inherited weakness that was repaired quietly is not in this note.
- **Runtime advisories against other pack ids** than the four queried.
- **`samples/` and `docs/`** were not examined.
