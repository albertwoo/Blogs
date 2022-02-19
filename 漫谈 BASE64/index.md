- id: 2a65da15-5fb0-4b4c-99f3-0bafb51e1c05
- title: 漫谈 BASE64
- keywords: 时间笔记,BASE64,面试,扎💔,F#
- description: 前几天参加了三年来第一次面试，二面面试回答如抢答，被问及base64的目的时顿时语塞
---


前几天参加了三年来第一次面试，二面面试回答如抢答，被问及base64的目的时顿时语塞。当时我只能说我用过的场景，比如html img, JWT, query string里使用等。


> In programming, Base64 is a group of binary-to-text encoding schemes that represent binary data (more specifically, a sequence of 8-bit bytes) in an ASCII string format by translating the data into a radix-64 representation.

👆wikipedia上是的解释


所以目的就是为了把二进制数据转化为字符串的形式。而转化出来的字符串只有64个字符可供选择，英文字母大小写总共52个，加上阿拉伯数字10个，另外一个是”+“，一个是“/“。还有个“=”用作扩充后缀，根据不同的实现和场景各异，非必须。并广泛应用于web, SMTP 邮件等。

原理其实很简单，就是把原始的数据二进制展开后，6位一组。比如3个8位的字符，展开就是24位，如果按照6位一组重新划分就可以得到4组。而2的6次方也就是64，所以只需要把每组的6位换算成一个数字，再按顺序从64个字符中取一个，最后把取出来的字符拼接成字符串即可。

用f#做个最简单的实现可以如下：

```fsharp
open System
open System.Text

// 定义一个字符映射表
let base64CharTable =
    [
        yield! ['A'..'Z']
        yield! ['a'..'z']
        yield! ['0'..'9']
        '+'
        '/'
    ]
    |> List.indexed
    |> Map.ofList

let toBase64 (bytes: byte[]) =
    bytes
    // 将每个byte中的每一个bit取出来以供选择
    |> Seq.map (fun by -> [| for i in 0..7 -> by >>> i &&& 1uy |] |> Seq.rev)
    |> Seq.concat
    // 6个bit分为一组
    |> Seq.chunkBySize 6
    |> Seq.map (fun bits ->
        // 把每组bit最后几位补0，再进行位操作合并成一个byte
        let by = 
            (bits |> Seq.rev |> Seq.toList)@[ for _ in 1..8-bytes.Length -> 0uy ]
            |> Seq.mapi (fun i by -> by <<< i)
            |> Seq.reduce (|||)
        // 合并后的byte即可转为一个int用来从映射表中取出相应的字符
        let num = Convert.ToInt32 by
        Map.find num base64CharTable)
    // 最后把所有字符组合成一个字符串即所谓的base64 string了
    |> String.Concat

let fromBase64 (str: string) =
    str
    // 重映射表中把对应字符的index找出来
    |> Seq.map (fun c -> base64CharTable |> Map.findKey (fun _ c' -> c = c'))
    // 将index的前6个bit取出来并组合起来
    |> Seq.map (fun num -> [| for i in 0..5 -> num >>> i &&& 1 |] |> Seq.rev)
    |> Seq.concat
    // 取8个bit为一组
    |> Seq.chunkBySize 8
    // 将每组的bit按顺序进行位操作，还原最初的byte
    |> Seq.map (Seq.rev >> Seq.mapi (fun i bit -> (byte bit) <<< i) >> Seq.reduce (|||))
    |> Seq.toArray

let base64Result =
    "Cool programmer"
    |> Encoding.UTF8.GetBytes
    |> toBase64

let originalString =
    base64Result
    |> fromBase64
    |> Encoding.UTF8.GetString
```

最后结果是：

    val base64Result : string = "Q29vbCBwcm9ncmFtbWVy"
    val originalString : string = "Cool programmer"

性能肯定是不行，但是结果还是满足预期的。


## 总结

看了原理，并进行简单的实现之后确实对base64有了更新的认识。不过如果让我当时面试的时候来实现估计还是只能写伪代码，对一些细节认识的不足还是导致我在实现的时候遇见了一些问题，比如大小端的问题。刚开始做的时候我用的是：

```fsharp
"M" 
|> Encoding.UTF8.GetBytes 
|> Seq.map (fun by -> [| for i in 0..7 -> by >>> i &&& 1uy |])
```

出来的结果是

    [[|1uy; 0uy; 1uy; 1uy; 0uy; 0uy; 1uy; 0uy|]]

但其实应该反排一下，否则生成的字符串和通常的base64编码出来的就不一样。真的是细节决定成败啊，服！！！