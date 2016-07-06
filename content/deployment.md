---
title: 部署和监控
order: 41
description: 如何部署，运行，监控 Meteor 产品应用。
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

你建立了一个 Meteor 应用，并且通过了测试，现在是时候把它放到网上让全世界人民欣赏了。部署一个 Meteor 应用跟部署其他 websocket-based Node.js 应用类似，但在某些方面存在不同。

部署一个应用跟大部分的发行软件有本质上的区别，我们可以多次部署。不需要等待用户换取软件的新版本，因为服务器会把更新的信息推送给用户。

使用优秀的质量保证(QA)流程对更改进行测试仍然是非常重要的。即使修复 bugs 很容易，这些错误仍然可能给客户造成严重困扰，并可能导致数据丢失！

<h3 id="environments">部署环境</h3>

在 web 应用部署中最常见的三种运行环境：

1. **Development.** 开发环境，这是指你开发新功能和运行本地测试的机器。
2. **Staging.** 中间环境，类似于 Production，但对应用的用户不可见。可以用于测试和 QA.
3. **Production.** 生产环节，用户使用中的 app 的真实部署环境。

中间环境主要是为了提供一个用户不可见的测试环境，在框架上跟真实部署环境无限接近。新代码在开发环境中不会出现问题，但到了真实生产环节中就出问题了。以客户端和服务器端之间的延迟问题为例 —— 连接到本地开发服务器，这么小的延迟你可能都不会将它当作一个问题。

因为这个原因，开发者会尽量让中间环节接近实际的生产环节。这意味着我们下面提到的在生产环节部署的步骤，也非常适用于中间环境。

<h3 id="environment">环境变量和设置</h3>

代码之外有两种方式配置你的应用：

1. **Environment variables.** 环境变量，这是运行过程中需要设置的一系列 `ENV_VARS`.
2. **Settings.** 设置，位于 JSON 对象中，通过 Meteor 命令 `--settings` 设置或字符串化到 `METEOR_SETTINGS` 环境变量。

设置用于设置环境(中间环境或生产环境)中具体事项，例如用于连接谷歌的访问标志和密钥。在特定环境中运行应用时这些设置不会改变

