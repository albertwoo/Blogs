- id: ec88a7bb-9931-4023-b3ef-9f15408b5ea4
- title: 使用 Blazor 重写了个人博客后的感想
- keywords: blazor,fsharp
- description: 近年来 Blazor 越发火热，虽然不及 react, vue, angular等，但是在 .net 生态，它却是如此特别和重要
---

Blazor 有多种模式运行，比如基于 websocket 的服务端，基于 WASM 的客户端，以及混合以上两种的模式。另外在此基础上，在 .NET 6 里面还可以做原生应用开发，无缝访问原生 api。这些简单，一致，高性能的体验是在其他平台少见的。

我的这个小博客，是基于学习的目的，用 react 开发，由于是fsharp的爱好者，所以是用的 fable，把fsharp的代码编译为js，然后通过parceljs打包，有一个 asp.net core 的后台服务器提供静态文件，以及相关的一些 api。

由于前后端都是fsharp，所以可以有一个共享的项目来放后端接口的模型。总的来说很容易实现很好的工程化，对于大项目来说尤其不错。但是对我这个博客来说就有点过渡设计，并且为了做服务端渲染也需要额外的很多代码。当然可以用 nodejs 来作为前端的宿主，并用现成的 react 生态里的东西来做 SSR。但是我就得发布并维护两个站点。当然还可以前后端都用 nodejs，然而我并不是这方面的爱好者。

作为.net生态的爱好者，我一直关注着Blazor，前不久我写了一个简单的 nuget 包，可以基于 bolero 来让我用fsharp写 Blazor 程序。写了一个大概之后，得拿个东西再练练手啊，所以这次又用我的小博客开刀了。

这次改造和重构之后，可以用“十分清爽”来形容。没什么前后端共享，不用担心 api 的暴露，不用担心服务端渲染，不用担心 js 打包，不用担心过重的状态管理。

唯一让我担心的就是网速。

但是这个问题也得好好想想。使用服务端模式的 Blazor，所有交互操作基本都是基于 websocket，所以网络延迟直接影响应用交互的流畅性。比如一个用例：

点击按钮 >=> 显示一个正在加载的控件 >=> 访问数据库加载数据，一起把数据显示出来并隐藏加载控件

而 “显示一个正在加载的控件”，对于服务端模式的 Blazor 就需要在 websocket 里有一回数据交换，所以至少得两个来回。而对一般的spa，第一个数据的来回是不需要的。



个人觉得，使用场景很重要：

* 对于企业内部应用而言，网速并不是问题，SSR也不需要，可以使用 WASM 模式的 Blazor.
* 对于工具性业务，比如设计类的 app，对性能要求较高，通常也不需要SSR，可以使用 WASM 模式的 Blazor，首次加载的问题完全可以通过购买相应的 CDN 产品来加速。
* 对于一般的企业门户网站，注重 SEO, SSR，首页加载速度，用服务端模式的 Blazor 非常合适。尽管有可能会因为网络导致交互卡顿的问题，但是可以通过就近部署，来降低网络延迟。不是非常完美，但是维护和开发简单，并且 SSR 非常简单。
* 当然还有一种混合模式，也有一定的应用场景，首次加载可以做 SSR，然后下载相应的 WASM 文件，加载成功后再接管客户端的渲染。


总体使用下来就是清爽，简单。

最后，再分享一些基于 Fun.Blazor 的代码片段：

```fsharp
// 可以简单注入需要的服务
let blogDetail (id: int64) = html.inject (fun (db: Slaveoftime.Db.SiteDbContext
                                              ,hook: IComponentHook
                                              ,ctx: IHttpContextAccessor
                                              ,store: IShareStore
                                              ,nav: NavigationManager) ->

    // 由 IShareStore 创建的状态是对用户所有组件共享的
    let entry = store.EntryUrl(nav.Uri)
    // 由 IComponentHook 创建的状态是和组件生命周期挂钩的
    let doc = hook.UseStore DeferredState.Loading
    let relatedDocs = hook.UseStore DeferredState.Loading

    // 如果是直接访问博客详情页面，则作同步调用，以便服务端渲染
    // 否则则是用户在浏览器上的后期操作而产生的访问，这种情况只需要在组件加载后再请求数据并渲染就好了，由下面的
    // hook.OnAfterRender.Add 添加一个异步调用就可以了
    if entry.Current = nav.Uri then
        getBlog db id |> Async.RunSynchronously |> DeferredState.Loaded |> doc.Publish
    
    // 使用 IObservable 来做响应式的状态管理，简单熟悉
    let getRelatedDocs id =
        getRelatedDocs db id 
        |> Observable.ofAsync 
        |> Observable.subscribe (DeferredState.Loaded >> relatedDocs.Publish)
        |> hook.AddDispose

    hook.OnAfterRender.Add (function
        | false // 再次渲染后
        | true -> // 首次渲染后
            match doc.Current.Value with
            | Some (Some doc) ->
                increaseCount db doc // 能进入此处，通常说明不是网络爬虫，可以做一个简单的访问计数
                getRelatedDocs doc.Id  // 加载相关联的博客

            | _ ->
                // 此处就是前面所说的，异步调用
                getBlog db id
                |> Observable.ofAsync
                |> Observable.subscribe (DeferredState.Loaded >> doc.Publish)
                |> hook.AddDispose

                doc.Observable
                |> Observable.add (function
                    | DeferredState.Loaded (Some doc) -> 
                        increaseCount db doc
                        getRelatedDocs doc.Id
                    | _ -> ())
            )
```

用 Fun.Blazor 写界面的代码片段：

```fsharp
// 监控 document （本质上是一个IObservable）, 如果发生变更则重新渲染相应的组件
html.watch (document, fun data ->
    match data.Value with
    | None when data.IsLoadingNow -> mudProgressLinear.create()
    | None ->
        mudAlert.create [
            mudAlert.childContent "Document not found or you do not have permission to access this document"
            mudAlert.severity Severity.Error
            mudAlert.icon Icons.Filled.Error
        ]
    | Some data ->
        let doc = data.Value
        mudForm.create [
            mudForm.childContent [
                mudSwitch<bool>.create [
                    mudSwitch.label "Publish current documments"
                    mudSwitch.checked' doc.IsPublish
                    mudSwitch.color (if doc.IsPublish then Color.Success else Color.Default)
                    mudSwitch.checkedChanged (fun x -> changeDocument (fun doc -> doc.IsPublish <- x))
                ]
                spaceV2
                mudTextField<string>.create [
                    mudTextField.label "Title"
                    mudTextField.fullWidth true
                    mudTextField.value doc.Title
                    mudTextField.valueChanged (fun x -> changeDocument (fun doc -> doc.Title <- x))
                ]
    ...
```

使用上面的写法，优缺点也是很明显的：

容易用熟悉的语法和逻辑控制 DOM 结构，不需要学习 razor 模板语法，都是最基本和简单的语法
目前还不支持 fluent api 所以看起来有点啰嗦。不过有 IDE 的智能提醒，其实还可以
fsharp bolero 目前还不支持 .NET 6 里最新的热更新


怎么说呢，目前感觉很不错，不过还得时间来说话，因为，每过一段时间之后，我都发觉自己以前写的代码很挫。。。