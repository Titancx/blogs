IE浏览器的阻止XSS攻击功能
====
昨天接到一个专利搜索结果页面的bug：在某个搜索条件下，IE10浏览器不能正常显示。

### 1. 重现问题

我们这里没有IE10浏览器，一般都是通过Windows8.1上的IE11浏览器模拟实现，所以我立即在IE11上验证，没有想到IE11竟然也重现了这个问题，所以我也没有必要去模拟IE10了。
>开发环境：我们组有三台服务器，一台装Unbuntu用来做普通服务器用，剩下两台都用来测试网站在IE上兼容性，一台是Windows Server 2012安装了IE8，一台是Windows 8.1安装了IE11。IE9，IE10都是通过IE11模拟测试的。

重现的时候发现两个问题，一个是页面顶部给出提示“Invalid input”(非法输入)，这是我们系统给出的错误提示。另一个就是IE给的提示信息“Internet Explorer has modified this page to help prevent cross-site scripting.”

### 2. 从错误提示着手进行调试

因为有我们系统“非法输入”的错误提示，所以我决定通过调试来确定是哪里出错了。

根据经验猜测这个错误很可能是由某个AJAX请求出错造成的，在锁定了AJAX之后，又对比该请求在Chrome和IE11浏览器中的不同，发现某个字段中括号（英文的左右括号）被替换成了#，由于该字段是由后台写在`<script></script>`，所以首先去查看网页源代码，令人吃惊的是网页源代码中竟然括号也变成#，这不得不让我怀疑是后台针对IE浏览器做了特殊处理。虽然我们知道后台不应该这么做，并且得到后台确切的答案（他们不会这么做）以后，我们决定通过Fiddler代理来查看请求和响应数据，而通过Fidder查看到的响应是正确的，难道是IE浏览器在搞鬼？

### 3. 从IE浏览器的关于XSS的提示着手
在网上搜索了“Internet Explorer has modified this page to help prevent cross-site scripting.”，找到了一些[文章](http://answers.microsoft.com/en-us/ie/forum/ie9-windows_7/internet-explorer-9-has-modified-the-page-to-help/84157078-964f-e011-8dfc-68b599b31bf5?tab=MoreHelp&auth=1)，大致说这是由于IE检测到页面中可能带有跨站脚本攻击代码，而对页面代码做了一些修改，以保护用户免受跨站脚本攻击。
并且提供了一些解决方案：点击`Tools->Internet Options->Security->Custom level`，在弹出的对话框中禁用XSS Filter。这样设置以后页面确实正常显示了。可见该问题确实是有IE误判造成的。

但是我们做为在线网络服务的提供商，不能依靠用户修改浏览器设置来解决问题，并且该设置并不是针对某个具体网站，而是整个互联网，意味着用户彻底放弃了IE针对XSS攻击的保护，得不偿失。还要继续寻找其他方案。终于在[另一篇文章](https://msdn.microsoft.com/zh-cn/library/dd565647%28v=vs.85%29.aspx?f=255&MSPPError=-2147217396)中找到答案。给页面添加一个响应头
```
X-XSS-Protection: 0
```
这样就通过禁用响应头彻底解决了该问题。

###4. 解决问题以后的一些思考
问题解决了，但是还是有写疑问，我们很容易找到IE误判为XSS攻击的代码，却不知道为什么会被判断为XSS攻击代码。只在[这篇文章](http://p42.us/ie8xss/Abusing_IE8s_XSS_Filters.pdf)中找了一些示例，但是我们的页面源代码中都不符合这些规则。估计IE官方也不会给出这些规则，否则很容易被黑客利用，设计能够避开这些规则的XSS攻击代码。

另外还想谈谈IE的阻止XSS攻击的功能，我们有时也会在Chrome看到相似的提示，但是Chrome给出另外一种选择，大概意思是我已经了解可能的风险，并且要继续访问，通过次选项用户可以比较方便地继续浏览其非常信任的网站。在这件事件上，IE显得非常保守，没有给出第二种选择，用户遇到这种情况，只会怀疑网站除了问题，也不会知道怎样才能查看网页内容。经过IE修改后的代码一般很难正确运行，而IE却尝试继续运行。远不如Chrome实在，检测到危险代码，干脆放弃执行代码，并给用户醒目提示。

IE对（疑似）XSS代码处理也真够不遗余力的，连查看源代码都不放弃，个人认为这是IE设计上的缺陷，查看源代码，我看的是**“源”**代码，将响应当做普通的不可执行的文档处理，不可能出现XSS攻击，还有就此时IE修改了源代码，却没有给出任何提示。

打完收工，睡觉。