环境变量用于设置应用进程事项，可以根据应用的不同实例进行改变。例如，你可以为各个进程设置不同的 `KADIRA_OPTIONS_HOSTNAME` 来确保 [kadira](#kadira) 日志记录有用的主机名。

在存储这些设置上还有一点需要说明：不要把这些设置文件跟你的应用代码存放在同一个栈。阅读相关文章了解这些设置最好存放在什么地方 [安全性章节](security.html#api-keys).

<h2 id="other-considerations">其他考虑</h2>

将应用部署到生产主机前还有其他事项需要考虑。记住，如果可能的话请在生产环境和中间环境中进行这些操作。

<h3 id="domain-name">域名</h3>

用户通过什么 URL 访问你的网站？你可能需要向域名注册商注册一个域名，并设置 DNS 指向特定网站(这要看你如何部署，请看下文)。如果你在 Galaxy 部署，在测试应用的时候可以使用 `x.meteorapp.com` 域名。

<h3 id="ssl">SSL 认证</h3>

在 Meteor 应用中使用 SSL 总是一件好事(要了解为什么请阅读[安全性章节](security.html#ssl))。一旦你注册了一个域名，就会需要根据域名证书颁发机构生成一个 SSL 证书。

<h3 id="cdn">CDN</h3>

虽然不是严格要求，但为你的网站建立内容分布网络(CDN)会是一个好主意。CDN 是托管网站静态资产(如 JavaScript, CSS, 和 images)的服务器网络，它分布在世界上各个地方，并使用最靠近用户的服务器为用户提供这些静态资产，目的是加速访问速度。例如，如果网站的服务器在 USA 的东海岸，而你的用户在澳大利亚，CDN 可以在澳大利亚，甚至在用户所在的城市托管一份该网站 JavaScript 的副本。这将极大缩短网站的初始加载时间。

CDN 最基本的语法就是把文件上传到 CDN 并改变 URL 指向 CDN (例如你的 Meteor 应用位于 `http://myapp.com`，则将图片的 URL 从 `<img src="http://myapp.com/cats.gif">` 改为 `<img src="http://mycdn.com/cats.gif">`). 但是这对于 Meteor 来说有点困难，因为最大的文件夹 —— Javascript 组件会因为你更改应用而经常改变。

对于 Meteor 来讲，我们推荐使用内部支持的 CDN (例如 [CloudFront](http://joshowens.me/using-a-cdn-with-your-production-meteor-app/))，跟提前上传文件相比，CDN 自动从你的服务器获取文件。你把文件放在 `public/` 文件夹(在这个例子中是 `public/cats.gif`)，这样当澳大利亚的用户通过 CDN 访问 `http://mycdn.com/cats.gif` 时，CDN 会访问 `http://myapp.com/cats.gif` 获取文件然后发送给用户。虽然这比直接访问 `http://myapp.com/cats.gif` 获取文件会慢一点，但这只会发生一次，因为 CDN 会保存这些文件，接下来澳大利亚用户访问都可以快速获取文件。

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

如果你的应用中包含网络字体并通过CDN获取，您可能需要配置字体的 header，以允许跨域资源共享(因为网络字体来自于你的网站之外的其他地方)。在 Meteor 可以通过添加一个 handler 轻松做到(你必须确保 CDN 遍历 header):

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

- Select your distribution
- Behavior tab
- Select your app origin
- Edit button
- Under "Whitelist Headers", scroll down to select "Origin"
- Add button
- "Yes, Edit" button

<h2 id="deployment-options">部署选项</h2>

Meteor 是一个开源平台，跟 regular Node.js 应用一样，你可以在上面运行 Meteor 应用。如果你要自己管理基础架构，并让 Meteor 应用正确运行，让所有人都可以使用，还是一件蛮棘手的事。这也是为什么我们推荐在 Galaxy 上部署 Meteor 应用。

<h3 id="galaxy">Galaxy (推荐)</h3>

The easiest way to operate your app with confidence is to use Galaxy, the service built by Meteor Development Group specifically to run Meteor apps.

Galaxy is a distributed system that runs on Amazon AWS. If you understand what it takes to run Meteor apps correctly and just how Galaxy works, you’ll come to appreciate Galaxy’s value, and that it will save you a lot of time and trouble. Most large Meteor apps run on Galaxy today, and many of them have switched from custom solutions they used prior to Galaxy’s launch.

In order to deploy to Galaxy, you'll need to [sign up for an account](https://www.meteor.com/galaxy/signup), and separately provision a MongoDB database (see below).

Once you've done that, it's easy to [deploy to Galaxy](http://galaxy-guide.meteor.com/deploy-guide.html). You just need to [add some environment variables to your settings file](http://galaxy-guide.meteor.com/environment-variables.html) to point it at your MongoDB, and you can deploy with:

```bash
DEPLOY_HOSTNAME=us-east-1.galaxy-deploy.meteor.com meteor deploy your-app.com --settings production-settings.json
```

In order for Galaxy to work with your custom domain (`your-app.com` in this case), you need to [set up your DNS to point at Galaxy](http://galaxy-guide.meteor.com/dns.html). Once you've done this, you should be able to reach your site from a browser.

You can also log into the Galaxy UI at https://galaxy.meteor.com. Once there you can manage your applications, monitor the number of connections and resource usage, view logs, and change settings.

<img src="images/galaxy-org-dashboard.png">

If you are following [our advice](security.html#ssl), you'll probably want to [set up SSL](http://galaxy-guide.meteor.com/encryption.html) on your Galaxy application with the certificate and key for your domain. The key things here are to add the `force-ssl` package and to use the Galaxy UI to add your SSL certificate.

Once you are setup with Galaxy, deployment is simple (just re-run the `meteor deploy` command above), and scaling is even easier---simply log into galaxy.meteor.com, and scale instantly from there.

<img src="images/galaxy-scaling.png">

<h4 id="galaxy-mongo">一起使用的 MongoDB 托管环境</h4>

If you are using Galaxy (or need a production quality, managed MongoDB for one of the other options listed here), it's usually a good idea to use a [MongoDB hosting provider](http://galaxy-guide.meteor.com/mongodb.html). There are a variety of options out there, but a good choice is [mLab](https://mlab.com/). The main things to look for are support for oplog tailing, and a presence in the us-east-1 or eu-west-1 AWS region.

<h3 id="mup">Meteor Up</h3>

[Meteor Up X](https://github.com/arunoda/meteor-up/tree/mupx), often referred to as "mupx", is an open source tool that can be used to deploy Meteor application to any online server over SSH. Mup handles some of the essential deployment requirements, but you will still need to do a lot of work to get your load balancing and version updates working smoothly - it's essentially a way to automate the manual steps of using `meteor build` and putting that bundle on your server.

You can obtain a server running Ubuntu or Debian from many generic hosting providers. Mup can SSH into your server with the keys you provide in the config. You can also [watch this video](https://www.youtube.com/watch?v=WLGdXtZMmiI) for a more complete walkthrough on how to do it.

<h3 id="custom-deployment">定制化部署</h3>

If you want to figure out your hosting solution completely from scratch, the Meteor tool has a command `meteor build` that creates a deployment bundle that contains a plain Node.js application. Any npm dependencies must be installed before issuing the `meteor build` command to be included in the bundle. You can host this application wherever you like and there are many options in terms of how you set it up and configure it.

**NOTE** it's important that you build your bundle for the correct architecture. If you are building on your development machine, there's a good chance you are deploying to a different server architecture. You'll want to specify the correct architecture with `--architecture`:

```bash
# for example if deploying to a Ubuntu linux server:
npm install --production
meteor build /path/to/build --architecture os.linux.x86_64
```

To run this application, you need to provide Node.js 0.10.x and a MongoDB server. The current release of Meteor has been tested with Node 0.10.43. You can then run the application by invoking `node`, a ROOT_URL, and the MongoDB endpoint.

```bash
cd my_directory
(cd programs/server && npm install)
MONGO_URL=mongodb://localhost:27017/myapp ROOT_URL=http://my-app.com node main.js
```

However, unless you have a specific need to roll your own hosting environment, the other options here are definitely easier, and probably make for a better setup than doing everything from scratch. Operating a Meteor app in a way that it works correctly for everyone can be complex, and [Galaxy](#galaxy) handles a lot of the specifics like routing clients to the right containers and handling coordinated version updates for you.

<h2 id="process">部署步骤</h2>

Although it's much easier to deploy a web application than release most other types of software, that doesn't mean you should be cavalier with your deployment. It's important to properly QA and test your releases before you push them live, to ensure that users don't have a bad experience, or even worse, data get corrupted.

It's a good idea to have a release process that you follow in releasing your application. Typically that process looks something like:

1. Deploy the new version of the application to your staging server.
2. QA the application on the staging server.
3. Fix any bugs found in step 2. and repeat.
4. Once you are satisfied with the staging release, release the *exact same* version to production.
5. Run final QA on production.

Steps 2. and 5. can be quite time-consuming, especially if you are aiming to maintain a high level of quality in your application. That's why it's a great idea to develop a suite of acceptance tests (see our [Testing Article](XXX) for more on this). To take things even further, you could run a load/stress test against your staging server on every release.

<h3 id="continuous-deployment">持续性部署</h3>

Continuous deployment refers to the process of deploying an application via a continuous integration tool, usually when some condition is reached (such as a git push to the `master` branch). You can use CD to deploy to Galaxy, as Nate Strauser explains in a [blog post on the subject](https://medium.com/@natestrauser/migrating-meteor-apps-from-modulus-to-galaxy-with-continuous-deployment-from-codeship-aed2044cabd9#.lvio4sh4a).

<h3 id="rolling-updates-and-data">滚动部署和数据版本</h3>

It's important to understand what happens during a deployment, especially if your deployment involves changes in data format (and potentially data migrations, see the [Collections Article](collections.html#migrations)).

When you are running your app on multiple servers or containers, it's not a good idea to shut down all of the servers at once and then start them all back up again. This will result in more downtime than necessary, and will cause a huge spike in CPU usage when all of your clients reconnect again at the same time. To alleviate this, Galaxy stops and re-starts containers one by one during deployment. There will be a time period during which some containers are running the old version and some the new version, as users are migrated incrementally to the new version of your app.

<img src="images/galaxy-deploying.png">

If the new version involves different data formats in the database, then you need to be a little more careful about how you step through versions to ensure that all the versions that are running simultaneously can work together. You can read more about how to do this in the [collections article](collections.html#migrations).

<h2 id="analytics">通过统计工具监控用户</h2>

It's common to want to know which pages of your app are most commonly visited, and where users are coming from. Here's a simple setup that will get you URL tracking using Google Analytics. We'll be using the [`okgrow:analytics`](https://atmospherejs.com/okgrow/analytics) package.

```
meteor add okgrow:analytics
```
Now, we need to configure the package with our Google Analytics key (the package also supports a large variety of other providers, check out the [documentation on Atmosphere](https://atmospherejs.com/okgrow/analytics)). Pass it in as part of [_Meteor settings_](#environment):

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

The analytics package hooks into Flow Router (see the [routing article](routing.html) for more) and records all of the page events for you.

You may want to track non-page change related events (for instance publication subscription, or method calls) also. To do so you can use the custom event tracking functionality:

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

To achieve a similar abstraction for subscriptions/publications, you may want to write a simple wrapper for `Meteor.subscribe()`.

<h2 id="apm">监控应用</h2>

When you are running an app in production, it's vitally important that you keep tabs on the performance of your application and ensure it is running smoothly.

<h3 id="meteor-performance">了解 Meteor 性能</h3>

Although a host of tools exist to monitor the performance of HTTP, request-response based applications, the insights they give aren't necessarily useful for a connected client system like a Meteor application. Although it's true that slow HTTP response times would be a problem for your app, and so using a tool like [Pingdom](https://www.pingdom.com) can serve a purpose, there are many kinds of issues with your app that won't be surfaced by such tools.

<h3 id="galaxy-apm">使用 Galaxy 监控</h3>

[Galaxy](#galaxy) offers turnkey Meteor hosting and provides tools that are useful to debug the current and past state of your application. CPU and Memory load graphs in combination with connected user counts can be vital to determining if your setup is handling the current load (or if you need more containers), or if there's some specific user action that's causing disproportionate load (if they don't seem to be correlated):

<img src="images/galaxy-metrics.png">

Galaxy's UI provides a detailed logging system, which can be invaluable to determine which action it is causing that extra load, or to generally debug other application issues:

<img src="images/galaxy-logs.png">

<h3 id="kadira">Kadira</h3>

If you really want to understand the ins and outs of running your Meteor application, you should give [Kadira](https://kadira.io) a try. Kadira is a full featured Application Performance Monitoring (APM) solution that's built from the ground up for Meteor. Kadira operates by taking regular client and server side observations of your application's performance as it conducts various activities and reporting them back to a master server.

When you visit the Kadira application, you can view current and past behavior of your application over various useful metrics. Kadira's [documentation](https://kadira.io/platform/kadira-apm/overview) is extensive and invaluable, but we'll discuss a few key areas here.

<h4 id="kadira-method-pub">Method 和 Publication 延迟</h4>

Rather than monitoring HTTP response times, in a Meteor app it makes far more sense to consider DDP response times. The two actions your client will wait for in terms of DDP are *method calls* and *publication subscriptions*. Kadira includes tools to help you discover which of your methods and publications are slow and resource intensive.

<img src="images/kadira-method-latency.png">

In the above screenshot you can see the response time breakdown of the various methods commonly called by the Atmosphere application. The median time of 56ms and 99th percentile time of 200ms seems pretty reasonable, and doesn't seem like too much of a concern

You can also use the "traces" section to discover particular cases of the method call that are particular slow:

<img src="images/kadira-method-trace.png">

In the above screenshot we're looking at a slower example of a method call (which takes 214ms), which, when we drill in further we see is mostly taken up waiting on other actions on the user's connection (principally waiting on the `searches/top` and `counts` publications). So we could consider looking to speed up the initial time of those subscriptions as they are slowing down searches a little in some cases.


<h4 id="kadira-livequery">现场查询监控</h4>

A key performance characteristic of Meteor is driven by the behavior of livequery, the key technology that allows your publications to push changing data automatically in realtime. In order to achieve this, livequery needs to monitor your MongoDB instance for changes (by tailing the oplog) and decide if a given change is relevant for the given publication.

If the publication is used by a lot of users, or there are a lot of changes to be compared, then these livequery observers can do a lot of work. So it's immensely useful that Kadira can tell you some statistics about your livequery usage:

<img src="images/kadira-observer-usage.png">

In this screenshot we can see that observers are fairly steadily created and destroyed, with a pretty low amount of reuse over time, although in general they don't survive for all that long. This would be consistent with the fact that we are looking at the `package` publication of Atmosphere which is started everytime a user visits a particular package's page. The behavior is more or less what we would expect so we probably wouldn't be too concerned by this information.

<h2 id="seo">SEO</h2>

If your application contains a lot of publicly accessible content, then you probably want it to rank well in Google and other search engines' indexes. As most webcrawlers do not support client-side rendering (or if they do, have spotty support for websockets), it's better to render the site on the server and deliver it as HTML in this special case.

To do so, we can use the [Prerender.io](https://prerender.io) service, thanks to the [`dfischer:prerenderio`](https://atmospherejs.com/dfischer/prerenderio) package. It's a simple as `meteor add`-ing it, and optionally setting your prerender token if you have a premium prerender account and would like to enable more frequent cache changes.

If you’re using [Galaxy to host your meteor apps](https://www.meteor.com/galaxy/signup), you can also take advantage of built-in automatic [Prerender.io](https://prerender.io) integration. Simply add [`mdg:seo`](https://atmospherejs.com/mdg/seo) to your app and Galaxy will take care of the rest.

Chances are you also want to set `<title>` tags and other `<head>` content to make your site appear nicer in search results. The best way to do so is to use the [`kadira:dochead`](https://atmospherejs.com/kadira/dochead) package. The sensible place to call out to `DocHead` is from the `onCreated` callbacks of your page-level components.
