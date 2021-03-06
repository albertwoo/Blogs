- id: 1ba77930-d671-4fe1-ac6c-e1a62eff964f
- title: 漫谈 XSS(跨站点脚本攻击)，CSRF(跨站点请求伪造，跨域攻击)
- keywords: 时间笔记,XSS,跨站点脚本攻击,CSRF,跨站点请求伪造,跨域攻击
- description: 我一直被站点安全相关的的名词困惑着，一方面是觉得它们很神秘，另一方面它们本身也很抽象。
- createTime: 2021-07-06
---

继续学习XSS(跨站点脚本攻击)，CSRF(跨站点请求伪造，跨域攻击)。

经过查寻各方面的资料，以及根据我自己的经验，我觉得还是XSS比较可怕。因为CSRF主要是基于浏览器对Cookie处理的特性来进行攻击的，所以如果你不用Cookie那也就没有这样的风险了。随着各种工程化的SPA网站的出现，它们更多的是使用js来进行业务逻辑的控制，不再像以前一样需要依赖浏览器对form表单处理的特性来发AJAX请求了，因为在那种不使用js的场景，我们需要浏览器自动把用户的信息带上，好让我们的服务端能够识别用户信息来进一步操作。但这都什么时代了，大家如果是用的React, Angular或Vuejs，那为什么还要依赖于Cookie呢，用Http Authorization配合JWT不香吗？我们完全可以把生成好的token放在Local storage里面，慢慢享用啊！

当然，老的网站还要继续维护，安全的防线可不能放松。而且Cookie的用法可能会特别多，毕竟存在了二十多年了，很多第三方服务可能需要它。我对于为什么Cookie还大行其道怀有困惑？有可能Cookie现在并没有大行其道了，而更有可能的是我还需要进一步的学习和受伤。

为什么我觉得XSS更可怕呢，因为就算我不用Cookie，只要攻击的人在我的用户访问我的网站时，浏览器加载了恶意的脚本，那么几乎它就可以做任何事情了。比如读取Cookie，比如读取local storage里的数据，以及其他任何我的网站自己的js可以做的任何事情。

如果你的Cookie配置了HttpOnly以及配合同源策略，加之Samesite的Cookie配置，而且用户的浏览器刚好也支持的话，貌似就万事大吉了。然而，转念一想也不一定。因为恶意脚本可以在你的网页的图片，按钮等上添加事件，带用户点击后触发AJAX请求给你的自己的服务器，从而恶意修改某些资源，比如银行的网页上，就发请求到银行说转账给谁多少钱。因为这样，浏览器和你的服务器都区分不出这个AJAX是你自己的脚本发起的还是恶意脚本发起的。

或许你觉得加一个CSRF Token在你的Form表单上，在服务端再去验证，就又可以万事大吉了。但是既然恶意脚本都已经注入了，那它完全也可以偷取你这CSRF Token啊。如果你是放在表单里的，那就搜寻你的DOM树；如果你用的是双重Cookie验证，那么肯定就不是HttpOnly的吧，因为这样你自己的js也无法获取，从而添加到query里，所以还是没有防到。


所以XSS最可怕！！！

写到这里，突然觉得有点恐怖，感觉浏览器上冲浪很容易就掉进水里了。


# Cross-site scripting (XSS) 

如果被恶意脚本注入了，我觉得差不多就凉了。但是XSS的防范总的来说其实并不难（但是每个细节都要做到的话其实还是挺难的），尤其是对有敏感功能的页面，最简单的就是检查表单输入的值有没有恶意脚本等。在asp.net core里面，用razor语法写的那些表单处理逻辑默认就有保护机制。它会对你输入的字符进行编码筛选，对不在合法字符集里的字符进行特殊编码，比如将不可信的数据放入html element的时候，字符 < 就会转化为 &lt。对于html attribute和javascript也都有类似的字符编码处理。处理之后恶意脚本就不会被浏览器意外加载启动了。

所以，这提醒我的就是，对于在浏览器上处理动态内容，尤其是任何有用户可以输入的地方，比如评论区，比如登录页面的输入框等等，都需要提高警惕。看来是时候为我这个小网站思考这方面的安全性了（以后再说吧😂）。

我个人React使用最多，其中有一个属性叫做 dangerouslySetInnerHTML， 它就非常容易产生XSS，所以使用的时候要多加小心。

但是现在的SPA框架（React, Angular, Vuejs），最多清理在你的应用里输入富文本时的XSS威胁，当攻击者直接调用你的后端api直接保存富文本的时候也还是有危险的。所以还是得在后端做过滤处理。

比如，我这个网站，前段富文本输入用的是quilljs，它本身就添加了XSS的防范处理，但是当调用我后端的asp.net core api保存文章时，还需要使用类似 mganss/HtmlSanitizer: Cleans HTML to avoid XSS attacks (github.com) 这样的库来进一步处理才能保存到数据库。否则攻击者直接调用了我的api保存了恶意脚本后，当用户浏览相应的文章后就会被攻击。


## 总结
学习和深入了解理论知识真的有好处，不说我理解地更深入了，至少发现了一些潜在威胁吧，继续！！！
