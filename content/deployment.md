---
title: 部署和监控
order: 41
description: 如何在生产环境中部署，运行，监控 Meteor 产品应用。
discourseTopicId: 19668
---

读完全文，你将能够：

1. 在部署一个 Meteor 应用之前应该考虑什么？
2. 如何部署到一些常见的 Meteor 应用托管环境。
3. 如何设计一个部署过程，以确保能够维护应用的质量。
4. 如何使用统计工具监控用户行为。
5. 如何使用 Kadira 监控你的应用。
6. 如何确保你的网站可以被搜索引擎收录。

<h2 id="deploying">部署 Meteor 应用</h2>

你建立了一个 Meteor 应用，并且通过了测试，现在是时候把它放到网上让全世界人民欣赏了。部署一个 Meteor 应用跟部署其他基于 WebScoket 的 Node.js 应用类似，但在某些方面存在不同。

部署一个 web 应用跟大部分的发行软件有本质上的区别，我们可以多次部署。不需要等待用户换取软件的新版本，因为服务器会把更新的信息推送给用户。

使用优秀的质量保证 (QA) 流程对更改进行测试仍然是非常重要的。即使修复 bugs 很容易，这些错误仍然可能给客户造成严重困扰，并可能导致数据丢失！

<h3 id="environments">部署环境</h3>

在 web 应用部署中最常见的三种运行环境：

1. **Development.** 开发环境，这是指你开发新功能和运行本地测试的机器。
2. **Staging.** 中间环境，类似于 Production，但对应用的用户不可见。可以用于测试和 QA.
3. **Production.** 生产环境，用户使用中的 app 的真实部署环境。

中间环境主要是为了提供一个用户不可见的测试环境，在框架上跟真实部署环境无限接近。新代码在开发环境中不会出现问题，但到了真实生产环境中就出问题了。以客户端和服务器端之间的延迟问题为例 —— 连接到本地开发服务器，这么小的延迟你可能都不会将它当作一个问题。

因为这个原因，开发者会尽量让中间环境接近实际的生产环境。这意味着我们下面提到的在生产环境部署的步骤，也非常适用于中间环境。

<h3 id="environment">环境变量和设置</h3>

代码之外有两种方式配置你的应用：

1. **Environment variables.** 环境变量，这是运行过程中需要设置的一系列 `ENV_VARS`.
2. **Settings.** 设置，位于 JSON 对象中，通过 Meteor 命令 `--settings` 设置或字符串化到 `METEOR_SETTINGS` 环境变量。

设置用于设置环境 (中间环境或生产环境) 中具体事项，例如用于连接谷歌的访问标志和密钥。在特定环境中运行应用时这些设置不会改变

