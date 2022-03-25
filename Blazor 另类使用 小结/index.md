- id: fadfef0a-0957-40bf-86ca-2ba93a919e79
- title: Blazor 另类使用 小结
- keywords: blazor,.NET
- description: 最近一段时间都在折腾 Blazor 的另类使用 Fun.Blazor，现在 2.0 beta 快差不多了，也是时候做一个小结了。
- createTime: 2022-03-25
---

当初为什么要折腾 Fun.Blazor 呢？直接用 C# 写，按照微软官方指导不香吗？当 C# 很多应用场景都支持开发时 hot-reload 的时候，而 F# 官方一直又没有什么说法，我是纠结和后悔过的。自己挖的坑，自己还是得填啊，后来自己也倒腾了一个简单的 hot-reload 之后，再回头来看，其实也没那么糟糕，至少很多事情最终按照我理想中的样子成型了。


## 我理想中的界面 DSL

- UI 控件的使用要简练，不冗余，没有任何模板语法，只有函数和数据
- 能做独立组件，以隔离自己的状态
- 能有显示的依赖注入
- 数据不变，UI不变
- 能有一个适应性的全局状态共享中心
- 能抽象业务逻辑，以重用或减少代码复杂度

```fsharp
// 在 F# 里变量名函数名等都可以用中文的，用 `` `` 引起来就好了
type ``模拟数据`` = { Length: int }

type IShareStore with

    member store.``共享数据`` =
        store.CreateCVal(nameof store.``共享数据``, LoadingState<``模拟数据``>.NotStartYet)

type IComponentHook with

    member hook.``加载数据``() =
        let store, http = hook.ServiceProvider.GetMultipleServices<IShareStore * MyHttpClient>()
        
        store.``共享数据``.Publish LoadingState.start
        
        http.Get<``模拟数据``>("/api/data/demo")
        |> Task.map (LoadingState.ofResult >> store.``共享数据``.Publish)
        |> ignore


let ``演示页面`` =
    html.inject (fun (store: IShareStore, hook: IComponentHook) ->
        let ``本地数据`` = ChangeableValue 0

        hook.OnFirstAfterRender.Add hook.``加载数据``

        let ``数据视图`` =
            adaptiview () { // 适应性的，只有当 data 内容变更才会重新渲染
                match! store.``共享数据`` with
                | LoadingState.NotStartYet ->
                    button {
                        onclick (fun _ -> hook.``加载数据``())
                        "加载数据"
                    }
                | LoadingState.Loading -> 
                    p { "数据加载中。。。" }
                | LoadingState.Loaded result -> 
                    p { $"数据总共有 {result.Length} 条" }
                | LoadingState.Reloading result ->
                    p { $"数据总共有 {result.Length} 条" }
                    p { "数据重载中。。。" }
            }

        let ``计数器`` =
            adaptiview () {
                let! count = ``本地数据``
                div { $"计数值 = {count}" }
            }

        let ``计数器按钮`` =
            button {
                onclick (fun _ -> ``本地数据``.Publish (fun x -> x + 1))
                "自增 1"
            }

        div {
            h2 {
                style { color "green" }
                "测试数据集"
            }
            ``数据视图``
            ``计数器``
            ``计数器按钮``
        }
    )


let ``主应用`` =
    div {
        header { "页面头部" }
        main { 
            html.route [
                routeCi "/demo" ``演示页面``
                routeAny (div { "没有找到相应页面！！！" })
            ]
        }
        footer { "页面尾部" }
    }
```


## 开发时 hot-reload

我是基于两年前别人做的一个社区项目改造而成，目前只能说简单可用，但是确实和 C# 的实现还是有一定的差距，就更别说和 React 比了。不过 C# 的实现其实也不怎么样，目前还不太稳定和可靠，随着项目的的增加，速度也明显下降。

对于我来说，hot-reload 主要就是调整界面细节，所以通常如果有大的更改，比如新建文件，重构，业务逻辑修改等，我都会 F5 重启一次。所以还可以（也只能）忍受。


## 最后

Fun.Blazor 版本2基本也就这样了，学了不少东西，也实现了不少东西，现在开始在兼职的项目中使用，虽然开发的工具链（hot-reload）不怎么样，但是总体出来的代码质量好了不少，后期的维护也简单了很多，所以可以在正式版发布后，告以段落。是时候换换口味了！
