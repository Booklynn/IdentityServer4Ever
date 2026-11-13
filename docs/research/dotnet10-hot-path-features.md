# .NET 10 runtime and library features for the hot paths

Research note for issue #7. Written 2026-11-13 against `main` at `5e40bd18` (`netcoreapp3.1`, IdentityServer4 4.1.2 code).
Scope is `src/` only: `IdentityServer4`, `Storage`, `EntityFramework.Storage`, `EntityFramework`, `AspNetIdentity`.

**How to read this note.** Nothing here was benchmarked. Every "expected benefit" is a hypothesis to be confirmed or rejected against a benchmark baseline before any change is kept. "Since" is the first .NET, ASP.NET Core or EF Core version that documents the API, taken from the version list on the Microsoft Learn API page. File references are `path:line` at the commit above.

## Summary

The code predates almost every allocation-reducing API added since .NET Core 3.1, but the largest costs on the hot paths are structural, and most of them do not need a new API to fix:

- **Discovery and JWKS are rebuilt on every request**: a resource-store read, an RSA or ECDSA parameter export per key, then reflection-based JSON to a string. Nothing is cached on the server.
- **Logging does work whether or not anyone is listening.** There are 489 logger calls across the five libraries and no `IsEnabled` check on any of them. A successful token request builds a log object and, at the Information level, serializes it as indented JSON.
- **The EF stores make more database round trips than needed**: ten per uncached client lookup, and a read before every grant write and delete.
- **JWTs go through the previous-generation handler** (`JwtSecurityTokenHandler`, pinned below 6.0) with Newtonsoft.Json on the claims path.
- **Grants are serialized with Newtonsoft.Json** on every code, refresh token and reference token operation.

The newer primitives (`SHA256.HashData`, `Base64Url`, `RandomNumberGenerator.GetHexString`, span splitting, frozen collections, `SearchValues`) do apply, but each removes a few small allocations per request. They are worth doing for clarity and for timing safety, and are unlikely to move throughput on their own.

Three candidate areas turned out not to apply: there is no `Regex` in `src/`, no sync-over-async on a request path, and locks appear only in the in-memory device flow store and one lazily built static page.

Retargeting alone changes the runtime: dynamic PGO is on by default since .NET 8, DATAS since .NET 9, and .NET 10 adds stack allocation and devirtualization work in the JIT. The benchmark baseline has to be taken after the retarget, or those gains will be credited to code changes.

## Ranked candidates

Ranked by expected benefit on the named path. All benefit sizes are hypotheses.

| # | Candidate | Since | Where it applies | Expected benefit (hypothesis) | Risk |
|---|---|---|---|---|---|
| 0 | Runtime defaults on retarget: dynamic PGO, DATAS, .NET 10 JIT work | .NET 8 / 9 / 10 | Whole process, no code change | Large in aggregate, but not attributable to any change we make | DATAS can lower throughput; measure both settings |
| 1 | Cache the serialized discovery and JWKS documents on the server | Any (output caching is ASP.NET Core 7, `HybridCache` is .NET 9) | `DiscoveryResponseGenerator`, `DiscoveryEndpoint`, `DiscoveryKeyEndpoint` | Large on those two endpoints | Issuer comes from the request host; key rotation staleness |
| 2 | `LoggerMessage` source generation plus explicit `IsEnabled` guards | .NET 6 | All 489 logger calls; first the token and authorize validators | Large on the token endpoint at Information level, moderate otherwise | Must keep the sensitive-value filters |
| 3 | EF Core: single or split no-tracking client query, `ExecuteDelete`/`ExecuteUpdate` for grants and cleanup | EF Core 5 / 7 | `ClientStore`, `PersistedGrantStore`, `TokenCleanupService` | Large for EF-backed deployments on cache miss and on every grant operation | Split-query consistency; cleanup notification contract; relational providers only |
| 4 | `JsonWebTokenHandler` (Microsoft.IdentityModel 7 or later) in place of `JwtSecurityTokenHandler` | IdentityModel 7.0 | `DefaultTokenCreationService`, `TokenValidator`, `JwtRequestValidator`, `PrivateKeyJwtSecretValidator` | Moderate to large on token creation and validation | Token wire format and claim typing can change; security review needed |
| 5 | System.Text.Json source generation for persisted grants | .NET 6 | `PersistentGrantSerializer` and its converters | Moderate on code, refresh and reference token flows | Stored data format; stricter parsing |
| 6 | Write responses as UTF-8 directly with source-generated metadata | ASP.NET Core 5 / .NET 6 | `HttpResponseExtensions.WriteJsonAsync`, all `Endpoints/Results` | Moderate to marginal (small payloads) | `Dictionary<string, object>` values of arbitrary type |
| 7 | `HybridCache` in place of `ICache<T>` over `IMemoryCache` | .NET 9 (preview at release, since generally available) | `DefaultCache`, `CachingClientStore`, `CachingResourceStore`, `CachingCorsPolicyService` | Moderate under concurrent cache misses, near zero on hits | Stale client and secret state; secrets in a distributed cache |
| 8 | One-shot hashing, `Base64Url`, `GetHexString`, `FixedTimeEquals` | .NET 5 / 9 / 8 / Core 2.1 | `HashExtensions`, `CryptoHelper`, `DefaultGrantStore`, secret validators, PKCE check | Marginal in latency, moderate in allocations per request | Timing-safe comparison; entropy of handles |
| 9 | `CancellationToken` through stores and services | Any | Every store and service interface | None on the happy path; protects capacity under overload | Wide public API change |
| 10 | Endpoint routing in place of the custom router middleware | ASP.NET Core 3.0 (minimal APIs 6, short-circuit 8) | `IdentityServerMiddleware`, `EndpointRouter` | Marginal to moderate; mainly an enabler for 1 and 11 | Middleware ordering for auth and CORS |
| 11 | Rate limiting middleware | ASP.NET Core 7 | Token, introspection, revocation, device endpoints | Protective, not a speed-up | Per-instance counters; partition key from untrusted input |
| 12 | Span-based splitting and joining, `SearchValues`, interpolated string handlers | .NET 8 / 9 / 6 | `StringsExtensions`, `BasicAuthenticationSecretParser`, `CachingResourceStore.GetKey` | Marginal | Low |
| 13 | `FrozenSet`/`FrozenDictionary` for constant tables | .NET 8 | `Constants.cs` lookups | Marginal, possibly none (tables hold 2 to 20 items) | Build only from trusted keys |
| 14 | `Environment.ProcessId` | .NET 5 | `DefaultEventService.cs:117` | Marginal; only when events are enabled | None |
| 15 | `TimeProvider` in place of `ISystemClock` | .NET 8 | 26 files | None for speed; removes an obsolete API | Sub-second precision differs |
| 16 | `ValueTask` and pooled async builders | Core 2.x / .NET 6 | `ICache<T>`, secret validators, in-memory stores | Marginal | Misuse (double await) |
| 17 | `ArrayPool` and `stackalloc` buffers for hashing and encoding | Core 2.1 | Same sites as 8 | Marginal | Secrets left in pooled buffers |

Not performance, but found on the same paths and worth carrying into the plan: `X509CertificateLoader` (item 18), JSON duplicate-property rejection (19), `RedirectHttpResult.IsLocalUrl` (20).

## What the hot paths do today

### Hosting and routing

