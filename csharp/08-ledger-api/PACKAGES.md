# Packages

Only the packages and versions listed here are available; there is no network. A reference to a lower version than the one listed resolves to the listed version, with the warning NU1603, which fails the build where warnings are errors; a higher version fails with NU1102, and a package not listed fails with NU1101; both errors name the package.
Install a package with the exact version listed: `<PackageReference Include="NAME" Version="VERSION" />`.
ASP.NET Core (Minimal API, its built-in validation, System.Text.Json) and the .NET base library come with SDK 10.0.403 as shared frameworks; they are not packages.

Cache snapshot: tree sha256 `0b265e7c7e6553932102ebd2861285b47cbba73807a3d37c5e4507c03888ac9d`, made 2026-11-07.

## Offered

| Package | Version | Purpose |
|---|---|---|
| `Microsoft.AspNetCore.Authentication.JwtBearer` | 10.0.14 | JWT bearer authentication for ASP.NET Core Minimal API |
| `Microsoft.AspNetCore.Mvc.Testing` | 10.0.14 | In-memory test host for ASP.NET Core Minimal API tests (WebApplicationFactory): a test project calls the app without a network |
| `Microsoft.Extensions.Hosting` | 10.0.14 | Generic Host for daemons: lifetime and graceful stop (IHostApplicationLifetime), dependency injection, configuration, ILogger |
| `Microsoft.NET.Test.Sdk` | 18.11.0 | VSTest host for dotnet test in VSTest mode |
| `xunit.runner.visualstudio` | 4.0.1 | xUnit adapter for the VSTest runner (dotnet test in VSTest mode) |
| `xunit.v3` | 4.0.2 | Test framework (xUnit v3), run on Microsoft Testing Platform: the test project is an executable |

## In the cache as dependencies of the offered packages

| Package | Version |
|---|---|
| `microsoft.applicationinsights` | 2.23.0 |
| `microsoft.aspnetcore.testhost` | 10.0.14 |
| `microsoft.bcl.asyncinterfaces` | 6.0.0 |
| `microsoft.bcl.cryptography` | 10.0.2 |
| `microsoft.codecoverage` | 18.11.0 |
| `microsoft.extensions.configuration` | 10.0.14 |
| `microsoft.extensions.configuration.abstractions` | 10.0.14 |
| `microsoft.extensions.configuration.binder` | 10.0.14 |
| `microsoft.extensions.configuration.commandline` | 10.0.14 |
| `microsoft.extensions.configuration.environmentvariables` | 10.0.14 |
| `microsoft.extensions.configuration.fileextensions` | 10.0.14 |
| `microsoft.extensions.configuration.json` | 10.0.14 |
| `microsoft.extensions.configuration.usersecrets` | 10.0.14 |
| `microsoft.extensions.dependencyinjection` | 10.0.14 |
| `microsoft.extensions.dependencyinjection.abstractions` | 10.0.14 |
| `microsoft.extensions.dependencymodel` | 10.0.14 |
| `microsoft.extensions.diagnostics` | 10.0.14 |
| `microsoft.extensions.diagnostics.abstractions` | 10.0.14 |
| `microsoft.extensions.fileproviders.abstractions` | 10.0.14 |
| `microsoft.extensions.fileproviders.physical` | 10.0.14 |
| `microsoft.extensions.filesystemglobbing` | 10.0.14 |
| `microsoft.extensions.hosting.abstractions` | 10.0.14 |
| `microsoft.extensions.logging` | 10.0.14 |
| `microsoft.extensions.logging.abstractions` | 8.0.0 |
| `microsoft.extensions.logging.abstractions` | 10.0.14 |
| `microsoft.extensions.logging.configuration` | 10.0.14 |
| `microsoft.extensions.logging.console` | 10.0.14 |
| `microsoft.extensions.logging.debug` | 10.0.14 |
| `microsoft.extensions.logging.eventlog` | 10.0.14 |
| `microsoft.extensions.logging.eventsource` | 10.0.14 |
| `microsoft.extensions.options` | 10.0.14 |
| `microsoft.extensions.options.configurationextensions` | 10.0.14 |
| `microsoft.extensions.primitives` | 10.0.14 |
| `microsoft.identitymodel.abstractions` | 8.19.2 |
| `microsoft.identitymodel.jsonwebtokens` | 8.19.2 |
| `microsoft.identitymodel.logging` | 8.19.2 |
| `microsoft.identitymodel.protocols` | 8.19.2 |
| `microsoft.identitymodel.protocols.openidconnect` | 8.19.2 |
| `microsoft.identitymodel.tokens` | 8.19.2 |
| `microsoft.testing.extensions.telemetry` | 2.5.1 |
| `microsoft.testing.extensions.trxreport.abstractions` | 2.5.1 |
| `microsoft.testing.platform` | 2.5.1 |
| `microsoft.testing.platform.msbuild` | 2.5.1 |
| `microsoft.testplatform.objectmodel` | 18.11.0 |
| `microsoft.testplatform.testhost` | 18.11.0 |
| `microsoft.win32.registry` | 5.0.0 |
| `system.diagnostics.eventlog` | 10.0.14 |
| `system.identitymodel.tokens.jwt` | 8.19.2 |
| `system.security.accesscontrol` | 6.0.1 |
| `xunit.analyzers` | 2.2.0 |
| `xunit.v3.assert` | 4.0.2 |
| `xunit.v3.common` | 4.0.2 |
| `xunit.v3.core.mtp-v2` | 4.0.2 |
| `xunit.v3.extensibility.core` | 4.0.2 |
| `xunit.v3.mtp-v2` | 4.0.2 |
| `xunit.v3.runner.common` | 4.0.2 |
| `xunit.v3.runner.inproc.console` | 4.0.2 |
