- id: 2109cd23-f528-4c85-b4cc-dd7092c1df4b
- title: Improve computation expression style DSL for Fun.Blazor
- keywords: computation_expression,fsharp,dsl
- description: Still trying to mess around with the frontend DSL powered by fsharp computation expression
- createTime: 2021-07-18
---

Before in the last post I said there are three limitations when use computation expression for UI DSL:

1. The intelliense is not very good
2. Fsharp can not always infer the types, sometimes you need to call CAST manually
3. For CE (computation expression) style, F# do not support override CustomOperation


Now the 2 and 3 are solved.



## For "2. Fsharp can not always infer the types, sometimes you need to call CAST manually"


I solved by add below code to the base class:

```fsharp
type FunBlazorContext<'Component when 'Component :> Microsoft.AspNetCore.Components.IComponent> () =
    ...
    member this.Run _ = this :> IFunBlazorNode
    ...
```

So every CE builder inherit from FunBlazorContext can be automatically cast to IFunBlazorNode. In this way we do not need compiler to infer and auto cast its type. So no matter where you compose your UI controls, you do not need below CAST tail call:

```fsharp
MudText'() {
    Color Color.Primary
    Typo Typo.h5
    childContent "Have fun ✌"
    CAST
}
```


## For "3. For CE (computation expression) style, F# do not support override CustomOperation"


In fsharp 5 preview I noticed that we have enabled CustomOperation override.

Now the duplicate of generated DslCE is reduced a lot. Take MudBlazor as example, before it will generate 3934 lines of code and now it reduced to 2332 (Some refactor also helps this reducing). Removed the redundant code makes me feel much better now.



## Last but not least


I changed some conventions.

Before I use lowercase for the DslCE, for example mudText will be the CE builder for the MudText. All the properties are transformed by lowerFirstCase too. There are two problems with this:

1. When you copy their demo code (for example from the MubBlazor docs website), you need to type more code to transform
2. Some controls can inherit from base dom, because it can accept all the base dom properties like onclick, value, href etc. Then the property override may not behave as you may expected. For example:
In the base class we have a CustomOperation "value", it accept obj as argument

```fsharp
[<CustomOperation("value")>] member this.value (_: FunBlazorContext<'Component>, v: obj) = 
    "value" => v |> BoleroAttr |> this.AddProp
```

In MudSelectItemBuilder, we also have a CustomOperation "value" (if we lowered the first case), but it accept a generic argument then the compiler may not select this property instead it will select the property from the base class. In base class the "value" name is lowercase because blazor`s basic dom define it in that way, but in MudBlazor it accept uppercase as the parameter. And they are totally two different things. A little confusing right? Me too, because when I write the cli, I made that mistake.

```fsharp
[<CustomOperation("value")>] member this.value (_: FunBlazorContext<'FunBlazorGeneric>, x: 'T) =
    "Value" => x |> BoleroAttr |> this.AddProp
```

Anyway, in the last I thought there is no need to change the case of perperty or the class name for CE style DSL. Only transform one property "ChildContent", because sometimes a control inherit from the base class which has "childContent" but sometimes it does not need to inheirt it and its default name will be "ChildContent" if I do not transform it but they are the same thing. So I think change them all to "childContent" is more consistency.



So now the generated CE style code can be consumed like this:

```fsharp
	MudDrawer'() {
	    Open isOpen
	    Elevation 25
	    Variant DrawerVariant.Persistent
	    childContent [
	        MudDrawerHeader'() {
	            LinkToIndex true
	            childContent [
	                MudText'() {
	                    Color Color.Primary
	                    Typo Typo.h5
	                    childContent "Have fun ✌"
	                }
	            ]
	        }
	        navmenu
	    ]
	}
```

Before it looks like this:

```fsharp
    mudDrawer() {
        open' isOpen
        elevation 25
        variant DrawerVariant.Persistent
        childContent [
            mudDrawerHeader() {
                linkToIndex true
                childContent [
                    mudText() {
                        color Color.Primary
                        typo Typo.h5
                        childContentStr "Have fun ✌"
                    }
                ]
            }
            navmenu
        ]
        CAST
    }
```

For more information please check the repo.
