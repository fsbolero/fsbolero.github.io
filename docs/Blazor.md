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
        div { "Hello, world!" }
```

To add parameters to the component, use a property with the `Parameter` attribute from namespace `Microsoft.AspNetCore.Blazor`.

```fsharp
type MyComponent() =
    inherit Component()

    [<Parameter>]
    member val Who = "" with get, set

    override this.Render() =
        div { $"Hello, {this.Who}!" }
```

#### Using a component

This section documents how to use a Blazor Component, either referenced from a C# Razor project, or created in F# by inheriting from `Component`.

To instantiate a Blazor component, use the `comp` computation expression builder. It is parameterized by the component type, and takes attributes and child nodes in its CE body.
To set a parameter, pass it by name as an attribute using the `=>` operator.
If there are no parameters to pass, use `attr.empty()`.

```fsharp
let myElement : Node =
    comp<MyComponent> { "Who" => "world" }

let myElementWithDefaultWho : Node =
    comp<MyComponent> { attr.empty() }
```

When inserting a component without parameters inside a parent element, `{ attr.empty() }` can be omitted:

```fsharp
let myComponentInAnElement : Node =
    div {
        attr.id "greeting"
        comp<MyComponent>
    }
```

Additionally, some parameter types must be handled specially:

* Parameters of type `EventCallback<T>` can be passed using one of these functions:

    * `attr.callback`, which takes a function `T -> unit`;
    * `attr.async.callback`, which takes a function `T -> Async<unit>`;
    * `attr.task.callback`, which takes a function `T -> Task`.
    
    Here is an example using the library [MatBlazor](https://www.matblazor.com/):
    
    ```fsharp
    open MatBlazor
    
    let myButton model dispatch =
        comp<MatButton> {
          attr.callback "OnClick" (fun _ -> dispatch ButtonClicked)
          "Click me!"
        }
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
        comp<MatTable> {
            "Items" => model.Cars

            attr.fragment "MatTableHeader" (
                concat {
                    th { "Name" }
                    th { "Price" }
                    th { "Horsepower" }
                }
            )

            attr.fragmentWith "MatTableRow" (fun (car: Car) ->
                concat {
                    td { car.Name } 
                    td { $"%.2f{car.Price}" }
                    td { $"{car.Horsepower}" }
                }
            )
        }
    ```

#### Cascading
You can use [Cascading Values and Cascading Parameters](https://docs.microsoft.com/en-us/aspnet/core/blazor/components/cascading-values-and-parameters) as well
```fsharp
let view state dispatch = 
   comp<CascadingValue<int>> { "Value" => 42; "Name" => "MeaningOfLife"; body }
```
in this case `body` is the content of your view somewhere down the hierarchy you may have something like this
```fsharp
type MyProgramComponent() =
    inherit ProgramComponent<State, Msg>()

    [<CascadingParameter(Name = "MeaningOfLife")>]
    member val MeaningofLife: int = 0 with get, set
    
    override this.Program =
        Program.mkProgram init update view
        |> Program.runWith this.MeaningOfLife
```
or a blazor component as well
```fsharp
type FooComponent() =
    inherit Component()

    [<CascadingParameter>]
    member val MeaningOfLife = 0 with get, set

    override this.Render() =
        concat {
            h1 { "Foo component" }
            p { "The meaning of life is {this.MeaningOfLife}" }
        }
```
### NavLink

The function `navLink` is a helper to create a Blazor `NavLink` component. This component creates an `<a>` tag which dynamically receives the `"active"` CSS class whenever the current page URL matches its own `href`. The match is customized by passing `NavLinkMatch.All` (to only match the full URL path) or `NavLinkMatch.Prefix` (to match any URL that starts with the `navLink`'s `href`).

```fsharp
let myMenu =
    ul {
        li { navLink NavLinkMatch.All { attr.href "/"; "Home" } }
        li { navLink NavLinkMatch.Prefix { attr.href "/blog"; "Blog" } }
    }
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
        button {
            on.task.click (fun _ ->
                this.JSRuntime.InvokeVoidAsync("console.log", model).AsTask())

            $"{model}"
        }
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
Inside `update`, the commands located in the module `Cmd.OfJS` do a JavaScript call and transform its return value into a message.
Just like standard Elmish commands, `Cmd.OfJS.either` also transforms potential exceptions into a message, whereas `Cmd.OfJS.perform` ignores such exceptions.

```fsharp
open Microsoft.JSInterop

type Message =
    | CallMyJSFunc of MyJSFuncArgType
    | CalledMyJSFunc of MyJSFuncReturnType
    | Error of exn

let update (js: IJSRuntime) message model =
    match message with
    | CallMyJSFunc data ->
        let cmd = Cmd.OfJS.either js "MyJsLib.myJSFunc" [| data |] CalledMyJSFunc Error
        model, cmd
    // ...

type MyApp() =
    inherit ProgramComponent<Model, Message>()
    
    override this.Program =
        let update = update this.JSRuntime
        Program.mkProgram init update view
```

These functions really are simple wrappers around `Cmd.OfTask.either`/`perform`.
For example, the above call is equivalent to:

```fsharp
let cmd =
    Cmd.OfTask.either
        (fun args -> js.InvokeAsync("MyJsLib.myJSFunc", args).AsTask())
        [| data |] CalledMyJSFunc Error
```

#### HTML element references

[Blazor's type `ElementReference`](https://docs.microsoft.com/en-us/aspnet/core/blazor/call-javascript-from-dotnet?view=aspnetcore-3.1#capture-references-to-elements) allows passing a reference to a rendered HTML element to JavaScript.
This is useful for interacting with JavaScript libraries that insert themselves in a given element, creating for example a map or a rich text editor; or libraries that interact with more fundamental JavaScript APIs, like focusing an element.

In Bolero, the type `HtmlRef` is a small utility that makes working with `ElementReference` from F# simple.

1. The element to bind must be inside a Component class.
2. Create an `HtmlRef` as a field of this class.
3. To bind it to an element, list it in the computation expression after attributes and before child nodes.
4. To use it, use its `.Value` property.

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

    let inputRef = HtmlRef() // (2)

    [<Inject>]
    member val JSRuntime = Unchecked.defaultof<IJSRuntime> with get, set

    override this.View model dispatch =
        concat {
            input {
                bind.input.string model dispatch
                inputRef // (3)
            }
            button {
                on.task.click (fun _ ->
                    this.JSRuntime.InvokeVoidAsync("MyJsLib.focus", inputRef.Value).AsTask() // (4)
                )

                "Focus this input box"
            }
        }
```

### Blazor component references

Just like with HTML elements, it is possible to capture a reference to an instantiated Blazor component.
Whereas HTML element references are mostly useful with JavaScript interop, Blazor component references are used directly in F# to eg. call methods on the component itself.

Capturing a Blazor component reference is done exactly the same way as capturing an HTML element reference, except that the reference type is `Ref<Component>` instead of `HtmlRef`, where `Component` is the component type.
For example, given the following component:

```fsharp
type MyComponent() =
    inherit Component()
    
    override this.Render() =
        div { "This is my component" }
        
    member this.Refresh() =
        Console.WriteLine("Refreshing this component!")
```

you can get a reference to an instance of it as follows:

```fsharp
type MyApplication() =
    inherit Component()
    
    let myComponentRef = Ref<MyComponent>()
    
    override this.Render() =
        div {
            comp<MyComponent> { myComponentRef }
            button {
                on.click (fun _ -> myComponentRef.Value.Refresh())

                "Refresh my component!"
            }
        }
```
