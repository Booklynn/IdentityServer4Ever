# Breaking changes between .NET Core 3.1 and .NET 10 that affect `src/`

Research for issue #2. Written 2026-11-13 against commit `5e40bd18` (`main`). Scope is `src/` only.

## Summary

- **The .NET 10 runtime and the ASP.NET Core 10 shared framework break almost nothing in this code.** With only the target framework changed to `net10.0` and every package left at its 3.1-era version, the core library compiles with no errors and its tests pass: 724 of 724 unit tests and 293 of 293 integration tests (1 skipped).
- **The real breaks come from three package upgrades that the retarget forces.**
  1. EF Core 3.1 does not compile against `net10.0` (15 errors in `src/EntityFramework.Storage/src/Stores` and `TokenCleanup`). EF Core must move to 10.0.x in the same change as the target framework.
  2. `Microsoft.AspNetCore.Authentication.OpenIdConnect` 10.0.15 requires Microsoft.IdentityModel 8.19.2 or later (the repo pins 5.6.0). IdentityModel 7 replaced Newtonsoft.Json with System.Text.Json. Everything still compiles, but two product behaviours break silently: the `cnf` claim in issued JWTs is corrupted, and object- or array-valued parameters in JWT request objects are dropped. JWK parsing also becomes strict JSON.
  3. AutoMapper 10.x throws `TypeInitializationException` on .NET 7 and later, which disables every mapper in `IdentityServer4.EntityFramework.Storage`. The fix shipped in AutoMapper 12.0.0.
- **With those packages bumped, the whole of `src/` builds with 0 errors.** Twelve tests then fail, all because of IdentityModel: ten through test fixtures that are not strict JSON, two through real product defects. The build reports about 150 compiler and analyzer warnings under nine diagnostic IDs (section 3), plus NuGet audit warnings. The largest groups are `ISystemClock` (59 sites in 31 files), `IDictionary.Add` on headers (44 sites) and `WebHostBuilder`/`IWebHost` (32 sites, tests and migration hosts only).
- **EF Core 3.1 to 10 is benign for this model.** The existing migrations and snapshots show no pending model changes under EF Core 10 tools, and the EF tests pass on the in-memory, SQLite and SQL Server LocalDB providers.
- **SDK:** `global.json` pins SDK `3.1.402`, so no `dotnet` command runs in the repo today. The .NET 10 SDK also audits transitive packages for vulnerabilities by default and warns about package references that the framework already provides.
- **Pin:** SDK `10.0.404`, runtime and ASP.NET Core `10.0.15`, both released 2026-11-10. The local machine has SDK `10.0.403` and runtime `10.0.14`, one patch behind.

## How this was established

Two kinds of evidence are used. Each claim below names which one it rests on.

**Documentation.** Microsoft's breaking-change lists for .NET 5 to 10, the ASP.NET Core lists for 5 to 10, the EF Core lists for 5.0 to 10.0, and the Microsoft.IdentityModel 7.x and 8.x notes. Every title in those lists was compared with what `src/` uses. Index pages:

- .NET: <https://learn.microsoft.com/en-us/dotnet/core/compatibility/5.0>, [6.0](https://learn.microsoft.com/en-us/dotnet/core/compatibility/6.0), [7.0](https://learn.microsoft.com/en-us/dotnet/core/compatibility/7.0), [8.0](https://learn.microsoft.com/en-us/dotnet/core/compatibility/8.0), [9.0](https://learn.microsoft.com/en-us/dotnet/core/compatibility/9.0), [10](https://learn.microsoft.com/en-us/dotnet/core/compatibility/10)
- ASP.NET Core: `https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/{5..10}/overview`, for example <https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/10/overview>
- EF Core: `https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-{5.0..10.0}/breaking-changes`, for example <https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/breaking-changes>

**Probe.** A throwaway copy of `src/` (made with `git archive main`, outside the working directory) was retargeted and built with SDK 10.0.403 on Windows 11, runtime 10.0.14. Nothing in the repository was changed. The cross-library `PackageReference`s (`IdentityServer4`, `IdentityServer4.Storage`, and so on) were turned into `ProjectReference`s so that one solution could build all 22 projects.

| Stage | Change from the previous stage | Result |
|---|---|---|
| A | `netcoreapp3.1` and `netstandard2.0` become `net10.0`. Packages unchanged. | 15 errors, all in `src/EntityFramework.Storage/src`. `IdentityServer4`, `Storage`, `AspNetIdentity`, their hosts and tests compile. `IdentityServer.UnitTests` 724/724. `IdentityServer.IntegrationTests` 293 passed, 1 skipped. |
| B | Microsoft.AspNetCore.\*, Microsoft.EntityFrameworkCore.\* and Microsoft.Extensions.\* to 10.0.15. Microsoft.IdentityModel to 8.19.2. `System.Security.Principal.Windows` reference removed. Test SDK bumped. | 0 errors, 212 warnings. Unit 723/724. Integration 282/293. EF tests blocked by AutoMapper. |
| C | AutoMapper 12.0.1, and `AssertConfigurationIsValid<T>()` changed to `AssertConfigurationIsValid()` in five test files. | `EntityFramework.UnitTests` 16/16. `EntityFramework.IntegrationTests` 40/40. `IdentityServer4.EntityFramework.Tests` 6/6 on in-memory, SQLite and LocalDB. |
| D | JWK test fixtures made strict JSON. | Integration 291/293. The two remaining failures are product defects (section 2). |

Also run in the probe: `dotnet ef migrations has-pending-model-changes` with `dotnet-ef` 10.0.15, and a smoke run of `src/IdentityServer4/host` on `net10.0` in the Development environment (discovery document, JWKS, login page and a client-credentials token request all worked).

## Versions to pin

