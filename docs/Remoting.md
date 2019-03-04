---
title: Remoting
---

# Remoting

Bolero.Remoting provides the ability to easily call server-side functions from the client side.

Remote calls are `POST` requests to a specific URL. Arguments and return values are automatically serialized to JSON.

## Defining the service

A set of server-side functions is defined as a record called a *remote service*. Each function is a field in this record, and must take one argument and return `Async<_>`. If you need to pass several arguments to a server-side function, use a tuple.

The record should implement `IRemoteService` to define the URL for its functions. Each function is served at the path `{service.BasePath}/{fieldName}`.

For example, here is the definition of a service for a simple key-value pair storage:

```fsharp
open Bolero.Remoting

type MyService =
    {
        getEntry : string -> Async<option<string>>  // Served at /myService/getEntry
        setEntry : string * string -> Async<unit>   // Served at /myService/setEntry
        deleteEntry : string -> Async<unit>         // Served at /myService/deleteEntry
    }

    interface IRemoteService with
        member this.BasePath = "/myService"
```

## Calling on the client side

On the client side, you will typically want to call these functions in the `update` of the Elmish app. See [the Elmish documentation](https://elmish.github.io/elmish/basics.html) to learn how to run commands in `update`.

1. In your Blazor startup (`Client/Startup.fs`), add support for remoting:

    ```fsharp
    open Bolero.Remoting

    type Startup() =

        member __.ConfigureServices(services: IServiceCollection) =
            services.AddRemoting()
            |> ignore
    ```

2. Retrieve the client-side service in the `ProgramComponent` by using `this.Remote`:

    ```fsharp
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
    type Model = { latestRetrievedEntry : string * string }

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
            model, []
    ```

## Defining on the server side

On the server side, Bolero.Remoting is registered as a service and added as ASP.NET Core middleware. There are several ways to do so.

### A simple service

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
                .UseBlazor<Client.Startup>()
            |> ignore
    ```

    Note that `UseRemoting` (and any other middleware) must be called *before* `UseBlazor`, because `UseBlazor` unconditionally catches all requests.

### Using dependency injection

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

### Using several services

You can of course define several remote services in the same application. Each of them needs to be registered by a separate call to `AddRemoting` in `ConfigureServices`. A single call to `UseRemoting` is enough in `Configure`.
