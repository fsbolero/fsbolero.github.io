---
title: Dealing with JSON
subtitle: Use the inbuilt [extensions](https://github.com/Tarmil/FSharp.SystemTextJson) to System.Text.Json for handling F# specific types
---

> Bolero uses [System.Text.Json](https://docs.microsoft.com/en-us/dotnet/standard/serialization/system-text-json-overview) and [FSharp.SystemTextJson](https://github.com/Tarmil/FSharp.SystemTextJson) for JSON handling. You are free to use any other JSON library of your choosing if you so wish.

### Configuration
For *hosted Blazor WebAssembly* & *Blazor server* apps, Bolero is already configured to use `JsonFSharpConverter` for remoting by default.

For *standalone Blazor WebAssembly* app, use either `JsonSerializerOptions` with  `JsonFSharpConverter` or add `JsonFSharpConverterAttribute`.

Learn more about the [Blazor hosting models](https://docs.microsoft.com/en-us/aspnet/core/blazor/hosting-models?view=aspnetcore-3.1).

#### Using options

Add `JsonFSharpConverter` to the converters in `JsonSerializerOptions`, and the format will be applied to all F# types. 
```fsharp
open System.Text.Json
open System.Text.Json.Serialization

let options = JsonSerializerOptions()
options.Converters.Add(JsonFSharpConverter())
JsonSerializer.Serialize({| x = "Bolero" |}, options)
```

#### Using attributes

Add `JsonFSharpConverterAttribute` to the type that needs to be serialized.
```fsharp
open System.Text.Json
open System.Text.Json.Serialization

[<JsonFSharpConverter>]
type Example = { x: string }
JsonSerializer.Serialize({x = "Bolero" })
```

#### Advantages and disadvantages

The options way is generally recommended because it applies the format to all F# types. In addition to your defined types, this also includes:

* Types defined in referenced libraries that you can't modify to add an attribute. This includes standard library types such as `option` and `Result`.
* Anonymous records.

The attribute way cannot handle the above cases.

The advantage of the attribute way is that it allows calling `Serialize` and `Deserialize` without having to pass options every time. This is particularly useful if you are passing your own data to a library that calls these functions itself and doesn't take options.

### Named records

Named record fields are serialized in the order in which they were declared in the type declaration.
```fsharp
type Example = { x: string; y: string }

JsonSerializer.Serialize({ x = "Hello"; y = "world!" })
// --> {"x":"Hello","y":"world!"}
JsonSerializer.Deserialize("""{"x":"Hello","y":"World!"}""")
// --> { x = "Hello"; y = "world!" }
```

### Anonymous records

Anonymous record fields are serialized in alphabetical order.
```fsharp
JsonSerializer.Serialize({| x = "Hello"; y = "world!" |}, options)
// --> {"x":"Hello","y":"world!"}

JsonSerializer.Deserialize<{| x : string ; y : string |}>("""{"x":"Hello","y":"World!"}""", options)
// --> {| x = "Hello"; y = "world!" |}
```

### More info
For examples using standard library types such as unions, tuples, `option`, `Map`, `list` & `Set`, more information is on the [FSharp.SystemTextJson](https://github.com/Tarmil/FSharp.SystemTextJson) project site. 