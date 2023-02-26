---
title: Running Initialization Logic in Program.cs
tags: [til, blazor, webassembly, initialization, performance]
readtime: true
---

## Problem

[Microsoft documentation](https://docs.microsoft.com/en-us/aspnet/core/blazor/fundamentals/dependency-injection?view=aspnetcore-5.0&pivots=webassembly#add-services-to-an-app) provides an example of running application-wide initialization logic in Program.cs prior to rendering content:

```cs
public static async Task Main(string[] args)
{
    var builder = WebAssemblyHostBuilder.CreateDefault(args);
    ...
    builder.Services.AddSingleton<WeatherService>();
    ...
    var host = builder.Build();
    var weatherService = host.Services.GetRequiredService<WeatherService>();
    await weatherService.InitializeWeatherAsync();
    await host.RunAsync();
}
```

Notice that `host.RunAsync()` only runs after `weatherService.InitializeWeatherAsync()` completes; the application startup time increases by the duration of the initialization logic (even if it runs asynchronously)!

I have created a [webpage](https://zhiyuan-amos.github.io/BlazorWasmPreRenderInitialization/) demonstrating the delay ([source code](https://github.com/Zhiyuan-Amos/BlazorWasmPreRenderInitialization)), where `InitializeWeatherAsync` looks like this:

```cs
public async Task InitializeWeatherAsync()
{
    Console.WriteLine("WeatherService: Initialization started");
    await Task.Delay(10000); // simulate asynchronous work that takes 10s to complete
    Console.WriteLine("WeatherService: Initialization completed");
}
```

Observe the following sequence of events when loading the webpage:

1. Application starts up; Browser shows "Loading..."
1. Console prints "WeatherService: Initialization started"
1. After 10s, Console prints "WeatherService: Initialization completed"
1. Browser renders application UI

This is bad for User Experience as the user is stuck at the loading page for 10s before being able to see anything meaningful and interact with the application.

## Solution

If you really have to perform additional initialization logic before rendering content, then you are out of luck.

However, if that's not necessary because the user can meaningfully interact with the application before the additional initialization logic completes, then there's 2 ways about it:

1. Run the asynchronous method without `await`-ing it

    If `weatherService.InitializeWeatherAsync()` is not `await`-ed, then the thread will not wait for it to return before running `host.RunAsync()`. You can also use `ContinueWith` to process the result or perform exception handling etc.

    This solution however, still introduces delay to application startup. As of today, WASM [runs in a single-thread](https://github.com/dotnet/aspnetcore/issues/17730), so the additional initialization logic still takes up CPU cycles that would have been otherwise used to startup the application (especially so if the additional initialization logic contains a lot of code that are executed synchronously).

1. Callback

    This implementation references the pattern described in [Microsoft's documentation on In-memory State Container Service](https://docs.microsoft.com/en-us/aspnet/core/blazor/state-management?pivots=webassembly&view=aspnetcore-5.0#in-memory-state-container-service-wasm). In summary:

    1. Create a State Container with `bool _isFirstRendered` and a delegate `event Action OnChange`.

    1. Update `WeatherService`'s constructor to add `InitializeWeatherAsync()` to `OnChange`.

    1. Create a parent Component that all Components inherit from, with the following code:

        ```cs
        protected override void OnAfterRender(bool firstRender)
        {
            if (firstRender)
            {
                FirstRenderStateContainer.SetRendered(); // sets `_isFirstRendered` to true and invokes methods in `OnChange`
            }
        }
        ```

    The results of this implementation can be found on this [webpage](https://zhiyuan-amos.github.io/BlazorWasmPostRenderInitialization/) ([source code](https://github.com/Zhiyuan-Amos/BlazorWasmPostRenderInitialization)), with the startup time shortened by 10s compared to the earlier example, since the additional initialization logic is executed after the UI has been rendered rather than before.

    Again, observe the output on Console for the sequence of events.

## Conclusion

Mobile devices with limited processing power like mine (Samsung Galaxy A31) takes about 4 seconds to load a barebone Blazor WASM application, which is really long. Therefore, reduce additional initialization logic in Program.cs as far as possible.

Also, if you have a better idea on how this problem can be resolved, do create an issue [here](https://github.com/Zhiyuan-Amos/BlazorWasmPostRenderInitialization) and let's discuss about it!
