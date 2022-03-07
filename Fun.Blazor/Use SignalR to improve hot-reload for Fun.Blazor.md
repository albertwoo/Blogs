- id: 7853850c-31f8-4162-b9ff-f610e45511e6
- title: Use SignalR to improve hot-reload for Fun.Blazor
- keywords: fsharp,blazor,SignalR
- description: SignalR client can support blazor wasm, so if use that we may have hot-reload for Fun.Blazor in wasm mode.
- createTime: 2022-03-07
---

Before I implemented limited hot-reload for Fun.Blazor. Some limitations are:

1. Not all fsharp expression are supported
2. Need explicitly specify which file you want to hot-reload
3. Only support blazor server mode

Now I am trying to improve the 3rd limitation.

The idea is like this:

- Create a shared library **Fun.Blazor.HotReload** which can support server and wasm mode
- Create **HotReloadComponent**

    After first render of this component, we can create a SignalR connection with **Microsoft.AspNetCore.SignalR.Client** to receive code changes.

    We can use it in multiple places, for example, if we have multiple projects for our application, then we can setup multiple entries for hot-reload which can improve hot-reload performance too. But of course it needs some extra work.

    The setup is very simple, you just pick an entry and call **html.hotReloadComp**, **FullNamespace.comp** can be located in any places, even in other projects. The next step is just use cli to watch that project.

    ```fsharp
    #if DEBUG
        html.hotReloadComp (FullNamespace.comp, "FullNamespace.comp")
    #else
        FullNamespace.comp // must defined in a module and without parameters
    #endif
    ```

- **Fun.Blazor.Cli**

    This project will become the server project which will host a SignalR hub. It watch a specific project changes and send changes to all the SignalR client connections.

    ```bash
    fun-blazor watch "xxxxx.fsproj"
    ```

## It works for more modes, but for wasm is very slow

This works for more modes, for blazor server of course. For hybrid mode, like blazor + WPF, it works the same.

But for blazor **wasm**, you cannot say it does not work. The only problem is that the **json serialization** is very slow compared to other modes. So after it received the code changes, sometimes it spend 10 seconds to parse the data, other steps are very quick, like SignalR data receiving, code evaluation etc.

Before I thought, blazor wasm only have performance issue for startup, and not expect it has performance issue when it is actually running, because it is almost native code right? How could it be slower.


## Multiple hot-reload entry

**html.hotReloadComp** accept an option argument **host**, so if we have multiple project and want hot-reload at the same time, we can do this:

In project (normally startup project), we can set the hot-reload entry for other projects:

```fsharp
let myApp =
    div {
    #if DEBUG
        html.hotReloadComp (Project1.comp, "Project1.comp", host = "http://localhost:8091")
    #else
        Project1.comp
    #endif

    #if DEBUG
        html.hotReloadComp (Project2.comp, "Project2.comp", host = "http://localhost:8092")
    #else
        Project2.comp
    #endif
    }
```

Then startup two watchers:

```bash
fun-blazor watch Project1.fsproj --server http://localhost:8091
fun-blazor watch Project2.fsproj --server http://localhost:8092
```

Last not not least, we can **dotnet run** our startup project.

With this, when we edit any files (with **// hot-reload** at its top) which related to the entry function, and get hot-reload.


## Finally

By default, the setup complexity is same as before or even simpler. When the json serialization is improved, it can be way simpler for blazor wasm mode. Another benefit is that we can easily get it works for large solutions with multiple projects.

So far so happy 😊
