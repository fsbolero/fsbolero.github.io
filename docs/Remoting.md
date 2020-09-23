---
title: Remoting
subtitle: Easily call server-side functions from the client side.
---

### Defining the service

A set of server-side functions is defined as a record called a *remote service*. Each function is a field in this record, and must take one argument and return `Async<_>`. If you need to pass several arguments to a server-side function, use a tuple.

The record should implement `IRemoteService` to define the URL for its functions. Each function is served at the path `{service.BasePath}/{fieldName}`.

For example, here is the definition of a service for a simple key-value pair storage:

```fsharp
open Bolero.Remoting

type MyService =
    {
        getEntry : string -> Async<string option>   // Served at /myService/getEntry
        setEntry : string * string -> Async<unit>   // Served at /myService/setEntry
        deleteEntry : string -> Async<unit>         // Served at /myService/deleteEntry
    }

    interface IRemoteService with
        member this.BasePath = "/myService"
```

Remote calls are `POST` requests to the function's URL. Arguments and return values are automatically serialized to JSON.

### Calling on the client side

On the client side, you will typically want to call these functions in the `update` of the Elmish app. See [the Elmish documentation](https://elmish.github.io/elmish/basics.html) to learn how to run commands in `update`.

1. In your Blazor startup (`Client/Startup.fs`), add support for remoting:

    ```fsharp
    open Bolero.Remoting.Client

    type Startup() =

        member __.ConfigureServices(services: IServiceCollection) =
            services.AddRemoting()
            |> ignore
    ```

2. Retrieve the client-side service in the `ProgramComponent` by using `this.Remote`:

    ```fsharp
    open Bolero.Remoting

    type App() =
        inherit ProgramComponent<Model, Message>()

        override this.Program =
            // Retrieve the service
            let myService = this.Remote<MyService>()
            // Pass it to `update`
            Program.mkProgram (fun _ -> initModel, []) (update myService) view
    ```

3. In `update`, use the service in `Cmd`s:

    ```fsharp
    type Model =
      { latestRetrievedEntry : string * string
        latestError : exn option }

    type Message =
        // Trigger a `getEntry` request
        | GetEntry of key: string
        // Received response of a `getEntry` request
        | GotEntry of key: string * value: string
        // A request threw an error
        | Error of exn

    let update myService message model =
        match message with
        | GetEntry key ->
            model,
            Cmd.ofAsync
                myService.getEntry key              // async call and argument
                (fun value -> GotEntry(key, value)) // message to dispatch on response
                Error                               // message to dispatch on error
        | GotEntry(key, value) ->
            { model with latestRetrievedEntry = (key, value) }, []
        | Error exn ->
            { model with latestError = Some exn }, []

    ```

### Defining on the server side

On the server side, Bolero.Remoting is registered as a service and added as ASP.NET Core middleware. There are several ways to do so.

#### A simple service

Here is how to implement a remote service without any dependencies.

1. Implement the service as a value:

    ```fsharp
    // A simple global map as storage.
    // A real-world app would probably use a database instead.
    let mutable storage = Map.empty

    let myService =
        {
            getEntry = fun key -> async {
                return Map.tryFind key
            }
            setEntry = fun (key, value) -> async {
                storage <- Map.add key value storage
            }
            deleteEntry = fun key -> async {
                storage <- Map.remove key storage
            }
        }
    ```

2. In your ASP.NET Core startup (`Server/Startup.fs`), register the service:

    ```fsharp
    open Bolero.Remoting.Server

    type Startup() =

        member this.ConfigureServices(services: IServiceCollection) =
            services.AddRemoting(myService)
            |> ignore
    ```

3. In your ASP.NET Core startup, start the remoting middleware:

    ```fsharp
    type Startup() =

        member this.Configure(app: IApplicationBuilder) =
            app.UseRemoting()
                // .OtherMethods()...
            |> ignore
    ```

    Note that `UseRemoting` (and any other middleware) should be called *before* `UseRouting` to make sure that it catches the requests to its endpoints.

#### Using dependency injection

You might need to use injected dependencies in a remote service: a logger, a database connection, etc. For this, you need a different approach.

1. Implement the service as a class inheriting from `RemoteHandler`. Dependencies can be injected from the constructor.

    ```fsharp
    type MyServiceHandler(log: ILogger<MyServiceHandler>) =
        inherit RemoteHandler<MyService>()

        let mutable storage = Map.empty

        override this.Handler =
            {
                getEntry = fun key -> async {
                    log.LogInformation("Retrieving {0}", key)
                    return Map.tryFind key
                }
                setEntry = fun (key, value) -> async {
                    log.LogInformation("Setting {0} to {1}", key, value)
                    storage <- Map.add key value storage
                }
                deleteEntry = fun key -> async {
                    log.LogInformation("Deleting {0}", key)
                    storage <- Map.remove key storage
                }
            }
    ```

2. In your ASP.NET Core startup, register the service by type rather than by instance:

    ```fsharp
    type Startup() =

        member this.ConfigureServices(services: IServiceCollection) =
            services.AddRemoting<MyServiceHandler>()
            |> ignore
    ```

3. In your ASP.NET Core startup, start the remoting middleware, just like for a simple service.

#### IRemoteContext

> Introduced in v0.8.

Bolero remoting provides a value of type `IRemoteContext` for multiple purposes. For example, its `HttpContext` property gives access to the ASP.NET Core `Microsoft.AspNetCore.Http.HttpContext` of the request.

Here is how to obtain an `IRemoteContext`:

* If you use [dependency injection](#using-dependency-injection), then simply inject `IRemoteContext` into the constructor:

    ```fsharp
    type MyServiceHandler(ctx: IRemoteContext) =
        // ...
    ```

* If you are not using dependency injection, you can replace your handler record value with a function taking `IRemoteContext` as argument and returning a record.

#### Using several services

You can of course define several remote services in the same application. Each of them needs to be registered by a separate call to `AddRemoting` in `ConfigureServices`. A single call to `UseRemoting` is enough in `Configure`.

### Authentication and authorization

> Introduced in v0.4.
>
> This has changed significantly in v0.8; [see the old documentation](https://github.com/fsbolero/website/blob/7a4e957/src/Website/docs/Remoting.md#authentication-and-authorization) for authentication and authorization in versions 0.4 through 0.7.

Bolero includes facilities for remote function authentication and authorization. They are based on standard ASP.NET Core functionality.

* **Authentication** means signing in, signing out and identifying the current user in remote functions.
* **Authorization** means specifying that a given remote function can only be used by authenticated users, optionally with additional criteria such as "is admin".

#### Authentication

Authentication is done using standard ASP.NET Core authentication features. Enabling it is therefore done like a usual ASP.NET Core application. Here is an example setup for the server-side `Startup.fs` using cookie authentication:

* In `ConfigureServices`, use the following:

    ```fsharp
    services
        .AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
            .AddCookie()
            .Services
        //.OtherMethods()...
    |> ignore
    ```

* In `Configure`, use the following:

    ```fsharp
    app.UseAuthentication()
        .UseRemoting()
        // .OtherMethods()...
    |> ignore
    ```

To learn more about ASP.NET Core authentication, you can check the official documentation [here](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/cookie?view=aspnetcore-2.2).

Authentication in a remote function uses the `Microsoft.AspNetCore.Http.HttpContext` provided by `IRemoteContext` ([see above](#iremotecontext)).

`HttpContext` has a lot of methods and properties. The extension methods added by Bolero and relevant for authentication are:

* `AsyncSignIn()` is used to sign the user in. It takes a `username: string` argument, and a number of optional arguments:
    * `persistFor: TimeSpan` decides for how long the signin lasts. If unset, the signin lasts for the duration of the user's browser session.
    * `claims: seq<Claim>` adds identity claims to the user, in addition to the `Name` claim that is created automatically for the username. Learn more about identity claims [here](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/claims?view=aspnetcore-2.2). You can see it in used below when discussing `Remote.authorizeWith`.
    * `properties: AuthenticationProperties` adds authentication properties. `IsPersistent` and `ExpiresUTC` are overridden by `persistsFor` if that is used.
    * `authenticationType: string` sets the authentication type for the identity principal. The default is `"Bolero.Remoting"`.
* `AsyncSignOut()` is used to sign the user out. It takes a single optional argument, `properties: AuthenticationProperties`.
* `TryUsername()` retrieves the current user's username, as a `string option`.
* `TryIdentity()` retrieves the current user's ASP.Net Core identity, as a `ClaimsIdentity option`.

For example, the following remote service implements simple login, logout and username retrieval:

```fsharp
type LoginService =
    {
        signIn : string -> Async<unit>
        signOut : unit -> Async<unit>
        getUsername : unit -> Async<string option>
    }

let loginService (ctx: IRemoteContext) =
    {
        signIn = fun username -> async {
            if password = "password" then // Replace this with a proper check!
                return! ctx.HttpContext.AsyncSignIn(username)
        }
        signOut = fun () -> async {
            return! ctx.HttpContext.AsyncSignOut()
        }
        getUsername = fun () -> async {
            return! ctx.HttpContext.TryUsername()
        }
    }
```

#### Authorization

Authorization also uses standard ASP.NET Core features. It is enabled in the startup class's `ConfigureServices` method:

```fsharp
member this.ConfigureServices(services: IServiceCollection) =
    services
        .AddAuthorization()
        //.OtherMethods()...
    |> ignore
```

You can then mark a remote function as authorized, ie. callable only by authenticated users, by wrapping it in a call to the method `Authorize` on [IRemoteContext](#iremotecontext).

```fsharp
type UserDataService =
    {
        getSecretData : unit -> Async<string>
    }

let userDataService (ctx: IRemoteContext) =
    {
        getSecretData = ctx.Authorize <| fun () -> async {
            // User is guaranteed to be authenticated here.
            return "Secret user data!"
        }
    }
```

You can use more fine-tuned authorization policies using `AuthorizeWith`. This method takes a list of ASP.NET Core `AuthorizeAttribute`s that specifies the authorization policy for this function. The following example can only be called by a user who was signed in as admin:

```fsharp
type UserDataService =
    {
        signIn : string * string -> Async<string>
        getSecretData : unit -> Async<string>
    }

let userDataService (ctx: IRemoteContext) =
    {
        signIn = fun (username, password) -> async {
            if password = "password" then
                // If the user is "administrator", add the "admin" role
                let claims =
                    match username with
                    | "administrator" -> [Claim(ClaimTypes.Role, "admin")]
                    | _ -> []
                return! ctx.HttpContext.AsyncSignIn(username, claims = claims)
        }

        // Only an admin can call this function
        getSecretData = ctx.AuthorizeWith [AuthorizeAttribute(Roles = "admin")] <| fun () -> async {
            return "Super secret data for admin eyes only!"
        }
    }
```

#### From the client side

You can call an authorized function from the client side with the standard `Cmd.ofAsync`. If the user is not authorized, then the call will return an error with the exception `RemoteUnauthorizedException`.

```fsharp
type Model =
  { secretData : string option
    latestError : exn option }

type Message =
    // Trigger a `getSecretData` request
    | GetSecretData
    // Received response of a `getSecretData` request
    | GotSecretData of data: string
    // A request threw an error or was unauthorized
    | Error of exn

let update myService message model =
    match message with
    | GetSecretData ->
        model,
        Cmd.ofAsync myService.getSecretData () GotSecretData Error
    | GotSecretData data ->
        { model with secretData = Some data }, []
    | Error RemoteUnauthorizedException ->
        // Tried to getSecretData, but the user was not signed in
        { model with secretData = None }, []
    | Error exn ->
        // Another error happened (eg. the server was unavailable)
        { model with latestError = Some exn }, []
```

This way is particularly convenient if you have several remote functions that need to handle authorization errors the same way (eg by showing a login popup), as they can all use the same `Error` message.

Alternatively, you can use the function `Cmd.ofAuthorized`. This function is similar to `Cmd.ofAsync`, except that it handles both success and unauthorized call with the same message by passing a value of type `option<'resp>`. This value is `None` if the user is not authorized, or `Some` with the returned value if the user is authorized.

This way is more convenient for a remote function that needs to handle authorization errors in a specific way.

```fsharp
type Model =
  { secretData : string option
    latestError : exn option }

type Message =
    // Trigger a `getSecretData` request
    | GetSecretData
    // Received response of a `getSecretData` request
    | GotSecretData of option<string>
    // A request threw an error
    | Error of exn

let update myService message model =
    match message with
    | GetSecretData ->
        model,
        Cmd.ofRemote myService.getSecretData () GotSecretData Error
    | GotSecretData (Some data) ->
        { model with secretData = Some data }, []
    | GotSecretData None ->
        // Tried to getSecretData, but the user was not signed in
        { model with secretData = None }, []
    | Error exn ->
        // Another error happened (eg. the server was unavailable)
        { model with latestError = exn }, []
```

The variant `Cmd.performRemote` doesn't take an `Error` message and ignores non-authorization-related errors.