- `Hosting/IdentityServerMiddleware.cs:43-93` runs for every request in the host application, not only protocol requests. Each call resolves four services by method injection (`IEndpointRouter`, `IUserSession`, `IEventService`, `IBackChannelLogoutService`), calls `session.EnsureSessionIdCookieAsync()` (which authenticates the cookie scheme, `Services/Default/DefaultUserSession.cs:203`), and allocates an `OnStarting` closure.
- `Hosting/EndpointRouter.cs:27-46` matches the path by a linear scan over the registered endpoints, then resolves the handler from DI by `Type`.
- Almost every service is registered transient (`Configuration/DependencyInjection/BuilderExtensions/Core.cs:121-258`), so one token request constructs the full validator and generator graph.
- No store or service interface in `src/` takes a `CancellationToken`. The only uses are in `EntityFramework/src/TokenCleanupHost.cs`.
- The only sync-over-async call is at startup (`Configuration/IdentityServerApplicationBuilderExtensions.cs:79`).

### Token endpoint

- `Endpoints/TokenEndpoint.cs:86` copies the parsed form into a `NameValueCollection` (`Extensions/IReadableStringCollectionExtensions.cs:18-27`).
- `Validation/Default/BasicAuthenticationSecretParser.cs:55-147`: `Substring`, `Convert.FromBase64String`, `Encoding.UTF8.GetString`, two more `Substring` calls, then `value.Replace("+", "%20")` and `Uri.UnescapeDataString` for each part.
- `Validation/Default/SecretValidator.cs:45-54` enumerates the secret list three times with LINQ and reads the clock for each secret.
- `Validation/Default/HashedSharedSecretValidator.cs:43-44,66-67` allocates two result tasks up front and computes both SHA-256 and SHA-512 of the presented secret before it knows which one the stored secret needs. Each hash goes through `SHA256.Create()`, `Encoding.UTF8.GetBytes`, `ComputeHash` and `Convert.ToBase64String` (`Extensions/HashExtensions.cs:22-69`).
- Secrets and PKCE values are compared with `IdentityModel.TimeConstantComparer.IsEqual` (`HashedSharedSecretValidator.cs:93,97`, `PlainTextSharedSecretValidator.cs:69`, `TokenRequestValidator.cs:779,786`). That helper, from the IdentityModel 4.4.0 package, copies both strings to `char[]`, returns early on a length mismatch, and branches on each character comparison. [C10]
- `Validation/Default/TokenRequestValidator.cs:814-840`: every `LogError`, `LogWarning`, `LogInformation` and `LogSuccess` first constructs a `TokenRequestValidationLog`, before any level check, and concatenates the message template at run time. `LogSuccess` logs at Information. When a provider formats the entry, the object's `ToString()` runs `LogSerializer.Serialize`, which is reflection-based and indented (`Logging/LogSerializer.cs:15-33`).
- `Validation/Default/TokenValidator.cs:169,240` builds a claims dictionary for the log object on every validation, again regardless of level.
- `Services/Default/DefaultTokenService.cs:128,197,206`: `ToUnixTimeSeconds().ToString()` and a hex `jti` from `CryptoRandom.CreateUniqueId`, which is `BitConverter.ToString(bytes).Replace("-", "")`. [C10]
- `Extensions/TokenExtensions.cs:34-150` builds the JWT payload with a dozen LINQ passes over the claims (`Where`, three chained `Except`, `GroupBy`), and parses JSON-valued claims with Newtonsoft `JRaw.Parse` (line 92).
- `Services/Default/DefaultTokenCreationService.cs:136` creates a new `JwtSecurityTokenHandler` per token. Line 104 computes the certificate hash and Base64Url-encodes it per token.
- `Stores/Default/DefaultGrantStore.cs:79` hashes `value + ":" + GrantType` for every grant read, write and delete. `Storage/src/Stores/Serialization/PersistentGrantSerializer.cs:33-46` serializes grants with Newtonsoft.Json and custom `Claim` and `ClaimsPrincipal` converters.
- `Endpoints/Results/TokenResult.cs:42` goes through `Extensions/HttpResponseExtensions.cs:18-30`: serialize to a `string`, write the string (a UTF-16 to UTF-8 transcode), flush, then flush again in the outer overload.
- `Services/Default/DefaultEventService.cs:117` calls `Process.GetCurrentProcess().Id` for every event and does not dispose the `Process`. Events are off by default.

### Token validation (introspection, userinfo, local API, `id_token_hint`)

- `Validation/Default/TokenValidator.cs:248-255` creates a new `JwtSecurityTokenHandler` and `TokenValidationParameters` per call. Line 416 parses the JWT a second time to read the audience.
- Lines 309-327 copy the claims and split space-delimited scope claims with `string.Split`.
- Validation looks the client up twice per token, each time through the client store: lines 302 and 180 for a JWT access token, 374 and 180 for a reference token, 87 and 302 for an identity token.

### Discovery and JWKS

- `Endpoints/DiscoveryEndpoint.cs` and `Endpoints/DiscoveryKeyEndpoint.cs` call the generator on every request. `DiscoveryOptions.ResponseCacheInterval` only sets a `Cache-Control` header for clients, and defaults to `null`.
- `ResponseHandling/Default/DiscoveryResponseGenerator.cs:95-361` rebuilds a `Dictionary<string, object>` each time. That includes `GetAllEnabledResourcesAsync` (a store read, plus the duplicate-name validation in `Extensions/IResourceStoreExtensions.cs:41-88`), about seventeen LINQ calls and several `ToArray` copies of constant lists.
- `DiscoveryResponseGenerator.cs:367-480` exports key parameters (`rsa.ExportParameters(false)`, `ecdsa.ExportParameters(false)`), computes the certificate hash and Base64-encodes the raw certificate for every key on every JWKS request.
- Both results are serialized by reflection through `WriteJsonAsync(object)`; the JWKS result wraps the keys in an anonymous type (`Endpoints/Results/JsonWebKeysResult.cs:60`).

### Authorize pipeline

- `Endpoints/AuthorizeEndpoint.cs:41,50` copies the query or form into a `NameValueCollection`.
- `Validation/Default/AuthorizeRequestValidator.cs` has 24 LINQ calls. It splits `response_type`, `scope`, `prompt` and `acr_values` with `Split` followed by `Distinct().ToList()` (lines 468, 560, 682, 780), and looks constants up with `List<string>.Contains` (lines 380-437, 519, 683, 720). `ResponseTypeEqualityComparer.cs:38-39,69` splits both operands on every comparison.
- `Validation/Default/JwtRequestValidator.cs:33,208-212` validates request objects with `JwtSecurityTokenHandler` and Newtonsoft `JObject`/`JArray`.
- `Extensions/NameValueCollectionExtensions.cs:36-62,140-164` builds redirect query strings with `StringBuilder` and `UrlEncoder`, then patches `+` to `%20` with `Replace`.
- `Extensions/HttpContextExtensions.cs:135-153` rebuilds the issuer URI by string concatenation on every call. It is called several times per request (token service, token validator, discovery).
- `Extensions/ValidatedAuthorizeRequestExtensions.cs:111-128` and `Models/Messages/ConsentRequest.cs:91-96` hash with `SHA256.Create()`.

### Stores and caching

