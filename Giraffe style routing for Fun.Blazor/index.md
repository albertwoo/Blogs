- id: 85a8cfe4-5323-4f00-a254-3f8f65b9fb6a
- title: Giraffe style routing for Fun.Blazor
- keywords: fsharp,blazor,giraffe
- description: Before I experimented Feliz.Router with Fun.Blazor and find some inconvenience so this time I tried to use giraffe style to see how it is going on.
---

Before I like the Feliz.Router very much, because it is very simple to start:

```fsharp
html.route (function
      | [ "router" ]
      | [ _; "router" ] -> html.text "Router"
                  
      | [ "router"; Route.Query [ "name", name; "age", Route.Int age ] ]
      | [ _; "router"; Route.Query [ "name", name; "age", Route.Int age ] ] -> html.text $"name is: {name}, age is: {age}"

      | _ -> html.text "Not my concern."
)
```

It is easily to extract data from url and also very explicity, its concept is very straightforward. But recently I found it is not easy to manage a little bit complex routes, and composing longer segments makes it hard to read. For example I can do

```fsharp
html.route (function
    | [ Route.Ci "document"; Route.Int docId; Route.Ci "comments"; ....  ]
)
```

I use Giraffe on backend a lot and like its style very much. So copy its code and integrate with blazor could make things funny. Now I can write things like:

```fsharp
html.route [
    routeCi "/whatever"
    subRouteCi "/router" [
        routeCi "/document" (html.text "Dcoument page")
        routeCif "/document/%i" (fun x -> html.text $"Document {x}")
        routeCiWithQueries "/documents" (fun queries -> html.text $"Documents with query: {formatQueries queries}")
        routeCifWithQueries "/documents/%s" (fun param queries -> html.text $"Documents(Param: {param}) with query: {formatQueries queries}")
    ]
]]
/// Or
let routes = [
    routeCi "/antdesign"            demoAntDesign
    routeCi "/fluentui"             demoFluentUI
    routeCi "/mudblazor"            demoMudBlazor
    subRouteCi "/router"            [ routeAny Router.Router.router ]
    routeCi "/elmish"               Elmish.Elmish.elmish
    routeCi "/helper-functions"     HelperFunctions.HelperFunctions.helperFunctions
    routeCi "/cli-usage"            CliUsage.CliUsage.cliUsage
    routeAny QuickStart.QuickStart.quickStart
]
```

Extracting the queries is not gonna difficult, I may write some helper functions (like Zaid-Ajaj/Giraffe.QueryReader: HttpHandler for easily working with query string parameters within Giraffe apps. (github.com)) to do that if necessary. Or just use json serializer to handle dictionary directly to bind to some model. Now I just expose a map of the query.