环境变量用于设置应用进程事项，可以根据应用的不同实例进行改变。例如，你可以为各个进程设置不同的 `KADIRA_OPTIONS_HOSTNAME` 来确保 [kadira](#kadira) 日志记录有用的主机名。

在存储这些设置上还有一点需要说明：不要把这些设置文件跟你的应用代码存放在同一个栈。阅读[安全性章节](security.html#api-keys)了解这些设置最好存放在什么地方。

<h2 id="other-considerations">其他考虑</h2>

将应用部署到生产主机前还有其他事项需要考虑。记住，如果可能的话请在生产环境和中间环境中进行这些操作。

<h3 id="domain-name">域名</h3>

用户通过什么 URL 访问你的网站？你可能需要向域名注册商注册一个域名，并设置 DNS 指向特定网站 (这要看你如何部署，请看下文)。如果你在 Galaxy 部署，在测试应用的时候可以使用 `x.meteorapp.com` 域名。

<h3 id="ssl">SSL 认证</h3>

在 Meteor 应用中使用 SSL 总是一件好事 (要了解为什么请阅读[安全性章节](security.html#ssl))。一旦你注册了一个域名，就会需要根据域名证书颁发机构生成一个 SSL 证书。

<h3 id="cdn">CDN</h3>

虽然不是严格要求，但为你的网站建立内容分布网络 (CDN) 会是一个好主意。CDN 是托管网站静态资产 (如 JavaScript, CSS, 和 images) 的服务器网络，它分布在世界上各个地方，并使用最靠近用户的服务器为用户提供这些静态资产，目的是加速访问速度。例如，如果网站的服务器在 USA 的东海岸，而你的用户在澳大利亚，CDN 可以在澳大利亚，甚至在用户所在的城市托管一份该网站 JavaScript 的副本。这将极大缩短网站的初始加载时间。

CDN 最基本的语法就是把文件上传到 CDN 并改变 URL 指向 CDN (例如你的 Meteor 应用位于 `http://myapp.com`，则将图片的 URL 从 `<img src="http://myapp.com/cats.gif">` 改为 `<img src="http://mycdn.com/cats.gif">`). 但是这对于 Meteor 来说有点困难，因为最大的文件夹 —— Javascript 组件会因为你更改应用而经常改变。

对于 Meteor 来讲，我们推荐使用内部支持的 CDN (例如 [CloudFront](http://joshowens.me/using-a-cdn-with-your-production-meteor-app/))，跟提前上传文件相比，CDN 自动从你的服务器获取文件。你把文件放在 `public/` 文件夹 (在这个例子中是 `public/cats.gif`)，这样当澳大利亚的用户通过 CDN 访问 `http://mycdn.com/cats.gif` 时，CDN 会访问 `http://myapp.com/cats.gif` 获取文件然后发送给用户。虽然这比直接访问 `http://myapp.com/cats.gif` 获取文件会慢一点，但这只会发生一次，因为 CDN 会保存这些文件，接下来澳大利亚用户访问都可以快速获取文件。

让 Meteor 使用 CDN 存储 Javascript 和 CSS 文件需要在服务器调用 `WebAppInternals.setBundledJsCssPrefix("http://mycdn.com")`。这也会包含 CSS 文件中包含的图片 URL。如果你需要使用动态前缀，可以从一个函数返回前缀并传递给 `WebAppInternals.setBundledJsCssUrlRewriteHook()`.

对于文件夹 `public/` 中的所有文件，将它们的 URL 指向 CDN. 可以使用名为 `assetUrl` 的helper.

原始文件:

```html
<img src="http://myapp.com/cats.gif">
```

URL 指向 CDN:

```js
Template.registerHelper("assetUrl", (asset) => {
  return "http://mycdn.com/" + asset
});
```

```html
<img src="{{assetUrl 'cats.gif'}}">
```

<h4 id="cdn-webfonts">CDNs 和 webfonts</h4>

如果你的应用中包含网络字体并通过CDN获取，您可能需要配置字体的 header，以允许跨域资源共享 (因为网络字体来自于你的网站之外的其他地方)。在 Meteor 可以通过添加一个 handler 轻松做到 (你必须确保 CDN 遍历 header):

```js
import { WebApp } from 'meteor/webapp';

WebApp.rawConnectHandlers.use(function(req, res, next) {
  if (req._parsedUrl.pathname.match(/\.(ttf|ttc|otf|eot|woff|font\.css|css)$/)) {
    res.setHeader('Access-Control-Allow-Origin', /* your hostname, or just '*' */);
  }
  next();
});
```

以 Cloudfront 为例，你将能够：

- 选择分布
- Behavior 选项
- 选择应用原型
- 编辑按钮
- 在 "Whitelist Headers" 下拉选择 "Origin"
- 添加按钮
- "Yes, Edit"按钮

<h2 id="deployment-options">部署选项</h2>

Meteor 是一个开源平台，跟 regular Node.js 应用一样，你可以在上面运行 Meteor 应用。如果你要自己管理基础架构，并让 Meteor 应用正确运行，让所有人都可以使用，还是一件蛮棘手的事。这也是为什么我们推荐在 Galaxy 上部署 Meteor 应用。

<h3 id="galaxy">Galaxy (推荐)</h3>

部署你的应用最简单的就是使用 Galaxy, Galaxy 是 Meteor Development Group 建立的专门用于部署 Meteor 应用的。

Galaxy 是在 Amazon AWS 上运行的分布式系统。如果你了解 Galaxy 是如何工作以及如何正确运行 Meteor 应用的，你一定会体会到 Galaxy 的价值，Galaxy 可以节约大量的时间和避免不必要的麻烦。目前很多大型 Meteor 应用都是在 Galaxy 上运行，其中很多都是从定制化的部署方案转换到 Galaxy.

如果要在 Galaxy 上部署，需要[注册一个账号](https://www.meteor.com/galaxy/signup),并分开提供 MongoDB 数据库 (参考下文)。

注册好账号后，[部署到 Galaxy](http://galaxy-guide.meteor.com/deploy-guide.html)就很简单了。只需要[在设置文件中添加环境变量](http://galaxy-guide.meteor.com/environment-variables.html)指向你的 MongoDB, 然后就可以根据下面的命令部署了：

```bash
DEPLOY_HOSTNAME=us-east-1.galaxy-deploy.meteor.com meteor deploy your-app.com --settings production-settings.json
```

为了让 你的域名 (在这个例子中是 `your-app.com`) 在 Galaxy 也可以工作，你需要[设置 DNS 指向 Galaxy](http://galaxy-guide.meteor.com/dns.html)。完成之后，就应该可以通过浏览器访问你的网站。

你也可以通过 https://galaxy.meteor.com 进入 Galaxy UI,在 Galaxy UI 可以管理你的应用，监控连接数和资源使用，查看日志，更改设置。

<img src="images/galaxy-org-dashboard.png">

如果你根据[我们的建议](security.html#ssl)，你可能会想要在 Galaxy 应用上通过域名的证书和密钥[设置 SSL](http://galaxy-guide.meteor.com/encryption.html)。这里的关键点是添加 `force-ssl` 包，并通过 Galaxy UI 添加 SSL 证书。

一旦设置好 Galaxy, 部署就很简单了 (重新运行上面的 `meteor deploy` 命令)，监控就更简单了 —— 登入 galaxy.meteor.com, 在那里就可以监控。

<img src="images/galaxy-scaling.png">

<h4 id="galaxy-mongo">一起使用的 MongoDB 托管环境</h4>

如果你正在使用 Galaxy (或者使用下面提到的服务，并需要管理 MongoDB),建议使用[MongoDB 托管服务](http://galaxy-guide.meteor.com/mongodb.html)。可以有很多种选择，但我们推荐[mLab](https://mlab.com/)。需要支持 oplog tailing，并且存在于 AWS 的 us-east-1 或 eu-west-1 区域。

<h3 id="mup">Meteor Up</h3>

[Meteor Up X](https://github.com/arunoda/meteor-up/tree/mupx)，也被叫做 "mupx",是一个开源工具可以通过 SSH 用来部署 Meteor 应用到任何在线服务器。Mup 可以满足一些关键的部署条件，但你还是需要做很多工作确保加载平衡和版本更新工作可以顺利进行 —— 实质上就是将 `meteor build` 这一步自动化，并将组件放在服务器上。

你可以从很多通用的托管服务商获取在 Ubuntu 或 Debian 上运行的服务器。Mup 可以通过你在配置中提供的密钥 SSH 到你的服务器。请[观看该视频](https://www.youtube.com/watch?v=WLGdXtZMmiI)了解更完整的操作。

<h3 id="custom-deployment">定制化部署</h3>

如果你想从头开始完全弄清楚你的托管解决方案，Meteor 通过命令行 `meteor build` 创建一个包含一个简单 Node.js 应用的部署组件。在执行 `meteor build` 前需要先安装 npm 所依赖的任何组件。你可以按照自己喜欢的方式托管你的应用，从设置和配置方面来讲是有很多选择的。

**注意** 在正确的架构上构建你的组件是很重要的。如果在自己部署的机器上构建，有很大可能会部署到其他服务器架构上。你会需要使用 `--architecture` 指定正确的架构：

```bash
# 例如部署到 Ubuntu linux 服务器：
npm install --production
meteor build /path/to/build --architecture os.linux.x86_64
```

运行该应用需要提供 Node.js 0.10.x 和 MongoDB 服务器。最新发布的 Meteor 已经通过 Node 0.10.43 测试。然后通过调用 `node`, ROOT_URL, 和 MongoDB 就可以运行该应用。

```bash
cd my_directory
(cd programs/server && npm install)
MONGO_URL=mongodb://localhost:27017/myapp ROOT_URL=http://my-app.com node main.js
```

除非你有特殊需求，需要推出自己的托管环境，不然这里的其他选择都是很简单的，也比什么都从头开始构建会有一个更好的起点。正确运行一个 Meteor 应用并让所有人都可以访问有时会很困难，[Galaxy](#galaxy)可以处理很多具体工作，比如通过路由链接客户端到正确的 containers, 并能够处理协调版本。

<h2 id="process">部署步骤</h2>

虽然部署一个 web 应用比大多数的软件发布简单，但也不能掉以轻心。在正式发布前需要做 QA 和测试以确保用户体验良好，避免数据损害这种糟糕的情况。

发布应用程序时遵循一个发布步骤将会是一个好主意。常见发布步骤我们按点列出：

1. 将新版本的应用部署到中间环境。
2. 在中间环境进行QA
3. 修复步骤二中发现的 bug，然后重新执行步骤二。
4. 如果你对中间环境部署的应用满意了，可以部署到生产环境。
5. 在生产环节进行 QA.

步骤二和步骤五可能会非常耗时，特别是当你想让应用保持高质量的时候。这就是为什么我们会需要开发一套验收测试 (阅读[测试文章](XXX) 了解更多)。更进一步说，你可以在每次发布新版本时在临时服务器进行负载/压力测试。

<h3 id="continuous-deployment">持续性部署</h3>

持续性部署是指在一定条件下，使用持续集成工具部署应用的过程 (就像使用 git push 命令推送到 `master` 分支)。你可以使用 CD 命令部署到 Galaxy,阅读[博客文章的主题](https://medium.com/@natestrauser/migrating-meteor-apps-from-modulus-to-galaxy-with-continuous-deployment-from-codeship-aed2044cabd9#.lvio4sh4a)了解更多。

<h3 id="rolling-updates-and-data">滚动部署和数据版本</h3>

理解部署中发生了什么事是很重要的，特别是当部署涉及到数据格式改变 (和潜在的数据迁移，查看[集合章节](collections.html#migrations))。

当在多个服务器或容器上运行你的应用，最好不要一次性关闭所有服务器然后再重新打开备份。这样做停机时间会更长，而且当所有用户再次同时连接时会导致 CPU 使用率急剧上升。为了缓和这种情况，Galaxy 在部署的时候会将一个容器停止和重启，然后进行下一个容器的停止和重启。会有一段时间一些容器在旧版本上运行，一些容器在新版本上运行，用户逐渐从旧版本切换到新版本。

<img src="images/galaxy-deploying.png">

如果新版本在数据库中包含不同的数据格式，那么需要多加小心，通过如何加强版本以确保所有同时运行的版本可以一起工作。了解应该如何操作请阅读[集合章节](collections.html#migrations)。

<h2 id="analytics">通过统计工具监控用户</h2>

我们经常想知道的是应用程序的哪个页面访问量最大，访客的来源。通过简单的设置可以使用 Google Analytics 跟踪 URL.我们会使用这个包[`okgrow:analytics`](https://atmospherejs.com/okgrow/analytics)

```
meteor add okgrow:analytics
```
现在我们需要使用 Google Analytics key 配置包 (该包除了支持 Google Analytics, 也支持其他类似工具，查看[Atmosphere 文档](https://atmospherejs.com/okgrow/analytics)). 作为 [_Meteor settings_](#environment) 的一部分进行传递：

```js
{
  "public": {
    "analyticsSettings": {
      // Add your analytics tracking id's here
      "Google Analytics" : {"trackingId": "Your tracking ID"}
    }
  }
}
```

这个统计包 hook 进 Flow Router (阅读[路由章节](routing.html)了解更多) 并记录所有的页面事件。

你可能还需要记录页面没有发生变化的事件 (例如发布订阅，或者调用方法)，要实现这个功能可以改变事件追踪函数：

```js
export const updateText = new ValidatedMethod({
  ...
  run({ todoId, newText }) {
    // We use `isClient` here because we only want to track
    // attempted method calls from the client, not server to
    // server method calls
    if (Meteor.isClient) {
      analytics.track('todos.updateText', { todoId, newText });
    }

    // ...
  }
});
```

为了实现 subscriptions/publications 的抽象化，你可能会需要为 `Meteor.subscribe()` 写一个简单的封装包。

<h2 id="apm">监控应用</h2>

在生产环境中运行应用时，确保性能良好和保证顺利运行是非常重要的。

<h3 id="meteor-performance">了解 Meteor 性能</h3>

尽管有很多工具可以用于监控应用的 HTTP 请求 —— 回应性能，但是对于 Meteor 应用来说不一定有用。缓慢的 HTTP 回应对一个应用来说确实是一个问题，所有使用类似[Pingdom](https://www.pingdom.com)的根据可以解决一些问题，但你的应用可能还存在其他问题是这些工具无法解决的。

<h3 id="galaxy-apm">使用 Galaxy 监控</h3>

[Galaxy](#galaxy) 提供 turnkey Meteor 托管和其他一些非常有用的工具用于 debug. 如果你的设置是用于处理当前负载(或者你需要更多容器)，决定设置的关键将是 CPU，内存负载图和连接用户数，或者一些不相称的用户过载行为 (看起来不相关)：

<img src="images/galaxy-metrics.png">

Galaxy's UI提供了详细的登录系统，这在决定哪种行为造成了过高负载，或者用于应用的 debug 是非常有用的：

<img src="images/galaxy-logs.png">

<h3 id="kadira">Kadira</h3>

如果你真的想知道运行 Meteor 应用的来龙去脉，可以尝试[Kadira](https://kadira.io)。Kadira 是专为 Meteor 建造的全功能应用性能监控 (APM)  解决方案。Kadira 在应用运行时会监控和观察客户端和服务器端，然后把发生的行为发送给主服务器。

当访问 Kadira 应用时，可以从多个指标观察你的应用过去和现在的行为。Kadira's [文档](https://kadira.io/platform/kadira-apm/overview)非常全面非常有价值，我们在这里只讨论几点。

<h4 id="kadira-method-pub">Method 和 Publication 延迟</h4>

与其监控 HTTP 响应时间，在 Meteor 应用中监控 DDP 响应时间会更有价值。根据 DDP,客户端会等待的行为包括 *method calls* 和 *publication subscriptions*. Kadira 提供根据帮助你发现哪些 methods 和 publications 非常慢以及占用资源。

<img src="images/kadira-method-latency.png">

在上面的截图中可以看到 Atmosphere 应用调用不同 method 的响应时间分解。中值是 56ms, 99% 在200ms 以内，看起来很合理，不必过分担心。

使用 "traces" section 可以发现哪些 Method 的调用执行非常缓慢： 

<img src="images/kadira-method-trace.png">

在上面的截图中我们看到一个很慢的 method 调用 (用了214ms)，我们深入研究，发现大部分时间用于等待用户连接行为 (主要是 `searches/top` 和 `counts` publications)。所有我们会考虑加速 subscriptions 的初始化时间，因为在某些情况下它会使搜索变慢。


<h4 id="kadira-livequery">现场查询监控</h4>

Meteor 的主要性能特点是通过 livequery 行为实现的，livequery 允许通过 publications 实现数据实时自动更新。为了达到这个目的，livequery 需要监控你的 MongoDB 实例来监控变化并决定这种变化是否跟给定的 publication 相关。

如果很多用户使用同一个 publication，或者有很多变化需要比较，那么 livequery 将帮忙做很多工作。所以 Kadira 提供的停机 livequery 使用情况的功能非常有用：

<img src="images/kadira-observer-usage.png">

在这张截图上我们可以看到 observers 的创建和销毁都相当稳定，少些会循环使用，但实际上它们并不会持久存在。这跟用户在 Atmosphere 上访问特定的页面， `package` publication 页面会重新加载是一样的。这张行为在外面的意料之中，所以我们也无需过多关注。

<h2 id="seo">SEO</h2>

如果你的应用包含了很多可以公开访问的内容，那么你应该希望你的应用可以在 Google 或其他搜索引擎有一个好的排名。因为很多网络爬虫不支持客户端呈现  (如果支持的话可能 websockets 也会参差不齐)，所以最好是在服务器上呈现网站然后将其作为 HTML 发送。

我们可以使用[Prerender.io](https://prerender.io)，名字为[`dfischer:prerenderio`](https://atmospherejs.com/dfischer/prerenderio)的包来实现这种目的。使用 `meteor add` 命令即可添加，如果你有额外的预渲染账号并希望进行频繁的缓存更改，可以自由设置你的预渲染口令。

如果你正在使用 [Galaxy 托管你的应用](https://www.meteor.com/galaxy/signup)，你还可以使用内置的自动[Prerender.io](https://prerender.io)集成。只需要在应用中简单添加 [`mdg:seo`](https://atmospherejs.com/mdg/seo)，Galaxy 就会知道怎么做了。

你还有可能需要添加 `<title>` 标签和其他 `<head>` 内容来实现搜索引擎优化。最好的方法是使用[`kadira:dochead`](https://atmospherejs.com/kadira/dochead)包。调用 `DocHead` 的最佳地方是页面级组件中的 `onCreated` 回调。