- `Services/Default/DefaultCache.cs:32-35` builds every cache key as `typeof(T).FullName + ":" + key`. `Extensions/ICacheExtensions.cs:34-60` is get, then load, then set, with no coordination: concurrent misses for the same key all hit the store.
- `Stores/Caching/CachingResourceStore.cs:66-70` builds the key for every lookup with `OrderBy` and `Aggregate` string concatenation.
- Default expiry for the client, resource and CORS caches is 15 minutes (`Configuration/DependencyInjection/Options/CachingOptions.cs:14`). There is no invalidation path.
- `Stores/InMemory/InMemoryClientStore.cs:43-48` and `InMemoryResourcesStore.cs` are linear LINQ scans.

### EF stores

- `EntityFramework.Storage/src/Stores/ClientStore.cs:54-69` runs ten queries per client lookup: one tracked query for the client, then nine `LoadAsync` calls, one per child collection.
- `EntityFramework.Storage/src/Stores/PersistedGrantStore.cs`: `StoreAsync` reads before it writes (line 49). `RemoveAsync` reads, then deletes (105-115). `RemoveAllAsync` loads every matching row, then deletes them one by one (133-142). Each read is `ToArrayAsync()` followed by an in-memory `SingleOrDefault`.
- `EntityFramework.Storage/src/Stores/ResourceStore.cs` uses `AsNoTracking` except for identity resources in `GetAllResourcesAsync` (173-175). Each method filters in memory a second time after the query.
- `EntityFramework.Storage/src/TokenCleanup/TokenCleanupService.cs:77-93,110-126` loads each batch of expired rows, calls `RemoveRange`, then `SaveChanges` (one `DELETE` per row). It reads `DateTime.UtcNow` directly (78, 111).
- All entity-to-model mapping goes through AutoMapper (11 files).
- The contexts are registered with `AddDbContext` (`EntityFramework.Storage/src/Configuration/ServiceCollectionExtensions.cs:48-57,91-101`), and the stores depend on the `IConfigurationDbContext` and `IPersistedGrantDbContext` interfaces rather than a concrete context type.

## Features in detail

### 0. Runtime defaults that arrive with the retarget

- **Dynamic PGO** is enabled by default since .NET 8. Microsoft reports about 15% on average across its benchmark suite. [R1] [R2]
- **Server GC** is the default for ASP.NET Core apps. [R7] **DATAS** was opt-in in .NET 8 and is on by default in .NET 9. It sizes the heap to the live data rather than to the machine, and the documentation says to disable it if throughput drops. [R3] [R6]
- **.NET 10 JIT**: stack allocation of small arrays, escape analysis for delegates and struct fields, and devirtualization of array interface methods, which the performance post notes also speeds up LINQ over arrays. [R4] [R5]
- **ASP.NET Core 10**: the Kestrel memory pool now returns idle memory automatically. [R9]

*Where:* everywhere, with no source change. The many small LINQ chains and closures listed above are the kind of code these JIT changes target, so part of their cost may already be gone after the retarget.

*Benefit (hypothesis):* large in aggregate. *Reasoning:* it applies to all code, including ASP.NET Core and EF Core themselves.

*Risk:* DATAS trades some throughput for memory. Benchmark with DATAS on and off before fixing the baseline.

### 1. Server-side caching of discovery and JWKS

*What:* keep the finished response bytes and reuse them until the inputs change. Three ways to do it:

- A cached `byte[]` built with `JsonSerializer.SerializeToUtf8Bytes` (available since .NET Core 3.0) inside the generator or the result. This needs no new API.
- The **output caching middleware** (ASP.NET Core 7). It caches only 200 responses to GET or HEAD, skips responses that set cookies and responses to authenticated requests, and can serve `304` from an ETag. [H4]
- **`HybridCache`** (item 7) as the store for the bytes.

*Where:* `ResponseHandling/Default/DiscoveryResponseGenerator.cs`, `Endpoints/DiscoveryEndpoint.cs`, `Endpoints/DiscoveryKeyEndpoint.cs`, `Endpoints/Results/DiscoveryDocumentResult.cs`, `Endpoints/Results/JsonWebKeysResult.cs`.

*Benefit (hypothesis):* large on these endpoints. *Reasoning:* each request today does a store read, an asymmetric key export per key and a reflection serialization, and all of it produces the same bytes until configuration or keys change. Relying parties poll both documents.

*Risks:*

- The issuer and every endpoint URL are derived from the request scheme, host and path base unless `IssuerUri` is set (`Extensions/HttpContextExtensions.cs:135-153`). The cache key must include them. A cache keyed too loosely lets one request's `Host` header shape the document served to others.
- A cached JWKS delays the appearance of a new signing key and the disappearance of a retired one. The lifetime has to be tied to the key rotation design.
- The documents vary by `Origin` for CORS today (`SetCache(maxAge, "Origin")`).
- Output caching must run after authentication and authorization, or it can serve content cached for one user to another. [H4] Protocol endpoints are not routed endpoints today, so per-endpoint output-cache policies have nothing to attach to (see item 10).

### 2. `LoggerMessage` source generation and `IsEnabled` guards

*What:* partial methods marked `[LoggerMessage]` get a generated implementation. The documentation says this removes boxing, temporary allocations and run-time template parsing, and recommends it for .NET 6 and later. [L1] [L2] Analyzer rule CA1848 flags the extension-method calls. [L3]

*Where:* all 489 `Log*` calls. In priority order: `TokenRequestValidator.cs:814-840`, `TokenValidator.cs:437-445` and the eager `_log.Claims` assignments at 117, 169 and 240, `AuthorizeRequestValidator.cs:838-844`, `IdentityServerMiddleware.cs:72,78`, `EndpointRouter.cs`, the secret parsers and validators, and `Services/Default/DefaultEventSink.cs:40`.

*Benefit (hypothesis):* large on the token endpoint when Information is enabled for the `IdentityServer4` categories, moderate otherwise. *Reasoning:* the request-log object is built on every request, and at Information it is serialized as indented JSON by reflection, which is likely to cost more than the rest of request validation. The remaining calls each allocate a `params object[]` and box value types even when the level is off.

*Caveat:* source generation removes the cost of the call, not the cost of computing its arguments. The eager `new TokenRequestValidationLog(...)` and `ToClaimsDictionary()` need an explicit `IsEnabled` check around them. That guard needs no new API and is probably the larger half of the gain.

*Risk:* the log objects pass through `TokenRequestSensitiveValuesFilter` and `AuthorizeRequestSensitiveValuesFilter`. A rewrite must keep the scrubbing; tokens are already logged at Trace (`TokenEndpoint.cs` `LogTokens`). The logging documentation makes the same point about redaction. [L1]

### 3. EF Core query and command improvements

*What and since:*

- **Split queries** (`AsSplitQuery`, EF Core 5) load each included collection in its own query and avoid the row multiplication of sibling joins. [E2]
- **No-tracking queries** are "generally quicker to execute", and `AsNoTrackingWithIdentityResolution` exists since EF Core 5. [E4] The ASP.NET Core best-practices page recommends them for read-only access. [A3]
- **`ExecuteDelete` and `ExecuteUpdate`** (EF Core 7) issue one set-based statement without loading entities or using the change tracker. [E1]
- **Compiled queries** (`EF.CompileAsyncQuery`, since EF Core 2.0, so not new) skip the query-cache lookup. [E3]
- **DbContext pooling** (`AddDbContextPool`, not new) reuses context instances. [E3]
- **EF Core 10** changes how `Contains` over a parameter list is translated: one scalar parameter per value by default, where EF Core 8 and 9 sent a JSON array. [E5]

