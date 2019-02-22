---
title: Routing
---

Bolero provides facilities to bind the page's URL to the Elmish model.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Prerequisites](#prerequisites)
- [Inferred router](#inferred-router)
  - [Format](#format)
- [Custom router](#custom-router)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Prerequisites

In order to use routing, you need to make sure that the page's base URL is set correctly. In most cases, you simply need to add the following inside the `<head>` of your page:

```html
<base href="/">
```

If the root of your application is a subpath, then you need to set that subpath as the base, **including the trailing slash**. For example, if you have an application served at `https://someone.github.io/myrepo/`, then you need the following:

```html
<base href="/myrepo/">
```

# Inferred router

The easiest way to create a router is by using an inferred router. In this mode of operation, you create an endpoint type which has a 1-to-1 correspondance with your supported URLs, and store it in the Elmish model.

Here are the steps to set up an inferred router:

1. Create the endpoint type. Typically, it will be an F# union type where each case corresponds to a path prefixed by the case's name and the case's arguments are the consecutive fragments of the path. For example:

    ```fsharp
    type Page =
        | Home                                  // -> /Home
        | BlogArticle of id: int                // -> /BlogEntry/42
        | BlogList of user: string * page: int  // -> /BlogList/tarmil/1
    ```
    
    See [Format](#format) for all the supported types and how to customize the corresponding path.

2. Add a field in the Elmish model that stores the endpoint:

    ```fsharp
    type Model =
        {
            page: Page
            // other fields...
        }
    ```

3. Add an Elmish message that sets the endpoint:

    ```fsharp
    type Message =
        | SetPage of Page
        // other messages...

    let update message model =
        match message with
        | SetPage page -> { model with page = page }
        // other messages...
    ```

4. Create the router with `Router.infer`:

    ```fsharp
    let router = Router.infer SetPage (fun m -> m.page)
    ```

5. Attach the router to the Elmish program:

    ```fsharp
    Program.mkSimple initModel update view
    |> Program.withRouter router
    ```

The router has a few helpful utilities:

* `router.Link` takes an endpoint value and returns the corresponding URL.

    ```fsharp
    a [attr.href (router.Link Home)] [text "Go to Home"]
    ```

* `router.HRef` takes an endpoint and returns an `href` attribute pointing to the corresponding URL.

    ```fsharp
    a [router.HRef Home] [text "Go to Home"]
    ```

## Format

`Router.infer` supports the following types:

* Standard library types:

    * `string`
    * `bool`
    * integer: `int8` (aka `sbyte`), `uint8` (aka `byte`), `int16`, `uint16`, `int32` (aka `int`), `uint32`, `int64`, `uint64`
    * `decimal`
    * float: `single`, `double` (aka `float`)

* F# union types. Each case corresponds to a path prefixed by the case's name. The case's arguments are the consecutive fragments of the path:

    ```fsharp
    type Page =
        | Home                                  // -> /Home
        | BlogArticle of id: int                // -> /BlogEntry/42
        | BlogList of user: string * page: int  // -> /BlogList/tarmil/1
    ```

    To customize the path, you can use the `EndPoint` attribute, with parameters indicated by `{curly_braces}`.

    ```fsharp
    type Page =
        | [<EndPoint "/">]
          Home                                  // -> /
        | [<EndPoint "/article/{id}">]
          BlogArticle of id: int                // -> /article/42
        | [<EndPoint "/list/{user}/{page}">]
          BlogList of user: string * page: int  // -> /list/tarmil/1
    ```
    
    An `{*asterisk}` indicates that this parameter catches the rest of the path. It must be the last parameter in the path, and correspond to a value of type `string`, `list` or `array`.
    
    ```fsharp
    type Page =
        | [<EndPoint "/list/{user}/tagged/{*tags}">]
          ListTagged of user: string * tags: list<string>
          
    Tagged("tarmil", ["bolero"; "webassembly"])
    // -> /list/tarmil/tagged/bolero/webassembly
    ```
    
    Several cases can share a common prefix in their paths, as long as any parameters in this common prefix have the same type.
    
    ```fsharp
    type Page =
        | [<EndPoint "/user/{user}/favorites">]
          Favorites of user: string
        | [<EndPoint "/user/{user}/comments">]
          Comments of user: string
    ```
    
    The path must contain a `{parameter}` for each of its case's values. Alternatively, the path can consist of a single non-parameter fragment; in this case, the values are appended in order as separate fragments.
    
    ```fsharp
    type Page =
        | [<EndPoint "/list">]                  // Equivalent to /list/{user}/{page}
          BlogList of user: string * page: int  // -> /list/tarmil/1
    ```

* Tuples. The values of the tuple are the consecutive fragments of the path.

    ```fsharp
    type Page = int * string    // -> /42/abc
    ```

* F# record types. The fields of the record are the consecutive fragments of the path.

    ```fsharp
    type Page =
        {
            x: int
            y: string
        }
    // -> /42/abc
    ```

* Lists and arrays. The first fragment of the path is the number of items, and the items themselves are the consecutive fragments.

    ```fsharp
    type Page = list<string>   // -> /3/abc/def/ghi
    ```

* Any combination of the above, including recursive types.

    ```fsharp
    type Page =
        | [<EndPoint "/article">]
          Article of ArticleId              // -> /article/123/announcing-bolero
        | [<EndPoint "/list/{*tags}">]
          List of tags: list<string>        // -> /list/bolero/blazor
        | [<EndPoint "/login/{page}">]
          LoginAndRedirectTo of page: Page  // -> /login/list/bolero/blazor
          
    and ArticleId =
        {
            uid: int
            slug: string
        }
    ```

# Custom router

To have more control over the exact shape of your URLs, you can create a custom router like follows.

```fsharp
let customRouter : Router<Page, Model, Message> =
    {
        // getEndPoint : Model -> Page
        getEndPoint = fun m -> m.page
        // setRoute : string -> option<Message>
        setRoute = fun path ->
            match path.Trim('/').Split('/') with
            | [||] -> Some Home
            | [|"article"; id|] -> Some (BlogArticle (int id))
            | [|"list"; user; page|] -> Some (BlogList (user, int page))
            | _ -> None
            |> Option.map SetPage
        // getRoute : Page -> string
        getRoute = function
            | Home -> "/"
            | BlogArticle(id) -> sprintf "/article/%i" id
            | BlogList(user, page) -> sprintf "/list/%s/%i" user page
    }
```

Note: if the URL is changed in such a way that `setRoute` returns `None`, then the model is not updated.

You can also create a router that maps directly to the model without an intermediary endpoint type. However this router type doesn't have some utilities such as `Link` and `HRef`.

```fsharp
let customRouter2 : Router<Model, Message> =
    {
        // setRoute : string -> option<Message>
        setRoute = fun path ->
            match path.Trim('/').Split('/') with
            | [||] -> Some Home
            | [|"article"; id|] -> Some (BlogArticle (int id))
            | [|"list"; user; page|] -> Some (BlogList (user, int page))
            | _ -> None
            |> Option.map SetPage
        // getRoute : Model -> string
        getRoute = fun model ->
            match model.page with
            | Home -> "/"
            | BlogArticle(id) -> sprintf "/article/%i" id
            | BlogList(user, page) -> sprintf "/list/%s/%i" user page
    }
```
