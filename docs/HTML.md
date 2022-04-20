---
title: Writing HTML
subtitle: Create elements, attributes and event handlers in plain F#
---

All of the functions described here are defined in the module `Bolero.Html`.

See also how to create HTML elements using [HTML templates](Templating).

> Note: the syntax for HTML has significantly changed in Bolero 0.20.
> See [HTML list functions](HTML-list-functions) for the pre-0.20 list-based syntax.

## Elements

HTML elements are created using [Computation Expressions](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions).
The CE builder is the name of the element, and the body of the CE lists child content.

```fsharp
let myElement (name: string) : Node =
    div {
        h1 { "My app" }
        p { $"Hello {name} and welcome to my app!" }
    }
```

To create a custom element for which there isn't a builder, use `elt`.

```fsharp
let myElement : Node =
    elt "data-paragraph" {
        "This is in a <data-paragraph> element."
    }
```

In addition to representing an HTML node, the type `Node` can also represent a (possibly empty) sequence of nodes. This is done using the `concat` builder.

```fsharp
let myElements : Node =
    concat {
        p { "First paragraph" }
        p { "Second paragraph" }
    }
```

The function `text` takes a string and represents a text node without having to wrap it in an HTML element.

```fsharp
let myTextNode (name: string) : Node =
    text $"Hello {name}!"
```

The function `empty()` represents an empty sequence of nodes.
This doesn't seem very useful at first, but it is actually important for conditional elements.

### Conditional elements

Due to the way that Blazor compares the rendered DOM when a change is applied, the returned HTML must always have the same structure: conditional elements can't be simply added. For example, the following may cause runtime errors:

```fsharp
// May fail at runtime.
let myButton (label: option<string>) : Node =
    button {
        if label.IsSome then
            label.Value
    }
```

Rendering such conditional content must be done with the `cond` function instead.

* `cond` can take a boolean value, and a function to call on this value returning a Node. For example, the following is correct:

    ```fsharp
    let myButton (label: option<string>) : Node =
        button {
            cond label.IsSome <| function
                | true -> text label.Value
                | false -> empty()
        }
    ```
    
    You can also see here why `empty` is a useful function.

* `cond` can also take a value whose type is an F# union, and a function that matches over the cases of this union. For example, `option<'T>` is an F# union, so the following is correct:

    ```fsharp
    let myButton (label: option<string>) : Node =
        button {
            cond label <| function
                | Some l -> text l
                | None -> empty()
        }
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
    let showLikes (users: UserList) : Node =
        concat {
            cond users <| function
                | One uname -> b { uname }
                | Two (uname1, uname2) ->
                    concat {
                        b { uname1 }
                        " and "
                        b { uname2 }
                    }
                | Many (uname1, uname2, others) ->
                    concat {
                        b { uname1 }
                        ", "
                        b { uname2 }
                        $" and {others} others"
                    }
            cond users <| function
                | One _ -> text " likes this."
                | _     -> text " like this."
        }
    ```

### Collection elements

Similarly, rendering collections using a function such as `List.map` to create a list of nodes can cause runtime errors. Instead, collections of items should be rendered using the function `forEach`.

```fsharp
let listUsers (names: string list) : Node =
    p {
        "Here are the users:"
        ul {
            forEach names <| fun name ->
                li { name }
        }
    }
```

Alternatively, inside HTML computation expressions, the `for` keyword renders collections safely.

```fsharp
let listUsers (names: string list) : Node =
    p {
        "Here are the users:"
        ul {
            for name in names do
                li { name }
        }
    }
```

## Attributes

Attributes are available in the `attr` submodule.
They are added to an element by listing them inside the computation expression, *before* child nodes.

```fsharp
let myElement : Node =
    p {
        attr.style "color: blue;"
        attr.``class`` "paragraph"

        "Hello and welcome to my app!"
    }
```

To create a custom attribute for which there isn't a function, use the `=>` operator.

```fsharp
let myElement : Node =
    p {
        "data-kind" => "paragraph"

        "Hello and welcome to my app!"
    }
```

### Conditional attributes

Like with elements (see [Conditional elements](#conditional-elements)), naively adding conditional attributes can lead to runtime errors.

```fsharp
// May fail at runtime.
let myElement (isBlue: bool) : Node =
    p {
        if isBlue then
            attr.style "color: blue;"

        "Hello and welcome to my app!"
    }
```

Instead, if an attribute may or may not need to be added depending on a condition, always add the attribute and give it a value of `false` or `null` when it should be omitted.

```fsharp
let myElement (isBlue: bool) : Node =
    p {
        attr.style (if isBlue then "color: blue;" else null)

        "Hello and welcome to my app!"
    }
```

## Event handlers

Event handlers are available in the `on` submodule.

```fsharp
let myElement : Node =
    button {
        on.click (fun _ -> printfn "Clicked!")

        "Click me!"
    }
```

The argument passed to the callback has type `UIEventArgs` from Blazor. Specific events have corresponding subtypes of `UIEventArgs`: for example, `on.click` uses `UIMouseEventArgs`.

```fsharp
let myElement : Node =
    button {
       on.click (fun e ->
            printfn $"Clicked at ({e.ClientX}, {e.ClientY})")

        "Click me!"
    }
```

To create a custom event handler for which there isn't a function, use `on.event`.

```fsharp
let myElement : Node =
    button {
        on.event "customevent" (fun _ -> printfn "Custom event!")

        "Click me!"
    }
```

Asynchronous event handlers are also available in the submodules `on.task` and `on.async`. These modules contain functions that are identical to the ones directly in `on`, except that their callbacks return `Task` and `Async<unit>`, respectively.

```fsharp
let myElement (js: IJSRuntime) : Node =
    button {
        on.task.click (fun e ->
            js.InvokeVoidAsync("console.log", e.ClientX).AsTask())

        "Click me!"
    }
```

## Data bindings

Attributes defined in the `bind` module define two-way binding with the element's value. These functions take two arguments:

1. The current value, which generally comes from the [Elmish](Elmish) model.
2. A setter function, which generally calls the Elmish dispatch function.

Here is an example using `bind.input.string`:

```fsharp
type Model = { username: string }

type Message =
    | SetUsername of string

let hello (model: Model) (dispatch: Dispatch<Message>) : Node =
    input {
        bind.input.string model.username (fun n -> dispatch (SetUsername n))

        // Equivalent but more concise, using the composition operator:
        bind.input.string model.username (dispatch << SetUsername)
    }
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

let hello (model: Model) (dispatch: Dispatch<Message>) : Node =
    input {
        attr.``type`` "checkbox"
        bind.checked model.isChecked (fun c -> dispatch (SetChecked c))
    }
```

For radio buttons, you can use `bind.change.*` like so:

```fsharp
type Color = Red | Green | Blue
type Model = { color: Color }

type Message =
    | SetColor of Color
    
let hello (model: Model) (dispatch: Dispatch<Message>) : Node =
    forEach [Red; Green; Blue] <| fun color ->
        input {
            attr.``type`` "radio"

            // use the same name for the 3 radio buttons to group them.
            attr.name "select-color"

            // HTML requires each button to have a different string value,
            // but you don't have to use this string in the event handler
            // if you have a better typed value at hand (here, `color`).
            bind.change.string (string color) (fun _ -> dispatch (SetColor color))
        }
```