*Where:*

- `ClientStore.cs:54-69`: replace ten round trips with one no-tracking query with includes, split or single. Nine sibling collections is the cartesian-explosion case the split-query page describes, so split is the likely choice.
- `PersistedGrantStore.cs:103-149`: `RemoveAsync` and `RemoveAllAsync` become one `ExecuteDeleteAsync` each.
- `PersistedGrantStore.cs:47-73`: `StoreAsync` reads before writing. A conditional `ExecuteUpdateAsync` followed by an insert when no row was affected would save the read.
- `TokenCleanupService.cs:71-130`: one `ExecuteDeleteAsync` per batch, or one for the whole expired set.
- `ResourceStore.cs:171-199`: add `AsNoTracking` for identity resources.
- `ResourceStore.cs:51-165`: all four finder methods use `names.Contains(x.Name)`, so the EF Core 10 translation change applies to them.

*Benefit (hypothesis):* large for EF-backed deployments. *Reasoning:* database round trips cost far more than any in-process change here. A code exchange touches the grant store several times. An uncached client lookup costs ten queries, and the token validator looks the client up twice per token.

*Risks:*

- Split queries are not atomic. The documentation notes that concurrent updates can produce inconsistent results across the queries. [E2] For a client, that could mean a secret list from one version and a redirect URI list from another.
- `ExecuteDelete` and `ExecuteUpdate` take effect immediately, bypass the change tracker, do not start a transaction on their own, and cannot return the affected rows. [E1] `TokenCleanupService` passes the removed entities to `IOperationalStoreNotification` (lines 91-93, 124-126), so a set-based delete cannot honour that contract as it stands.
- The current code treats `DbUpdateConcurrencyException` as a benign race. With `ExecuteDelete` the signal becomes "zero rows affected". For single-use authorization codes and refresh token rotation, zero rows means "someone else already consumed it" and must not be treated as success by the caller.
- These methods "currently only work with relational database providers". [E1] The test projects run against the EF in-memory provider (`EntityFramework.Storage/test/IntegrationTests/DatabaseProviderBuilder.cs:17`), which Microsoft discourages for testing. [E6]
- Compiled queries take a concrete `DbContext` type as their first parameter and "may only be used against a single EF Core model". [E3] The stores hold interface-typed contexts and the context classes are generic, so compiled queries need a design change first. Their benefit is marginal next to removing round trips.

### 4. `JsonWebTokenHandler`

*What:* the current-generation JWT handler in `Microsoft.IdentityModel.JsonWebTokens`. ASP.NET Core 8 switched its own handlers to it and lists "30% performance improvement" and async processing among the reasons. [A4] IdentityModel 7.0 replaced Newtonsoft.Json with System.Text.Json internally. [A5]

This is a library change, not a runtime feature. It is listed because it sits in the middle of token creation and validation.

*Where:* `Services/Default/DefaultTokenCreationService.cs:71-138`, `Extensions/TokenExtensions.cs:34-150`, `Validation/Default/TokenValidator.cs:246-349,412-426`, `Validation/Default/JwtRequestValidator.cs:33`, `Validation/Default/PrivateKeyJwtSecretValidator.cs:110`, `Validation/Default/JwtBearerClientAssertionSecretParser.cs:110`. The package pin is `System.IdentityModel.Tokens.Jwt` `[5.6.0,6.0)` in `src/Directory.Build.targets`.

*Benefit (hypothesis):* moderate to large on JWT creation and validation, excluding the signature operation itself. *Reasoning:* Microsoft's own figure, plus the removal of the Newtonsoft `JRaw` parsing and of the second JWT parse in `GetClientIdFromJwt`.

*Risks:*

- The IdentityModel 7 notes warn of "subtle differences in serialization", and unknown complex claim values now surface as `JsonElement`. [A5] `TokenExtensions.cs` places Newtonsoft `JToken` objects into the payload, which a System.Text.Json-based writer will not serialize the same way.
- Claim ordering, number formatting and JSON-valued claims (`cnf`, `address`) should be pinned by tests before the switch.
- Validation defaults differ between handler generations and must be compared setting by setting.

### 5. System.Text.Json for persisted grants

*What:* the System.Text.Json source generator (.NET 6) emits type metadata at compile time. The documentation says this improves performance and reduces private memory, and that the optional fast path raises serialization throughput. There is no fast path for deserialization. [J1] [J2]

*Where:* `Storage/src/Stores/Serialization/PersistentGrantSerializer.cs`, `ClaimConverter.cs`, `ClaimsPrincipalConverter.cs`, `CustomContractResolver.cs`, and the grant models (`AuthorizationCode`, `RefreshToken`, `Token`, `Consent`).

*Benefit (hypothesis):* moderate. *Reasoning:* one serialize or deserialize per grant operation, on objects that carry a `ClaimsPrincipal` and a claim list.

*Risks:*

- The JSON written to the `Data` column would change. ADR 0001 removes any duty to read IdentityServer4 data, but grants written by an earlier build of this product would also stop deserializing. For refresh tokens that means every user signs in again.
- System.Text.Json is stricter by default: case-sensitive property names, no comments, no trailing commas. [J3] `CustomContractResolver` currently skips read-only properties on write, and that behaviour needs an explicit equivalent.
- Source generation in .NET 10 and earlier does not support private or protected members or accessors. [J2]

### 6. Direct UTF-8 response writing

*What:*

- `HttpResponse.WriteAsJsonAsync` (ASP.NET Core 5) serializes straight to the response body. [J5]
- `JsonSerializer.SerializeAsync` has `PipeWriter` overloads since .NET 9. [J7]
- `Utf8JsonWriter.WriteRawValue` (.NET 6) can embed a pre-serialized fragment. [J8]
- `JsonSerializerOptions.IgnoreNullValues` is obsolete since .NET 6 (warning SYSLIB0020). The replacement is `DefaultIgnoreCondition`. [J4]

*Where:* `Extensions/HttpResponseExtensions.cs:18-30`, `Infrastructure/ObjectSerializer.cs`, and every result in `Endpoints/Results`: `TokenResult`, `TokenErrorResult`, `IntrospectionResult`, `UserInfoResult`, `DeviceAuthorizationResult`, `BadRequestResult`, `TokenRevocationErrorResult`. Also `Logging/LogSerializer.cs:17` for the obsolete option.

*Benefit (hypothesis):* moderate to marginal. *Reasoning:* it removes one string of the payload size, one transcode and one redundant flush per response. Token responses are a few kilobytes at most.

*Risks:*

- Several payloads are `Dictionary<string, object>`: introspection, userinfo, discovery, and the `[JsonExtensionData]` `Custom` property on token responses. With source generation, the run-time type of every `object` value must be declared. [J1] Custom entries and custom claims are caller-supplied, so a source-generation-only resolver would throw on an unregistered type.
- Native AOT is not a requirement, so the practical shape is source-generated metadata for the fixed DTOs with the reflection resolver kept as a fallback (`TypeInfoResolverChain`, .NET 8). [J9]
- Fast-path serialization is not used for async stream serialization except for small payloads. [J1]

