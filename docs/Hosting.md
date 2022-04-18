---
title: Hosting models
subtitle: WebAssembly or server-side
---

Bolero applications can run in two different modes: WebAssembly (also referred to as client-side Bolero) and server-side.

In WebAssembly mode, dynamic content runs on the client side using [WebAssembly](https://webassembly.org/).
It is still a full .NET implementation, but running directly in the user's browser.

In server-side mode, dynamic content runs on the server side and is shared with the user's browser via a [SignalR](https://dotnet.microsoft.com/apps/aspnet/signalr) connection.

You can learn more about these hosting models in [the official Blazor documentation](https://docs.microsoft.com/en-us/aspnet/core/blazor/hosting-models?view=aspnetcore-5.0).

### Hosting models

#### Plain WebAssembly mode

To create a plain WebAssembly application, use the [dotnet project template](index#creating-a-project) with the following arguments:

```sh
dotnet new bolero-app --server=false
```

In plain WebAssembly mode, the static content of the application is a simple `.html` file, and the dynamic Bolero content is rendered in a tag of this file.
This file must contain the following:

```html
<script src="_framework/blazor.webassembly.js"></script>
```

as well as a container element for the dynamic content, for example `<div id="main"></div>`.
This container is selected in `Startup.fs`:

```fsharp
builder.RootComponents.Add<Main.MyApp>("#main")
```

Note that prerendering is not possible in plain WebAssembly mode.

#### Hosted WebAssembly mode

Hosted WebAssembly is the default hosting mode created by the [dotnet project template](index#creating-a-project):

```sh
dotnet new bolero-app
```

In hosted WebAssembly mode, the static content of the application is rendered on the ASP.NET Core server side.
[See below](#static-content-generation) for the different ways it can be generated.

Like in plain WebAssembly mode, the static content must include a container element matching the selector used in `Startup.fs`.

Unlike in plain WebAssembly mode, prerendering is possible based on the [configuration](#configuring-hosted-modes).

#### Server-side mode

To create a server-side Bolero app, create a default [dotnet project template](index#creating-a-project):

```sh
dotnet new bolero-app
```

and in `src/AppName.Server/Startup.fs`, replace the line:

```fsharp
.AddBoleroHost()
```

with:

<!-- TODO: .AddServerSideBlazor()? if we remove it from template -->
```fsharp
.AddBoleroHost(server = true)
```

The static content is rendered exactly like hosted WebAssembly mode, [see below](#static-content-generation).

### Configuring hosted modes

Hosted WebAssembly and server-side are known collectively as hosted modes, and they share a lot of configuration and features.

#### Configuration

Hosted modes are configured by using `AddBoleroHost` in the (server-side) dependency injection.

```fsharp
type Startup() =

    member this.ConfigureServices(services: IServiceCollection) =
        services.AddBoleroHost() |> ignore
        // ... other dependencies ...
```

This method takes several optional arguments:

* `server: bool` determines whether this is a hosted WebAssembly (`server = false`) or server-side application (`server = true`).

    The default is `false`.
    
    Note: when setting to `true`, make sure that `services.AddServerSideBlazor() |> ignore` is also called.

* `prerendered: bool` determines whether the dynamic Bolero content is prerendered.

    If true, then the content is rendered on the server side (even in WebAssembly mode) and the resulting HTML is included in the static content served by ASP.NET Core.
    This prevents the "pop-in" effect where the user first sees an empty container, and once the page is ready (ie WebAssembly has started or SignalR has connected), the content suddenly appears.

    The default is `true`.

* `devToggle: bool` determines whether the user can choose between hosted WebAssembly and server-side mode by passing `?server=false` or `?server=true`, respectively, in the URL.

    This feature is intended for development only.
    For example, when writing a hosted WebAssembly application, it can be convenient to temporarily switch to server-side mode for debugging.
    
    This setting is only active when the ASP.NET Core application runs in the `Development` environment; in any other environment, it has no effect.

    The default is `true`.

#### Static content generation

There are multiple ways available to generate the static HTML content that contains a hosted Bolero app.

##### Bolero HTML

> Introduced in v0.17.

The recommended way to generate static content for Bolero is to use the same [HTML functions](HTML) that are used by dynamic content.
It is the default method used by the dotnet project template, or can be explicitly used with `--hostpage=bolero`.

A few additional functions are available in the module `Bolero.Server.Html`:

* `doctypeHtml` creates a `<html>` tag preceded by a standard doctype declaration.
    Like other element functions, it takes as arguments a list of attributes and a list of child elements.
    
* `rootComp<T>` inserts a component of type `T` as dynamic content.

    This may include prerendered content, depending on the `prerendered` value passed to `AddBoleroHost`.

    In WebAssembly mode, this must be inserted inside a container element matched by the selector used in `Startup.fs`.

    In WebAssembly mode without prerendering, this actually doesn't insert any content.
    Indeed, in this mode, all that is needed is the container element and the `boleroScript` (see below).
    So if you know that you will always run your app in WebAssembly mode without prerendering, then you can simply not insert the `rootComp`.
    This way, you won't need a reference from the server project to the client project.

    Note that this is the only way to insert dynamic content in a static page.
    If you use functions like `comp` or `ecomp`, the component's initial HTML will be inserted in the page, but no dynamic behavior will take place.

* `boleroScript` inserts a `<script>` tag pointing to the JavaScript file that starts the application.
    It knows which script to insert (either `blazor.webassembly.js` or `blazor.server.js`) based on the `server` and `devToggle` values passed to `AddBoleroHost`.

Here is an example page using all of the above:

```fsharp
open Bolero.Html
open Bolero.Server.Html

let myPage = doctypeHtml {
    head {
        title { "Hello, world!" }
    }
    body {
        div { "This is the body of the page" }
        div {
            attr.id "main" 
            rootComp<Client.Main.MyApp>
        }
        boleroScript
    }
}
```

Such a page can be served as follows in the server-side Startup:

```fsharp
type Startup() =

    member this.Configure(app: IApplicationBuilder) =
        app.UseStaticFiles()
            .UseRouting()
            .UseBlazorFrameworkFiles() // Necessary in hosted WebAssembly mode
            .UseEndpoints(fun endpoints ->
                endpoints.MapBlazorHub() |> ignore // Necessary in server-side mode
                endpoints.MapFallbackToBolero(myPage) |> ignore
            )
        |> ignore
```

Alternatively, it can be used as the content of an ASP.NET Core MVC controller:

```fsharp
open Microsoft.AspNetCore.Mvc
open Bolero.Server

type MyController() =
    inherit Controller()
    
    member this.Index() =
        this.BoleroPage(myPage)
```

##### Razor

Another option is to use a Razor page.
This is particularly convenient when integrating the application in an ASP.NET Core application written in C#.
It can also be used with an F# application with [Razor runtime compilation](https://docs.microsoft.com/en-us/aspnet/core/mvc/views/view-compilation?view=aspnetcore-5.0&tabs=visual-studio#enable-runtime-compilation-in-an-existing-project); this was the mode used by the dotnet template until Bolero 0.16.
Starting with 0.17, it can be used with `--hostpage=razor`.

Bolero provides the type `IBoleroHostConfig` and a few extension methods on Razor's `Html` to help render components and Blazor JavaScript tags using the configuration from `AddBoleroHost`:

* `RenderComponentAsync<T>(IBoleroHostConfig)` inserts a component of type `T`.

    This may include prerendered content, depending on the `prerendered` value passed to `AddBoleroHost`.

    In WebAssembly mode, this must be inserted inside a container element matched by the selector used in `Startup.fs`.
    
* `RenderBoleroScript(IBoleroHostConfig)` inserts a `<script>` tag pointing to the JavaScript file that starts the application.
    It knows which script to insert based on the `server` and `devToggle` values passed to `AddBoleroHost`.

Here is an example page using all of the above:

```razor
@page "/"
@namespace MyApp.Server
@using Bolero.Server
@inject IBoleroHostConfig BoleroHostConfig
<!DOCTYPE html>
<html>
  <head>
    <title>Hello, world!</title>
  </head>
  <body>
    <div>This is the body of the page</div>
    <div id="main">
      @(await Html.RenderComponentAsync<MyApp.Client.Main.MyApp>(BoleroHostConfig))
    </div>
    @Html.RenderBoleroScript(BoleroHostConfig)
  </body>
</html>
```

Such a page saved as `Pages/_Host.cshtml` can be served as follows in the server-side Startup:

```fsharp
type Startup() =

    member this.Configure(app: IApplicationBuilder) =
        app.UseStaticFiles()
            .UseRouting()
            .UseBlazorFrameworkFiles() // Necessary in hosted WebAssembly mode
            .UseEndpoints(fun endpoints ->
                endpoints.MapBlazorHub() |> ignore // Necessary in server-side mode
                endpoints.MapFallbackToPage("/_Host") |> ignore
            )
        |> ignore
```

Or in a C# application:

```csharp
public class Startup
{
    public void Configure(IApplicationBuilder app)
    {
        app.UseStaticFiles()
            .UseRouting()
            .UseBlazorFrameworkFiles() // Necessary in hosted WebAssembly mode
            .UseEndpoints(endpoints =>
            {
                endpoints.MapBlazorHub(); // Necessary in server-side mode
                endpoints.MapFallbackToPage("/_Host");
            });
    }
}
```
