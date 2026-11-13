# Built-in .NET 10 replacements for each third-party dependency

Research for issue #6. Facts as of **2026-11-13**. Scope is `src/` only (five libraries, their tests, hosts, migrations and build projects).

This note presents facts and trade-offs. It does not decide replace-or-keep for any dependency; that belongs to a later conversation.

How the facts were gathered:

- **Usage counts** come from `grep` over `src/**/*.cs` and the `src/**/*.csproj` files on `main` (commit `5e40bd18`), excluding `bin/` and `obj/`. Counts are of files or textual matches, so treat them as sizes, not exact call-site inventories.
- **Versions, licences, target frameworks, deprecations and advisories** come from the NuGet registration API (`https://api.nuget.org/v3/registration5-gz-semver2/<id>/index.json`), read on 2026-11-13. The matching human-readable page is `https://www.nuget.org/packages/<id>/<version>`.
- **Repository ownership, archive status and licence files** come from the GitHub API for each project's own repository.
- **.NET API availability** was checked against the locally installed .NET 10 reference packs (`Microsoft.NETCore.App.Ref` and `Microsoft.AspNetCore.App.Ref` 10.0.14). The learn.microsoft.com link given beside each API is where it is documented.
- **Behavioural claims marked "probe"** were observed by running a throwaway `net10.0` console app against `System.IdentityModel.Tokens.Jwt` 8.23.0 and `Newtonsoft.Json` 13.0.4 (see [Appendix A](#appendix-a-probe-results)). The probe is not committed.

"Supports net10.0" below means the package ships a `net10.0` asset, or an asset NuGet treats as compatible (`net8.0`, `net6.0`, `netstandard2.0`). Apart from the probe, nothing here was compiled against `net10.0`.

## Summary

| Dependency (pin in `src/`) | Usage in `src/` | Built-in / first-party candidate | Main gap for this codebase | Licence and maintenance, 2026-11-13 |
| --- | --- | --- | --- | --- |
| **Newtonsoft.Json** 12.0.2 | 24 files: 4 Storage (persisted-grant serializer and converters), 3 core (JWT payload, JAR payload, dev key file), 3 hosts (one line each), 14 integration tests | `System.Text.Json` (in-box), optionally source-generated | Default deserialization replaces pre-initialised collections, dropping `Token.Claims`' `ClaimComparer`; `Claim` and `ClaimsPrincipal` still need converters; stored grant JSON should be assumed to change | MIT. Active: 13.0.4 (2025-09-16), 14.0.1-beta2 (2026-09-21) adds a `net10.0` asset. Pinned 12.0.2 has a high-severity advisory |
| **AutoMapper** `[10.0.0,11.0)` | 11 files, 473 lines, all in `EntityFramework.Storage/src/Mappers`; 12 call sites in 3 stores | None in .NET. Hand-written mapping code | Five profile classes (about 210 lines) become explicit code for 5 aggregates (Client alone has 47 model properties); the `AssertConfigurationIsValid` safety net goes away | **No longer MIT.** 15.0+ is RPL-1.5 or commercial (Lucky Penny Software). Every MIT version, including the pinned range, is inside a high-severity advisory with no MIT fix |
| **IdentityModel** 4.4.0 | `using IdentityModel` in 155 files; ~340 `OidcConstants` and ~190 `JwtClaimTypes` references in library code; 19 integration-test files use `IdentityModel.Client` | BCL covers the helpers (`Base64Url`, `FixedTimeEquals`, `GetHexString`, `X509Store`). No in-box equivalent for the protocol constants or the test client | Constants would have to be owned in-repo (the two source files are Apache-2.0); integration tests would need a hand-written protocol client | **Deprecated and unlisted** on NuGet; repo archived. Successor `Duende.IdentityModel` 8.1.0, Apache-2.0, `net10.0` asset, active |
| **System.IdentityModel.Tokens.Jwt** `[5.6.0,6.0)` and **Microsoft.IdentityModel.Protocols.OpenIdConnect** 5.6.0 | 17 core files, 1 host file, 12 test files. `JwtSecurityTokenHandler` created in 4 places | Same family at 8.x; `JsonWebTokenHandler` / `JsonWebToken` is the newer handler. .NET itself has no JWT API | 8.x swapped Newtonsoft for `System.Text.Json`: the current payload code **silently emits wrong JSON** for JSON-valued claims, and JAR JSON parameters are silently dropped (probe). Public extension points expose `JwtHeader`/`JwtPayload`/`JwtSecurityToken` | MIT, Microsoft. 8.23.0 (2026-09-18) has a `net10.0` asset. 5.6.0 is listed as out of support and has a medium advisory. 8.x arrives transitively through the ASP.NET Core OIDC handler anyway |
| **Polly** via `Microsoft.Extensions.Http.Polly` | **0 files.** Version pin only; no project references it | `Microsoft.Extensions.Http.Resilience` (Microsoft, built on Polly) | Nothing to replace. Deleting the pin removes it | Polly: BSD-3-Clause, 8.9.1 (2026-11-07). Both Microsoft packages current, MIT |
| **Serilog** (+ AspNetCore, Sinks.Console, Sinks.File) | 4 files in 3 hosts: bootstrap logger, `UseSerilog()`, one `UseSerilogRequestLogging()` | `Microsoft.Extensions.Logging` console provider; `UseHttpLogging()` | No output-template formatting, no built-in file provider (file sink is already commented out) | Apache-2.0, active. Serilog 4.4.0, Serilog.AspNetCore 10.0.0, both with `net10.0` assets |
| **System.Security.Principal.Windows** 4.7.0 | 3 host csproj references; **0 code uses** | In the shared framework | None. On `net10.0` the reference raises NU1510 | MIT. Last package 5.0.0 (2020-11-09); the assembly ships in-box |
| **Bullseye** 3.3.0 and **SimpleExec** 6.2.0 | 5 `build.csproj`, one shared 104-line `Program.Partial.cs`, 9 targets | Plain `dotnet build/test/pack`; a .NET 10 file-based app; MSBuild targets | Target dependency graph and CLI are Bullseye's; a script would order steps by hand | Apache-2.0, active. Bullseye 6.3.0, SimpleExec 13.1.0, both with `net10.0` assets |
| **MinVer** 2.3.0 | 1 package reference applied to every project, plus one MSBuild target reading `MinVer*` properties | None in the SDK for tag-derived versions. Static `<Version>` or CI-supplied `-p:Version` | Version would no longer follow git tags automatically | Apache-2.0, active. 8.0.0 (2026-09-05); README prerequisite is .NET SDK 10.0 or later |
| **xunit** 2.4.1 (+ runner, `Microsoft.NET.Test.Sdk`) | 5 test projects, 129 files, 920 test methods | `xunit.v3` on Microsoft.Testing.Platform (xunit is not Microsoft's); MSTest is the Microsoft-authored framework | 24 `async void` tests fail fast in v3; projects become executables. MSTest would mean rewriting attributes on 920 tests | Apache-2.0, .NET Foundation. v2 packages **deprecated** (security fixes only). `xunit.v3` 4.0.3 (2026-11-09) |
| **FluentAssertions** 5.10.2 | 121 files, 2,657 `.Should()` calls | xunit's own `Assert` | Mechanical rewrite of 2,657 assertions; 92% use six methods | **8.x is not open source** (Xceed, paid for commercial use). 7.x stays Apache-2.0 (7.2.2, 2026-03-16) and gets fixes. Pinned 5.10.2 is Apache-2.0 |

Also pinned in `src/Directory.Build.targets` but worth a line each:

- `Microsoft.AspNetCore.Mvc.NewtonsoftJson`: pinned, referenced by no project in `src/`. There is no Newtonsoft MVC formatter in `src/`.
- `Microsoft.SourceLink.GitHub` 1.0.0: referenced by all five libraries. Source Link has been part of the .NET SDK since .NET 8, so the reference is redundant on a .NET 10 SDK ([breaking-change note](https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/8.0/source-link)).

## Cross-cutting findings

1. **.NET 11 shipped three days ago.** .NET 11.0.0 was released 2026-11-10 as an STS release. Microsoft's support page lists both .NET 10 (LTS) and .NET 11 (STS) with the same end of support, 2028-11-14, and .NET 8 and 9 reached end of life on 2026-11-10 ([support policy](https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core), [release index](https://github.com/dotnet/core/blob/main/release-notes/releases-index.json)). Several packages already publish `net11.0` assets (AutoMapper 17.0.0, `Microsoft.Extensions.Http.Resilience` 11.0.0). Whether "`net10.0` only" still stands in light of this is a question for the plan, not for this note.
2. **Three pins are dead weight.** `Microsoft.Extensions.Http.Polly`, `Microsoft.AspNetCore.Mvc.NewtonsoftJson` and (in code terms) `System.Security.Principal.Windows` have no users in `src/`. Removing them needs no replacement.
3. **Microsoft.IdentityModel 8.x is not optional.** `Microsoft.AspNetCore.Authentication.OpenIdConnect` 10.0.15 depends on `Microsoft.IdentityModel.Protocols.OpenIdConnect >= 8.19.2`, which depends on `System.IdentityModel.Tokens.Jwt`. The core library references that ASP.NET Core package, so the 8.x family, including `System.IdentityModel.Tokens.Jwt`, is in the graph whatever is decided about direct use.
4. **The Newtonsoft and Microsoft.IdentityModel questions are one question.** The core library uses Newtonsoft `JToken` types only because Microsoft.IdentityModel 5.x used Newtonsoft internally. Moving to 8.x forces those three files onto `System.Text.Json` types.
5. **Known advisories on current pins:** Newtonsoft.Json 12.0.2 ([GHSA-5crp-9r3c-p9vr](https://github.com/advisories/GHSA-5crp-9r3c-p9vr), high, fixed in 13.0.1), System.IdentityModel.Tokens.Jwt 5.6.0 ([GHSA-59j7-ghrg-fj52](https://github.com/advisories/GHSA-59j7-ghrg-fj52), medium, fixed in 5.7.0 / 6.34.0 / 7.1.2), AutoMapper 10.x ([GHSA-rvv3-g6hj-g44x](https://github.com/advisories/GHSA-rvv3-g6hj-g44x), high, fixed only in 15.1.1 and 16.1.1).

---

## Newtonsoft.Json

### Usage in `src/`

24 `.cs` files. `IdentityServer4.csproj` is the only project with a direct `PackageReference`. `IdentityServer4.Storage` uses Newtonsoft without referencing it: the package arrives transitively through IdentityModel 4.4.0, whose only dependencies are `Newtonsoft.Json >= 11.0.2` and `System.Text.Encodings.Web` ([NuGet](https://www.nuget.org/packages/IdentityModel/4.4.0)).

| Where | Files | Kind of use |
| --- | --- | --- |
| `src/Storage/src/Stores/Serialization/` | `PersistentGrantSerializer.cs`, `ClaimConverter.cs`, `ClaimsPrincipalConverter.cs`, `CustomContractResolver.cs` | The persisted-grant serializer. `JsonSerializerSettings` with two custom `JsonConverter`s (`Claim` and `ClaimsPrincipal`, both via `ClaimLite` / `ClaimsPrincipalLite` DTOs) and a `DefaultContractResolver` subclass that keeps only writable properties. Serializes `AuthorizationCode`, `RefreshToken`, `Token`, `Consent`, `DeviceCode`. Consumed through `IPersistentGrantSerializer` by 6 core stores and services and by `EntityFramework.Storage`'s `DeviceFlowStore` |
| `src/IdentityServer4/src/Extensions/TokenExtensions.cs` | 1 | `JRaw.Parse` on JSON-valued claims, then `JTokenType`, `JArray`, `JToken` to merge objects and arrays into the `JwtPayload` |
| `src/IdentityServer4/src/Validation/Default/JwtRequestValidator.cs` | 1 | `case JObject` / `case JArray` on `JwtSecurityToken.Payload` values, re-serialized with `Formatting.None` |
| `src/IdentityServer4/src/Configuration/DependencyInjection/BuilderExtensions/Crypto.cs` | 1 | `JsonConvert.SerializeObject(jwk)` to write `tempkey.jwk` |
| `src/*/host/Quickstart/Diagnostics/DiagnosticsViewModel.cs` | 3 (identical) | One `JsonConvert.DeserializeObject<string[]>` call each |
| `src/IdentityServer4/test/IdentityServer.IntegrationTests/` | 14 | `JObject` (17), `JArray` (41), `JsonConvert` (19) to inspect token payloads and responses, much of it because IdentityModel 4.x exposes responses as `JObject` |

There is no custom MVC formatter and no `AddNewtonsoftJson()` call in `src/`.

The core library already uses `System.Text.Json` in 7 files, including the response writers (`TokenResult.cs`, `TokenErrorResult.cs` with `[JsonExtensionData]`), `Infrastructure/ObjectSerializer.cs` and `Logging/LogSerializer.cs`. So the codebase is already half-migrated; Newtonsoft remains only where Microsoft.IdentityModel 5.x or the grant store forces it.

### Replacement: `System.Text.Json`

In the shared framework, no package needed. Microsoft's migration guide marks every feature the grant serializer uses as supported ([migrate from Newtonsoft.Json](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/migrate-from-newtonsoft)):

| Newtonsoft feature used here | `System.Text.Json` status per the guide |
| --- | --- |
| Custom converter registration | Supported; "order of precedence differs" |
| `DefaultContractResolver` to ignore properties | Supported via `DefaultJsonTypeInfoResolver` ([custom contracts](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/custom-contracts)) |
| `ObjectCreationHandling` global setting | Supported: "reuse rather than replace properties" |
| `JObject` / `JArray` / `JToken` | `JsonNode`, `JsonDocument`, `JsonElement` ([DOM](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/use-dom)) |
| Support for a broad range of types | Partial: "some types require custom converters" |
| Deserialize inferred type to `object` properties | Not supported; values arrive as `JsonElement` |

Source generation (`JsonSerializerContext`, `[JsonSerializable]`) is available and documented ([source generation](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/source-generation)); it is what makes a serializer trim- and AOT-safe.

### Gaps for this codebase

1. **Pre-initialised collections lose their comparer by default.** `Token.Claims` is declared `ICollection<Claim>` and initialised to `new HashSet<Claim>(new ClaimComparer())` (`src/Storage/src/Models/Token.cs:114`). Newtonsoft fills the existing set. `System.Text.Json` by default allocates a new collection and assigns it. Probe, with the same shape (`ICollection<string>` initialised to a case-insensitive `HashSet`): Newtonsoft returned `HashSet` with duplicates collapsed, `System.Text.Json` returned `List` with duplicates kept. Setting `PreferredObjectCreationHandling = JsonObjectCreationHandling.Populate` (available since .NET 8) restored the `HashSet` and its comparer ([populate properties](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/populate-properties)). The same page notes populate "currently doesn't work for types that have a parameterized constructor".
2. **Computed properties are written unless told otherwise.** `Token` and `RefreshToken` expose get-only computed properties (`SubjectId`, `SessionId`, `Scopes`, and on `RefreshToken` also `ClientId`, `Description`). `CustomContractResolver` exists to drop them. The `System.Text.Json` equivalent is `IgnoreReadOnlyProperties = true` (probe confirmed it drops a get-only property), so the custom resolver class has no reason to survive.
3. **`Claim` and `ClaimsPrincipal` still need converters.** Probe: `JsonSerializer.Deserialize<Claim>` throws `NotSupportedException` (no usable constructor). The existing `ClaimLite` / `ClaimsPrincipalLite` DTO design carries over; only the converter base class changes.
4. **Stored JSON format.** The grant payload's property names and null handling follow Newtonsoft defaults today. Whether a `System.Text.Json` serializer would read rows written by the current one was not tested. ADR 0001 says no compatibility is owed, so this is a note, not a blocker.
5. **Obsolete API already in use.** `ObjectSerializer.cs` sets `JsonSerializerOptions.IgnoreNullValues`, obsolete since .NET 6 ([SYSLIB0020](https://learn.microsoft.com/en-us/dotnet/fundamentals/syslib-diagnostics/syslib0020)); the replacement is `DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull`.
6. **The three core files move with Microsoft.IdentityModel, not on their own.** See the next-but-two section.
7. **Tests.** The 14 integration-test files parse JWT payloads into `Dictionary<string, object>` with Newtonsoft and cast values to `JObject` / `JArray`. With `System.Text.Json` those values are `JsonElement`, so each assertion on a nested value is rewritten.

### Status

- Licence MIT ([NuGet](https://www.nuget.org/packages/Newtonsoft.Json/13.0.4), [repo](https://github.com/JamesNK/Newtonsoft.Json)).
- Latest stable 13.0.4, published 2025-09-16, assets for `net6.0` and `netstandard2.0`. Latest prerelease 14.0.1-beta2, published 2026-09-21, adds `net8.0` and `net10.0` assets. Repository last pushed 2026-11-10. So it is maintained, at a slow cadence.
- Pinned 12.0.2 (2019) is affected by [GHSA-5crp-9r3c-p9vr](https://github.com/advisories/GHSA-5crp-9r3c-p9vr) (CVE-2024-21907, high), fixed in 13.0.1.

---

## AutoMapper

### Usage in `src/`

Confined to one folder of one library: `src/EntityFramework.Storage/src/Mappers/`, 11 files, 473 lines.

- 5 `Profile` classes (`ClientMapperProfile`, `ApiResourceMapperProfile`, `ScopeMapperProfile`, `IdentityResourceMapperProfile`, `PersistedGrantMapperProfile`).
- 5 static classes, each holding a private `IMapper` and exposing `ToModel()` / `ToEntity()` (and `UpdateEntity()` for persisted grants).
- 1 value converter, `AllowedSigningAlgorithmsConverter` (comma-separated string to `ICollection<string>` and back).

Features used across the profiles: `CreateMap` with `MemberList.Destination` / `MemberList.None`, `ReverseMap` (22), `ForMember` (20), `ConstructUsing` (14), `MapFrom` (13), `ConvertUsing` (4), `Condition` (3). No `ProjectTo`, no DI registration, no `IMapper` injection.

Call sites of `ToModel` / `ToEntity` / `UpdateEntity` outside the folder: 12 in three store classes (`ClientStore.cs` 1, `PersistedGrantStore.cs` 4, `ResourceStore.cs` 7), 4 in `src/EntityFramework/migrations/SqlServer/SeedData.cs`, about 56 in tests. Five mapper unit-test files (332 lines) each call `AssertConfigurationIsValid`.

Size of what is mapped: `Models.Client` has 47 settable properties against 53 on `Entities.Client`, including nine child collections, six of which map between `string` and a one-column entity. The other four aggregates are much smaller (`PersistedGrant` is 10 properties, one-to-one).

### Replacement

.NET has no object-to-object mapper, in the box or as a Microsoft package. The built-in option is hand-written mapping methods behind the existing `ToModel()` / `ToEntity()` signatures, which would leave the 72 call sites untouched.

For reference, not as a built-in: Mapperly is a Roslyn source generator that emits such methods at compile time. It is third-party (Riok), Apache-2.0, latest stable 4.3.1 published 2025-12-22 ([NuGet](https://www.nuget.org/packages/Riok.Mapperly/4.3.1), [repo](https://github.com/riok/mapperly)).

### Gaps

- The convention-based matching of 47 `Client` properties becomes explicit assignments in each direction, plus the nine collection projections. A new property on a model is silently unmapped unless a test catches it; today `AssertConfigurationIsValid` catches that for destination members.
- `ReverseMap` means each profile line currently defines two mappings; hand-written code states both.
- Nothing in the profiles depends on AutoMapper runtime behaviour that code cannot express directly.

### Status

- **Ownership and licence changed.** The repository is now `LuckyPennySoftware/AutoMapper`. Its `LICENSE.md` states that the code is governed by the Reciprocal Public License 1.5, and that those who do not wish to release their own source under those terms may use it under a commercial licence ([LICENSE.md](https://github.com/LuckyPennySoftware/AutoMapper/blob/main/LICENSE.md)).
- The author's launch post (2025-07-02) names v15.0 as the first release under the dual licence and describes a free "Community" tier of the commercial licence for organisations under USD 5,000,000 gross annual revenue, non-profits under the same budget, education, and non-production use. A licence key is still required "for auditing purposes"; a missing key does not stop the library working but produces log messages ([post](https://www.jimmybogard.com/automapper-and-mediatr-commercial-editions-launch-today/)).
- NuGet metadata agrees: versions up to 14.0.0 (2025-02-14) carry licence expression `MIT`; 15.0.1 onward carry a licence file instead of an SPDX expression. Latest is 17.0.0, published 2026-11-12, with `net10.0` and `net11.0` assets ([NuGet](https://www.nuget.org/packages/AutoMapper/17.0.0)). 17.0.0 also takes a dependency on `Microsoft.IdentityModel.JsonWebTokens`.
- **Advisory with no MIT fix.** [GHSA-rvv3-g6hj-g44x](https://github.com/advisories/GHSA-rvv3-g6hj-g44x) (CVE-2026-32933, high, published 2026-03-13): denial of service by stack overflow when mapping deeply nested, self-referential object graphs. Affected: all versions below 15.1.1, and 16.0.0 to 16.1.0. Patched: 15.1.1 and 16.1.1. Every MIT-licensed version, including the pinned `[10.0.0,11.0)` range and 14.0.0, is in the affected range. Whether this codebase's maps are reachable by that attack was not assessed; the mapped models are not self-referential.
- The last MIT version, 14.0.0, ships only a `net8.0` asset (usable from `net10.0`).
- Whether RPL-1.5 is compatible with distributing this Apache-2.0 library is a legal question this note does not answer.

---

## IdentityModel

### Usage in `src/`

The most widely used third-party dependency by file count. `using IdentityModel` (or a sub-namespace) appears in 155 files: 68 in the core library, 3 in Storage, 3 in AspNetIdentity, 1 in EntityFramework.Storage, 61 in core tests, 2 in EF tests, 15 in hosts, 2 in migrations.

In library code it is used in four distinct ways:

| Kind | Size | Examples |
| --- | --- | --- |
| Protocol constants, `OidcConstants.*` | ~340 references across 22 nested classes; top five are `TokenErrors` (67), `AuthorizeErrors` (46), `AuthorizeRequest` (37), `Discovery` (31), `ResponseTypes` (30) | `src/IdentityServer4/src/Validation/Default/AuthorizeRequestValidator.cs`, `TokenRequestValidator.cs`, `ResponseHandling/Default/DiscoveryResponseGenerator.cs` |
| Claim-type constants, `JwtClaimTypes.*` | ~190 references | `src/Storage/src/Models/Token.cs`, `src/Storage/src/IdentityServerUser.cs`, `src/AspNetIdentity/src/UserClaimsFactory.cs` |
| Crypto and encoding helpers | `Base64Url.Encode/Decode` at 19 sites, `CryptoRandom.CreateUniqueId` at 8, `TimeConstantComparer.IsEqual` at 5, `X509.CurrentUser/LocalMachine...Find` at 4 | `Configuration/CryptoHelper.cs`, `Validation/Default/TokenRequestValidator.cs`, `Validation/Default/HashedSharedSecretValidator.cs`, `Services/Default/DefaultHandleGenerationService.cs` |
| HTTP protocol client, `IdentityModel.Client` | 19 integration-test files only; none in library code | `RequestClientCredentialsTokenAsync` (35), `RequestPasswordTokenAsync` (27), `AuthorizeResponse` (28), `IntrospectTokenAsync` (12), `RevokeTokenAsync` (12) |

### Replacement

The helpers have BCL equivalents, all present in the .NET 10 reference pack:

| IdentityModel API | .NET 10 API |
| --- | --- |
| `Base64Url.Encode` / `Decode` | [`System.Buffers.Text.Base64Url.EncodeToString` / `DecodeFromChars`](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.text.base64url) |
| `TimeConstantComparer.IsEqual` | [`CryptographicOperations.FixedTimeEquals`](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.cryptographicoperations.fixedtimeequals) (takes byte spans, so strings are encoded first) |
| `CryptoRandom.CreateUniqueId(n, Hex)` | [`RandomNumberGenerator.GetHexString`](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.randomnumbergenerator.gethexstring) or `RandomNumberGenerator.GetBytes` plus `Convert.ToHexString` |
| `X509.CurrentUser.My.Thumbprint.Find(...)` | `X509Store` plus [`X509Certificate2Collection.Find`](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.x509certificate2collection.find) |

Duende made the first of these substitutions itself: Duende.IdentityModel 8.0.0 removed its `Base64Url` class "using SDK/Runtime provided type instead" and removed its epoch-time extensions in favour of `DateTimeOffset.ToUnixTimeSeconds()` ([release notes](https://github.com/DuendeSoftware/foss/releases/tag/im-8.0.0)). So moving to the successor package also forces the `Base64Url` change at all 19 sites.

For the constants there is no in-box equivalent. Microsoft.IdentityModel has partial overlaps ([`JwtRegisteredClaimNames`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.identitymodel.jsonwebtokens.jwtregisteredclaimnames), [`OpenIdConnectParameterNames`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.identitymodel.protocols.openidconnect.openidconnectparameternames)), but nothing shaped like `OidcConstants.TokenErrors` or `OidcConstants.Discovery`. Whether those two classes cover every constant this codebase uses was not checked member by member.

### Gaps

- **Constants.** Dropping the package means owning `OidcConstants` and `JwtClaimTypes` in this repo. They are two plain source files of string constants under Apache-2.0 (`identity-model/src/IdentityModel/OidcConstants.cs` and `JwtClaimTypes.cs` in [DuendeSoftware/foss](https://github.com/DuendeSoftware/foss)). Hosts use the same constants (15 host files in `src/` import `IdentityModel`), and today they get them transitively from this library's IdentityModel dependency.
- **`CreateUniqueId` output shape.** The replacement must produce the same length and alphabet where handles are stored or compared; not checked in detail.
- **Test client.** Nothing in .NET replaces `IdentityModel.Client`. The 19 integration-test files would need either the Duende package kept as a test-only dependency or a small hand-written client over `HttpClient` and `FormUrlEncodedContent`.
- **`IdentityModel` 4.x to `Duende.IdentityModel` 8.x is not a rename only.** The namespace is now `Duende.IdentityModel`, response objects expose `System.Text.Json` types where 4.x exposed `JObject`, and `Base64Url` is gone. The 155 files change either way.

### Status

- **The `IdentityModel` package is dead.** Every version on NuGet is unlisted and marked deprecated with the message "This package have been rebranded and replaced with Duende.IdentityModel", alternate package `Duende.IdentityModel` ([NuGet](https://www.nuget.org/packages/IdentityModel/4.4.0)). Its last version was 7.0.0. The repository is archived under `DuendeArchive/IdentityModel`, last pushed 2024-11-12 ([repo](https://github.com/DuendeArchive/IdentityModel)).
- **Successor.** `Duende.IdentityModel` 8.1.0, published 2026-03-17, licence expression `Apache-2.0`, assets for `net8.0`, `net10.0`, `netstandard2.0`; the `net10.0` asset has no package dependencies ([NuGet](https://www.nuget.org/packages/Duende.IdentityModel/8.1.0)). Source lives in `DuendeSoftware/foss`, Apache-2.0, last pushed 2026-11-04. The repo README says the rename "doesn't indicate other big changes" and that Duende is "committed to keeping them open source" ([README](https://github.com/DuendeSoftware/foss)).
- Pinned 4.4.0 targets `netstandard2.0` and drags in Newtonsoft.Json.
- Duende Software is the vendor of the commercial successor to IdentityServer4. The licence is Apache-2.0 today; the ownership fact is recorded because the ticket asked for it.

---

## System.IdentityModel.Tokens.Jwt and Microsoft.IdentityModel.*

### Usage in `src/`

17 files in the core library, 1 host file, 12 test files. `IdentityServer4.csproj` references `Microsoft.IdentityModel.Protocols.OpenIdConnect` directly and gets `System.IdentityModel.Tokens.Jwt` through it; the integration-test project references `System.IdentityModel.Tokens.Jwt` directly.

| Kind | Where |
| --- | --- |
| Creating JWTs: `JwtHeader`, `JwtPayload`, `JwtSecurityToken`, `JwtSecurityTokenHandler.WriteToken` | `Services/Default/DefaultTokenCreationService.cs`, `Extensions/TokenExtensions.cs` |
| Validating JWTs: `JwtSecurityTokenHandler.ValidateToken` with `TokenValidationParameters` | `Validation/Default/TokenValidator.cs`, `PrivateKeyJwtSecretValidator.cs`, `JwtRequestValidator.cs` |
| Reading a JWT without validating: `new JwtSecurityToken(string)` | `TokenValidator.cs:416`, `JwtBearerClientAssertionSecretParser.cs:110` |
| Key model: `SecurityKey`, `SigningCredentials`, `RsaSecurityKey`, `ECDsaSecurityKey`, `X509SecurityKey`, `JsonWebKey`, `JsonWebKeyConverter`, `SecurityAlgorithms` | `Configuration/CryptoHelper.cs`, `Configuration/DependencyInjection/BuilderExtensions/Crypto.cs`, `Services/Default/DefaultKeyMaterialService.cs`, `Stores/ISigningCredentialStore.cs`, `Models/SecurityKeyInfo.cs`, `ResponseHandling/Default/DiscoveryResponseGenerator.cs`, `IdentityServerConstants.cs` |
| Claim-type maps: `InboundClaimTypeMap.Clear()`, `MapInboundClaims = false`, `DefaultOutboundClaimTypeMap` | `TokenValidator.cs:249`, `JwtRequestValidator.cs:35`, `Test/TestUserStore.cs:106` |

Several of these types are part of this library's own public and protected API: `ISigningCredentialStore` returns `SigningCredentials`; `DefaultTokenCreationService` has protected virtual methods returning `JwtHeader`, `JwtPayload` and taking `JwtSecurityToken`; `JwtRequestValidator.ValidateJwtAsync` returns `JwtSecurityToken`; `TokenExtensions.CreateJwtPayload` is public.

### Replacement

There is no JWT implementation in .NET itself. ASP.NET Core's JwtBearer and OpenIdConnect handlers depend on this same package family. So the choice is which version and which handler, not whether to have the dependency.

Within the family there are two generations:

- `JwtSecurityTokenHandler` / `JwtSecurityToken` in `System.IdentityModel.Tokens.Jwt`. Still shipped in 8.23.0 and not marked obsolete in the reference docs ([API](https://learn.microsoft.com/en-us/dotnet/api/system.identitymodel.tokens.jwt.jwtsecuritytokenhandler)).
- `JsonWebTokenHandler` / `JsonWebToken` in `Microsoft.IdentityModel.JsonWebTokens` ([API](https://learn.microsoft.com/en-us/dotnet/api/microsoft.identitymodel.jsonwebtokens.jsonwebtokenhandler)). ASP.NET Core 8 switched its handlers to this one by default, calling `JwtSecurityToken` "the previous generation" and citing a 30% performance improvement and async processing ([breaking change](https://learn.microsoft.com/en-us/aspnet/core/breaking-changes/8/securitytoken-events)).

### Gaps for this codebase

Moving from 5.6 to 8.x at all (even staying on `JwtSecurityTokenHandler`):

1. **JSON-valued claims are silently corrupted.** IdentityModel 7 replaced Newtonsoft with `System.Text.Json` ([IdentityModel 7x wiki](https://github.com/AzureAD/azure-activedirectory-identitymodel-extensions-for-dotnet/wiki/IdentityModel-7x)). `TokenExtensions.CreateJwtPayload` puts Newtonsoft `JToken` values into the `JwtPayload`. Probe on 8.23.0: a `JObject` value `{"x5t#S256":"abc"}` under `cnf` was written as `"cnf":[["abc"]]`, with no exception. The same value supplied as a `JsonElement` was written correctly. The 8.23.0 source says of its payload writer: "not meant to be a general object serializer. If a user needs to serialize a special value, then serialize the value into a `JsonElement`" ([source](https://github.com/AzureAD/azure-activedirectory-identitymodel-extensions-for-dotnet/blob/8.23.0/src/Microsoft.IdentityModel.Tokens/Json/JsonSerializerPrimitives.cs)). This affects the `cnf` confirmation claim and every claim with value type `json`.
2. **JAR JSON parameters are silently dropped.** `JwtRequestValidator.ProcessPayloadAsync` copies payload values that are `string`, `JObject` or `JArray`. Probe: on 8.23.0, object and array values in `JwtSecurityToken.Payload` are `JsonElement`, so neither Newtonsoft case matches and those parameters would vanish without error.
3. **Developer key file format changes.** `Crypto.cs` writes `tempkey.jwk` with `JsonConvert.SerializeObject(jwk)`. In 8.x `JsonWebKey` carries `System.Text.Json` attributes, not Newtonsoft ones. Probe: Newtonsoft then writes PascalCase names plus `AdditionalData` (`{"AdditionalData":{},"Alg":null,...`), while `JsonSerializer.Serialize(jwk)` writes standard JWK names. Both happened to round-trip through `new JsonWebKey(json)` in the probe.
4. **Obsoletions.** `JwtPayload.Exp`, `.Iat`, `.Nbf` are obsolete in favour of `long`-typed members; the wiki also lists the synchronous `ValidateToken` as deprecated in favour of `ValidateTokenAsync`, and the 8.23.0 docs mark `JsonWebTokenHandler.ValidateToken` obsolete.

Additionally, if moving to `JsonWebTokenHandler`:

5. **Token creation is shaped differently.** There is no header or payload object to mutate. `CreateToken` takes a `SecurityTokenDescriptor`, or a raw JSON payload string plus `SigningCredentials` and extra header claims. Probe: `CreateToken(payloadJson, creds, { "typ": "at+jwt" })` produced header `{"alg":"RS256","kid":"k1","typ":"at+jwt"}`. Passing `x5t` as an extra header claim threw `IDX14116` ("cannot contain the following claims: 'alg, kid, x5t, enc, zip'. These values are added by default (if necessary)"). `DefaultTokenCreationService` currently sets `header["x5t"]` by hand for X.509 keys; whether `JsonWebTokenHandler` emits `x5t` on its own for an `X509SecurityKey` was not tested.
6. **Public API breaks** on every member listed above that exposes `JwtHeader`, `JwtPayload` or `JwtSecurityToken`. ADR 0001 permits this.
7. **Name collision.** `ValidateTokenAsync` returns `Microsoft.IdentityModel.Tokens.TokenValidationResult` (probe); this codebase has its own `IdentityServer4.Validation.TokenValidationResult`.
8. **Claim mapping default flips.** `JsonWebTokenHandler.MapInboundClaims` defaults to `false` (docs and probe); `JwtSecurityTokenHandler`'s defaults to `true`. The explicit `InboundClaimTypeMap.Clear()` calls become unnecessary.
9. **Validation does not throw.** `ValidateTokenAsync` returns a result with `IsValid` and `Exception`; the current code relies on `try/catch` around `ValidateToken` (for example the `SecurityTokenExpiredException` branch in `TokenValidator`).

The key model (`SecurityKey`, `SigningCredentials`, `JsonWebKey`, `SecurityAlgorithms`) is shared by both handlers and unaffected by the handler choice.

### Status

- Licence MIT, author Microsoft, repo `AzureAD/azure-activedirectory-identitymodel-extensions-for-dotnet`, last pushed 2026-11-12.
- Latest version of all four packages checked (`System.IdentityModel.Tokens.Jwt`, `Microsoft.IdentityModel.JsonWebTokens`, `Microsoft.IdentityModel.Tokens`, `Microsoft.IdentityModel.Protocols.OpenIdConnect`) is **8.23.0**, published 2026-09-18, each with an explicit `net10.0` asset ([NuGet](https://www.nuget.org/packages/System.IdentityModel.Tokens.Jwt/8.23.0)). No 9.x exists.
- Support policy: 8.x "Active", tied to .NET 9 and .NET 10 lifetimes, to about November 2028. 7.x was supported through 2026-11-10. For 5.x only 5.7.0 and later is supported; **5.6.0 is listed under "out of support versions"** ([supportPolicy.md](https://github.com/AzureAD/azure-activedirectory-identitymodel-extensions-for-dotnet/blob/dev/supportPolicy.md)). The policy file is dated 2025-05-08 and does not mention .NET 11.
- Pinned 5.6.0 is marked deprecated on NuGet ("Move to latest") and is affected by [GHSA-59j7-ghrg-fj52](https://github.com/advisories/GHSA-59j7-ghrg-fj52) (CVE-2024-21319, medium).
- `Microsoft.AspNetCore.Authentication.OpenIdConnect` 10.0.15 requires `Microsoft.IdentityModel.Protocols.OpenIdConnect >= 8.19.2` ([NuGet](https://www.nuget.org/packages/Microsoft.AspNetCore.Authentication.OpenIdConnect/10.0.15)).

---

## Polly (Microsoft.Extensions.Http.Polly)

### Usage in `src/`

None. `src/Directory.Build.targets` has a version pin (`PackageReference Update`), but no `.csproj` in `src/` includes the package and no `.cs` file mentions Polly, `AddPolicyHandler` or `AddTransientHttpErrorPolicy`.

The core library does register named `HttpClient`s and returns the `IHttpClientBuilder` to the host (`AddBackChannelLogoutHttpClient`, `AddJwtRequestUriHttpClient` in `Configuration/DependencyInjection/BuilderExtensions/Additional.cs`), so a host can attach any handler it likes. That extension point needs only `Microsoft.Extensions.Http`, which is in the ASP.NET Core shared framework.

### Replacement

Nothing needs replacing. If resilience were wanted later, the Microsoft package is `Microsoft.Extensions.Http.Resilience` (`AddStandardResilienceHandler()`: rate limiter, total timeout, retry, circuit breaker, attempt timeout). It is not Polly-free: the docs say it "relies on the `Microsoft.Extensions.Resilience` library and Polly" ([docs](https://learn.microsoft.com/en-us/dotnet/core/resilience/http-resilience)).

### Status

- Polly: BSD-3-Clause, 8.9.1 published 2026-11-07, `net10.0` asset, repo active ([NuGet](https://www.nuget.org/packages/Polly/8.9.1)).
- `Microsoft.Extensions.Http.Polly`: MIT, still published in step with .NET (10.0.x and 11.0.0), `netstandard2.0` only, not marked deprecated on NuGet; depends on Polly 7.2.4 or later.
- `Microsoft.Extensions.Http.Resilience`: MIT, 10.11.0 (2026-10-14) on the 10.x line and 11.0.0 (2026-11-10), `net10.0` asset.

---

## Serilog (hosts only)

### Usage in `src/`

Three host projects, no library code. `IdentityServer4/host` references `Serilog.AspNetCore`; the `AspNetIdentity` and `EntityFramework` hosts also reference `Serilog`, `Serilog.Sinks.Console` and `Serilog.Sinks.File` explicitly.

Four files: the three `Program.cs` files are near-identical (static `Log.Logger` with minimum-level overrides per namespace, `Enrich.FromLogContext()`, a console sink with a custom output template and ANSI theme, a commented-out file sink, `try/catch/finally` with `Log.Fatal` and `Log.CloseAndFlush()`, then `.UseSerilog()`), and `IdentityServer4/host/Startup.cs` calls `app.UseSerilogRequestLogging()`.

The libraries log through `Microsoft.Extensions.Logging.ILogger<T>` only.

### Replacement

- Console output: the built-in console provider with `AddSimpleConsole` or `AddJsonConsole` ([console log formatting](https://learn.microsoft.com/en-us/dotnet/core/extensions/console-log-formatter)).
- Per-category minimum levels: `Logging:LogLevel` configuration or `AddFilter`.
- Request logging: `UseHttpLogging()` from `Microsoft.AspNetCore.HttpLogging`, in the shared framework ([HTTP logging](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/http-logging/)).

### Gaps

- No output template. `SimpleConsoleFormatterOptions` offers `TimestampFormat`, `SingleLine`, `ColorBehavior`, `IncludeScopes`, `UseUtcTimestamp`; anything else needs a custom `ConsoleFormatter`.
- No file provider. The built-in providers are Console, Debug, EventSource and EventLog; Microsoft's own list sends file logging to third-party providers ([logging providers](https://learn.microsoft.com/en-us/dotnet/core/extensions/logging/providers)). The file sink in the hosts is commented out, so nothing in use is lost.
- `UseHttpLogging` logs differently from Serilog's one-line request summary. Equivalence was not compared.
- These are development hosts; none of this ships in a package.

### Status

Apache-2.0, actively maintained. Serilog 4.4.0 (2026-07-10, `net10.0` asset), Serilog.AspNetCore 10.0.0 (2025-11-28, `net10.0` asset; an 11.0.0 dev build was published 2026-11-13), Serilog.Sinks.Console 6.1.1, Serilog.Sinks.File 7.0.0 ([NuGet](https://www.nuget.org/packages/Serilog.AspNetCore/10.0.0), [repo](https://github.com/serilog/serilog)). Pins are 2.9.0 / 3.2.0 / 3.1.1 / 4.1.0 from 2019.

---

## System.Security.Principal.Windows

### Usage in `src/`

A `PackageReference` in each of the three host projects. No `.cs` file in `src/` uses `WindowsIdentity`, `WindowsPrincipal` or any other type from the package. (`PrincipalExtensions.cs` imports `System.Security.Principal` for `IPrincipal` and `IIdentity`, which are not in this package.)

### Replacement

The assembly is part of the .NET shared framework: `System.Security.Principal.Windows.dll` is in the `Microsoft.NETCore.App.Ref` 10.0.14 pack, and that pack's `PackageOverrides.txt` lists `System.Security.Principal.Windows|5.0.0` as framework-provided ([WindowsIdentity API](https://learn.microsoft.com/en-us/dotnet/api/system.security.principal.windowsidentity)).

### Gaps

None. Starting with the .NET 10 SDK, a direct reference to a framework-provided package from a `net10.0` project raises warning NU1510 ([NU1510](https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/10.0/nu1510-pruned-references)).

### Status

MIT, Microsoft. Last package version 5.0.0, published 2020-11-09; no later package exists because the code ships with the runtime ([NuGet](https://www.nuget.org/packages/System.Security.Principal.Windows/5.0.0)).

---

## Bullseye and SimpleExec (build)

### Usage in `src/`

Each of the five library folders has a `build/build.csproj` (a `netcoreapp3.1` console app referencing both packages) started by `build.cmd` / `build.sh` / `build.ps1` with `dotnet run --project build`. All five share one linked file, `src/build/Program.Partial.cs` (104 lines), plus a per-library `Program.cs` holding a prefix constant.

The script defines nine targets (`clean-build-output`, `build`, `sign-binary`, `test`, `clean-pack-output`, `pack`, `sign-package`, `copy-pack-output`, and the `default` / `quick` / `sign` aggregates). Bullseye supplies `Target`, `DependsOn`, `RunTargetsAndExit`; SimpleExec supplies `Run("dotnet", ...)`. The signing targets shell out to a `SignClient` tool configured for IdentityServer4's .NET Foundation signing identity (`signClient.json` at the repo root).

### Replacement

No Microsoft package provides a target runner. Built-in options:

- Call `dotnet build`, `dotnet test`, `dotnet pack` directly from CI or a shell script.
- A .NET 10 file-based app: the .NET 10 SDK can run a single `.cs` file without a project, with `#:package` and `#:property` directives ([file-based apps](https://learn.microsoft.com/en-us/dotnet/core/sdk/file-based-apps)). That removes the five `build.csproj` files whether or not the packages stay.
- MSBuild targets in a `.proj` file.

### Gaps

- Bullseye's dependency ordering, target listing, `--dry-run`-style CLI and coloured summary would be lost or re-implemented.
- With signing removed, what remains is build, test, pack and copy: four sequential commands.
- The root `global.json` pins SDK 3.1.402, and five separate solutions each own a build project; consolidating them is a larger question than the packages.

### Status

Both by the same maintainer (`adamralph`), Apache-2.0, repos pushed within the last three days.

- Bullseye 6.3.0, published 2026-10-05, assets `net8.0`, `net9.0`, `net10.0` ([NuGet](https://www.nuget.org/packages/Bullseye/6.3.0), [repo](https://github.com/adamralph/bullseye)).
- SimpleExec 13.1.0, published 2026-09-05, same assets ([NuGet](https://www.nuget.org/packages/SimpleExec/13.1.0), [repo](https://github.com/adamralph/simple-exec)).
- Both READMEs on `main` say "Platform support: .NET 10.0 and later", which is narrower than the latest published packages; the next release may drop `net8.0` and `net9.0`.
- Pins (3.3.0 and 6.2.0) are from 2019 to 2020; the API has moved from the static `Bullseye.Targets` methods used here, so an upgrade edits the script too. The exact changes were not enumerated.

---

## MinVer

### Usage in `src/`

One line in `src/Directory.Build.targets`, written as `PackageReference Include` (not `Update`), so every project under `src/` gets it. One custom target, `SetAssemblyVersion`, runs after MinVer's target and sets `AssemblyVersion` from `MinVerMajor`, `MinVerMinor`, `MinVerPatch`. No MinVer options (tag prefix, minimum version) are set anywhere in `src/`.

### Replacement

The .NET SDK has no feature that derives a version from git tags. Built-in means are a static `<Version>` or `<VersionPrefix>` in a props file, or passing `-p:Version=...` from CI. Source Link, now in the SDK, adds the commit hash to `InformationalVersion` but does not compute a version number ([Source Link in the SDK](https://learn.microsoft.com/en-us/dotnet/core/compatibility/sdk/8.0/source-link)).

### Gaps

- Versions stop tracking tags unless CI passes them in.
- The existing tags are IdentityServer4's (4.1.2 and earlier). MinVer would compute this fork's versions from them unless a tag prefix or new tags are introduced. That is a release-policy matter for whoever decides.

### Status

Apache-2.0, same maintainer as Bullseye. Latest 8.0.0, published 2026-09-05 (8.1.0-alpha.1 on 2026-09-29). It is a build-time-only package with no target-framework assets; its README lists ".NET SDK 10.0 or later" and Git as prerequisites ([NuGet](https://www.nuget.org/packages/MinVer/8.0.0), [repo](https://github.com/adamralph/minver)). The pin, 2.3.0, is from 2020; behaviour differences across six major versions were not enumerated.

---

## xunit

### Usage in `src/`

Five test projects, each referencing `xunit`, `xunit.runner.visualstudio` and `Microsoft.NET.Test.Sdk`.

- 129 files import `Xunit`.
- 920 test methods: 818 `[Fact]`, 102 `[Theory]`. By project: core unit tests 605, core integration tests 257, EF.Storage integration 40, EF.Storage unit 16, EF tests 2.
- Features: `[InlineData]` (216), `[Trait("Category", ...)]` (507), `[MemberData]` with `TheoryData<T>` (EF tests, for database providers), `IClassFixture<T>` (2), one `[Fact(Skip = ...)]`, two `xunit.runner.json` files. About 90 direct `Assert.*` calls.
- No `ITestOutputHelper`, no `IAsyncLifetime`, no custom attributes or orderers. One file imports `Xunit.Sdk`.
- **24 test methods are `async void`.**

### Replacement

xunit is a .NET Foundation project, not a Microsoft product, so there are two different readings of "first-party":

- **xunit v3 on Microsoft.Testing.Platform (MTP).** `xunit.v3` "includes built-in support for the new Microsoft Testing Platform" ([xunit MTP docs](https://xunit.net/docs/getting-started/v3/microsoft-testing-platform)). MTP is Microsoft's replacement for VSTest ([MTP intro](https://learn.microsoft.com/en-us/dotnet/core/testing/microsoft-testing-platform-intro)). The .NET 10 SDK added an MTP-native `dotnet test` mode, enabled in `global.json` with `"test": { "runner": "Microsoft.Testing.Platform" }` ([dotnet test](https://learn.microsoft.com/en-us/dotnet/core/testing/unit-testing-with-dotnet-test)). In that mode `Microsoft.NET.Test.Sdk` and `xunit.runner.visualstudio` are needed only for VSTest-based tooling.
- **MSTest.** The Microsoft-authored framework (MIT, 4.5.3 published 2026-11-11), also MTP-based ([MSTest](https://learn.microsoft.com/en-us/dotnet/core/testing/unit-testing-mstest-intro)).

### Gaps

For xunit v2 to v3, per the migration guide ([migration](https://xunit.net/docs/getting-started/v3/migration)):

- Package rename `xunit` to `xunit.v3`; test projects must set `OutputType` to `Exe`; minimum target is .NET 8.
- `async void` tests "are fast-failed at runtime". The 24 here must return `Task`.
- `Xunit.Abstractions` types moved; this codebase does not use them.
- The guide lists no changes to `[Fact]`, `[Theory]`, `[InlineData]`, `[MemberData]`, `[Trait]` or `IClassFixture`. That is absence of a listed change, not a confirmed compatibility statement.
- xunit.v3 4.0 (2026-08-14) made MTP v2 the default and discontinued MTP v1 support ([4.0.0 notes](https://xunit.net/releases/v3/4.0.0)).

For MSTest: every `[Fact]`, `[Theory]`, `[InlineData]`, `[Trait]` and fixture on 920 tests is rewritten. No feature this suite uses is missing from MSTest as far as the docs show, but that was not checked feature by feature.

### Status

- Licence Apache-2.0, copyright .NET Foundation and contributors ([repo](https://github.com/xunit/xunit)).
- `xunit` (v2) latest is 2.9.3, published 2025-01-08, and **every v2 version is marked deprecated**: "This package will only be updated for security issues. All future feature work has moved onto v3" ([NuGet](https://www.nuget.org/packages/xunit/2.9.3)).
- `xunit.v3` latest is 4.0.3, published 2026-11-09, assets `net8.0` and `net472`. `xunit.runner.visualstudio` 4.0.1 (2026-10-09). Repo pushed 2026-11-10.
- `Microsoft.NET.Test.Sdk` 18.12.0 (2026-11-11), MIT. The pin is 16.2.0 from 2019.

---

## FluentAssertions

### Usage in `src/`

121 test files; 2,657 `.Should()` calls.

| Assertion | Count |
| --- | --- |
| `Be` | 1,046 |
| `BeTrue` | 347 |
| `NotBeNull` | 293 |
| `Contain` | 282 |
| `BeFalse` | 249 |
| `BeNull` | 216 |
| `BeEquivalentTo` | 58 |
| `StartWith` | 45 |
| `Throw` | 33 |
| 18 others | 88 |

The first six account for 92%. Structural comparison (`BeEquivalentTo`, 58 uses, 3 with `Excluding`) and `.And.` chaining (15) are the only non-trivial features.

### Replacement

xunit's own `Assert` class, which the test projects already have and already use about 90 times. Most of the table maps one-to-one (`Assert.Equal`, `Assert.True`, `Assert.NotNull`, `Assert.Contains`, `Assert.Null`, `Assert.StartsWith`, `Assert.Throws`). xunit has `Assert.Equivalent` for structural comparison.

Not built-in, for reference: AwesomeAssertions is a community fork of FluentAssertions, Apache-2.0, 9.6.0 published 2026-08-20; its README states "the license will never change" ([NuGet](https://www.nuget.org/packages/AwesomeAssertions/9.6.0), [repo](https://github.com/AwesomeAssertions/AwesomeAssertions)). Whether its current version keeps the `FluentAssertions` namespace was not verified.

### Gaps

- A mechanical rewrite of 2,657 call sites across 121 files.
- `Assert.Equivalent` is not option-for-option the same as `BeEquivalentTo`; the 58 uses, and the 3 with `Excluding`, need individual attention. Not analysed further.
- Failure messages differ in style.

### Status

- **Version 8 and later are not open source.** The repository's `LICENSE` is the "Xceed Software Inc. Community License Agreement" for non-commercial use; any other use "requires a paid Commercial License from Xceed" ([LICENSE](https://github.com/fluentassertions/fluentassertions/blob/main/LICENSE)). The licence text counts "developing or testing open-source software" as non-commercial use, and separately excludes use "by or for" an organisation that earns revenue. How those two clauses apply to this project and its contributors is a legal question this note does not answer.
- NuGet metadata agrees: 8.0.0 (2025-01-14) onward carries a licence file, authors now include Xceed, latest 8.11.0 (2026-09-14).
- **Version 7 remains Apache-2.0 and maintained.** Xceed's FAQ: "Version 7 will remain fully open-source indefinitely and receive bugfixes and other important corrections" ([FAQ](https://xceed.com/fluent-assertions-faq/)). NuGet shows 7.2.1 (2026-02-07) and 7.2.2 (2026-03-16), both `Apache-2.0`.
- The pinned 5.10.2 (2020) is Apache-2.0 and has `netstandard2.0` / `netcoreapp2.1` assets, so it still restores on `net10.0`. Source-breaking changes between 5.x and 7.x were not enumerated.
- An unpinned or floating upgrade would cross into 8.x and the Xceed licence.

---

## Not in the ticket's list, noted in passing

- **HybridCache** (named in the ticket as a candidate replacement). No third-party caching library is used in `src/`. The code uses `IMemoryCache` (2 files) and `IDistributedCache` (5 files) behind its own `ICache<T>` abstraction (7 files). `HybridCache` is a Microsoft package, `Microsoft.Extensions.Caching.Hybrid`, not part of the shared framework, documented for ASP.NET Core 9 and later ([docs](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/hybrid)). It is a possible simplification of first-party code, not a dependency replacement.
- **Entity Framework Core, ASP.NET Core Identity, ASP.NET Core authentication packages.** First-party; version alignment only.

---

## What could not be verified

- **Compilation.** Nothing in `src/` was built against `net10.0`. "Supports net10.0" rests on NuGet asset metadata and NuGet's compatibility rules.
- **Persisted-grant format compatibility** between the current Newtonsoft serializer and a `System.Text.Json` one was not tested.
- **`x5t` header behaviour** of `JsonWebTokenHandler` with an `X509SecurityKey` was not tested; only that passing it explicitly throws.
- **AutoMapper advisory reachability** for this codebase's maps was not assessed.
- **Licence compatibility** questions (RPL-1.5 with Apache-2.0; the Xceed community licence for an open-source project with commercial users) are legal questions and are not answered here.
- **Lucky Penny's licence agreement page** (`luckypennysoftware.com/license`) was not read; the terms above are from the repository's `LICENSE.md` and the author's launch post.
- **AwesomeAssertions namespace**, **Bullseye / SimpleExec / MinVer breaking changes** across the pinned-to-latest gap, and **FluentAssertions 5 to 7 breaking changes** were not enumerated.
- **Constant coverage.** Whether Microsoft.IdentityModel's constant classes cover every `OidcConstants` / `JwtClaimTypes` member used here was not checked.
- **MSTest feature parity** for this suite was not checked beyond the docs' overview.
- **Microsoft.IdentityModel and .NET 11.** The support policy file predates .NET 11; how 8.x's support window relates to .NET 11 is not stated there.
- The `IdentityModel-7x` wiki page was read through a summarising fetch; its statements are paraphrased, and the behaviour that matters here was confirmed independently by the probe.

## Appendix A: probe results

A throwaway `net10.0` console app referencing `System.IdentityModel.Tokens.Jwt` 8.23.0 and `Newtonsoft.Json` 13.0.4, run with .NET SDK 10.0.403. Observed output, abridged:

| # | Case | Result |
| --- | --- | --- |
| 1a | `JwtPayload.Add("cnf", JRaw.Parse("{\"x5t#S256\":\"abc\"}"))`, then `JwtSecurityTokenHandler.WriteToken` | Payload contains `"cnf":[["abc"]]`. No exception |
| 1b | `JwtPayload.Add("arr", JToken[] { {"a":1}, {"a":2} })` | Payload contains `"arr":[[["1"]],[["2"]]]`. No exception |
| 2a | Same values as `JsonElement`; `scope` as `string[]` | `"cnf":{"x5t#S256":"abc"},"arr":[{"a":1},2],"scope":["a","b"]` |
| 2b | `ReadJwtToken` on 2a, runtime types of payload values | `exp=Int64, iss=String, cnf=JsonElement, arr=JsonElement, scope=JsonElement` |
| 3a | `JsonWebTokenHandler.CreateToken(payloadJson, creds, { typ })` | Header `{"alg":"RS256","kid":"k1","typ":"at+jwt"}`; payload passed through unchanged. With `x5t` added to the header dictionary: throws `SecurityTokenException` IDX14116 |
| 3b | `JsonWebTokenHandler.ValidateTokenAsync` | `IsValid=True`; result type `Microsoft.IdentityModel.Tokens.TokenValidationResult`; `cnf` claim value is `JsonElement`; `MapInboundClaims` default `False`; claim types unmapped (`iss,aud,exp,cnf`) |
| 4a | `Newtonsoft.Json.JsonConvert.SerializeObject(JsonWebKey)` | `{"AdditionalData":{},"Alg":null,"Crv":null,"D":"...`; `new JsonWebKey(json)` still read `Kty=RSA`, `Kid=k1` |
| 4b | `System.Text.Json.JsonSerializer.Serialize(JsonWebKey)` | `{"d":"...` with standard JWK names; round-trips |
| 5a | `System.Text.Json` default, into `ICollection<string>` initialised to `HashSet<string>(OrdinalIgnoreCase)`, input `["a","A"]` | `List`, count 2 |
| 5b | Same with `PreferredObjectCreationHandling = Populate` | `HashSet`, count 1 |
| 5c | Same with Newtonsoft defaults | `HashSet`, count 1 |
| 5d | `System.Text.Json` serializing a type with a computed get-only property | Written by default; omitted with `IgnoreReadOnlyProperties = true` |
| 5f | `JsonSerializer.Deserialize<System.Security.Claims.Claim>` | Throws `NotSupportedException` (no usable constructor) |