### 7. `HybridCache`

*What:* a cache abstraction over an in-process primary cache and an optional `IDistributedCache` secondary. It guarantees that only one concurrent caller per key runs the factory, supports tags for group invalidation, and ships as the `Microsoft.Extensions.Caching.Hybrid` package. It was a preview at the .NET 9 release and has since become generally available. [H1] [H2] [H3]

*Where:* `Services/ICache.cs`, `Services/Default/DefaultCache.cs`, `Extensions/ICacheExtensions.cs`, and the three caching decorators in `Stores/Caching`.

*Benefit (hypothesis):* moderate under concurrent cache misses, near zero in steady state. *Reasoning:* the current get-then-set extension lets every concurrent request for a cold key reach the store, which with the EF `ClientStore` is ten queries each. A cache hit is already a dictionary lookup.

*Risks:*

- **Stale security state.** A disabled client, a removed secret or a narrowed redirect URI list stays effective until the entry expires (15 minutes by default today). `HybridCache` adds tag invalidation, but removal reaches only the current server and the shared secondary cache; other servers' in-memory copies are not affected. [H1]
- **Never cache persisted grants or reference tokens.** Revocation and one-time use depend on the store being authoritative. Only configuration data (clients, resources, CORS origins) is a candidate.
- **Secrets at rest.** A secondary cache serializes the cached object. `Client` contains `ClientSecrets`, so a Redis or SQL secondary would hold them, and `Client` would need to round-trip through the serializer exactly.
- **Object reuse.** By default every read deserializes a fresh copy for safety; reusing instances requires the type to be sealed and marked immutable. `Client` and `Resources` are mutable today. [H1]
- **Key hygiene.** The documentation warns against raw external input in keys. [H1] Client IDs, scope lists and CORS origins all come from the request. Today's `DefaultCache` has the same exposure to key flooding.

### 8. Hashing, encoding, randomness and comparison primitives

*What and since:*

| API | Since | Replaces |
|---|---|---|
| `SHA256.HashData` / `TryHashData` (and SHA-384, SHA-512) [C1] | .NET 5 | `SHA256.Create()` + `ComputeHash` + dispose |
| `CryptographicOperations.HashData(HashAlgorithmName, ...)` [C7] | .NET 9 | The algorithm switch in `CryptoHelper.GetHashAlgorithmForSigningAlgorithm` |
| `System.Buffers.Text.Base64Url` [C5] | .NET 9 | IdentityModel `Base64Url`, which is `ToBase64String` + `Split('=')` + two `Replace` calls [C10] |
| `Convert.ToHexString` / `ToHexStringLower` [C6] | .NET 5 / .NET 9 | `BitConverter.ToString(bytes).Replace("-", "")` |
| `RandomNumberGenerator.GetHexString(int, bool)` [C3] | .NET 8 | `CryptoRandom.CreateUniqueId(n, Hex)` |
| `RandomNumberGenerator.GetBytes(int)` (static) [C4] | .NET 6 | `CryptoRandom.CreateRandomKey` |
| `CryptographicOperations.FixedTimeEquals` [C2] | .NET Core 2.1 | `TimeConstantComparer.IsEqual` |

`FixedTimeEquals` is not new: it was already available on .NET Core 3.1.

*Where:*

- `Extensions/HashExtensions.cs:22-69`
- `Configuration/CryptoHelper.cs:64-93` (`at_hash`, `c_hash`)
- `Stores/Default/DefaultGrantStore.cs:79`
- `Validation/Default/HashedSharedSecretValidator.cs:66-97`
- `Validation/Default/TokenRequestValidator.cs:776-786` (PKCE)
- `Services/Default/DefaultHandleGenerationService.cs:23`
- `Services/Default/DefaultTokenService.cs:197` (`jti`)
- `Services/Default/DefaultUserSession.cs:165` (session ID)
- `Services/Default/DefaultTokenCreationService.cs:104` and `Extensions/X509CertificateExtensions.cs:25` (`x5t`, `x5t#S256`)
- `Extensions/ValidatedAuthorizeRequestExtensions.cs:111-128`
- `Models/Messages/ConsentRequest.cs:91-96`
- `Stores/Default/ProtectedDataMessageStore.cs:53,77`

*Benefit (hypothesis):* marginal in latency, moderate in allocations. *Reasoning:* the hash computation itself is unchanged; what goes away is a `HashAlgorithm` instance, a few intermediate arrays and strings per operation. Microsoft's own benchmark for the static `GetBytes` shows it about level with the instance pattern (ratio 0.97, same allocation). [C9]

Two changes of logic matter more than the API swap:

- `HashedSharedSecretValidator` should compute only the hash whose length matches the stored secret, not both.
- Comparing the hash bytes with `FixedTimeEquals`, not Base64 strings, removes the string conversions and gives a documented fixed-time guarantee.

*Risks:*

- **Timing.** `FixedTimeEquals` returns early only when the lengths differ and is fixed-time otherwise. [C2] Any rewrite of secret, PKCE or handle comparison must go through it and must not introduce `SequenceEqual`, `==` or `string.Equals` on secret material. The current helper branches per character and is weaker than the BCL primitive. [C10]
- **Entropy.** `CreateUniqueId(16, Hex)` means 16 random bytes, 32 hex characters. `GetHexString(int)` takes the string length in characters. [C3] Passing 16 would halve the entropy of handles, `jti` values and session IDs.
- **Format.** `BitConverter.ToString` is upper-case. `GetHexString` and `ToHexString` are upper-case by default too, but the lower-case variants are one argument away. Handles are hashed into store keys, so case is part of the key.
- **Padding.** The exact handling of `=` padding by `Base64Url` decoding was not checked here (see "Not verified").

### 9. `CancellationToken` plumbing

*What:* pass `HttpContext.RequestAborted` through validators, stores and outbound HTTP calls. Not a new feature.

*Where:* every interface in `Storage/src/Stores`, `IdentityServer4/src/Stores`, `Services` and `Validation`; the EF stores (`ToArrayAsync()`, `SaveChangesAsync()` without tokens); `Services/Default/BackChannelLogoutHttpClient.cs:42`; `Services/Default/DefaultJwtRequestUriHttpClient.cs:44`; `TokenCleanupService`.

*Benefit (hypothesis):* none on a happy-path benchmark. Under overload or client disconnects it stops abandoned requests from holding database connections. A latency benchmark will not show this; it needs a load test with aborted requests.

*Risks:*

- Cancelling between "grant consumed" and "token issued" must not leave a consumed authorization code or a rotated refresh token with no response. State-changing sections should run to completion once started.
- It changes almost every public interface. ADR 0001 permits that.

### 10. Endpoint routing

*What:* register protocol endpoints with `MapGet`/`MapPost` in place of the custom path-matching middleware. Endpoint routing dates from ASP.NET Core 3.0, `WebApplication` and minimal APIs from 6, and `ShortCircuit()` from 8, which runs an endpoint immediately after routing and skips the remaining middleware. [A1]

*Where:* `Hosting/IdentityServerMiddleware.cs`, `Hosting/EndpointRouter.cs`, `Hosting/Endpoint.cs`, `Hosting/MutualTlsEndpointMiddleware.cs` (which rewrites paths), `Hosting/CorsPolicyProvider.cs:97-100` (its own path list), and `Configuration/IdentityServerApplicationBuilderExtensions.cs:30-50`.

