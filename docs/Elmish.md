---
title: Using Elmish
subtitle: Leverage the Model-View-Update architecture using the [Elmish](https://elmish.github.io/elmish/) library
---

> Note: this page documents how to integrate Elmish in a Bolero application. It does not aim at explaining the Elmish architecture; you can learn about it [on Elmish's website](https://elmish.github.io/elmish/).

## Program component

The main component for a Bolero application is `ProgramComponent<'model, 'msg>`. This is a Blazor component whose content is defined by an Elmish program.

For example, given the following simple Elmish counter program:

```fsharp
type Model = { value: int }
let initModel = { value = 0 }

type Message = Increment | Decrement
let update message model =
    match message with
    | Increment -> { model with value = model.value + 1 }
    | Decrement -> { model with value = model.value - 1 }

let view model dispatch =
    div {
        button { on.click (fun _ -> dispatch Decrement); "-" }
        string model.value
        button { on.click (fun _ -> dispatch Increment); "+" }
    }

let program =
    Program.mkSimple (fun _ -> initModel) update view
```

You can wrap it in a Blazor component as follows:

```fsharp
open Elmish
open Bolero

type MyApp() =
    inherit ProgramComponent<Model, Message>()

    override this.Program = program
```

## View components

Bolero works by calling the `view` on every `update`, and then diffing the returned node against the previously rendered page. This means that a part of the page that doesn't change after a given update still takes some computation to realize that it hasn't changed.

Take the following example:

```fsharp
type Model = { firstName: string; lastName: string }
let initModel = { firstName = ""; lastName = "" }

type Message = SetFirstName of string | SetLastName of string
let update message model =
    match message with
    | SetFirstName n -> { model with firstName = n }
    | SetLastName n -> { model with lastName = n }

let viewInput model setValue =
    input {
        attr.value model
        on.change (fun e -> setValue (unbox e.Value))
    }

let view model dispatch =
    div {
        viewInput model.firstName (fun n -> dispatch (SetFirstName n))
        viewInput model.lastName (fun n -> dispatch (SetLastName n))
        $"Hello, {model.firstName} {model.lastName}!"
    }
```

This displays two input boxes prompting the user for a first name and a last name. On every change, `view` is called, and therefore `viewInput` is called twice, even though only one field has changed. Of course this is a tiny application so this doesn't cause any noticeable performance issue, but as the app gets bigger, this kind of unnecessary work may become a problem.

This can be improved using `ElmishComponent<'model, 'msg>`. This component type only calls its view if its model has changed. It is typically used by passing a sub-model, rather than the full model; for example, in our case, we'll just pass the relevant field: either `model.firstName` or `model.lastName`.

```fsharp
type Input() =
    // The first `string` is the model type.
    // The second `string` is the message type.
    inherit ElmishComponent<string, string>()

    override this.View model dispatch =
        input {
            attr.value model
            on.change (fun e -> dispatch (unbox e.Value))
        }
```

Instantiating an `ElmishComponent` is done using the function `ecomp`.
This function is parameterized by the component type, and takes two arguments: a model and a dispatch function.
It returns a computation expression builder to pass parameters to the component (see [Components](Blazor#using-a-component)).
If there are no parameters to pass, use `attr.empty()`.

```fsharp
let view model dispatch =
    div {
        ecomp<Input,_,_> model.firstName (fun n -> dispatch (SetFirstName n)) { attr.empty() }
        ecomp<Input,_,_> model.lastName (fun n -> dispatch (SetLastName n)) { attr.empty() }
        $"Hello, {model.firstName} {model.lastName}!"
    }
```

### Customizing the model update check

Let's improve our `Input` component by adding a label. We can change the model to a record that contains both the label and the value.

```fsharp
type InputModel = { label: string; value: string }

type Input() =
    inherit ElmishComponent<InputModel, string>()
```

Since we know that the label of a given component will always be the same, we'd like `Input` to only look at the value when checking whether the model has changed. By default, `ElmishComponent` checks whether the model has changed by calling `obj.ReferenceEquals` on the old and new models. To specify a different check, we can override `ShouldRender(oldModel, newModel)`.

```fsharp
type InputModel = { label: string; value: string }

type Input() =
    inherit ElmishComponent<InputModel, string>()

    // Check for model changes by only looking at the value.
    override this.ShouldRender(oldModel, newModel) =
        oldModel.value <> newModel.value

    override this.View model dispatch =
        label {
            model.label
            input {
                attr.value model.value
                on.change (fun e -> dispatch (unbox e.Value))
            }
        }

let view model dispatch =
    div {
        ecomp<Input,_,_>
            { label = "First name: "; value = model.firstName }
            (fun n -> dispatch (SetFirstName n))
            { attr.empty() }
        ecomp<Input,_,_>
            { label = "Last name: "; value = model.lastName }
            (fun n -> dispatch (SetLastName n))
            { attr.empty() }
        text (sprintf "Hello, %s %s!" model.firstName model.lastName)
    }
```
