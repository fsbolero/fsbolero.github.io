---
title: Using Blazor features
subtitle: ""
---

This page documents some useful features of Blazor and how to best use them from a Bolero application.

### Dependency injection

Blazor provides the ability to work with the dependency injection mechanism introduced by ASP.NET Core. [Learn more about it on the official documentation](https://docs.microsoft.com/en-us/aspnet/core/blazor/dependency-injection).

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

Dependencies are injected in the client-side host builder:

```fsharp
module Program =

    [<EntryPoint>]
    let Main args =
        let builder = WebAssemblyHostBuilder.CreateDefault(args)

        builder.Services.AddSingleton<IMyDependency, MyDependencyImpl>() |> ignore

        builder.RootComponents.Add<MyApp>("#main")
        builder.Build().RunAsync() |> ignore
        0
```

### JavaScript interoperability

It is possible to use Blazor's JavaScript interoperability interface `IJSRuntime` in Bolero using dependency injection. [Learn more about it on the official documentation](https://docs.microsoft.com/en-us/aspnet/core/blazor/call-javascript-from-dotnet).

```fsharp
open Microsoft.JSInterop

type MyComponent() =
    inherit ElmishComponent<Model, Message>()

    [<Inject>]
    member val JSRuntime = Unchecked.defaultof<IJSRuntime> with get, set

    override this.View model dispatch =
        button [
            on.task.click (fun _ ->
                this.JSRuntime.InvokeVoidAsync("console.log", model).AsTask())
        ] [
            text model
        ]
```

#### In `ProgramComponent`

It is already injected in `ProgramComponent`, so you can use it directly without injecting it beforehand.

```fsharp
open Microsoft.JSInterop

type MyApp() =
    inherit ProgramComponent<Model, Message>()

    override this.Program =
        Program.mkSimple init update view
        |> Program.withTrace (fun msg model ->
            this.JSRuntime.InvokeVoidAsync("console.log", msg, model) |> ignore)
```

#### In Elmish update commands

It is common to need JavaScript interoperation in the `update` function to call external functionality. The `IJSRuntime` can be passed to it from the `ProgramComponent`. Inside `update` the command `Cmd.ofTask` is particularly suitable to perform a JavaScript call and transform its return value into a message.

```fsharp
open Microsoft.JSInterop

type Message =
    | CallMyJSFunc of MyJSFuncArgType
    | CalledMyJSFunc of MyJSFuncReturnType
    | Error of exn

let update (js: IJSRuntime) message model =
    match message with
    | CallMyJSFunc data ->
        let cmd =
            Cmd.ofTask
                (fun args -> js.InvokeAsync("MyJsLib.myJSFunc", args).AsTask())
                [| data |] CalledMyJSFunc Error
        model, cmd
    // ...

type MyApp() =
    inherit ProgramComponent<Model, Message>()
    
    override this.Program =
        let update = update this.JSRuntime
        Program.mkProgram init update view
```

#### HTML element references

[Blazor's type `ElementReference`](https://docs.microsoft.com/en-us/aspnet/core/blazor/call-javascript-from-dotnet?view=aspnetcore-3.1#capture-references-to-elements) allows passing a reference to a rendered HTML element to JavaScript.
This is useful for interacting with JavaScript libraries that insert themselves in a given element, creating for example a map or a rich text editor; or libraries that interact with more fundamental JavaScript APIs, like focusing an element.

In Bolero, the type `ElementReferenceBinder` is a small utility that makes working with `ElementReference` from F# simple.

1. The element to bind must be inside a Component class.
2. Create an `ElementReferenceBinder` as a field of this class.
3. To bind it to an element, pass it to the attribute `attr.ref` of this element.
4. To use it, use its `.Ref` property.

For example, given this small JavaScript function that can focus a DOM element it receives as argument:

```javascript
const MyJsLib = {
    focus: function(elt) {
        elt.focus();
    }
}
```

This function can be called as follows from a Bolero component:

```fsharp
type MyInputWithFocusButton() = // (1)
    inherit ElmishComponent<string, string>()

    let inputRef = ElementReferenceBinder() // (2)

    [<Inject>]
    member val JSRuntime = Unchecked.defaultof<IJSRuntime> with get, set

    override this.View model dispatch =
        concat [
            input [
                attr.ref inputRef // (3)
                bind.input.string model dispatch
            ]
            button [
                on.task.click (fun _ ->
                    this.JSRuntime.InvokeVoidAsync("MyJsLib.focus", inputRef.Ref) // (4)
                )
            ] [
                text "Focus this input box"
            ]
        ]
```
