---
title: Writing HTML
---

# Writing HTML

Bolero provides F# functions to create HTML elements, attributes and event handlers, and also instantiate Blazor components. All of these are defined in the module `Bolero.Html`.

See also how to create HTML elements using [HTML templates](Templating).

## Elements

To create an HTML element, just call the function with its name. It takes two arguments: a list of attributes and a list of child elements, and returns a value of type `Node`.

Additionally, the function `text` creates a text node, and `textf` creates a text node using `printf`-style formatting.

```fsharp
let myElement name =
    div [] [
        h1 [] [text "My app"]
        p [] [textf "Hello %s and welcome to my app!" name]
    ]
```

Elements that can't have children, such as `input` or `br`, only take attributes as argument.

```fsharp
let myElement =
    p [] [
        text "First line of the paragraph."
        br []
        text "Second line of the paragraph."
    ]
```

To create a custom element for which there isn't a function, use `elt`.

```fsharp
let myElement =
    elt "data-paragraph" [] [
        text "This is in a <data-paragraph> element."
    ]
```

In addition to representing an HTML node, the type `Node` can also represent a (possibly empty) sequence of nodes. This is done using the `concat` function.

```fsharp
let myElements =
    concat [
        p [text "First paragraph"]
        p [text "Second paragraph"]
    ]
```

`empty` represents an empty sequence of nodes: it is equivalent to `concat []`. This doesn't seem very useful at first, but it is actually important for conditional elements.

### Conditional elements

Due to the way that Blazor compares the rendered DOM when a change is applied, the returned HTML must always have the same structure: conditional elements can't be simply added. For example, the following may cause runtime errors:

```fsharp
// May fail at runtime.
let myButton (label: option<string>) =
    button [] [
        if label.IsSome then
            yield text label.Value
    ]
```

Rendering such conditional content must be done with the `cond` function instead.

* `cond` can take a boolean value, and a function to call on this value returning a Node. For example, the following is correct:

    ```fsharp
    let myButton (label: option<string>) =
        button [] [
            cond label.IsSome <| function
                | true -> text label.Value
                | false -> empty
        ]
    ```
    
    You can also see here why `empty` is a useful value.

* `cond` can also take a value whose type is an F# union, and a function that matches over the cases of this union. For example, `option<'T>` is an F# union, so the following is correct:

    ```fsharp
    let myButton (label: option<string>) =
        button [] [
            cond label <| function
                | Some l -> text l
                | None -> empty
        ]
    ```

    Here's an example with a union defined in your code:
    
    ```fsharp
    /// A list of usernames, truncated to two + number of others
    type UserList =
        | One of string
        | Two of string * string
        | Many of string * string * int
    
    /// Shows one of the following, depending on the number of users:
    /// * "*Alice* likes this"
    /// * "*Alice* and *Bob* like this"
    /// * "*Alice*, *Bob* and 12 others like this"
    let showLikes (users: UserList) =
        concat [
            cond users <| function
                | One uname -> b [] [text uname]
                | Two (uname1, uname2) ->
                    concat [
                        b [] [text uname1]
                        text " and "
                        b [] [text uname2]
                    ]
                | Many (uname1, uname2, others) ->
                    concat [
                        b [] [text uname1]
                        text ", "
                        b [] [text uname2]
                        textf " and %i others" others
                    ]
            cond users <| function
                | One _ -> text " likes this."
                | _     -> text " like this."
        ]
    ```

### Collection elements

Similarly, rendering collections using a function such as `List.map` to create a list of nodes can cause runtime errors. Instead, collections of items should be rendered using the function `forEach`.

```fsharp
let listUsers (names: string list) =
    p [] [
        text "Here are the users:"
        ul [] [
            forEach names <| fun name ->
                li [] [text name]
        ]
    ]
```

## Attributes

Attributes are available in the `attr` submodule.

```fsharp
let myElement =
    p [
        attr.style "color: blue;"
        attr.``class`` "paragraph"
    ] [
        text "Hello and welcome to my app!"
    ]
```

To create a custom attribute for which there isn't a function, use the `=>` operator.

```fsharp
let myElement =
    p ["data-kind" => "paragraph"] [
        text "Hello and welcome to my app!"
    ]
```

### Conditional attributes

