- id: 15936233-ccd2-4b40-97b0-cde439360d28
- title: 使用 tailwindcss 为我的 blazor 博客添加黑暗模式
- keywords: blazor,tailwindcss,暗黑模式
- description: 最近几年黑暗模式越来越流行，不管是系统成面还是 UI 设计趋势，感觉不光是程序员喜欢黑色主题，大家都好这口，可能是因为大家都越来越喜欢晚睡了吧
---

## 首先来对比一下最终效果:

![dark](./5fb26e69-1e5f-4752-bdd7-cd11c3a45566)
![light](./44ecf194-9279-42e6-90ee-61696f391c65)

实验还是发生在我这个小博客上，先前用 Blazor 重写过，这次看看加一个暗黑模式会不会太复杂。我下面举的例子使用 FSharp + Blazor 写的，虽然语法稍有不同，但是原理类似。



根据 tailwindcss 官方文档，要实现这个还是很简单的：



## 首先就是要在 tailwind.config.js 文件启用：

```js
module.exports = {
    darkMode: 'class',
    ....
}
```

它有两种模式，一个是 class 另一个是 media。class 模式的意思就是在最上层 DOM 标签的 class 属性里添加 dark 或 light。而 media 就是跟随系统配置。这两种模式的不同，会导致 tailwindcss 生成的 css 有些微不同，所以在最开始选择的时候最好想清楚自己适合哪一种，不然后来可能会要额外改一些 css。我选的是 class 模式，就遇见过这样的问题，随后解释。

## 其次就是要给你的顶层 DOM 元素的 class 属性设置 dark/light

```fsharp
// 创建一个共享变量
let isDarkMode = store.IsDark true

// Blazor 在浏览器上第一次渲染后
hook.OnFirstAfterRender.Add (fun () ->
    task {
        // 读取浏览器 localStorage 是否配置过，如果配置过就读取相应的值并使用
        let! isDarkSetted = localStorage.ContainKeyAsync DarkModeKey
        if isDarkSetted then
            let! isDark = localStorage.GetItemAsync<bool>(DarkModeKey)
            isDarkMode.Publish isDark
    } |> ignore
)

html.watch (isDarkMode, fun isDark ->
    div() {
        // 默认是 light 所以我只需要显式配置 dark 即可
        classes [ if isDark then "dark" ]
        childContent [
            ...
        ]
    }
)
```

tailwindcss 官网示例上是设置在了 html 标签上的，对我而言不太方便，就配在了 blazor 的 root 组件里了。


改变它这个共享变量的地方，就是我们的开关咯，我是放在页面最上方，最明显的地方：

```fsharp
// 引用 isDarkMode 这个共享的变量
let isDarkMode = store.IsDark()
html.watch (isDarkMode, fun isDark ->
    MudIconButton'() {
        Icon (if isDark then Icons.Filled.Brightness3 else Icons.Filled.Brightness5)
        Size Size.Small
        OnClick (fun _ ->
            let newMode = not isDark
            // 将用户的选择保存在浏览器的 localStorage 里面
            localStorage.SetItemAsync(DarkModeKey, newMode) |> ignore
            // 通知 Blazor 当前 session 里共享的 isDarkMode 值的改变，以重新渲染
            isDarkMode.Publish newMode)
    }
)
```

接着就是具体的 css 的使用了


我就用我博客里的文章标题举例：


```fsharp
a() {
    childContent doc.Title
    href $"/blog/{doc.Id}"
    classes [
        Tw.``text-lg``; Tw.``font-semibold``; Tw.``cursor-pointer``; Tw.truncate
        // light 模式的 css            dark 模式的 css
        Tw.``text-gray-darker``; Tw.``dark:text-gray-lighter``
        // 此处就是我上面提到的，如果你不是用的 class 模式，那就不需要最后那个 css
        // 此处应该是由于 class 模式下生成的 css 优先级问题
        // 导致必须要显式地指明 dark 模式下，在鼠标悬停时的 css
        Tw.``hover:text-warning``; Tw.``dark:hover:text-warning``
    ]
}
```

## 最后


理论上这就可以了，用 postcss 再处理一下就可以发布了。总体的 css 的体量当然是增加了很多，在非压缩的情况下，先前大概是 17 kb，加了黑暗模式后变成了 34 kb，翻了倍，不过还是可以接受的。



非理论上来说，就是因为我还使用了 MudBlazor （Material design 库），所以也还得切换一下它的模式，不过就简单了很多，毕竟别人是有 js 背书的：


```fsharp
MudThemeProvider'() {
    Theme (if isDark then darkTheme else lightTheme)
}
```


darkTheme 和 lightTheme 具体的样子我就不共享的，毕竟就是实例化一个对象，配置了几个颜色而已。