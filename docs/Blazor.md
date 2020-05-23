---
title: Using Blazor features
subtitle: Blazor Components and their interoperability
---

### Components

#### Creating a component

> Note: this section describes how to create and use plain Blazor components. It is recommended to use Elmish components whenever possible; see [Using Elmish](Elmish).

You can create plain Blazor components by inheriting from the `Component` type.

```fsharp
type MyComponent() =
    inherit Component()

    override this.Render() =
        div [] [text "Hello, world!"]
```

To add parameters to the component, use a property with the `Parameter` attribute from namespace `Microsoft.AspNetCore.Blazor`.

```fsharp
type MyComponent() =
    inherit Component()

    [<Parameter>]
    member val Who = "" with get, set

    override this.Render() =
        div [] [text (sprintf "Hello, %s!" this.Who)]
```

#### Using a component

This section documents how to use a Blazor Component, either referenced from a C# Razor project, or created in F# by inheriting from `Component`.

To instantiate a Blazor component, use the `comp` function. It is parameterized by the component type, and takes attributes and child nodes as arguments.
To set a parameter, pass it by name as an attribute using the `=>` operator.

```fsharp
let myElement =
    comp<MyComponent> ["Who" => "world"] []
```

However, some parameter types must be handled specially:

* Parameters of type `EventCallback<T>` can be passed using one of these functions:

    * `attr.callback`, which takes a function `T -> unit`;
    * `attr.async.callback`, which takes a function `T -> Async<unit>`;
    * `attr.task.callback`, which takes a function `T -> Task`.
    
    Here is an example using the library [MatBlazor](https://www.matblazor.com/):
    
    ```fsharp
    open MatBlazor
    
    let myButton model dispatch =
        comp<MatButton>
          [ attr.callback "OnClick" (fun _ -> dispatch ButtonClicked) ]
          [ text "Click me!" ]
    ```
    
    In Blazor, these parameters would be passed as `Action<T>`.

* Parameters of type `RenderFragment` and `RenderFragment<T>` can be passed using the function `attr.fragment` and `attr.fragmentWith`, respectively. The former takes a `Node`, and the latter a function `T -> Node`.

    Here is again an example using MatBlazor:
    
    ```fsharp
    open MatBlazor
    
    type Car =
      { Name: string
        Price: decimal
        Horsepower: int }
        
    type Model =
      { Cars: Car[] }
    
    let myTable model dispatch =
        comp<MatTable>
          [ "Items" => model.Cars

            attr.fragment "MatTableHeader" (
              concat
                [ th [] [ text "Name" ]
                  th [] [ text "Price" ]
                  th [] [ text "Horsepower"]
                ]
            )

            attr.fragmentWith "MatTableRow" (fun (car: Car) ->
              concat
                [ td [] [ text car.Name ]
                  td [] [ textf "%.2f" car.Price ]
                  td [] [ textf "%i" car.Horsepower ]
                ]
            )
          ]
          []
    ```

### NavLink

The function `navLink` is a helper to create a Blazor `NavLink` component. This component creates an `<a>` tag which dynamically receives the `"active"` CSS class whenever the current page URL matches its own `href`. The match is customized by passing `NavLinkMatch.All` (to only match the full URL path) or `NavLinkMatch.Prefix` (to match any URL that starts with the `navLink`'s `href`).

```fsharp
let myMenu =
    ul [] [
        li [] [navLink NavLinkMatch.All [attr.href "/"] [text "Home"]]
        li [] [navLink NavLinkMatch.Prefix [attr.href "/blog"] [text "Blog"]]
    ]
```

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

It is common to need JavaScript interoperation in the `update` function to call external functionality. The `IJSRuntime` can be passed to it from the `ProgramComponent`.
Inside `update`, the commands `Cmd.ofJS` and `Cmd.performJS` do a JavaScript call and transform its return value into a message.
`ofJS` also transforms potential exceptions into a message, whereas `performJS` ignores such exceptions.

```fsharp
open Microsoft.JSInterop

type Message =
    | CallMyJSFunc of MyJSFuncArgType
    | CalledMyJSFunc of MyJSFuncReturnType
    | Error of exn

let update (js: IJSRuntime) message model =
    match message with
    | CallMyJSFunc data ->
        let cmd = Cmd.ofJS js "MyJsLib.myJSFunc" [| data |] CalledMyJSFunc Error
        model, cmd
    // ...

type MyApp() =
    inherit ProgramComponent<Model, Message>()
    
    override this.Program =
        let update = update this.JSRuntime
        Program.mkProgram init update view
```

These functions really are simple wrappers around `Cmd.ofTask`/`Cmd.performTask`.
For example, the above call is equivalent to:

```fsharp
let cmd =
    Cmd.ofTask
        (fun args -> js.InvokeAsync("MyJsLib.myJSFunc", args).AsTask())
        [| data |] CalledMyJSFunc Error
```

#### HTML element references

[Blazor's type `ElementReference`](https://docs.microsoft.com/en-us/aspnet/core/blazor/call-javascript-from-dotnet?view=aspnetcore-3.1#capture-references-to-elements) allows passing a reference to a rendered HTML element to JavaScript.
This is useful for interacting with JavaScript libraries that insert themselves in a given element, creating for example a map or a rich text editor; or libraries that interact with more fundamental JavaScript APIs, like focusing an element.

In Bolero, the type `ElementReferenceBinder` is a small utility that makes working with `ElementReference` from F# simple.

1. The element to bind must be inside a Component class.
2. Create an `ElementReferenceBinder` as a field of this class.
3. To bind it to an element, pass it to the attribute `attr.bindRef` of this element.
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
                attr.bindRef inputRef // (3)
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
