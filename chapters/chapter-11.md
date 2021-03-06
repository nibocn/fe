如何优化前端应用性能
===

> 我开始写前端应用的时候，并不知道一个 Web 应用需要优化那么多的东西。编写应用的时候，运行在本地的机器上，没有网络问题，也没有多少的性能问题。可当我把自己写的博客部署到服务器上时，我才发现原来我的应用在生产环境上这么脆弱。

我的第一个真正意义上的 Web 应用——开发完应用，并可供全世界访问，是我的博客。它运行在一个共享 256 M 内存的 VPS 服务器上，并且服务器是在国外，受限于网络没有备案的缘故。于是，在这个配置不怎样的、并且在国外的机器上，打开我的博客可是要好几分钟。

因此，我便不断地想着办法去优化打开速度，在边学习前端知识的时候，也边玩着各种 Web 运维的知识。直到我毕业的时候，我才有财力将博客迁往 Amazon 的AWS 上，才大大降低了应用的打开速度。

虽然处理速度上去了，带宽等条件也没有好多少。随着能力的提供，发现最开始觉得的服务器又在国外、解析域名 DNS 影响了网站打开速度，还有一系列的问题需要优化。

所以，它给我的第一个经验是：**当更好的服务器可以解决问题时，就不会花费人力了。**

后来吧，随着对于 Web 开发了解的深入，我开始对这性能优化有了更深入的了解。

博客优化经验：速度优化
---

当我的博客迁移到 AWS，服务器的性能提升之后，我便开始着手解决网络带来的问题。因此，首先就是要确认问题到底是来自网络的哪一部分。这时，我们就可以借助于 Chrome 浏览器，来查看一下问题的来源。

打开 Chrome 浏览器的 Network 标签，点击最后的 **Waterfall** 就可以看到相应的内容：

![Chrome 网络工具](../images/network-performance.png)

从图中，我们就可以看到影响加载速度的主要因素是：

 - Waiting (TTFB)。TTFB，即Time To First Byte，指的是：请求发出后，到收到响应的第一个字节所花费的时间。
 - Content Download。即下载内容所需要的时间。

用户下载内容所需要的时间，受限于服务器的资源、资源的大小以及用户的网络速度。因此，我们暂时不讨论这方面的内容。

我们可以看到这里的主要限制因素是，TTFB。而要对 TTFB 优化的话，就需要关心两部分：

 - 服务器。比如：如果有复杂的防火墙规则或路由问题，则TTFB时间可能很大。又或者是你的服务器性能不好，但是你启用了 GZIP 压缩，那么它也将增加 TTFB 所需要的时间。
 - 应用程序。比较简单的作法是和我一样，将这部分交给 New Relic 去处理，我们就可以知道应用中哪些地方比较占用资源。

### TTFB 优化

而对于早期我的博客来说，还有一个主要的限制因素是 DNS 查询所需要的时间——即查询这个域名对应的服务器地址的时间。这主要是受限于**域名服务器提供的 DNS 服务器比较慢**，作一个比较简单的对比：

![淘宝 vs www.phodal.com](../images/ping-results.png)

通过使用 ``traceroute`` 工具，我们就可以发现经过不同网关所需要的时间了：

![traceroute 结果](../images/traceroute.png)

而这还是在我采用了 DNSPod 这样的工具之后的结果，原先的域名服务器则需要更长的时间。可惜，我这么穷已经不能付钱给更好的 DNS 服务提供商了。

### 服务器优化

后来，我发现我的博客还有一个瓶颈是在服务器上。于是，我使用 APM 工具 NewRelic 发现了另外一个性能瓶颈，服务器默认使用的 Python 版本是 2.6。

对于如我博客这样性能的服务器来说，应用的一个很大的瓶颈就是：大量的数据查询。

![NewRelic Result](../images/newrelic.png)

如当用户访问博客的列表页时，大概需要 500+ ms 左右的时间，而一篇详情页则差不多是 200ms+。对于数据查询来说，除了使用更多、更好的服务器，还可以减少对数据的查询——即缓存数据结果。

而在当时，我并没有注意博客对于缓存的控制，主要是因为使用的静态资源比较少。这一点直到我实习的时候才发现。

项目优化经验：缓存优化
---

当我试用了 PageSpeed 以及 YSlow 之后， 我发现光只使用 Nginx 来启用压缩 JS 和 CSS 是不够的，我们还需要：

 - CSS和JavaScript压缩、合并、级联、内联等等
 - 设置资源的缓存时间。将资源缓存到服务器里，减少浏览器对资源的请求。
 - 对图片进行优化。转化图片的格式为 JPG 或者 WEBP 等等的格式，降低图片的大小，以加快请求的速度。
 - 对 HTML 重写、压缩空格、去除注释等。减少 HTML 大小，加快速度。
 - 缓存 HTML 结果。缓存请求过的内容，可以减少应用计算的时间。
 - 等等

对于 CSS 和 JS 压缩这部分来说，我们可以从工程上来实现，也可以使用诸如  Nginx Pagespeed 这样的工具来实现。但是我想使用工程手段更容易进行控制，并且不依赖于 HTTP 服务器。

设置资源的缓存时间就比较简单了，弄清楚 Last-Modified / Etag / Expires / Cache-Control 几种缓存机制，再依据自己的上线策略做一些调整即可。

至于缓存  HTML 结果来说，比较简单的做法就是采用诸如  Squid 和 Varnish 这样的缓存服务器。

当然了，还有一些极其有意思的方法，如将 JavaScript 存储在 LocalStorage 中。

在今天看来，很多对于 Web 应用的优化就是：只要你想优化一个常见的应用，那么它一定是会有工作的。

移动优化经验：用户体验优化
---

受限于移动应用的种种条件限制，我们会有选择的对移动应用进行缓存。并且，它还有助于我们改善移动应用的用户体验。移动应用与 Web 应用受限于网络条件，会有一些不同之处：

 - 移动应用或者移动 Web 应用，都是先响应用户的行为，再去获取数据。而桌面应用则可以先获取数据，再响应用户的行为。
 - 移动应用或单页面应用，在进行页面跳转后，为了加快返回上一页的速度，都会考虑数据或者页面。
 - Web 应用的用户有着更稳定的网页条件，而移动应用则容易遇到网络问题。
 - 等等

因此，在完成移动应用的时候，我们都会缓存 API 的结果。并在页面的生命周期内，对页面进行优化。

### 缓存 API 结果

### 生命周期优化

优化中的反最佳实践
---

在对应用进行优化的过程中，还会遇到一个非常有意思的问题：你将采用的优化方案，往往是和业界所推荐的最佳模式相违背的。如 inline css 对于用户来说，可以获得更好的体验效果。又如更快的 Google AMP，则能在打开速度上有更大的提升，但是却和最佳实践相去甚远。

待续~~