*Benefit (hypothesis):* marginal to moderate. *Reasoning:* the linear path scan is cheap. The saving is that non-protocol requests stop paying for four service resolutions, a cookie authentication and a closure. The larger value is as an enabler: routed endpoints can carry rate-limiting, output-cache, CORS and authorization metadata.

*Risks:*

- Ordering is security-relevant. The current pipeline fixes the order of base-URL capture, CORS, authentication, mTLS and the protocol middleware.
- A short-circuited endpoint skips authentication and CORS middleware. That suits discovery only if its CORS headers are produced another way, and it is wrong for any endpoint that reads the user session.
- ASP.NET Core 10 changes cookie authentication to return 401 or 403, not a redirect, for endpoints marked as API endpoints. [R9] The authorize endpoint depends on redirects and must not acquire that metadata.

### 11. Rate limiting middleware

*What:* `Microsoft.AspNetCore.RateLimiting` (ASP.NET Core 7) provides fixed window, sliding window, token bucket and concurrency limiters, applied globally or per endpoint. [A2]

*Where:* token, introspection, revocation and device authorization endpoints. The device flow already has its own throttle (`Services/Default/DistributedDeviceFlowThrottlingService.cs`).

*Benefit (hypothesis):* protective. It bounds the cost of client-secret guessing and of expensive requests; it does not make any request faster.

*Risks:*

- Counters are in memory per instance. With N instances the effective limit is N times the configured one. [A2]
- The documentation says it is not a defence against distributed attacks. [A2]
- The partition key (client ID, IP address) is attacker-influenced, so a careless choice lets an attacker create unbounded partitions or lock out a legitimate client.
- Per-endpoint policies need routed endpoints (item 10); without them only a global limiter applies.

### 12. String handling

*What and since:*