Source: the official release metadata, <https://builds.dotnet.microsoft.com/dotnet/release-metadata/releases-index.json> and <https://builds.dotnet.microsoft.com/dotnet/release-metadata/10.0/releases.json>, read on 2026-11-13.

| Item | Latest | Notes |
|---|---|---|
| .NET SDK | **10.0.404** (4xx band); 10.0.115 (1xx band) | Released 2026-11-10. Ships C# 14.0. |
| .NET runtime | **10.0.15** | Security release. [Release notes](https://github.com/dotnet/core/blob/main/release-notes/10.0/10.0.15/10.0.15.md). |
| ASP.NET Core runtime | **10.0.15** | Same release. |
| Installed locally | SDK 10.0.403, runtime 10.0.14 | One patch behind (10.0.14 was released 2026-10-25). |
| Support | .NET 10 is LTS, active, end of life 2028-11-14 | .NET 8 and 9 reached end of life on 2026-11-10. .NET 11.0.0 (STS) shipped the same day. |

Package versions on nuget.org on 2026-11-13: `Microsoft.AspNetCore.*`, `Microsoft.EntityFrameworkCore.*` and `Microsoft.Extensions.*` are at 10.0.15 in the 10.x line (11.0.0 also exists, so "latest" is no longer 10.x; pin the major). `dotnet-ef` 10.0.15. `Microsoft.IdentityModel.*` and `System.IdentityModel.Tokens.Jwt` are at 8.23.0; the floor imposed by ASP.NET Core 10.0.15 is 8.19.2.

`global.json` roll-forward: when no `rollForward` is set the policy is `patch`, which is why the current file fails (section 7). Source: <https://learn.microsoft.com/en-us/dotnet/core/tools/global-json>.

## 1. Changes that stop restore or build

### 1.1 `global.json` pins an SDK that is not installed

- **What breaks:** every `dotnet` command in the repository. Observed in the working directory: "A compatible .NET SDK was not found. Requested SDK version: 3.1.402".
- **Where:** `global.json` (repository root; it governs `src/`).
- **Replacement:** set the version to `10.0.404` and choose a `rollForward` policy. Source: <https://learn.microsoft.com/en-us/dotnet/core/tools/global-json>.

### 1.2 Target framework monikers

- **What breaks:** .NET Core 3.1 reached end of life on 2022-12-13 (release metadata, see "Versions to pin"). The ASP.NET Core and EF Core 10.0.15 packages that `src/` needs ship `net10.0` assets only, so neither `netcoreapp3.1` nor `netstandard2.0` projects can reference them (checked in the nuspecs of `Microsoft.EntityFrameworkCore.Relational`, `Microsoft.EntityFrameworkCore.SqlServer`, `Microsoft.EntityFrameworkCore.Sqlite`, `Microsoft.AspNetCore.Authentication.OpenIdConnect`, `Microsoft.AspNetCore.Authentication.Certificate` and `Microsoft.AspNetCore.Identity.EntityFrameworkCore` on nuget.org).
- **Where:** all 22 project files. `netstandard2.0`: `src/Storage/src/IdentityServer4.Storage.csproj`, `src/EntityFramework.Storage/src/IdentityServer4.EntityFramework.Storage.csproj`. `netcoreapp3.1`: every other `.csproj` under `src/`, including the five `build/build.csproj` runners.
- **Replacement:** `<TargetFramework>net10.0</TargetFramework>`. The moniker format changed from `netcoreappX.Y` to `netX.Y` in .NET 5: <https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/5.0/targetframework-name-change>.
- **Probe:** the default C# version moves from 8 to 14 with this change (<https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-versioning>). No source file failed to compile because of a newer language version.

### 1.3 EF Core 3.1 does not compile on `net10.0`

- **What breaks:** .NET 10 adds `System.Linq.AsyncEnumerable` to the core libraries. In EF Core 3.1 `DbSet<T>` implements both `IQueryable<T>` and `IAsyncEnumerable<T>`, so every LINQ operator applied directly to a `DbSet` is ambiguous. Stage A gives 7 × CS0121, 4 × CS1940 and 4 follow-on CS1061.
- **Where:** `src/EntityFramework.Storage/src/Stores/ClientStore.cs:55`, `DeviceFlowStore.cs:110,141`, `PersistedGrantStore.cs:49,105`, `ResourceStore.cs:57,68,94,105,124,133,151,160`, and `src/EntityFramework.Storage/src/TokenCleanup/TokenCleanupService.cs:78,111`.
- **Replacement:** move to EF Core 10 in the same change. `DbSet` stopped implementing `IAsyncEnumerable` in EF Core 6.0, which removes the ambiguity; no source change is needed (stage B compiles these files unchanged).
- **Sources:** <https://learn.microsoft.com/en-us/dotnet/core/compatibility/core-libraries/10.0/asyncenumerable>; <https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-6.0/breaking-changes#dbset-iasyncenumerable>.

### 1.4 The Microsoft.IdentityModel pin conflicts with ASP.NET Core 10 packages

- **What breaks:** restore. `Microsoft.AspNetCore.Authentication.OpenIdConnect` 10.0.15 depends on `Microsoft.IdentityModel.Protocols.OpenIdConnect` >= 8.19.2 (nuspec on nuget.org). The repo pins `Microsoft.IdentityModel.Protocols.OpenIdConnect` 5.6.0 and `System.IdentityModel.Tokens.Jwt` `[5.6.0,6.0)`. Probe: bumping only the ASP.NET Core packages gives `NU1605: Detected package downgrade: Microsoft.IdentityModel.Protocols.OpenIdConnect from 8.19.2 to 5.6.0`, which NuGet treats as an error (<https://learn.microsoft.com/en-us/nuget/reference/errors-and-warnings/nu1605>).
- **Where:** `src/Directory.Build.targets` (the two version entries). The dependency exists because the core library references the OpenID Connect handler: `src/IdentityServer4/src/IdentityServer4.csproj`, used in `src/IdentityServer4/src/Configuration/DependencyInjection/ConfigureOpenIdConnectOptions.cs` and `IdentityServerServiceCollectionExtensions.cs:86`.
- **Replacement:** Microsoft.IdentityModel 8.19.2 or later for every IdentityModel package ("all the IdentityModel libraries must have the same version", <https://github.com/AzureAD/azure-activedirectory-identitymodel-extensions-for-dotnet/wiki/IdentityModel-8x>). This is a three-major-version jump and is the source of every finding in section 2.2. Microsoft's own note on the first of these bumps says "it's possible there are breaking changes in the packages themselves": <https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/5/security-identitymodel-nuget-package-versions-updated>.

## 2. Changes that compile but fail at run time

### 2.1 AutoMapper 10.x cannot initialise on .NET 7 or later

- **What breaks:** the static initialiser of every mapper class throws `TypeInitializationException` wrapping `ArgumentException: GenericArguments[0], 'System.Char', on 'T MaxFloat[T](IEnumerable<T>)' violates the constraint of type 'T'`. The cause is AutoMapper reflecting over `System.Linq.Enumerable` methods that gained generic-math constraints in .NET 7. Every store in `IdentityServer4.EntityFramework.Storage` that maps an entity is unusable. Probe stage B: `EntityFramework.UnitTests` 3/16, `EntityFramework.IntegrationTests` 28/40. Version 10.1.1, the newest inside the repo's `[10.0.0,11.0)` range, fails the same way.
- **Where:** `src/EntityFramework.Storage/src/Mappers/*.cs` (`ClientMappers`, `ApiResourceMappers`, `IdentityResourceMappers`, `ScopeMappers`, `PersistedGrantMappers`), version range in `src/Directory.Build.targets`.
- **Replacement:** AutoMapper 12.0.0 or later, which contains the fix, or removal of AutoMapper (issue #6 decides). Moving to 12 removes the generic `AssertConfigurationIsValid<TProfile>()` overload used by five tests in `src/EntityFramework.Storage/test/UnitTests/Mappers/` (CS0308). With 12.0.1 and that one-line test change all EF tests pass (stage C). Note that NuGet audit reports advisory GHSA-rvv3-g6hj-g44x against both 10.0.0 and 12.0.1.
- **Sources:** <https://github.com/LuckyPennySoftware/AutoMapper/issues/3988> and the fix <https://github.com/LuckyPennySoftware/AutoMapper/pull/3999> (milestone 12.0.0); runtime side <https://github.com/dotnet/runtime/issues/69119>.

### 2.2 Microsoft.IdentityModel 5.6 to 8.x: System.Text.Json replaces Newtonsoft.Json

IdentityModel 7.0 "used Newtonsoft for JSON serialization, but now it uses `System.Text.Json`", and "complex unknown types are exposed as `JsonElement`". Source: <https://github.com/AzureAD/azure-activedirectory-identitymodel-extensions-for-dotnet/wiki/IdentityModel-7x>. Stage A proves these are package effects and not runtime effects: with IdentityModel 5.6.0 on .NET 10 all tests pass.

**a. JSON-valued claims in issued JWTs are corrupted (silent).**

- **What breaks:** `CreateJwtPayload` parses JSON claims with Newtonsoft (`JRaw.Parse`) and puts `JToken` objects into the `JwtPayload`. IdentityModel 8 serialises the payload with System.Text.Json and does not understand `JToken`. A minimal reproduction against `System.IdentityModel.Tokens.Jwt` 8.19.2 turns `{"x5t#S256":"foo"}` into `"cnf":[["foo"]]`. No exception is thrown. This affects the `cnf` claim of certificate-bound access tokens (mutual TLS) and every claim with value type `json`.
- **Where:** `src/IdentityServer4/src/Extensions/TokenExtensions.cs:87-144`. Failing test: `IdentityServer.IntegrationTests.Clients.ClientCredentialsClient.Valid_request_with_confirmation_should_return_expected_payload`.
- **Replacement:** add `System.Text.Json` values (`JsonElement`, or plain dictionaries and lists) to the payload instead of `JToken`.

**b. Object and array parameters in JWT request objects are dropped (silent).**

- **What breaks:** `ProcessPayloadAsync` switches on `string`, `JObject` and `JArray`. Payload values are now `string`, numbers, `bool` or `JsonElement`, so the object and array cases never match and the parameter disappears (for example the OpenID Connect `claims` parameter).
- **Where:** `src/IdentityServer4/src/Validation/Default/JwtRequestValidator.cs:193-219`. Failing test: `JwtRequestAuthorizeTests.authorize_should_accept_complex_objects_in_request_object`.
- **Replacement:** handle `JsonElement` (`GetRawText()` for objects and arrays).

**c. `new JsonWebKey(json)` accepts strict JSON only.**

- **What breaks:** JWK text with single quotes or a trailing comma, and JWK text produced by `JsonConvert.SerializeObject(jsonWebKey)`, now throws `IDX10805`. For client secrets of type `JsonWebKey` the exception is caught and logged as "Could not parse client secrets", and the request object is rejected.
- **Where (product):** `src/IdentityServer4/src/Extensions/ClientExtensions.cs:47`. **Where (test fixtures):** `src/IdentityServer4/test/IdentityServer.IntegrationTests/Endpoints/Authorize/JwtRequestAuthorizeTests.cs:34,72,78,116,122` (10 tests fail through these; 9 pass once the fixtures are strict JSON, and the tenth then fails for reason b), and `src/IdentityServer4/test/IdentityServer.UnitTests/Extensions/IdentityServerBuilderExtensionsCryptoTests.cs:24-37` (trailing comma, 1 test; not re-run after repair).
- **Replacement:** strict JSON in fixtures; serialise `JsonWebKey` with System.Text.Json (stage D did exactly this). Whether to accept the stricter parsing in the product is question 2 in section 10.

**d. Smaller IdentityModel items.**

- `JwtPayload.Exp` is obsolete ("use `long? Expiration`", year-2038 fix): `src/IdentityServer4/src/Validation/Default/PrivateKeyJwtSecretValidator.cs:120` (CS0618 in the probe). Source: the IdentityModel-7x page above.
- The synchronous `ValidateToken` is described as deprecated in favour of `ValidateTokenAsync` on the same page. It is still callable and produced no compiler warning in the probe. Call sites: `JwtRequestValidator.cs:183`, `PrivateKeyJwtSecretValidator.cs:111`, `TokenValidator.cs:269`.
- ASP.NET Core 8 made the JwtBearer and OpenID Connect handlers produce `JsonWebToken` instead of `JwtSecurityToken` in their events (<https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/8/securitytoken-events>). Library code in `src/` does not read those events. The hosts configure `AddOpenIdConnect` (`src/IdentityServer4/host/Startup.cs:176-223`, `src/AspNetIdentity/host/Startup.cs:44`) without downcasting, so they are not affected.

## 3. Obsoletions: the code compiles with warnings

No project sets `TreatWarningsAsErrors`, so none of these stop the build today. Counts are from stage B.

| # | Diagnostic | What is obsolete | Where in `src/` | Documented replacement |
|---|---|---|---|---|
| 3.1 | CS0618 | `ISystemClock`, `SystemClock`, and the `AuthenticationHandler` constructor that takes a clock (ASP.NET Core 8) | 59 sites in 31 files. Product: 26 files under `src/IdentityServer4/src/` (list below). Tests: `IdentityServer.UnitTests/Common/MockSystemClock.cs`, `StubClock.cs`, `Extensions/JwtPayloadCreationTests.cs`, `Validation/Setup/Factory.cs`, `ResponseHandling/AuthorizeInteractionResponseGenerator/AuthorizeInteractionResponseGeneratorTests_Custom.cs` | `System.TimeProvider`; drop the clock parameter from handler constructors. <https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/8/isystemclock-obsolete> |
| 3.2 | ASPDEPR004, ASPDEPR008 | `WebHostBuilder`, `IWebHost`, `WebHost`, `TestServer(IWebHostBuilder)` (.NET 10) | 32 sites. Tests: 13 files under `src/IdentityServer4/test/IdentityServer.IntegrationTests/` (`Clients/*.cs`, `Common/IdentityServerPipeline.cs:79,101`, `Endpoints/Introspection/IntrospectionTests.cs:33`). Migration hosts: `src/{EntityFramework.Storage,EntityFramework,AspNetIdentity}/migrations/SqlServer/Program.cs` | `HostBuilder` with `ConfigureWebHost(b => b.UseTestServer())` and `host.GetTestServer()`, or `WebApplicationBuilder`. <https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/10/webhostbuilder-deprecated> |
| 3.3 | SYSLIB0057 | `X509Certificate2` constructors that take bytes or a file path (.NET 9) | Product: `src/IdentityServer4/src/Extensions/ClientExtensions.cs:58`. Host: `src/IdentityServer4/host/Startup.cs:140,147,258`. Tests: `IdentityServer.IntegrationTests/Common/TestCert.cs:15`, `IdentityServer.UnitTests/Common/TestCert.cs:16` | `X509CertificateLoader.LoadCertificate` for X.509 content, `LoadPkcs12FromFile` for PFX. <https://learn.microsoft.com/en-us/dotnet/fundamentals/syslib-diagnostics/syslib0057> |
| 3.4 | SYSLIB0023 | `RNGCryptoServiceProvider` (.NET 6) | `src/IdentityServer4/src/Services/Default/NumericUserCodeGenerator.cs:51` | `RandomNumberGenerator` static methods. <https://learn.microsoft.com/en-us/dotnet/fundamentals/syslib-diagnostics/syslib0023> |
| 3.5 | SYSLIB0020 | `JsonSerializerOptions.IgnoreNullValues` (.NET 6) | `src/IdentityServer4/src/Infrastructure/ObjectSerializer.cs:13`, `src/IdentityServer4/src/Logging/LogSerializer.cs:17` | `DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull`. <https://learn.microsoft.com/en-us/dotnet/fundamentals/syslib-diagnostics/syslib0020> |
| 3.6 | CS0618 | `HttpRequestMessage.Properties` (.NET 5) | `src/IdentityServer4/src/Services/Default/DefaultJwtRequestUriHttpClient.cs:42` | `HttpRequestMessage.Options`. <https://learn.microsoft.com/en-us/dotnet/api/system.net.http.httprequestmessage.properties> |
| 3.7 | CS0618 | `JwtPayload.Exp` | See 2.2 d | `JwtPayload.Expiration` |
| 3.8 | ASP0019 | `IHeaderDictionary.Add` (analyzer, on by default) | 44 sites. Product: `src/IdentityServer4/src/Extensions/HttpResponseExtensions.cs:42,61,70,116,120`, `Endpoints/Results/AuthorizeResult.cs:135`, `Endpoints/Results/ProtectedResourceErrorResult.cs:45,50`. Hosts: three copies of `Quickstart/SecurityHeadersAttribute.cs`. Tests: five files in `IdentityServer.UnitTests` | `Append` or the indexer. <https://learn.microsoft.com/en-us/aspnet/core/diagnostics/asp0019> |
| 3.9 | CA2017 | Logging template and argument count differ (analyzer, on by default) | `src/IdentityServer4/src/Validation/Default/JwtBearerClientAssertionSecretParser.cs:115`. The exception is passed as a format argument and is never logged. | Pass the exception as the first argument. <https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca2017> |
| 3.10 | EF1001 | EF Core internal API (`InMemoryOptionsExtension`) | `src/EntityFramework.Storage/test/IntegrationTests/Stores/DeviceFlowStoreTests.cs:127,170` | No public equivalent named in the warning; restructure the test. |
| 3.11 | CS0618, suppressed | `IndexBuilder.HasName` (EF Core 5.0) | `src/AspNetIdentity/migrations/SqlServer/Migrations/UsersDb/*.Designer.cs` and `ApplicationDbContextModelSnapshot.cs`. Generated files that disable warnings 612 and 618, so nothing is reported. | `HasDatabaseName`; regenerating the migration does it. <https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-5.0/breaking-changes#index-obsolete> |

Product files that use `ISystemClock` (3.1), all under `src/IdentityServer4/src/`: `Endpoints/Results/AuthorizeResult.cs`, `Endpoints/Results/EndSessionResult.cs`, `Extensions/HttpContextAuthenticationExtensions.cs`, `Extensions/HttpContextExtensions.cs`, `Extensions/TokenExtensions.cs`, `Hosting/IdentityServerAuthenticationService.cs`, `Hosting/LocalApiAuthentication/LocalApiAuthenticationHandler.cs`, `IdentityServerTools.cs`, `ResponseHandling/Default/AuthorizeInteractionResponseGenerator.cs`, `AuthorizeResponseGenerator.cs`, `DeviceAuthorizationResponseGenerator.cs`, `TokenResponseGenerator.cs`, `Services/Default/DefaultBackChannelLogoutService.cs`, `DefaultConsentService.cs`, `DefaultEventService.cs`, `DefaultIdentityServerInteractionService.cs`, `DefaultRefreshTokenService.cs`, `DefaultTokenCreationService.cs`, `DefaultTokenService.cs`, `DefaultUserSession.cs`, `DistributedDeviceFlowThrottlingService.cs`, `Test/TestUserResourceOwnerPasswordValidator.cs`, `Validation/Default/DeviceCodeValidator.cs`, `SecretValidator.cs`, `TokenRequestValidator.cs`, `TokenValidator.cs`.

Two facts about `ISystemClock` matter for the port:

- It is still registered. `AddAuthentication` in ASP.NET Core 10 calls `TryAddSingleton<ISystemClock, SystemClock>()` next to `TryAddSingleton(TimeProvider.System)`, so the three service-locator sites (`AuthorizeResult.cs:55`, `EndSessionResult.cs:56`, `HttpContextAuthenticationExtensions.cs:44`) keep working. Source: <https://github.com/dotnet/aspnetcore/blob/release/10.0/src/Security/Authentication/Core/src/AuthenticationServiceCollectionExtensions.cs>. The breaking-change page warns that it "may be removed from the container in a future version".
- The framework no longer reads it. Cookie and other handlers take time from `TimeProvider`, so replacing `ISystemClock` in DI no longer moves the framework's clock. `SystemClock` truncates to whole seconds and `TimeProvider` does not (same page).

## 4. Documented behaviour changes that reach code in `src/`

No test fails because of these. They are listed because the code path exists.

| # | Change | Where in `src/` | Effect and replacement |
|---|---|---|---|
| 4.1 | .NET 5: globalization uses ICU on Windows. Culture-sensitive string operations behave differently from NLS. <https://learn.microsoft.com/en-us/dotnet/core/compatibility/globalization/5.0/icu-globalization-api> | 16 culture-sensitive `StartsWith(string)` and `EndsWith(string)` calls in product code: `Extensions/StringsExtensions.cs:92,103,114,125,138,200`, `Extensions/HttpContextExtensions.cs:124,146`, `Extensions/HttpResponseExtensions.cs:85`, `Extensions/ValidatedAuthorizeRequestExtensions.cs:28,71`, `Infrastructure/MessageCookie.cs:68`, `Configuration/DependencyInjection/ConfigureInternalCookieOptions.cs:61`, `ResponseHandling/Default/DiscoveryResponseGenerator.cs:349`, `Validation/Default/BearerTokenUsageValidator.cs:69`, `Validation/Default/DefaultClientConfigurationValidator.cs:152` (all under `src/IdentityServer4/src/`) | Only Windows changes; Linux already used ICU on 3.1. These compare URLs, prefixes and the `Bearer` scheme, so ordinal comparison is the intent. Replacement: pass `StringComparison.Ordinal`. |
| 4.2 | .NET 10 and C# 14: `array.Contains(x)` inside an expression tree binds to `MemoryExtensions.Contains(ReadOnlySpan<T>, T)` instead of `Enumerable.Contains`. <https://learn.microsoft.com/en-us/dotnet/core/compatibility/core-libraries/10.0/csharp-overload-resolution> | The three EF queries whose list is a `string[]`: `src/EntityFramework.Storage/src/Stores/ResourceStore.cs:94,124,151` | EF Core had to learn the new shape (<https://github.com/dotnet/efcore/issues/35100>); EF Core 10 has it. Probe: the store tests for these three methods pass on the in-memory provider. Their SQL translation was not exercised (section 9). |
| 4.3 | .NET 9: `IHttpClientFactory` uses `SocketsHttpHandler` as the default primary handler. <https://learn.microsoft.com/en-us/dotnet/core/compatibility/networking/9.0/default-handler> | `src/IdentityServer4/src/Configuration/DependencyInjection/BuilderExtensions/Additional.cs:370-419` and `Core.cs:48` (back-channel logout and `request_uri` clients) | No code casts the primary handler to `HttpClientHandler`, so nothing breaks here. An application that configured these named clients by cast would break. |
| 4.4 | .NET 9: `IHttpClientFactory` logging redacts header values by default. <https://learn.microsoft.com/en-us/dotnet/core/compatibility/networking/9.0/redact-headers> | Same clients as 4.3 | Log content only. |
| 4.5 | .NET SDK 8: Source Link is built in and `InformationalVersion` gains `+<commit>`. <https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/8.0/source-link> | `src/IdentityServer4/src/Configuration/IdentityServerApplicationBuilderExtensions.cs:58` logs `InformationalVersion` at startup | Cosmetic: the startup log line shows the commit suffix. |

## 5. EF Core 3.1 to 10

The model is plain (tables, keys, indexes, max lengths, cascade deletes; no owned types, value converters, concurrency tokens or JSON columns), so most of the 100-odd documented EF Core breaks do not apply. The ones that touch `src/`:

| # | Change | Where | Result |
|---|---|---|---|
| 5.1 | EF Core 9, high impact: `Migrate()` and `dotnet ef database update` throw if the model differs from the last snapshot. <https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-9.0/breaking-changes#pending-model-changes> | `src/AspNetIdentity/migrations/SqlServer/SeedData.cs:24` calls `Migrate()`. Migrations from 2020 in `src/EntityFramework.Storage/migrations/SqlServer/Migrations/` and `src/AspNetIdentity/migrations/SqlServer/Migrations/`. | **Not hit.** Probe: `dotnet ef migrations has-pending-model-changes` (tools 10.0.15) reports no changes for `PersistedGrantDbContext`, `ConfigurationDbContext` and `ApplicationDbContext`. |
| 5.2 | EF Core 7, high impact: SqlClient defaults to `Encrypt=True`. <https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-7.0/breaking-changes#encrypt-true> | Connection strings in `src/{AspNetIdentity,EntityFramework}/host/appsettings.json`, `src/*/migrations/SqlServer/appsettings.json`, `src/EntityFramework.Storage/host/ConsoleHost/Program.cs:14`, and the two `test/**/DatabaseProviderBuilder.cs` files | LocalDB strings work unchanged (probe: tests pass against `(LocalDb)\MSSQLLocalDB` with SqlClient 6.1.7). The `Server=(local)\SQL2016;...User ID=sa` string in `src/EntityFramework/test/IdentityServer4.EntityFramework.Tests/DatabaseProviderBuilder.cs:39` targets a real instance and would need a trusted certificate or `Encrypt=False`. Not tested. |
| 5.3 | EF Core 8: `Contains` over a parameter list used `OPENJSON` (needs SQL Server compatibility level 130). EF Core 10 changes the default again to one parameter per value. <https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-8.0/breaking-changes#sqlserver-contains-compatibility>, <https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/breaking-changes#parameterized-collections> | `ResourceStore.cs:57,94,124,151` | Works on EF Core 10 (in-memory provider only in the probe). The generated SQL differs from 3.1, which inlined constants. Relevant to the benchmark baseline, not to correctness. |
| 5.4 | EF Core 6: `dotnet-ef` targets .NET 6 and later. <https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-6.0/breaking-changes#dotnet-ef> | `.config/dotnet-tools.json` pins `dotnet-ef` 3.1.0, used by `src/EntityFramework/migrations.bat`, `updatedb.bat`, `src/EntityFramework.Storage/migrations/SqlServer/*.bat`, `src/AspNetIdentity/migrations/SqlServer/builddb.bat` | Probe: `dotnet ef` 3.1.0 fails to start on a machine with only .NET 10 ("install Microsoft.NETCore.App 3.1.0"). Pin `dotnet-ef` 10.0.15. |
| 5.5 | EF Core 10: `Microsoft.Data.SqlClient` 6.1.x adds `Microsoft.Identity.Client.Broker`, which enlarges published output. <https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/breaking-changes#sqlclient-broker-dependency> | Hosts and tests that reference `Microsoft.EntityFrameworkCore.SqlServer` | Size only. The page names SqlClient 7.1.x as the mitigation. |
| 5.6 | EF Core 6: the in-memory provider validates required properties; SQLite connections are pooled. <https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-6.0/breaking-changes> | `src/EntityFramework.Storage/test/IntegrationTests/`, `src/EntityFramework/test/` | Not hit: all tests pass in stage C. |
| 5.7 | EF Core 9: tools need `Microsoft.EntityFrameworkCore.Design` to be resolvable. <https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-9.0/breaking-changes#tools-design> | `src/EntityFramework.Storage/migrations/SqlServer/SqlServer.csproj`, `src/AspNetIdentity/migrations/SqlServer/SqlServer.csproj` (both reference Design with `PrivateAssets="All"`) | Not hit: the tools ran in the probe. `src/EntityFramework/migrations/SqlServer/SqlServer.csproj` has no Design reference and no migrations; not exercised. |

Opportunity, not a break: `ExecuteDelete` (EF Core 7) can replace the load-then-`RemoveRange` loops in `TokenCleanupService.cs:78-125`. That belongs to the performance tickets.

## 6. ASP.NET Core Identity

`src/AspNetIdentity/src` is six files that use `UserManager`, `SignInManager`, `IUserClaimsPrincipalFactory`, `SecurityStampValidatorOptions.OnRefreshingPrincipal` and `CookieAuthenticationOptions`. It compiles against ASP.NET Core 10 with no errors and no compiler warnings.

- Types that derive from `SecurityStampValidator<TUser>` must drop the `ISystemClock` constructor parameter from .NET 8 (<https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/8/isystemclock-obsolete>). The library only sets `SecurityStampValidatorOptions.OnRefreshingPrincipal` (`src/AspNetIdentity/src/IdentityServerBuilderExtensions.cs:42-44`) and does not derive from it, so it is not affected.
- The Identity EF model for the host's `ApplicationDbContext` is unchanged between the 2020 snapshot and Identity 10.0.15 with default options (5.1).
- The stale entry `<PackageReference Update="Microsoft.AspNetCore.Identity" Version="$(FrameworkVersion)" />` in `src/Directory.Build.targets` refers to a package whose last version is 2.3.x; Identity has shipped in the shared framework since 3.0. No project includes it, so it is harmless and can go.
- There is no test project for `AspNetIdentity` in `src/`, and its host was not run. Run-time behaviour of this library on .NET 10 is therefore unverified (section 9).

## 7. SDK and MSBuild

| # | Change | Where | Result and replacement |
|---|---|---|---|
| 7.1 | `global.json` roll-forward | `global.json` | See 1.1. |
| 7.2 | .NET 10 SDK: NuGet audit covers transitive packages by default for projects that target `net10.0` (`NuGetAuditMode=all`). <https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/10.0/nugetaudit-transitive-packages>; audit itself arrived in SDK 8: <https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/8.0/dotnet-restore-audit> | Every project | Stage A reports NU1902/NU1903 for `Newtonsoft.Json` 11.0.2 and 12.0.2, `AutoMapper` 10.0.0, `Microsoft.Data.SqlClient` 1.0.19269.1, `System.IdentityModel.Tokens.Jwt` and `Microsoft.IdentityModel.JsonWebTokens` 5.6.0, `SQLitePCLRaw.lib.e_sqlite3` 2.0.2. After stage C only `Newtonsoft.Json` (GHSA-5crp-9r3c-p9vr, via `IdentityModel` 4.4.0 and the direct 12.0.2 pin) and `AutoMapper` (GHSA-rvv3-g6hj-g44x) remain. These are warnings; they become errors if warnings are ever treated as errors. Issue #3 owns the vulnerabilities. |
| 7.3 | .NET 10 SDK: NU1510 for a direct reference to a package the framework already provides. <https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/10.0/nu1510-pruned-references> | `System.Security.Principal.Windows` in `src/IdentityServer4/host/Host.csproj`, `src/EntityFramework/host/Host.csproj`, `src/AspNetIdentity/host/Host.csproj`, and its version entry in `src/Directory.Build.targets` | Remove the reference. |
| 7.4 | .NET 10 SDK: a `PackageReference` with no version is error NU1015 (it was warning NU1604). <https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/10.0/nu1015-packagereference-version> | The repo's pattern: `Include` without a version in each `.csproj`, `Update ... Version` in `src/Directory.Build.targets` | Not hit: restore succeeds because every included package has an `Update` entry. A package added without one now fails restore instead of warning. Central Package Management is the documented design that avoids this. |
| 7.5 | .NET SDK 8: Source Link is included in the SDK. <https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/8.0/source-link> | `Microsoft.SourceLink.GitHub` 1.0.0 in the five library `.csproj` files and `src/Directory.Build.targets` | The reference is redundant. |
| 7.6 | .NET SDK 8: smaller runtime-identifier graph (NETSDK1206). <https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/8.0/rid-graph> | Old `SQLitePCLRaw.lib.e_sqlite3` 2.0.2 brought in by EF Core SQLite 3.1 | Appears in stage A only. Gone once EF Core is 10.0.x. |
| 7.7 | .NET 10 SDK: `dotnet new sln` creates `.slnx`. <https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/10.0/dotnet-new-sln-slnx-default> | The five `.sln` files under `src/` | Existing `.sln` files keep working (the per-library solutions were not rebuilt individually; the probe used one generated `.sln`). Only matters when a new solution is created. |
| 7.8 | Local tools | `.config/dotnet-tools.json` | `dotnet-ef` 3.1.0 does not start (5.4). `SignClient` 1.2.17 does start on .NET 10 (probe). |
| 7.9 | Build runner and test packages | `src/*/build/build.csproj` (Bullseye 3.3.0, SimpleExec 6.2.0), MinVer 2.3.0, `Microsoft.NET.Test.Sdk` 16.2.0, `xunit` 2.4.1, `xunit.runner.visualstudio` 2.4.1, `FluentAssertions` 5.10.2 | All work on `net10.0` unchanged: the runner lists its targets, MinVer 2.3.0 runs (it fell back to its default version because the probe copy is not a git repository), and stage A ran 1,017 tests on the 2019-era test packages. Upgrading them is a choice, not a requirement of .NET 10. |
| 7.10 | SDK 7 and 8 CLI changes: `--output` is not valid for solution-level commands; `dotnet pack` defaults to Release. <https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/7.0/solution-level-output-no-longer-valid>, <https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/8.0/dotnet-pack-config> | `src/build/Program.Partial.cs:60` | Not hit: the script packs a project file, not a solution, and passes `-c Release` explicitly. |

## 8. Checked and not affected

Each of these appears in the official lists and touches an area the code uses. None changes behaviour here.

- **ASP.NET Core 10: cookie login redirects disabled for known API endpoints** (<https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/10/cookie-authentication-api-endpoints>). Applies to `[ApiController]`, minimal-API JSON endpoints and SignalR. `src/` has none; IdentityServer's protocol endpoints are middleware and issue their own redirects.
- **ASP.NET Core 7: a single registered authentication scheme becomes the default** (<https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/7/default-authentication-scheme>). `AddIdentityServer` always registers two cookie schemes and an explicit default (`src/IdentityServer4/src/Configuration/DependencyInjection/BuilderExtensions/Core.cs:60-62`).
- **ASP.NET Core 8.0.17, 9.0.6 and later: forwarded-headers middleware ignores `X-Forwarded-*` from unknown proxies** (<https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/8/forwarded-headers-unknown-proxies>). The change covers apps that did not enable `X-Forwarded-For`. The only use in `src/` is `src/IdentityServer4/host/Startup.cs:107-110`, which enables `XForwardedFor | XForwardedProto`, so the trusted-proxy check already applied to it. It sets no `KnownProxies`, so only a loopback proxy is trusted, as before. `KnownNetworks` is obsolete in .NET 10 in favour of `KnownIPNetworks` (<https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/10/ipnetwork-knownnetworks-obsolete>); the host does not use it.
- **.NET 9: `HostBuilder` turns on `ValidateOnBuild` and `ValidateScopes` in Development** (<https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/9/hostbuilder-validation>). Probe: `src/IdentityServer4/host` starts in Development on .NET 10 with no validation error, and the integration tests pass. The `EntityFramework` and `AspNetIdentity` hosts were not started (they need a database).
- **ASP.NET Core 9: middleware types with multiple constructors** (<https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/9/middleware-constructors>). `BaseUrlMiddleware`, `MutualTlsEndpointMiddleware` and `IdentityServerMiddleware` each have one constructor.
- **ASP.NET Core 6: `ClientCertificate` no longer triggers renegotiation on HttpSys** (<https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/6/clientcertificate-doesnt-trigger-renegotiation>). The code already calls `GetClientCertificateAsync()` (`DefaultTokenService.cs:242`, `MutualTlsSecretParser.cs:69`).
- **ASP.NET Core 10: `IActionContextAccessor` and Razor runtime compilation obsolete** (<https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/10/iactioncontextaccessor-obsolete>, <https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/10/razor-runtime-compilation-obsolete>). Neither is used in `src/`.
- **.NET 5: `Create()` overloads on cryptographic abstractions obsolete** (SYSLIB0007, <https://learn.microsoft.com/en-us/dotnet/core/compatibility/cryptography/5.0/instantiating-default-implementations-of-cryptographic-abstractions-not-supported>). The code calls `SHA256.Create()`, `SHA384.Create()`, `SHA512.Create()`, `RSA.Create(int)` and `ECDsa.Create(ECCurve)`, which are the supported forms (`src/IdentityServer4/src/Configuration/CryptoHelper.cs`, `Extensions/HashExtensions.cs`). No warning in the probe.
- **.NET 10: `X509Certificate` key parameters can be null; stricter `X500DistinguishedName` validation** (<https://learn.microsoft.com/en-us/dotnet/core/compatibility/cryptography/10.0/x509-publickey-null>). The code reads thumbprints, `RawData`, `GetCertHash` and `Subject` only.
- **System.Text.Json changes in .NET 7 and 10** (polymorphic `object` serialisation, <https://learn.microsoft.com/en-us/dotnet/core/compatibility/serialization/7.0/polymorphic-serialization>; property-name conflict checks, <https://learn.microsoft.com/en-us/dotnet/core/compatibility/serialization/10/property-name-validation>). The token, discovery, introspection and userinfo responses are covered by the integration tests, which pass.
- **C# 9 to 14 language changes.** No compile error in any of the 825 source files.

## 9. Not verified

- **Baseline on .NET Core 3.1.** The 3.1 runtime is not installed, so "the tests are green today" is assumed, not observed. Stage A (old packages, new runtime) is the nearest evidence.
- **SDK 10.0.404 and runtime 10.0.15.** The probe ran on SDK 10.0.403 and runtime 10.0.14 with 10.0.15 packages.
- **Linux and macOS.** Everything ran on Windows. Not checked: OpenSSL 1.1.1 minimum on Unix (<https://learn.microsoft.com/en-us/dotnet/core/compatibility/cryptography/10.0/openssl-version-requirement>), `PublicSign` builds, and the SQLite path of `EntityFramework.IntegrationTests`, which on Windows runs the in-memory provider only (`src/EntityFramework.Storage/test/IntegrationTests/IntegrationTest.cs:33-38`).
- **SQL Server other than LocalDB**, including the `Encrypt=True` default against a real instance and the SQL generated for the `Contains` queries in `ResourceStore.cs` (4.2, 5.3). Those queries ran on the in-memory provider only.
- **Mutual TLS end to end**, IIS and HttpSys hosting, and proxy deployments. Only code paths covered by the existing tests ran.
- **`AspNetIdentity` and `EntityFramework` hosts at run time**, and the `AspNetIdentity` library, which has no tests in `src/`.
- **The integration-test count after fixing 2.2 a and b.** The fixes themselves were not written; only their causes were reproduced.
- **Whether `ISystemClock` stays registered in future servicing releases.** It is registered in `release/10.0` today.
- **Performance.** No measurements were taken.

## 10. Questions that need a human decision

1. **Port in one step or two?** Stage A shows that the core library can move to `net10.0` with tests green while still on IdentityModel 5.6. That is an unsupported combination, but it would separate "runtime change" from "IdentityModel change" in the history. The EF libraries cannot do this (1.3).
2. **Lenient JWK JSON.** IdentityModel 8 rejects JWK secrets with single quotes or trailing commas (2.2 c). Accept the stricter behaviour and change the three fixtures, or add tolerant parsing? This changes tests, so the standing preference requires a recorded decision.
3. **AutoMapper.** Its 10.x line is dead on .NET 7 and later, and 12.0.1 carries an open advisory. Bump for the behaviour-preserving port and replace later, or replace during the port? (Feeds issues #6 and #14.)
4. **`ISystemClock` in the public API.** It appears in public constructors and in the public extension method `CreateJwtPayload(this Token, ISystemClock, ...)`. Moving to `TimeProvider` is a public API change. Do it in the port, or carry the warnings until the conventions sweep?
5. **Warnings policy.** About 150 compiler and analyzer warnings, plus NuGet audit warnings, exist after the port. Should the port adopt `TreatWarningsAsErrors`, and if so, should the NuGet audit codes (NU1901 to NU1904) be exempt so that a newly published advisory cannot break the build?
6. **`global.json` roll-forward policy** (`latestFeature`, `latestPatch` or `disable`), and which SDK feature band to pin (4xx or 1xx).
