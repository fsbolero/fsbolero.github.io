---
title: Upgrade guide
subtitle: How to update a project for newer releases
---

### From v0.15 to v0.16

Bolero v0.16 upgrades the dependency to .NET 5. Here are the associated upgrade steps:

* Install [the .NET SDK 5.0.100 or newer](https://dotnet.microsoft.com/download).

* If you are using Visual Studio, it is recommended to upgrade to version 16.8.

* In your *client-side* project files:

    * Replace the line:
    
        ```xml
        <Project Sdk="Microsoft.NET.Sdk.Web">
        ```
        
        with:
        
        ```xml
        <Project Sdk="Microsoft.NET.Sdk.BlazorWebAssembly">
        ```
        
    * Remove the dependency on the NuGet package `Microsoft.AspNetCore.Components.WebAssembly.Build`.

* In *all* your project files:

    * Replace the `TargetFramework` with `net5.0`.
    
    * Update the dependency version on all `Microsoft.AspNetCore.*` packages to `5.0.*` (for NuGet) or `~> 5.0.0` (for Paket).

### From v0.14 to v0.15

Bolero v0.15 doesn't change the SDK requirements. Here are the upgrade steps:

* Elmish has been updated to v3.0. As a consequence, some functions are now obsolete.
    For example, `Cmd.ofAsync` should now be replaced with `Cmd.OfAsync.either`.
    It is a simple renaming, you can simply follow the compiler warnings.
    
* The HTML element reference API has changed:
    * The type `ElementRefBinder` was renamed to `HtmlRef`;
    * its `Ref` property was renamed to `Value`;
    * The function `attr.bindRef` was renamed to `attr.ref`;
    * The function `attr.ref` taking a function as argument has been removed.

### From v0.13 to v0.14

Bolero 0.14 doesn't change the SDK requirements. Here are the upgrade steps:

* The module `Bolero.Json` has been removed.
    Instead, Bolero's remoting now uses the standard `System.Text.Json`, together with the library [`FSharp.SystemTextJson`](https://github.com/tarmil/fsharp.systemtextjson) to provide support for F# types such as records, unions, options, lists, etc.
    If you had JSON format customization based on Bolero.Json attributes, you must replace them with System.Text.Json-based customization.
    Otherwise, there is nothing to do.

### From v0.12 to v0.13

Bolero 0.13 upgrades the dependency on .NET Core SDK 3.1.300 or newer and on Blazor to 3.2.0. Here are the associated upgrade steps:

* Install [the .NET Core 3.1.300 SDK or newer](https://dotnet.microsoft.com/download/dotnet-core).

* If you are using Visual Studio, it is recommended to upgrade to version 16.6.

* If you use remoting, in your client-side startup, replace the following:

    ```fsharp
    builder.Services.AddRemoting() |> ignore
    ```

    with:

    ```fsharp
    builder.Services.AddRemoting(builder.HostEnvironment) |> ignore
    ```

    Note also that `AddRemoting` now adds a named `HttpClient` intended solely for use by remoting.
    If you were using the injected `HttpClient` for your own purposes, then you will need to inject one separately using the standard method:

    ```fsharp
    builder.Services.AddSingleton(new HttpClient(BaseAddress = Uri(builder.HostEnvironment.BaseAddress))) |> ignore
    ```

* There is a new `BoleroHostConfig` which simplifies the configuration of the server-side hosting.
    While the `_Host.cshtml` and `HostModel.fs` that were included in previous versions of the project template should continue to work correctly, it is recommended to switch to using `BoleroHostConfig`.

    * Remove `HostModel.fs`.

    * In `Pages/_Host.cshtml`, apply the following change:

        ```diff
        @page "/"
        @namespace HelloWorld
        -@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
        -@model HostModel
        +@using Bolero.Server.RazorHost
        +@inject IBoleroHostConfig BoleroHostConfig
        <!DOCTYPE html>
        <html>
        <head>
          <title>Hello world!</title>
          <meta charset="UTF-8" />
          <base href="/" />
        </head>
        <body>
        -  <div id="main">@(await Html.RenderComponentAsync<HelloWorld.Client.MyApp>(Model.IsServer ? RenderMode.ServerPrerendered : RenderMode.Static))</div>
        +  <div id="main">@(await Html.RenderComponentAsync<HelloWorld.Client.MyApp>(BoleroHostConfig))</div>
        -  <script src="_framework/blazor.@(Model.IsServer ? "server" : "webassembly").js"></script>
        +  @Html.RenderBoleroScript(BoleroHostConfig)
        </body>
        </html>
        ```

    * In `Startup.fs`, open the namespace `Bolero.Server.RazorHost` and inject `BoleroHostConfig` in `ConfigureServices`:

        ```fsharp
        services.AddBoleroHost() |> ignore
        ```

        This method takes optional arguments that configure the hosting of the application (server vs client, prerendered or not, etc).

### From v0.11 to v0.12

Bolero 0.12 upgrades the dependency on .NET Core SDK 3.1.102 or newer and on Blazor to 3.2-preview2. Here are the associated upgrade steps:

* Install [the .NET Core 3.1.102 SDK or newer](https://dotnet.microsoft.com/download/dotnet-core).

* If you are using Visual Studio, check on the above page for the corresponding version (16.4 or 16.5).

* Follow closely the upgrade guides for [Blazor 3.2-preview1](https://devblogs.microsoft.com/aspnet/blazor-webassembly-3-2-0-preview-1-release-now-available/) and [Blazor 3.2-preview2.](https://devblogs.microsoft.com/aspnet/blazor-webassembly-3-2-0-preview-2-release-now-available/) Note in particular the NuGet package renames in the latter. Regarding `WebAssemblyHost`, see the client-side changes below.

* In your client `.fsproj` file, inside the `<PropertyGroup>` tag, add the following:

    ```xml
    <RazorLangVersion>3.0</RazorLangVersion>
    ```

* The client-side startup has been completely revamped in Blazor 3.2; [see the Blazor 3.2-preview1 announcement for more details.](https://devblogs.microsoft.com/aspnet/blazor-webassembly-3-2-0-preview-1-release-now-available/). A startup file like the following:

    ```fsharp
    namespace HelloWorld.Client

    open Microsoft.AspNetCore.Blazor.Hosting
    open Microsoft.Extensions.DependencyInjection
    open Microsoft.AspNetCore.Components.Builder

    type Startup() =

        member __.ConfigureServices(services: IServiceCollection) =
            // [1] Services configuration:
            services.AddRemoting() |> ignore

        member __.Configure(app: IComponentsApplicationBuilder) =
            // [2] Component configuration:
            app.AddComponent<MyApp>("#main")

    module Program =

        [<EntryPoint>]
        let Main args =
            BlazorWebAssemblyHost.CreateDefaultBuilder()
                 .UseBlazorStartup<Startup>()
                 .Build()
                 .Run()
            0
    ```

    becomes:

    ```fsharp
    namespace HelloWorld.Client

    open Microsoft.AspNetCore.Components.WebAssembly.Hosting

    module Program =

        [<EntryPoint>]
        let Main args =
            let builder = WebAssemblyHostBuilder.CreateDefault(args)

            // [1] Services configuration:
            builder.Services.AddRemoting() |> ignore

            // [2] Component configuration:
            builder.RootComponents.Add<MyApp>("#main")

            builder.Build().RunAsync() |> ignore
            0
    ```

* In the server-side startup, replace the following:

    ```fsharp
    app.UseClientSideBlazorFiles<Client.Main.MyApp>()
    ```

    with:

    ```fsharp
    app.UseBlazorFrameworkFiles()
    ```

    and the following:

    ```fsharp
    .UseEndpoints(fun endpoints ->
        endpoints.MapDefaultControllerRoute() |> ignore
        endpoints.MapFallbackToClientSideBlazor<Client.Startup>("index.html") |> ignore)
    ```

    with:

    ```fsharp
    .UseEndpoints(fun endpoints ->
        endpoints.MapControllers() |> ignore
        endpoints.MapFallbackToFile("index.html") |> ignore)
    ```

    Additionally, in the `main` function, under `WebHost.CreateDefaultBuilder(args)`, add:

    ```fsharp
        .UseStaticWebAssets()
    ```

    See [the Blazor 3.2-preview2 announcement](https://devblogs.microsoft.com/aspnet/blazor-webassembly-3-2-0-preview-2-release-now-available/) for more details.

### From v0.10 to v0.11

Bolero 0.11 upgrades the dependency on .NET Core to 3.1 and on Blazor to 3.1-preview4. Here are the associated upgrade steps:

* Install [the .NET Core 3.1 SDK or newer](https://dotnet.microsoft.com/download/dotnet-core).

* If you are using Visual Studio, upgrade to version 16.4.

* Change the `<TargetFramework>` of your client `.fsproj` file from `netstandard2.0` to `netstandard2.1`.

* Change the `<TargetFramework>` of your server `.fsproj` file from `netcoreapp3.0` to `netcoreapp3.1`.

* The API for binders have been changed.  
  The `bind` module now contains submodules `bind.input` and `bind.change` which in turn contain functions for the type of value being bound: `string`, `int`, `int64`, `float`, `float32`, `decimal`, `dateTime` and `dateTimeOffset`.  
  Additionally, a module `bind.withCulture` contains the same submodules with functions taking an additional `CultureInfo` as argument to specify the culture to use to parse the value.

  For example, the following:

  ```fsharp
  concat [
      input [bind.input model.name (fun n -> dispatch (SetName n))]
      input [bind.inputInt model.age (fun a -> dispatch (SetAge a))]
  ]
  ```

  becomes:

  ```fsharp
  concat [
      input [bind.input.string model.name (fun n -> dispatch (SetName n))]
      input [bind.input.int model.age (fun a -> dispatch (SetAge a))]
  ]
  ```

### From v0.9 to v0.10

Bolero 0.10 doesn't change the dependencies from 0.9. It has one breaking change:

* The function `ecomp` now takes an additional list of attributes as first argument.

### From v0.8 to v0.9

Bolero 0.9 upgrades the dependency on .NET Core to 3.0 RTM and on Blazor to 3.0-preview9. It doesn't include any breaking changes. Here are the associated upgrade steps:

* Install [the .NET Core 3.0 SDK](https://dotnet.microsoft.com/download/dotnet-core).

* If you are using Visual Studio, upgrade to version 16.3.

### From v0.7 to v0.8

Bolero 0.8 upgrades the dependency on Blazor and .NET Core to 3.0-preview8. Here are the associated upgrade steps:

* Install [the .NET Core 3.0-preview8 SDK](https://dotnet.microsoft.com/download/dotnet-core).

* The server-side module `Remote` has been removed, with its functions `withHttpContext`, `authorize` and `authorizeWith`.

    Instead, a new type `IRemoteContext` is provided via dependency injection:

    ```fsharp
    type IRemoteContext =
        inherit IHttpContextAccessor // member HttpContext : HttpContext with get, set
        member Authorize : ('req -> Async<'resp>) -> ('req -> Async<'resp>)
        member AuthorizeWith : seq<IAuthorizeData> -> ('req -> Async<'resp>) -> ('req -> Async<'resp>)
    ```

    To obtain this new value:

    * If you use a `RemoteHandler` class with dependency injection, simply take `ctx: IRemoteContext` as additional argument in its constructor.

    * If you use a plain record value, make it instead a function that takes `ctx: IRemoteContext` as argument and returns the record.
        New overloads on `IServiceCollection.AddRemoting` can take such a function as argument.

    Then, to use it:

    * Replace `Remote.withContext <| fun http arg -> // ...` with a simple `fun arg -> ...`.

    * Replace `Remote.authorize <| fun http arg -> // ...` with `ctx.Authorize <| fun arg -> ...`.

    * Replace `Remote.authorizeWith attrs <| fun http arg -> // ...` with `ctx.AuthorizeWith attrs <| fun arg -> ...`.

    In all three cases, use `ctx.HttpContext` instead of `http` in the function.

### From v0.6 to v0.7

Bolero 0.7 updates the dependency on Blazor and .NET Core to 3.0-preview7. Here are the associated upgrade steps:

* Install [the .NET Core 3.0-preview7 SDK](https://dotnet.microsoft.com/download/dotnet-core).

* `Cmd.ofRemote`, `Cmd.performRemote` and the related type `RemoteResponse<'T>` are obsolete. Here is how to update:

    * If the remote call is to a non-authorized function, then simply use `Cmd.ofAsync` / `Cmd.performAsync`.
    
    * If the remote call is to an authorized function, then replace `RemoteResponse<'T>` with `option<'T>`, `Cmd.ofRemote` with `Cmd.ofAuthorized` and `Cmd.performRemote` with `Cmd.performAuthorized`.

### From v0.5 to v0.6

Bolero 0.6 updates the dependency on Blazor and .NET Core to 3.0-preview6. Here are the associated upgrade steps:

* Install [the .NET Core 3.0-preview6 SDK](https://dotnet.microsoft.com/download/dotnet-core).

* The server-side `Startup.fs` code has changed. In `Configure`, the method `UseBlazor` has been removed; replace it with the following:

    ```fsharp
        .UseClientSideBlazorFiles<Client.Startup>()
        .UseRouting()
        .UseEndpoints(fun endpoints ->
            endpoints.MapDefaultControllerRoute() |> ignore
            endpoints.MapFallbackToClientSideBlazor<Client.Startup>("index.html") |> ignore)
    ```

    This requires the addition of the following line in `ConfigureServices`:

    ```fsharp
        services.AddMvcCore() |> ignore
    ```

HTML hot reloading also received changes.

* A new NuGet package `Bolero.HotReload.Server` needs to be referenced by the `Server` project.

* `IAppBuilder.UseHotReload` is deprecated; instead, add `endpoints.UseHotReload()` at the top of the `UseEndPoints` callback:

    ```fsharp
        .UseClientSideBlazorFiles<Client.Startup>()
        .UseRouting()
        .UseEndpoints(fun endpoints ->
            endpoints.UseHotReload()
            endpoints.MapDefaultControllerRoute() |> ignore
            endpoints.MapFallbackToClientSideBlazor<Client.Startup>("index.html") |> ignore)
    ```

### From v0.4 to v0.5

![Blazor 3.0](../img/blazor-icon.png)

Bolero 0.5 doesn't bring any breaking API changes.
However, it upgrades the dependency on Blazor from 0.7 to 3.0-preview5; and by doing so, it also upgrades the dependency on .NET Core from 2.1 to 3.0-preview5.
This brings a number of necessary changes to projects. Here are the necessary steps:

* Install [the .NET Core 3.0-preview5 SDK](https://dotnet.microsoft.com/download/dotnet-core).

* If your solution contains a `global.json` file with an explicit SDK version, either remove the file or update its SDK version to `3.0.100-preview5`.

* In `paket.dependencies`:
  * In the `framework` declaration, replace `netcoreapp2.1` with `netcoreapp3.0`.
  * Remove the following `nuget` lines:
    * `Microsoft.AspNetCore.App` (In ASP.NET Core 3.0, it is automatically included by the SDK)
    * `Microsoft.AspNetCore.Razor.Design`
    * `Microsoft.AspNetCore.SignalR`
  * Replace `clitool Microsoft.AspNetCore.Blazor.Cli` with `nuget Microsoft.AspNetCore.Blazor.DevServer`.
  * For all the `Microsoft.AspNetCore.Blazor*` and `Bolero*` dependencies, remove version constraints if there are any (such as `~> 0.7.0`), and add `prerelease` as a version constraint.

* In `src/*.Client/paket.references`:
  * Replace `Microsoft.AspNetCore.Blazor.Cli` with `Microsoft.AspNetCore.Blazor.DevServer`.

* In `src/*.Server/paket.references`:
  * Remove `Microsoft.AspNetCore.App`.

* In both `*.fsproj` files, replace the `TargetFramework` from `netcoreapp2.1` to `netcoreapp3.0`. Also, remove the following lines from `*.Client.fsproj`:
    ```xml
    <RunCommand>dotnet</RunCommand>
    <RunArguments>blazor serve</RunArguments>
    ```

You can now run `.paket/paket update` to update the packages.
There are also a few code changes to apply:

* In `src/*.Client/Startup.fs`:
  * Replace `Microsoft.AspNetCore.Blazor.Builder` with `Microsoft.AspNetCore.Components.Builder`.
  * Replace `IBlazorApplicationBuilder` with `IComponentsApplicationBuilder`.

* In `src/*.Server/Startup.fs`:
  * Replace `IHostingEnvironment` with `IWebHostEnvironment`.

Your solution should now build and run successfully.
If your code uses Blazor APIs directly, please check the Blazor [0.8](https://devblogs.microsoft.com/aspnet/blazor-0-8-0-experimental-release-now-available/), [0.9](https://devblogs.microsoft.com/aspnet/blazor-0-9-0-experimental-release-now-available/) and [3.0](https://devblogs.microsoft.com/aspnet/blazor-now-in-official-preview/) upgrade guides for more code changes you might need to make.

If you are having any issues with this upgrade, don't hesitate to [post an issue on GitHub!](https://github.com/fsbolero/bolero/issues)