- `MemoryExtensions.Split` into a `Span<Range>` (.NET 8) and the allocation-free split enumerator (.NET 9). [T3]
- `SearchValues<char>` (.NET 8) for repeated "any of these characters" searches. [T1] [T6]
- Interpolated string handlers (C# 10 with .NET 6): existing `$"..."` expressions compile to cheaper code with no source change. [T5]
- `string.Create` with a span callback (available since .NET Core 2.1, so not new). [T4]
- `Dictionary.GetAlternateLookup<ReadOnlySpan<char>>` (.NET 9) for looking a span up without making a string. [T7]

*Where:*

- `Extensions/StringsExtensions.cs:19-34` (`ToSpaceSeparatedString` concatenates `element + " "` inside a loop, then trims; `string.Join(' ', list)` does this without new APIs), `:37-60` (`Split` + `ToList` + `Distinct` + `Sort`), `:90-131` (slash helpers use culture-sensitive `StartsWith(string)`)
- `Validation/Default/ResponseTypeEqualityComparer.cs:38-39,69`
- `Validation/Default/TokenValidator.cs:321`
- `Validation/Default/BasicAuthenticationSecretParser.cs:68-95,147`
- `Stores/Caching/CachingResourceStore.cs:66-70`
- `Extensions/HttpContextExtensions.cs:34,59-65,135-153`
- `Extensions/HttpResponseExtensions.cs:47`

*Benefit (hypothesis):* marginal. *Reasoning:* the inputs are short (a handful of scopes, one header), and each site saves a few small allocations.

*Risk:* low. `StartsWith("/")` without `StringComparison` is culture-sensitive today; the ordinal `char` overload is both faster and more correct, but it is a behaviour change on paper. Scope parsing feeds authorization decisions, so the empty-entry and duplicate rules must stay identical.

### 13. Frozen collections

*What:* `FrozenSet<T>` and `FrozenDictionary<TKey, TValue>` (.NET 8) are read-only collections that cost more to build and are optimized for lookup. [T2]

*Where:* the tables in `Constants.cs` (lines 22-132, 257-312) used by `AuthorizeRequestValidator.cs:380-437,519,570,683,720`, `DefaultClaimsService.cs:258-277`, `UserInfoRequestValidator.cs:77`, `JwtRequestValidator.cs:199`, `TokenRevocationRequestValidator.cs:86` and `ProtectedResourceErrorResult.cs:31-33`. Also the per-client collections (`AllowedScopes`, `RedirectUris`, `AllowedGrantTypes`) once a client is cached and immutable.

*Benefit (hypothesis):* marginal, possibly none. *Reasoning:* the tables hold between 2 and about 20 short strings, and a linear scan over an array that small is already fast. The claim-type filters in `DefaultClaimsService`, which run once per claim, are the only plausible win.

*Risk:* the documentation says a frozen dictionary "should only be initialized with trusted keys, as the details of the keys impacts construction time". [T2] Constant tables are fine. Freezing anything built from client registrations or request data is not.

### 14. `Environment.ProcessId`

*What:* a static property since .NET 5. [T8]

*Where:* `Services/Default/DefaultEventService.cs:117`.

*Benefit (hypothesis):* marginal, and only when events are enabled. It avoids creating a `Process` object, which is never disposed, for every event.

*Risk:* none.

### 15. `TimeProvider`

*What:* the framework time abstraction since .NET 8. ASP.NET Core 8 marked `ISystemClock` obsolete in its favour. [P1] [P2]

*Where:* 26 files inject `ISystemClock`. `TokenCleanupService.cs:78,111`, `Models/Messages/Message.cs:19` and `Validation/Models/GrantValidationResult.cs:89` read `DateTime.UtcNow` directly.

*Benefit (hypothesis):* none for speed. It removes obsolete-API warnings on .NET 10 and makes the cleanup service testable.

*Risk:* `TimeProvider` does not truncate to the second as `ISystemClock` did. [P1] Token `iat`, `nbf` and `exp`, grant creation times and expiry comparisons must be checked for off-by-one-second changes.

### 16. `ValueTask` and pooled async builders

*What:* `ValueTask<T>` avoids a `Task` allocation when a method completes synchronously. `PoolingAsyncValueTaskMethodBuilder` (.NET 6) pools the state machine when it does not. [V1]

*Where:* `ICache<T>.GetAsync` (always synchronous in `DefaultCache.cs:44-49`), `ISecretParser` and `ISecretValidator` (the default implementations use `Task.FromResult`), `IHandleGenerationService`, `ITokenCreationService`, the in-memory stores.

*Benefit (hypothesis):* marginal. One small allocation per call, across about ten calls per request.

*Risk:* a `ValueTask` must be awaited exactly once. These are public extension interfaces implemented by third parties, so the contract becomes easier to misuse.

### 17. Pooled and stack buffers

*What:* `ArrayPool<T>` and `stackalloc` with span-based encoding and hashing. Available since .NET Core 2.1, so not new.

*Where:* the same sites as item 8, where a secret or handle is encoded to UTF-8 before hashing.

*Benefit (hypothesis):* marginal on top of item 8.

*Risk:* `ArrayPool.Return` leaves the contents in place unless `clearArray: true` is passed. [C11] A rented buffer that held a client secret, a code verifier or a handle would be handed to the next renter with the secret still in it. Every such input fits in a small `stackalloc` buffer, cleared with `CryptographicOperations.ZeroMemory` [C11]; a pool is only justified for large inputs, and then it must be cleared on return.

### Correctness and hardening items found on the same paths

18. **`X509CertificateLoader`** (.NET 9). The `X509Certificate2` constructors that take bytes or a path are obsolete since .NET 9 (SYSLIB0057). [C8] `Extensions/ClientExtensions.cs:58` uses one, and re-parses each certificate secret every time it is called. .NET 10 also adds certificate lookup by SHA-256 thumbprint. [R8]
19. **Duplicate JSON properties.** .NET 10 adds `JsonSerializerOptions.AllowDuplicateProperties` and a `JsonSerializerOptions.Strict` preset; the release notes describe duplicate properties as a source of security vulnerabilities. [J6] [R8] This applies to `Infrastructure/ObjectSerializer.cs` (message cookies, authorization parameter messages) and to any new grant deserializer.
20. **Local URL check.** ASP.NET Core 10 adds `RedirectHttpResult.IsLocalUrl`. [A6] `Extensions/StringsExtensions.cs:147-191` carries a hand-written copy that guards redirects.

## Checked and found not applicable

- **Regex source generation** (.NET 7): there is no `Regex` use in `src/`.
- **Sync-over-async**: none on request paths. One `GetAwaiter().GetResult()` at startup.
- **`System.Threading.Lock`** (.NET 9): locks exist only in `Stores/InMemory/InMemoryDeviceFlowStore.cs` and `Endpoints/Results/CheckSessionResult.cs:53`. Not a hot path.
- **HMAC and PBKDF2 one-shot APIs** (.NET 6): `src/` uses neither.
- **Compiled models** (EF Core 6): aimed at start-up time for models with hundreds of entity types. [E3] The two contexts here have about 25.

## Risk flags

1. **Timing-safe comparison.** Secret, PKCE and handle comparisons must use `CryptographicOperations.FixedTimeEquals` on bytes. The current IdentityModel helper branches per character. (Items 8, 17)
2. **Handle entropy.** `GetHexString` takes a character count, where `CreateUniqueId` takes a byte count. A literal port halves entropy. (Item 8)
3. **Secrets in pooled buffers.** `ArrayPool` does not clear on return by default. (Item 17)
4. **Caching and revocation.** Cache configuration only, never grants or reference tokens. Any cache holds a disabled client or a withdrawn secret valid for its lifetime, and `HybridCache` invalidation does not reach other servers' memory. (Item 7)
5. **Secrets in a distributed cache.** A `HybridCache` secondary would store serialized `Client` objects with their secrets. (Item 7)
6. **Discovery cache key.** The issuer comes from the request host unless configured. The key must include scheme, host and path base, or the `Host` header can poison the document. (Item 1)
7. **JWKS staleness** during key rotation. (Item 1)
8. **Set-based deletes change race semantics.** "Zero rows affected" replaces the concurrency exception; for one-time-use grants the caller must treat it as "already consumed". (Item 3)
9. **Split queries are not atomic.** A client could be assembled from two versions of its configuration. (Item 3)
10. **JWT handler change.** Token wire format, claim typing and validation defaults can shift. (Item 4)
11. **Grant format change** invalidates stored refresh tokens and consents. (Item 5)
12. **Source-generated JSON with `object` values** throws on undeclared run-time types unless a reflection fallback is kept. (Item 6)
13. **Log redaction** must survive the logging rewrite. (Item 2)
14. **Cancellation** must not interrupt a grant between consumption and issuance. (Item 9)
15. **Short-circuit and ordering.** Routed endpoints change which middleware runs, and ASP.NET Core 10 changes cookie-redirect behaviour for API endpoints. (Item 10)
16. **Rate limiter partitions** keyed on untrusted input; limits are per instance. (Item 11)
17. **`TimeProvider` precision** differs from `ISystemClock`. (Item 15)
18. **Frozen collections** only from trusted keys. (Item 13)

## Questions that need a human decision

1. **What is the baseline?** The 3.1 code as it stands, or the same code retargeted to `net10.0` with no other change? Runtime gains arrive with the retarget, so code changes can only be judged fairly against the retargeted build. Should DATAS be on or off in that baseline?
2. **How stale may client and resource configuration be?** Today it is up to 15 minutes with no invalidation. Is that still acceptable, and may a distributed cache hold `Client` objects that include secrets?
3. **May the persisted grant format change** between builds of this product, given that it invalidates outstanding refresh tokens? ADR 0001 covers IdentityServer4 data only.
4. **Is the token cleanup notification contract** (`IOperationalStoreNotification` receives the removed entities) worth keeping? It blocks a set-based delete.
5. **Is the EF in-memory provider still a supported test target?** `ExecuteDelete` and `ExecuteUpdate` need a relational provider.
6. **Is replacing the custom router with endpoint routing in scope?** It changes public types (`IEndpointRouter`, `Endpoint`, `IEndpointHandler`) and the middleware order.
7. **Does rate limiting belong in the library**, or does it stay a host concern that the documentation recommends?
8. **Should the IdentityModel helpers** (`TimeConstantComparer`, `CryptoRandom`, `Base64Url`) be replaced by BCL APIs outright, removing that use of the package?
9. **What should a successful token request log at Information?** The current entry carries the full, filtered request details. Keeping it, shrinking it or demoting it to Debug changes both cost and what operators see.

## Not verified

- **No measurements.** No benchmark, profile or allocation trace was run. Every benefit size is a hypothesis.
- **Logging provider behaviour.** That the `{@details}` argument is rendered by calling the log object's `ToString()` is inferred from the `ToString` overrides in `Logging/Models`; the provider code path was not traced.
- **System.Text.Json source generation limits.** I did not confirm against documentation whether anonymous types (`JsonWebKeysResult`, `TokenRevocationErrorResult`) can be registered, or whether `[JsonExtensionData]` is supported on the serialization fast path. Both need a spike.
- **`JwtPayload` with Newtonsoft `JToken` values** on IdentityModel 7 or later was not tested. The serialization concern in item 4 follows from the release notes, not from an experiment.
- **DbContext pooling.** I could not find the constructor requirement for `AddDbContextPool` on the pages I checked. Both contexts take a second constructor argument (the store options), which may rule pooling out as written.
- **Output caching without routed endpoints.** I did not confirm how base policies treat responses produced by terminal middleware.
- **`HybridCache` packaging on .NET 10.** Confirmed as a NuGet package; not confirmed whether it is part of the ASP.NET Core shared framework.
- **`Base64Url` padding and whitespace rules** on decode were not compared with the IdentityModel helper, which accepts unpadded input only.
- **EF Core 10 parameter-list translation** was read from the release notes; its effect on the four `ResourceStore` queries was not measured.
- **DATAS** effect on this workload is unknown.
- **API introduction versions** come from the version list on each Microsoft Learn API page. `LoggerMessageAttribute`, `HybridCache` and `IBufferDistributedCache` are listed there only as packages, so their versions come from the conceptual articles instead.
- The .NET 7 performance post was not consulted (the address I tried returned 404).

## Sources

All Microsoft Learn, .NET Blog or first-party GitHub pages, read on 2026-11-13.

Runtime and platform

- [R1] What's new in the .NET 8 runtime: https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-8/runtime
- [R2] Runtime configuration options for compilation: https://learn.microsoft.com/en-us/dotnet/core/runtime-config/compilation
- [R3] Dynamic adaptation to application sizes (DATAS): https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/datas
- [R4] What's new in the .NET 10 runtime: https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-10/runtime
- [R5] Performance Improvements in .NET 10: https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-10/
- [R6] Performance Improvements in .NET 9: https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-9/
- [R7] Memory management and garbage collection in ASP.NET Core: https://learn.microsoft.com/en-us/aspnet/core/performance/memory
- [R8] What's new in .NET 10 libraries: https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-10/libraries
- [R9] What's new in ASP.NET Core 10: https://learn.microsoft.com/en-us/aspnet/core/release-notes/aspnetcore-10.0

Cryptography and encoding

- [C1] `SHA256.HashData`: https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.sha256.hashdata
- [C2] `CryptographicOperations.FixedTimeEquals`: https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.cryptographicoperations.fixedtimeequals
- [C3] `RandomNumberGenerator.GetHexString`: https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.randomnumbergenerator.gethexstring
- [C4] `RandomNumberGenerator.GetBytes`: https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.randomnumbergenerator.getbytes
- [C5] `Base64Url`: https://learn.microsoft.com/en-us/dotnet/api/system.buffers.text.base64url
- [C6] `Convert.ToHexString`: https://learn.microsoft.com/en-us/dotnet/api/system.convert.tohexstring and `ToHexStringLower`: https://learn.microsoft.com/en-us/dotnet/api/system.convert.tohexstringlower
- [C7] `CryptographicOperations.HashData`: https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.cryptographicoperations.hashdata
- [C8] SYSLIB0057: https://learn.microsoft.com/en-us/dotnet/fundamentals/syslib-diagnostics/syslib0057 and `X509CertificateLoader`: https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.x509certificateloader
- [C9] Performance Improvements in .NET 6 (cryptography section): https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-6/
- [C10] IdentityModel 4.4.0 source: https://github.com/IdentityModel/IdentityModel/blob/4.4.0/src/TimeConstantComparer.cs , https://github.com/IdentityModel/IdentityModel/blob/4.4.0/src/CryptoRandom.cs , https://github.com/IdentityModel/IdentityModel/blob/4.4.0/src/Base64Url.cs
- [C11] `ArrayPool<T>.Return`: https://learn.microsoft.com/en-us/dotnet/api/system.buffers.arraypool-1.return and `CryptographicOperations.ZeroMemory`: https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.cryptographicoperations.zeromemory

Strings and collections

- [T1] `SearchValues<T>`: https://learn.microsoft.com/en-us/dotnet/api/system.buffers.searchvalues-1
- [T2] `FrozenDictionary<TKey,TValue>`: https://learn.microsoft.com/en-us/dotnet/api/system.collections.frozen.frozendictionary-2
- [T3] `MemoryExtensions.Split`: https://learn.microsoft.com/en-us/dotnet/api/system.memoryextensions.split
- [T4] `String.Create`: https://learn.microsoft.com/en-us/dotnet/api/system.string.create
- [T5] The history of C# (version 10, interpolated string handlers): https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-version-history and String Interpolation in C# 10 and .NET 6: https://devblogs.microsoft.com/dotnet/string-interpolation-in-c-10-and-net-6/
- [T6] Performance Improvements in .NET 8: https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/
- [T7] `Dictionary<TKey,TValue>.GetAlternateLookup`: https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.dictionary-2.getalternatelookup
- [T8] `Environment.ProcessId`: https://learn.microsoft.com/en-us/dotnet/api/system.environment.processid

JSON

- [J1] System.Text.Json source generation: https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/source-generation
- [J2] Reflection versus source generation: https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/reflection-vs-source-generation
- [J3] Migrate from Newtonsoft.Json: https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/migrate-from-newtonsoft
- [J4] SYSLIB0020: https://learn.microsoft.com/en-us/dotnet/fundamentals/syslib-diagnostics/syslib0020
- [J5] `HttpResponseJsonExtensions.WriteAsJsonAsync`: https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.http.httpresponsejsonextensions.writeasjsonasync
- [J6] `JsonSerializerOptions.AllowDuplicateProperties`: https://learn.microsoft.com/en-us/dotnet/api/system.text.json.jsonserializeroptions.allowduplicateproperties
- [J7] `JsonSerializer.SerializeAsync`: https://learn.microsoft.com/en-us/dotnet/api/system.text.json.jsonserializer.serializeasync
- [J8] `Utf8JsonWriter.WriteRawValue`: https://learn.microsoft.com/en-us/dotnet/api/system.text.json.utf8jsonwriter.writerawvalue
- [J9] `JsonSerializerOptions.TypeInfoResolverChain`: https://learn.microsoft.com/en-us/dotnet/api/system.text.json.jsonserializeroptions.typeinforesolverchain

Logging

- [L1] Compile-time logging source generation: https://learn.microsoft.com/en-us/dotnet/core/extensions/logger-message-generator
- [L2] High-performance logging: https://learn.microsoft.com/en-us/dotnet/core/extensions/high-performance-logging
- [L3] CA1848: https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1848

Caching

- [H1] HybridCache library in ASP.NET Core: https://learn.microsoft.com/en-us/aspnet/core/performance/caching/hybrid
- [H2] Hello HybridCache (general availability): https://devblogs.microsoft.com/dotnet/hybrid-cache-is-now-ga/
- [H3] What's new in ASP.NET Core 9: https://learn.microsoft.com/en-us/aspnet/core/release-notes/aspnetcore-9.0
- [H4] Output caching middleware: https://learn.microsoft.com/en-us/aspnet/core/performance/caching/output

Time

- [P1] Breaking change: ISystemClock is obsolete: https://learn.microsoft.com/en-us/dotnet/core/compatibility/aspnet-core/8.0/isystemclock-obsolete
- [P2] What is TimeProvider: https://learn.microsoft.com/en-us/dotnet/standard/datetime/timeprovider-overview

EF Core

- [E1] ExecuteUpdate and ExecuteDelete: https://learn.microsoft.com/en-us/ef/core/saving/execute-insert-update-delete
- [E2] Single versus split queries: https://learn.microsoft.com/en-us/ef/core/querying/single-split-queries
- [E3] Advanced performance topics: https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics
- [E4] Tracking versus no-tracking queries: https://learn.microsoft.com/en-us/ef/core/querying/tracking
- [E5] What's new in EF Core 10: https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/whatsnew
- [E6] EF Core in-memory provider: https://learn.microsoft.com/en-us/ef/core/providers/in-memory/

ASP.NET Core and tokens

- [A1] Routing in ASP.NET Core (short-circuit): https://learn.microsoft.com/en-us/aspnet/core/fundamentals/routing
- [A2] Rate limiting middleware: https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit
- [A3] ASP.NET Core best practices: https://learn.microsoft.com/en-us/aspnet/core/fundamentals/best-practices
- [A4] Breaking change: Security token events return a JsonWebToken: https://learn.microsoft.com/en-us/dotnet/core/compatibility/aspnet-core/8.0/securitytoken-events
- [A5] IdentityModel 7x: https://github.com/AzureAD/azure-activedirectory-identitymodel-extensions-for-dotnet/wiki/IdentityModel-7x
- [A6] `RedirectHttpResult.IsLocalUrl`: https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.http.httpresults.redirecthttpresult.islocalurl

Async

- [V1] `PoolingAsyncValueTaskMethodBuilder`: https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.poolingasyncvaluetaskmethodbuilder
