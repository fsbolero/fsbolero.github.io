---
title: Writing HTML (Legacy)
subtitle: Create elements, attributes and event handlers in plain F#
---

All of the functions described here are defined in the module `Bolero.Html`.

See also how to create HTML elements using [HTML templates](Templating).

> Note: the syntax for HTML has significantly changed in Bolero 0.20.
> This page describes the pre-0.20 syntax; see [HTML](HTML) for the 0.20 computation expression-based syntax.

### Elements

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
        p [] [text "First paragraph"]
        p [] [text "Second paragraph"]
    ]
```

`empty` represents an empty sequence of nodes: it is equivalent to `concat []`. This doesn't seem very useful at first, but it is actually important for conditional elements.

#### Conditional elements

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

#### Collection elements

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

### Attributes

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

#### Conditional attributes

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

### Event handlers

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

Asynchronous event handlers are also available in the submodules `on.task` and `on.async`. These modules contain functions that are identical to the ones directly in `on`, except that their callbacks return `Task` and `Async<unit>`, respectively.

```fsharp
let myElement (js: IJSRuntime) =
    button [
        on.task.click (fun e ->
            js.InvokeVoidAsync("console.log", e.ClientX).AsTask())
    ] [
        text "Click me!"
    ]
```

### Data bindings

Attributes defined in the `bind` module define two-way binding with the element's value. These functions take two arguments:

1. The current value, which generally comes from the [Elmish](Elmish) model.
2. A setter function, which generally calls the Elmish dispatch function.

Here is an example using `bind.input.string`:

```fsharp
type Model = { username: string }

type Message =
    | SetUsername of string

let hello model dispatch =
    input [
        bind.input.string model.username (fun n -> dispatch (SetUsername n))

        // Equivalent but more concise, using the composition operator:
        bind.input.string model.username (dispatch << SetUsername)
    ]
```

The module `bind` contains the submodules `input` and `change`:

* Functions in `bind.input` bind to the `value` property of an element by listening to the `oninput` event. This means that the callback is called on every user interaction on the element that changes its value. For example, on a text input, it is triggered on every keystroke.

    They are suitable for `input` and `textarea` elements.

* Functions in `bind.change` are identical except that they listen to the `onchange` event. This means that the callback is called when a change is "committed" by the user. For example, on a text input, it is triggered when the user presses Enter or unfocuses the element after changing its value.

    They are suitable for `input`, `textarea` and `select` elements.

These submodules contain functions that bind to a value with the corresponding type:

* `string`
* `int`
* `int64`
* `float`
* `float32`
* `decimal`
* `dateTime`
* `dateTimeOffset`

The number-typed functions are particularly suitable with an input that has ```attr.``type`` "number"```.

Additionally, the module `bind` directly contains a function `checked` which binds to the `checked` property of a checkbox input. Note that you also need to add ```attr.``type`` "checkbox"``` to the input.

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

For radio buttons, you can use `bind.change.*` like so:

```fsharp
type Color = Red | Green | Blue
type Model = { color: Color }

type Message =
    | SetColor of Color

let hello model dispatch =
    forEach [Red; Green; Blue] <| fun color ->
        input [
            attr.``type`` "radio"

            // use the same name for the 3 radio buttons to group them.
            attr.name "select-color"

            // HTML requires each button to have a different string value,
            // but you don't have to use this string in the event handler
            // if you have a better typed value at hand (here, `color`).
            bind.change.string (string color) (fun _ -> dispatch (SetColor color))
        ]
```
