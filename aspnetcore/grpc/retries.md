---
title: Transient fault handling with gRPC retries
author: jamesnk
description: Learn how to make resilient, fault tolerant gRPC calls with retries in .NET.
monikerRange: '>= aspnetcore-3.0'
ms.author: jamesnk
ms.date: 03/18/2021
no-loc: [appsettings.json, "ASP.NET Core Identity", cookie, Cookie, Blazor, "Blazor Server", "Blazor WebAssembly", "Identity", "Let's Encrypt", Razor, SignalR]
uid: grpc/retries
---
# Transient fault handling with gRPC retries

By [James Newton-King](https://twitter.com/jamesnk)

gRPC retries is a feature that allows gRPC clients to automatically retry failed calls. This article discusses how to configure a retry policy to make resilient, fault tolerant gRPC apps in .NET.

gRPC retries requires [Grpc.Net.Client](https://www.nuget.org/packages/Grpc.Net.Client) version 2.36.0 or later.

## Transient fault handling

gRPC calls can be interrupted by transient faults. Transient faults include:

* Momentary loss of network connectivity.
* Temporary unavailability of a service.
* Timeouts due to server load.

When a gRPC call is interrupted, the client throws an `RpcException` with details about the error. The client app must catch the exception and choose how to handle the error.

```csharp
var client = new Greeter.GreeterClient(channel);
try
{
    var response = await client.SayHelloAsync(
        new HelloRequest { Name = ".NET" });

    Console.WriteLine("From server: " + response.Message);
}
catch (RpcException ex)
{
    // Write logic to inspect the error and retry
    // if the error is from a transient fault.
}
```

Duplicating retry logic throughout an app is verbose and error prone. Fortunately the .NET gRPC client has a built-in support for automatic retries.

## Configure a gRPC retry policy

A retry policy is configured once when a gRPC channel is created:

```csharp
var defaultMethodConfig = new MethodConfig
{
    Names = { MethodName.Default },
    RetryPolicy = new RetryPolicy
    {
        MaxAttempts = 5,
        InitialBackoff = TimeSpan.FromSeconds(1),
        MaxBackoff = TimeSpan.FromSeconds(5),
        BackoffMultiplier = 1.5,
        RetryableStatusCodes = { StatusCode.Unavailable }
    }
};

var channel = GrpcChannel.ForAddress("https://localhost:5001", new GrpcChannelOptions
{
    ServiceConfig = new ServiceConfig { MethodConfigs = { defaultMethodConfig } }
});
```

The preceding code:

* Creates a `MethodConfig`. Retry policies can be configured per-method and methods are matched using the `Names` property. This method is configured with `MethodName.Default`, so it's applied to all gRPC methods called by this channel.
* Configures a retry policy. This policy instructs clients to automatically retry gRPC calls that fail with the status code `Unavailable`.
* Configures the created channel to use the retry policy by setting `GrpcChannelOptions.ServiceConfig`.

gRPC clients created with the channel will automatically retry failed calls:

```csharp
var client = new Greeter.GreeterClient(channel);
var response = await client.SayHelloAsync(
    new HelloRequest { Name = ".NET" });

Console.WriteLine("From server: " + response.Message);
```

### gRPC retry options

The following table describes options for configuring gRPC retry policies:

| Option | Description |
| ------ | ----------- |
| `MaxAttempts` | The maximum number of call attempts, including the original attempt. This value is limited by `GrpcChannelOptions.MaxRetryAttempts` which defaults to 5. A value is required and must be greater than 1. |
| `InitialBackoff` | The initial backoff delay between retry attempts. A randomized delay between 0 and the current backoff determines when the next retry attempt is made. After each attempt, the current backoff is multiplied by `BackoffMultiplier`. A value is required and must be greater than zero. |
| `MaxBackoff` | The maximum backoff places an upper limit on exponential backoff growth. A value is required and must be greater than zero. |
| `BackoffMultiplier` | The backoff will be multiplied by this value after each retry attempt and will increase exponentially when the multiplier is greater than 1. A value is required and must be greater than zero. |
| `RetryableStatusCodes` | A collection of status codes. A gRPC call that fails with a matching status will be automatically retried. For more information about status codes, see [Status codes and their use in gRPC](https://grpc.github.io/grpc/core/md_doc_statuscodes.html). At least one retryable status code is required. |

## Hedging

Hedging is an alternative retry strategy. Hedging enables aggressively sending multiple copies of a single gRPC call without waiting for a response. Hedged gRPC calls may be executed multiple times on the server and the first successful result is used. It's important that hedging is only enabled for methods that are safe to execute multiple times without adverse effect.

Hedging has pros and cons when compared to retries: 

* An advantage to hedging is it might return a successful result faster. It allows for multiple simultaneously gRPC calls and will complete when the first successful result is available. 
* A disadvantage to hedging is it can be wasteful. Multiple calls could be made and all succeed. Only the first result is used and the rest are discarded.

## Configure a gRPC hedging policy

A hedging policy is configured like a retry policy. Note that a hedging policy can't be combined with a retry policy.

```csharp
var defaultMethodConfig = new MethodConfig
{
    Names = { MethodName.Default },
    HedgingPolicy = new HedgingPolicy
    {
        MaxAttempts = 5,
        NonFatalStatusCodes = { StatusCode.Unavailable }
    }
};

var channel = GrpcChannel.ForAddress("https://localhost:5001", new GrpcChannelOptions
{
    ServiceConfig = new ServiceConfig { MethodConfigs = { defaultMethodConfig } }
});
```

### gRPC hedging options

The following table describes options for configuring gRPC hedging policies:

| Option | Description |
| ------ | ----------- |
| `MaxAttempts` | The hedging policy will send up to this number of calls. `MaxAttempts` represents the total number of all attempts, including the original attempt. This value is limited by `GrpcChannelOptions.MaxRetryAttempts` which defaults to 5. A value is required and must be 2 or greater. |
| `HedgingDelay` | The first call will be sent immediately, but the subsequent hedging calls will be delayed by this value. When the delay is set to zero or `null`, all hedged calls are sent immediately. Default value is zero. |
| `NonFatalStatusCodes` | A collection of status codes which indicate other hedge calls may still succeed. If a non-fatal status code is returned by the server, hedged calls will continue. Otherwise, outstanding requests will be canceled and the error returned to the app. For more information about status codes, see [Status codes and their use in gRPC](https://grpc.github.io/grpc/core/md_doc_statuscodes.html). |

## Additional resources

* <xref:grpc/client>
* [Retry general guidance - Best practices for cloud applications](/azure/architecture/best-practices/transient-faults)
