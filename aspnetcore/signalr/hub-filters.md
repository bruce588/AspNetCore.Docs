---
title: Use hub filters in SignalR for ASP.NET Core
author: brecon
description: Learn how to use hub filters in ASP.NET Core SignalR.
monikerRange: '>= aspnetcore-5.0'
ms.author: brecon
ms.custom: mvc
ms.date: 05/22/2020
no-loc: [Blazor, "Identity", "Let's Encrypt", Razor, SignalR]
uid: signalr/hub-filters
---

# Use hub filters in SignalR for ASP.NET Core

Hub filters allow logic to run before and after hub methods are invoked by clients. This article will provide guidance for writing and using hub filters.

## Configure hub filters

Hub filters can be applied globally or per hub type. When adding filters the order the filters are added is the order that the filters will run in. Also, global hub filters will run before local hub filters.

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddSignalR(hubOptions =>
    {
        // Global filters will run first
        hubOptions.AddFilter<CustomFilter>();
    }).AddHubOptions<MyHub>(options =>
    {
        // Local filters will run second
        options.AddFilter<CustomFilter2>();
    });
}
```

There are 3 ways to add hub filters.
1. Add a filter by concrete type.

```csharp
hubOptions.AddFilter<TFilter>();
```

This will be resolved from DI or type activated.

2. Add a filter by runtime type.

```csharp
hubOptions.AddFilter(typeof(TFilter));
```

This will be resolved from DI or type activated.

3. Add a filter by instance.

```csharp
hubOptions.AddFilter(new MyFilter());
```

This instance will be used like a singleton, all hub method invocations will use the same instance.

Hub filters are created and disposed per hub invocation. If you want to store global state in the filter, or no state, then we recommend adding the hub filter type to DI as a singleton for better performance or add the filter as an instance if you can.

## Create hub filters

Create a filter by declaring a class that inherits from `IHubFilter`, and add the `InvokeMethodAsync` method. There is also `OnConnectedAsync` and `OnDisconnectedAsync` that can optionally be implemented to wrap the `OnConnectedAsync` and `OnDisconnectedAsync` hub methods respectively.

```csharp
public class CustomFilter : IHubFilter
{
    public async ValueTask<object> InvokeMethodAsync(HubInvocationContext invocationContext, Func<HubInvocationContext, ValueTask<object>> next)
    {
        Console.WriteLine($"Calling hub method '{invocationContext.HubMethodName}'");
        try
        {
            return await next(invocationContext);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Exception calling '{invocationContext.HubMethodName}'");
            throw ex;
        }
    }

    // Optional method
    public Task OnConnectedAsync(HubLifetimeContext context, Func<HubLifetimeContext, Task> next)
    {
        return next(context);
    }

    // Optional method
    public Task OnDisconnectedAsync(HubLifetimeContext context, Exception exception, Func<HubLifetimeContext, Exception, Task> next)
    {
        return next(context, exception);
    }
}
```

Filters are very similar to middleware, `next` will invoke the next filter and the final filter will invoke the hub method. Filters can also store the result from awaiting `next` and run logic after the hub method has been called before returning the result from `next`.

If a filter wants to skip a hub method invocation we recommend throwing a `HubException` and not calling `next` so the client will receive an error if it was expecting a result.

## Using hub filters

When writing the filter logic, try to make it generic by using attributes on hub methods instead of checking for hub method names.

For example, lets write a filter that will check an argument to a hub method for banned phrases and replace any phrases it finds with "***".
For the purposes of this example we'll assume a `LanguageFilterAttribute` class is defined and it has a property named `FilterArgument` that can be set when using the attribute.

First, place the attribute on the hub method that has a string argument that we want to clean up.
```csharp
public class ChatHub
{
    [LanguageFilter(filterArgument: 0)]
    public async Task SendMessage(string message, string username)
    {
        await Clients.All.SendAsync("SendMessage", $"{username} says: {message}");
    }
}
```

Next, define a hub filter that will check for the attribute and replace banned phrases in an argument to the hub method with "***".
```csharp
public class LanguageFilter : IHubFilter
{
    // populated from a file or inline
    private List<string> bannedPhrases = new List<string>();

    public async ValueTask<object> InvokeMethodAsync(HubInvocationContext invocationContext, Func<HubInvocationContext, ValueTask<object>> next)
    {
        var languageFilter = (LanguageFilterAttribute)Attribute.GetCustomAttribute(invocationContext.HubMethod, typeof(LanguageFilterAttribute));
        if (invocationContext.HubMethodArguments.Count > languageFilter.FilterArgument &&
            invocationContext.HubMethodArguments[languageFilter.FilterArgument] is string str)
        {
            foreach (var bannedPhrase in bannedPhrases)
            {
                str = str.Replace(bannedPhrase, "***");
            }
            if (invocationContext.HubMethodArguments is object[] arguments)
            {
                arguments[languageFilter.FilterArgument] = str;
            }
            else
            {
                arguments = invocationContext.HubMethodArguments.ToArray();
                arguments[languageFilter.FilterArgument] = str;
                invocationContext = new HubInvocationContext(invocationContext.Context, invocationContext.ServiceProvider, invocationContext.Hub, invocationContext.HubMethod, arguments);
            }
        }

        return await next(invocationContext);
    }
}
```

Finally, we need to register the hub filter. Because this filter relies on state outside of the normal hub method scope we need to register it as a singleton.
```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddSignalR(hubOptions =>
    {
        hubOptions.AddFilter<LanguageFilter>();
    });

    services.AddSingleton<LanguageFilter>();
}
```

## The HubInvocationContext object

The `HubInvocationContext` contains information for the current hub method invocation.

| Property | Description |
| ------ | ----------- |
| `Context ` | The `HubCallerContext` contains information about the connection. |
| `Hub` | The instance of the Hub being used for this hub method invocation. |
| `HubMethodName` | The name of the hub method being invoked. |
| `HubMethodArguments` | The list of arguments being passed to the hub method. |
| `ServiceProvider` | The scoped service provider for this hub method invocation. |
| `HubMethod` | The hub method information. |

## The HubLifetimeContext object

The `HubLifetimeContext` contains information for the `OnConnectedAsync` and `OnDisconnectedAsync` hub methods.

| Property | Description |
| ------ | ----------- |
| `Context ` | The `HubCallerContext` contains information about the connection. |
| `Hub` | The instance of the Hub being used for this hub method invocation. |
| `ServiceProvider` | The scoped service provider for this hub method invocation. |

## Authorization and filters

[Authorize attributes on hub methods](xref:signalr/authn-and-authz#use-authorization-handlers-to-customize-hub-method-authorization) run before hub filters.