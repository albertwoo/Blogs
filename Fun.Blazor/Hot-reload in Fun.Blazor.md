- id: d959e36a-f4fe-4a10-88af-5e738633db0f
- title: Hot-reload in Fun.Blazor
- keywords: fsharp,blazor,Fun.Blazor
- description: Hot-reload is very important for building frontend application, because it can save life. Here is the journey for how I make it work and hot to use it.
- createTime: 2022-02-28
---


I know csharp got hot-reload for blazor for a while, I also keep tracking on the [fsharp issue](https://github.com/dotnet/fsharp/issues/11636) and hope there is some miracle which can happen. But there is no progress here. So I decide to have a look for what I can do.  
To make `dotnet watch` work directly is too hard to me which required a lot of knowledge about FSharp.Compiler.Service and the IL format which csharp roslyn is using for patch the program for hot-reload.

But luckily I still remember [Fabulous](https://github.com/fsprojects/Fabulous) has hot-reload years ago, which is using [FSharp.Compiler.PortaCode](https://github.com/fsprojects/FSharp.Compiler.PortaCode). So I start to try on that. And it turns out to integrate it is pretty simple.


## The idea is:

1. Start a process to monitor the source code of target project
2. Use FCS (FSharp.Compiler.Server) to parse the syntax tree to some domain types which can be used to express the syntax and to be serialized.
3. Use HttpClient to send the syntax tree to the blazor server
4. Blazor server expose an endpoint to accept the syntax tree data
5. Blazor server has an root component to evaluate the syntax tree to real code and replace the root render function
6. Blazor server trigger re-render, then use the newly created render function to render all the stuff again. Then the rest is handled by blazor

Let's see the final output here, because I am very exited about the result:

![hot-reload-demo](./site-hot-reload.gif)


## There are couple of problems I have when I use `FSharp.Compiler.PortaCode`

1. The syntax tree conversion is not fully implemented, a lot of use cases cannot be handled. Like some syntax: Yield in computation expression, DU, struct tuple, generic. I spent some hard time to partially fix them, because to be frankly I just do not know too much about all that stuff.
2. It use HttpListener which require Admin mode. Luckily is I am using blazor server, so the server is already there. So I can just remove this part.
3. The speed is not fast if the file grows. I tried on the doc app which contains many files but less than 50, and the speed is not fast.

Maybe there are just too much limitation in its design, so that project is not maintained for two years. I do not expect to make the integration works perfectly, what I want to achieve is build a hot-reload feature in a limited way to fill the time gap. I know fsharp will have the hot-reload like csharp has in the future (maybe years 😒).


## The limited way

To avoid the performance issue and narrow down the failing possibility of converting syntax tree, I got bellow approach:

Explicitly mark the file you want have hot-reload

    Add comment line `// hot-reload` at the top of the source file

For example, if you have code file like below (ordered):

``` {highlight:[5]}
UI
  ├─── Controls.fs
  ├─── PostDetail.fs
  ├─── PostList.fs
  ├─── Main.fs
  ├─── Index.fs
```

`Main.fs` has the entry render function. And it must have `// hot-reload` at the top.

If you also want hot-reload for `PostDetail.fs`, you will need to add `// hot-reload` at its top, also add `// hot-reload` to `PostList.fs` if `PostDetail.fs` is used in here. But if `PostDetail.fs` is just used in `Main.fs`, then you only need add `// hot-reload` to itself.

Because that is the way it works, I need to make sure all the chained functions are evaluated with the new syntax tree.


## Only work in blazor server mode

So far, I only make it work for server mode. The idea of making it work for WASM mode is not popping up into my head so far. Maybe I will try that in the future. But workaround is simple: just create a server project and add WASM app as a reference then it should work. I set them up in the template.


## How to use it

I added hot-reload features to all the template projects.

> dotnet new --install Fun.Blazor.Templates::2.0.0-beta010
> dotnet tool install --global Fun.Blazor.Cli --version 2.0.0-beta020

I will go through the most complex one: `Blazor + shoelacejs + tailwindcss`

Create the demo project like below:

    dotnet new fun-blazor-wasm-sltw -o Demo

It will create stuff like this:

```
.vscode // contains some config for shoelace and tailwind intellisense

Demo // This is the real wasm app
  ├─── JsInterop.fs // This cannot be hot-reloaded, because some syntax cannot be converted
  ├─── App.fs // This is entry render function
        // hot-reload
  ├─── Startup.fs

Demo.Server // just for hot-reload
  ├─── Index.fs // setup the hot-reload
        type Index() =
        #if DEBUG
            inherit HotReloadComponent
                (
                    "Demo.App.app", // The full name for the entry render function
                    Demo.App.app,
                    staticAssetsDir = __SOURCE_DIRECTORY__ + "/../Demo/wwwroot" // this is for css hot-reload
                )
        #else
            inherit FunBlazorComponent()
            override _.Render() = Demo.App.app
        #endif
  ├─── Startup.fs // Just setup the dependency and boot up the server 
```

For this complex project, I need to run like below commands (but you can also have a script to run them at one time):

    cd to Demo // This is for tailwindcss jit mode 
    pnpm install
    pnpm run watch-css

    cd to Demo.Server // Can provide the endpoint to accept the syntax tree data
    dotnet run

    cd to Demo // The cli to watch the WASM project and send the syntax tree data to server
    fun-blazor watch .\Demo.fsproj -s https://localhost:5001


## Finally

Anyway, I made it 😊! I know it is far from perfect, I also did not expect to make it perfect. I am not an expert on any of those. I just want to make it more productive so I can save some time on my next project when I use Fun.Blazor.