Like with elements (see [Conditional elements](#conditional-elements)), naively adding conditional attributes can lead to runtime errors.

```fsharp
// May fail at runtime.
let myElement (isBlue: bool) =
    p [
        if isBlue then
            yield attr.style "color: blue;"
    ] [
        text "Hello and welcome to my app!"
    ]
```

Instead if an attribute may or may not need to be added depending on a condition, always add the attribute and give it a value of `false` or `null` when it should be omitted.

```fsharp
let myElement (isBlue: bool) =
    p [attr.style (if isBlue then "color: blue;" else null)] [
        text "Hello and welcome to my app!"
    ]
```

## Event handlers

Event handlers are available in the `on` submodule.

```fsharp
let myElement =
    button [on.click (fun _ -> printfn "Clicked!")] [
        text "Click me!"
    ]
```

The argument passed to the callback has type `UIEventArgs` from Blazor. Specific events have corresponding subtypes of `UIEventArgs`: for example, `on.click` uses `UIMouseEventArgs`.

```fsharp
let myElement =
    button [
       on.click (fun e ->
            printfn "Clicked at (%i, %i)" e.ClientX e.ClientY)
    ] [
        text "Click me!"
    ]
```

To create a custom event handler for which there isn't a function, use `on.event`.

```fsharp
let myElement =
    button [
        on.event "customevent" (fun _ -> printfn "Custom event!")
    ] [
        text "Click me!"
    ]
```

## Data bindings

Attributes defined in the `bind` module define two-way binding with the element's value. These functions take two arguments:

1. The current value, which generally comes from the [Elmish](Elmish) model.
2. A setter function, which generally calls the Elmish dispatch function.

Here is an example using `bind.input`:

```fsharp
type Model = { username: string }

type Message =
    | SetUsername of string

let hello model dispatch =
    input [
        bind.input model.username (fun n -> dispatch (SetUsername n))
    ]
```

The functions in the `bind` module are:

* `bind.input` binds to the `value` property of an element, as a string, by listening to the `oninput` event. This means that the callback is called on every user interaction on the element that changes its value. For example, on a text input, it is triggered on every keystroke.

    It is suitable for `input` and `textarea` elements.

* `bind.change` is identical to `bind.input` except that it listens to the `onchange` event. This means that the callback is called when a change is "committed" by the user. For example, on a text input, it is triggered when the user presses Enter or unfocuses the element after changing the value.

    It is suitable for `input`, `textarea` and `select` elements.

* `bind.inputInt` and `bind.changeInt` bind to the `value` property of an element as an `int`, by listening to the corresponding event. They are particularly suitable with an input that has ```attr.``type`` "number"```. If the value cannot be parsed as an `int` during an event, the setter is not called.

* `bind.inputFloat` and `bind.changeFloat` bind to the `value` property of an element as a `float`, by listening to the corresponding event. They are particularly suitable with an input that has ```attr.``type`` "number"```. If the value cannot be parsed as a `float` during an event, the setter is not called.

* `bind.checked` binds to the `checked` property of a checkbox input. Note that you also need to add ```attr.``type`` "checkbox"``` to the input.

> Note: Radio buttons (```attr.``type`` "radio"```) are not yet supported by Blazor, and therefore by Bolero; see [the issue on Blazor's tracker](https://github.com/aspnet/Blazor/issues/1322).

```fsharp
type Model = { isChecked: bool }

type Message =
    | SetChecked of bool

let hello model dispatch =
    input [
        attr.``type`` "checkbox"
        bind.checked model.isChecked (fun c -> dispatch (SetChecked c))
    ]
```

## Components

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

To instantiate a Blazor component, use the `comp` function. It is parameterized by the component type, and takes attributes and child nodes as arguments.

```fsharp
let myElement =
    comp<MyComponent> ["Who" => "world"] []
```

## NavLink

The function `navLink` is a helper to create a Blazor `NavLink` component. This component creates an `<a>` tag which dynamically receives the `"active"` CSS class whenever the current page URL matches its own `href`. The match is customized by passing `NavLinkMatch.All` (to only match the full URL path) or `NavLinkMatch.Prefix` (to match any URL that starts with the `navLink`'s `href`).

```fsharp
let myMenu =
    ul [] [
        li [] [navLink NavLinkMatch.All [attr.href "/"] [text "Home"]]
        li [] [navLink NavLinkMatch.Prefix [attr.href "/blog"] [text "Blog"]]
    ]
```
