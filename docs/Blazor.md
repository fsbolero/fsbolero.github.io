---
title: Using Blazor features
subtitle: ""
---

This page documents some useful features of Blazor and how to best use them from a Bolero application.

### Dependency injection

Blazor provides the ability to work with the dependency injection mechanism introduced by ASP.NET Core. [Learn more about it on the official documentation](https://docs.microsoft.com/en-us/aspnet/core/blazor/dependency-injection?view=aspnetcore-3.0).

#### Requiring a dependency

Any Blazor component can require a dependency. This includes the Bolero `ProgramComponent`, as well as any `Component` class you create. This is done by creating a mutable property with the attribute `Microsoft.AspNetCore.Components.Inject`:

```fsharp
open Microsoft.AspNetCore.Components
open Bolero

type MyApp() =
    inherit ProgramComponent<Model, Message>()
    
    [<Inject>]
    member val MyDependency = Unchecked.defaultof<IMyDependency> with get, set
    
    override this.Program =
        doSomethingWith this.MyDependency
```

#### Providing a dependency

Dependencies are injected from the client-side `Startup` class's `ConfigureServices` method:

```fsharp
type Startup() =

    member __.ConfigureServices(services: IServiceCollection) =
        services.AddSingleton<IMyDependency>(new MyDependency()) |> ignore
```
