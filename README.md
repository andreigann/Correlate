# Correlate

Correlate provides flexible .NET Core support for correlation ID in ASP.NET Core and HttpClient.

## Installation

Install Correlate via the Nuget package manager or `dotnet` cli.

```powershell
dotnet add package Correlate
```

For ASP.NET Core integration:

```powershell
dotnet add package Correlate.AspNetCore
```

---

[![Build status](https://ci.appveyor.com/api/projects/status/rwfdg9d4i3g0qyga/branch/master?svg=true)](https://ci.appveyor.com/project/skwasjer/correlate)
[![Tests](https://img.shields.io/appveyor/tests/skwasjer/Correlate/master.svg)](https://ci.appveyor.com/project/skwasjer/correlate/build/tests)

| | | |
|---|---|---|
| `Correlate` | [![NuGet](https://img.shields.io/nuget/v/Correlate.svg)](https://www.nuget.org/packages/Correlate/) [![NuGet](https://img.shields.io/nuget/dt/Correlate.svg)](https://www.nuget.org/packages/Correlate/) | Core library, including a `DelegatingHandler` for `HttpClient`. |
| `Correlate.Abstractions` | [![NuGet](https://img.shields.io/nuget/v/Correlate.Abstractions.svg)](https://www.nuget.org/packages/Correlate.Abstractions/) [![NuGet](https://img.shields.io/nuget/dt/Correlate.Abstractions.svg)](https://www.nuget.org/packages/Correlate.Abstractions/) | Abstractions library. |
| `Correlate.AspNetCore` | [![NuGet](https://img.shields.io/nuget/v/Correlate.AspNetCore.svg)](https://www.nuget.org/packages/Correlate.AspNetCore/) [![NuGet](https://img.shields.io/nuget/dt/Correlate.AspNetCore.svg)](https://www.nuget.org/packages/Correlate.AspNetCore/) | ASP.NET Core middleware. |
| `Correlate.DependencyInjection` | [![NuGet](https://img.shields.io/nuget/v/Correlate.DependencyInjection.svg)](https://www.nuget.org/packages/Correlate.DependencyInjection/) [![NuGet](https://img.shields.io/nuget/dt/Correlate.DependencyInjection.svg)](https://www.nuget.org/packages/Correlate.DependencyInjection/) | Extensions for registration in a `IServiceCollection` container. |

## Usage

In a typical ASP.NET Core (MVC) application, register the middleware and required services to handle incoming requests with a correlation id, and to enrich the response with a relevant correlation id.

When using `HttpClient` to call other services, you can use `HttpClientFactory` to attach a delegating handler to any `HttpClient` which will automatically add a correlation id header to the outgoing request for cross service correlation.

### Example ###

Configure your application:

```csharp
using Correlate.AspNetCore;
using Correlate.DependencyInjection;
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;

public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // Register services.
        services.AddCorrelate(options => 
            options.RequestHeaders = new []
            {
              // List of incoming headers possible. First that is set on given request is used and also returned in the response.
              "X-Correlation-ID",
              "My-Correlation-ID"
            }
        );

        // Register a typed client that will include the correlation id in outgoing request.
        services
            .AddHttpClient<IMyService, MyService>()
            .CorrelateRequests("X-Correlation-ID");

        services.AddMvcCore();
    }

    public void Configure(IApplicationBuilder app)
    {
        // Use middleware.
        app.UseCorrelate();

        app.UseMvc();
    }
}
```

Using this setup, for any incoming request that contains a `X-Correlation-ID` or `My-Correlation-ID` header, the correlation id from that header will be used throughout the request pipeline. For any incoming request without either header, a unique correlation id is generated instead.

Secondly, all responses will receive a matching response header.

Thirdly, for all outbound HTTP calls that are sent via the `HttpClient` provided to `MyService` instances, a `X-Correlation-ID` request header is added.

> In order to capture incoming requests with correlation ids as soon as possible, the middleware should be registered as the first middleware in the pipeline or at least near the top. Otherwise, you have middleware executing outside of the correlation context scope making them untrackable.

## Logging

Before a request flows down the pipeline, a log scope is created with a `CorrelationId` property, containing the correlation id.

Most popular log providers will be able to log the correlation id with minimal set up required.

Here's some providers that require no set up or custom code, only configuration:

- Serilog: `new LoggerConfiguration().Enrich.FromLogContext()`
  https://github.com/serilog/serilog/wiki/Enrichment#the-logcontext  
- NLog: https://github.com/NLog/NLog/wiki/EventProperties-Layout-Renderer

## ICorrelationContextAccessor - Getting the correlation id from anywhere

To access the correlation id anywhere in code, inject an instance of `ICorrelationContextAccessor` in your constructor. 

### Example

```csharp
public class MyService
{
    public MyService(ICorrelationContextAccessor correlationContextAccessor)
    {
        string correlationId = correlationContextAccessor.CorrelationContext.CorrelationId;
    }
}
```

> Note: `correlationContextAccessor.CorrelationContext` can be null, when `MyService` is not scoped to a request. Thus, when used outside of ASP.NET (not using the middleware component), you should create the context using `ICorrelationContextFactory` or `CorrelationManager` for each unique subprocess.

## CorrelationManager

To simplify managing correlation contexts, the `CorrelationManager` can be used. It takes care of alot of the logic to create the context properly. This is especially useful when running background tasks or applications other than ASP.NET Core which need to interact with external services.

### Example

```csharp
public class MyWorker
{
    private readonly CorrelationManager _correlationManager;
    private readonly MyService _myService;

    public MyWorker(CorrelationManager correlationManager, MyService myService)
    {
        _correlationManager = correlationManager;
        _myService = myService;
    }

    public async Task RunAsync()
    {
        while (true)
        {
            await _correlationManager.CorrelateAsync(async () => {
                // Do work in a scoped correlation context with its own correlation id.
                await myService.MakeHttpCallAsync();
            });

            await Task.Delay(5000);
        }
    }
}
```

In this example the `MakeHttpCallAsync()` call is executed in a scoped correlation context, for which a correlation id is automatically generated and attached to the outgoing request provided the `HttpClient` is set up with the delegating handler:

```csharp
services
    .AddHttpClient<IMyService, MyService>()
    .CorrelateRequests("X-Correlation-ID");
```

## Enabling Correlate

Correlate depends on logging (`ILogger`) and/or tracing (`DiagnosticsListener`), in a sense that a correlation context will only be created when one of either is enabled. This is intentional, because it does not make much sense to have the overhead of Correlate - no matter how minute - when nothing is logged/traced.

If the `IsEnabled()` method for both returns `false`, the `CorrelationManager` will simply execute the task without scoping it to a correlation context. This also means that `ICorrelationContextAccessor.CorrelationContext` will return `null`, since no context is created.

For ASP.NET Core no additional setup is required, and Correlate will just be enabled (unless you explicitly disabled logging/tracing). When working outside of ASP.NET Core, make sure logging or tracing is enabled via configuration/code. This includes in unit tests.

## ICorrelationIdFactory

By default, when generating a correlation id, the `GuidCorrelationIdFactory` will produce guid based correlation ids.

As an alternative, there's also a `RequestIdentifierCorrelationIdFactory` which produces base32 encoded correlation ids. To use this or a custom implementation  instead, simply register it manually in the service container.

### Example
```csharp
services.AddSingleton<ICorrelationIdFactory, RequestIdentifierCorrelationIdFactory>();
```

## More info

### Supported .NET targets
- .NET Standard 2.0
- ASP.NET Core 2.1/2.2

### Build requirements
- Visual Studio 2017
- .NET Core 2.2 SDK

#### Contributions
PR's are welcome. Please rebase before submitting, provide test coverage, and ensure the AppVeyor build passes. I will not consider PR's otherwise.

#### Contributors
- skwas (author/maintainer)
