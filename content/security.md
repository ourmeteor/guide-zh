---
title: "安全性"
order: 40
description: 如何保证 Meteor 应用的安全性.
discourseTopicId: 19667
---

阅读完本文，你将能够：

1. Meteor 应用的安全。
2. 如何保证 Meteor 方法，发布和源码的安全性。
3. 在部署和生产中往哪里存储密钥。
4. 如何按照一个安全性检查表来审核应用。

<h1 id="introduction">简介</h1>

Securing a web application is all about understanding security domains and understanding the attack surface between these domains. In a Meteor app, things are pretty simple:保证一个应用的安全就是理解安全域和安全域之间的攻击界面。在一个 Meteor 应用中，其实很简单：

1. 服务器上运行的代码可以被信任。
2. 其他：客户端上运行的代码，通过 Method 和 publication 参数传输的数据等，都是不能被信任的。

实际应用中，应该在这两个域之间的边界做大量的安全性和验证性的工作。简单来说：

1. 验证和检验所有来自客户端的输入。
2. 不要把任何私密信息泄露到客户端。

<h2 id="attack-surface">概念：攻击界面</h2>

大部分 Meteor 应用都是把客户端代码文件和服务器端代码文件放在一起，所以应该格外注意哪些是在客户端运行的，哪些是在服务器端运行的，两种的界限是什么。下面是 Meteor 应用安全性检查的完整列表：

1. **Methods**: 任何以 Method 参数带进来的数据都应该被验证有效性， Method 也不应该返回用户没有权限获得的数据。
2. **Publications**: 任何以 publication 参数带进来的数据都应该被验证有效性，publication 也不应该返回用户没有权限获得的数据。
3. **Served files**: 应该确保运行在客户端的源码或配置文件不能包含私密数据。

上面讲到的这些点我们下面会详细讲解：

<h3 id="allow-deny">避免使用 allow/deny</h3>

在这个教程中，我们非常不支持直接在客户端使用 [allow](http://docs.meteor.com/#/full/allow) 和 [deny](http://docs.meteor.com/#/full/deny) 查询 MongoDB 的数据。原因在上面的安全性完整列表。验证所有 MongoDB 数据库查询是非常困难的，随着 MongoDB 版本的更新，遇到的困难可能会更多。

有好几篇文章讲到在客户端执行 MongoDB 数据库更新的潜在缺陷，特别是 [Allow & Deny 安全性挑战](https://www.discovermeteor.com/blog/allow-deny-security-challenge/) 和它的 [results](https://www.discovermeteor.com/blog/allow-deny-challenge-results/)，可以在 Discover Meteor 博客上找到。

按照上面讲到的，我们推荐所有的 Meteor 应用都应该使用 Method 操作来自客户端的数据，并严格限制每个 Method 的参数。

下面的代码片段放在服务器端拒绝所有客户端对数据集的更新。这可以保证应用中也不可以使用 `allow`：

```js
// 拒绝所有客户端对数据集的更新
Lists.deny({
  insert() { return true; },
  update() { return true; },
  remove() { return true; },
});
```


<h2 id="methods">Methods</h2>

Method 是 Meteor 应用从外部接收输入和数据的方式，所以从应用安全性来讲是非常重要的。如果没有保证 Method 的安全性，用户可能会通过你不希望的方式修改数据库——修改其他用户的文件，删除数据，或者搞乱数据库进而搞垮你的应用。

<h3 id="validate-arguments">验证所有的参数</h3>

如果输入是正确的，那么写简洁欸的代码就会更容易，所有在运行任何代码前验证所有 Method 的参数是很重要的。你不会希望用户输入不正确的数据类型进而导致程序崩溃。

假设你正在写 Method 的单元测试，你需要检查所有可能的 Method 数据输入；验证参数可以把需要做单元测试的输入限定在一个范围内，减少代码量。自我记录也是很好的一个习惯；其他开发者可以通过查看你的代码了解 Method 所要求的参数类型。

下面这个例子可以说明不验证参数的话，可能会带来灾难性后果：

```js
Meteor.methods({
  removeWidget(id) {
    if (! this.userId) {
      throw new Meteor.Error('removeWidget.unauthorized');
    }

    Widgets.remove(id);
  }
});
```

如果用户传递一个非 ID 的选择器，如 `{}`，则整个数据库都会被删除。

<h3 id="validated-method">mdg:validated-method</h3>

为了帮助你写成全面验证参数的 Method，我们给 Method 写了一个闭包来强制要求参数验证。在[Methods 章节](methods.html#validated-method)了解如何使用。接下来我们写的代码都是在安装了这个包的基础上。如果没有，这些原则也适用只是写出来的代码会有点怪。

<h3 id="user-id-client">不要从客户端传递 userId</h3>

Meteor Method 的 `this` 语境可以包含一些当前连接的有用信息，最有用的是[`this.userId`](http://docs.meteor.com/#/full/method_userId)。这个属性是 DDP 登录系统在管理，由框架本身保证其安全性。

当前用户的用户 ID 可以根据 `this` 获取，所以不应该将当前用户的 ID 作为参数传递给 Method。因为这将使得任何客户端可以传递任何用户 ID。我们看下面这个例子：

```js
// #1: 错误！客户端可以传递任何用户的 ID 并更改其姓名。
setName({ userId, newName }) {
  Meteor.users.update(userId, {
    $set: { name: newName }
  });
}

// #2: 正确，客户端只可以设置当前登录用户的名字。
setName({ newName }) {
  Meteor.users.update(this.userId, {
    $set: { name: newName }
  });
}
```

需要传递任何用户 ID 作为参数的只有下面几种情况：

1. 只有管理员才可以接触到的 Method，用于编辑其他用户资料。请查看[用户角色](accounts.html##roles-and-permissions)章节了解如何判断用户的角色和权限。
2. 不用于修改其他用户的 Method，而是作为目标；例如，该 Methood 可以用于发送私信，或者添加其他用户为好友。

<h3 id="specific-action">每个行为用一个 Method</h3>

确保应用安全最好的办法就是理解所有的输入都可能来自不安全的渠道，所以要处理好这些输入。要了解什么样的输入可以来自客户端的最简单的方法就是将其限制在尽可能小的空间中。这意味着应用中的 Method 应该都是有具体行为的，而且不应该有多种选择可以改变这种行为。这样做的目标就是我们可以简单查看应用中的 Method 并且验证和测试它来确保安全性。下面是 Todos 应用中一个安全的 Method：

```js
export const makePrivate = new ValidatedMethod({
  name: 'lists.makePrivate',
  validate: new SimpleSchema({
    listId: { type: String }
  }).validator(),
  run({ listId }) {
    if (!this.userId) {
      throw new Meteor.Error('lists.makePrivate.notLoggedIn',
        'Must be logged in to make private lists.');
    }

    const list = Lists.findOne(listId);

    if (list.isLastPublicList()) {
      throw new Meteor.Error('lists.makePrivate.lastPublicList',
        'Cannot make the last public list private.');
    }

    Lists.update(listId, {
      $set: { userId: this.userId }
    });

    Lists.userIdDenormalizer.set(listId, this.userId);
  }
});
```

你可以看到这个 Method 在做一件很具体的事——将单个列表标记为私有。还可以使用另外一个 Method `setPrivacy`，也可以设置列表为公开或私有，但是在这个应用中对于 `makePrivate` 和 `makePublic` 的安全性考虑是很不同的。通过将操作分为不同的 Method，会更加简单清楚。从上面 Method 的定义可以清楚知道我们接受的参数，如何检测安全性，以及对于数据库的操作。

但是，这并不是 Method 的灵活性很差。我们来看一个例子：

```js
const Meteor.users.methods.setUserData = new ValidatedMethod({
  name: 'Meteor.users.methods.setUserData',
  validate: new SimpleSchema({
    fullName: { type: String, optional: true },
    dateOfBirth: { type: Date, optional: true },
  }).validator(),
  run(fieldsToSet) {
    Meteor.users.update(this.userId, {
      $set: fieldsToSet
    });
  }
});
```

上面的是一个很好的 Method 因为你可以掌控灵活性，可以有一些可选择的域但是只传递希望改变的域。这个方法之所以可行是因为设置用户的全名和生日的安全性考虑是一样的——我们不需要对不同的域设置不同的安全验证。请注意在 MongoDB 执行 `$set` 查询是在服务器产生的——我们不应该在客户端执行同样的 MongoDB 查询，因为这样很难验证，而且很可能会带来副作用。

<h4 id="reusing-security-rules">重构以重新使用安全性规则</h4>

在应用中可能会遇到多个 Method 使用同样的安全性验证。这可以通过将安全性验证代码分离出来成为一个模块，封装 Method 本身，或者拓展 `Mongo.Collection` 类，然后在服务器端的 `insert`, `update` 和 `remove` 里面检测。但是，通过特定的 Method 实施客户端的沟通会比从客户端发送随意的 `update` 操作要好，因为恶意用户不能发送未经检查的 `update` 操作。

<h3 id="rate-limiting">速率限制</h3>

跟 REST 端点一样，Meteor Methods 可以在任何地方调用——一个恶毒的项目，浏览器控制台的脚本等。这可以很容易在短时间内启动多个 Method 调用。这意味着黑客可以很轻易地测试所有的输入，然后找到关键输入。Meteor 的密码登录有内置的速率限制来防止暴力破解，但是你可以为你的其他方法设置速率限制。

在 Todos 应用中，我们使用下面的代码给所有的 Method 设置一个基础速率限制：

```js
// 获取列表的所有 method 名称
const LISTS_METHODS = _.pluck([
  insert,
  makePublic,
  makePrivate,
  updateName,
  remove,
], 'name');

// 每个链接每秒钟只允许 5 个列表操作
DDPRateLimiter.addRule({
  name(name) {
    return _.contains(LISTS_METHODS, name);
  },

  // 每个链接 ID 的速率限制
  connectionId() { return true; }
}, 5, 1000);
```

这将使得每个 Method 每秒钟每个链接只能调用 5 次。用户不应该注意到这种速率限制，但是可以阻止恶意脚本请求充斥服务器。你可以根据你的应用需要调整速率限制参数。

<h2 id="publications">Publications</h2>

Publications 是 Meteor 服务器提供数据给客户端的主要方式。虽然使用 Method 主要考虑是确保用户不能按照我们不希望的方式修改数据库，但是使用 publication 的主要原因是过滤返回的数据，这样恶意用户就不能看到我们不希望用户看到的数据。

#### 你不能在渲染层确保安全性

In a server-side-rendered framework like Ruby on Rails, it's sufficient to simply not display sensitive data in the returned HTML response. In Meteor, since the rendering is done on the client, an `if` statement in your HTML template is not secure; you need to do security at the data level to make sure that data is never sent in the first place. 在 Ruby on Rails 这种在服务器层渲染的价格，

<h3 id="method-rules">有关 Method 的规则仍然适用</h3>

上面讲 Method 时提到的点对 publication 也适用：

1. Validate all arguments using `check` or `aldeed:simple-schema`.
1. Never pass the current user ID as an argument.
1. Don't take generic arguments; make sure you know exactly what your publication is getting from the client.
1. Use rate limiting to stop people from spamming you with subscriptions.

<h3 id="fields">总是严格限制域</h3>

[`Mongo.Collection#find` has an option called `fields`](http://docs.meteor.com/#/full/find) which lets you filter the fields on the fetched documents. You should always use this in publications to make sure you don't accidentally publish secret fields.

For example, you could write a publication, then later add a secret field to the published collection. Now, the publication would be sending that secret to the client. If you filter the fields on every publication when you first write it, then adding another field won't automatically publish it.

```js
// #1: Bad! If we add a secret field to Lists later, the client
// will see it
Meteor.publish('lists.public', function () {
  return Lists.find({userId: {$exists: false}});
});

// #2: Good, if we add a secret field to Lists later, the client
// will only publish it if we add it to the list of fields
Meteor.publish('lists.public', function () {
  return Lists.find({userId: {$exists: false}}, {
    fields: {
      name: 1,
      incompleteCount: 1,
      userId: 1
    }
  });
});
```

If you find yourself repeating the fields often, it makes sense to factor out a dictionary of public fields that you can always filter by, like so:

```js
// In the file where Lists is defined
Lists.publicFields = {
  name: 1,
  incompleteCount: 1,
  userId: 1
};
```

Now your code becomes a bit simpler:

```js
Meteor.publish('lists.public', function () {
  return Lists.find({userId: {$exists: false}}, {
    fields: Lists.publicFields
  });
});
```

<h3 id="publications-user-id">Publications and userId</h3>

The data returned from publications will often be dependent on the currently logged in user, and perhaps some properties about that user - whether they are an admin, whether they own a certain document, etc.

Publications are not reactive, and they only re-run when the currently logged in `userId` changes, which can be accessed through `this.userId`. Because of this, it's easy to accidentally write a publication that is secure when it first runs, but doesn't respond to changes in the app environment. Let's look at an example:

```js
// #1: Bad! If the owner of the list changes, the old owner will still see it
Meteor.publish('list', function (listId) {
  check(listId, String);

  const list = Lists.findOne(listId);

  if (list.userId !== this.userId) {
    throw new Meteor.Error('list.unauthorized',
      'This list doesn\'t belong to you.');
  }

  return Lists.find(listId, {
    fields: {
      name: 1,
      incompleteCount: 1,
      userId: 1
    }
  });
});

// #2: Good! When the owner of the list changes, the old owner won't see it anymore
Meteor.publish('list', function (listId) {
  check(listId, String);

  return Lists.find({
    _id: listId,
    userId: this.userId
  }, {
    fields: {
      name: 1,
      incompleteCount: 1,
      userId: 1
    }
  });
});
```

In the first example, if the `userId` property on the selected list changes, the query in the publication will still return the data, since the security check in the beginning will not re-run. In the second example, we have fixed this by putting the security check in the returned query itself.

Unfortunately, not all publications are as simple to secure as the example above. For more tips on how to use `reywood:publish-composite` to handle reactive changes in publications, see the [data loading article](data-loading.html#complex-auth).

<h3 id="publication-options">Passing options</h3>

For certain applications, for example pagination, you'll want to pass options into the publication to control things like how many documents should be sent to the client. There are some extra considerations to keep in mind for this particular case.

1. **Passing a limit**: In the case where you are passing the `limit` option of the query from the client, make sure to set a maximum limit. Otherwise, a malicious client could request too many documents at once, which could raise performance issues.
2. **Passing in a filter**: If you want to pass fields to filter on because you don't want all of the data, for example in the case of a search query, make sure to use MongoDB `$and` to intersect the filter coming from the client with the documents that client should be allowed to see. Also, you should whitelist the keys that the client can use to filter - if the client can filter on secret data, it can run a search to find out what that data is.
3. **Passing in fields**: If you want the client to be able to decide which fields of the collection should be fetched, make sure to intersect that with the fields that client is allowed to see, so that you don't accidentally send secret data to the client.

In summary, you should make sure that any options passed from the client to a publication can only restrict the data being requested, rather than extending it.

<h2 id="served-files">Served files</h2>

Publications are not the only place the client gets data from the server. The set of source code files and static assets that are served by your application server could also potentially contain sensitive data:

1. Business logic an attacker could analyze to find weak points.
1. Secret algorithms that a competitor could steal.
1. Secret API keys.

<h3 id="secret-code">Secret server code</h3>

While the client-side code of your application is necessarily accessible by the browser, every application will have some secret code on the server that you don't want to share with the world.

Secret business logic in your app should be located in code that is only loaded on the server. This means it is in a `server/` directory of your app, in a package that is only included on the server, or in a file inside a package that was loaded only on the server.

If you have a Meteor Method in your app that has secret business logic, you might want to split the Method into two functions - the optimistic UI part that will run on the client, and the secret part that runs on the server. Most of the time, putting the entire Method on the server doesn't result in the best user experience. Let's look at an example, where you have a secret algorithm for calculating someone's MMR (ranking) in a game:

```js
// In a server-only file
MMR = {
  updateWithSecretAlgorithm(userId) {
    // your secret code here
  }
}
```

```js
// In a file loaded on client and server
const Meteor.users.methods.updateMMR = new ValidatedMethod({
  name: 'Meteor.users.methods.updateMMR',
  validate: null,
  run() {
    if (this.isSimulation) {
      // Simulation code for the client (optional)
    } else {
      MMR.updateWithSecretAlgorithm(this.userId);
    }
  }
});
```

Note that while the Method is defined on the client, the actual secret logic is only accessible from the server. Keep in mind that code inside `if (Meteor.isServer)` blocks is still sent to the client, it is just not executed. So don't put any secret code in there.

Secret API keys should never be stored in your source code at all, the next section will talk about how to handle them.

<h2 id="api-keys">Securing API keys</h2>

Every app will have some secret API keys or passwords:

1. Your database password.
1. API keys for external APIs.

These should never be stored as part of your app's source code in version control, because developers might copy code around to unexpected places and forget that it contains secret keys. You can keep your keys separately in [Dropbox](https://www.dropbox.com/), [LastPass](https://lastpass.com), or another service, and then reference them when you need to deploy the app.

You can pass settings to your app through a _settings file_ or an _environment variable_. Most of your app settings should be in JSON files that you pass in when starting your app. You can start your app with a settings file by passing the `--settings` flag:

```sh
# Pass development settings when running your app locally
meteor --settings development.json

# Pass production settings when deploying your app to Galaxy
meteor deploy myapp.com --settings production.json
```

Here's what a settings file with some API keys might look like:

```js
{
  "facebook": {
    "clientId": "12345",
    "secret": "1234567"
  }
}
```

In your app's JavaScript code, these settings can be accessed from the variable `Meteor.settings`.

[Read more about managing keys and settings in the Deployment article.](deployment.html#environment)

<h3 id="client-settings">Settings on the client</h3>

In most normal situations, API keys from your settings file will only be used by the server, and by default the data passed in through `--settings` is only available on the server. However, if you put data under a special key called `public`, it will be available on the client. You might want to do this if, for example, you need to make an API call from the client and are OK with users knowing that key. Public settings will be available on the client under `Meteor.settings.public`.

<h3 id="api-keys-oauth">API keys for OAuth</h3>

For the `accounts-facebook` package to pick up these keys, you need to add them to the service configuration collection in the database. Here's how you do that:

First, add the `service-configuration` package:

```sh
meteor add service-configuration
```

Then, upsert into the `ServiceConfiguration` collection:

```js
ServiceConfiguration.configurations.upsert({
  service: "facebook"
}, {
  $set: {
    clientId: Meteor.settings.facebook.clientId,
    loginStyle: "popup",
    secret: Meteor.settings.facebook.secret
  }
});
```

Now, `accounts-facebook` will be able to find that API key and Facebook login will work properly.

<h2 id="ssl">SSL</h2>

This is a very short section, but it deserves its own place in the table of contents.

**Every production Meteor app that handles user data should run with SSL.**

For the uninitiated, this means all of your HTTP requests should go over HTTPS, and all websocket data should be sent over WSS.

Yes, Meteor does hash your password or login token on the client before sending it over the wire, but that only prevents an attacker from figuring out your password - it doesn't prevent them from logging in as you, since they could just send the hashed password to the server to log in! No matter how you slice it, logging in requires the client to send sensitive data  to the server, and the only way to secure that transfer is by using SSL. Note that the same issue is present when using cookies for authentication in a normal HTTP web application, so any app that needs to reliably identify users should be running on SSL.

You can ensure that any unsecured connection to your app redirects to a secure connection by adding the `force-ssl` package.

#### Setting up SSL

1. On [Galaxy](deployment.html#galaxy), most things are set up for you, but you need to add a certificate. [See the help article about SSL on Galaxy](https://galaxy.meteor.com/help/using-ssl).
2. If you are running on your own [infrastructure](deployment.html#custom-deployment), there are a few options for setting up SSL, mostly through configuring a proxy web server. See the articles: [Josh Owens on SSL and Meteor](http://joshowens.me/ssl-and-meteor-js/), [SSL on Meteorpedia](http://www.meteorpedia.com/read/SSL), and [Digital Ocean tutorial with an Nginx config](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-meteor-js-application-on-ubuntu-14-04-with-nginx).

<h2 id="checklist">Security checklist</h2>

This is a collection of points to check about your app that might catch common errors. However, it's not an exhaustive list yet---if we missed something, please let us know or file a pull request!

1. Make sure your app doesn't have the `insecure` or `autopublish` packages.
1. Validate all Method and publication arguments, and include the `audit-argument-checks` to check this automatically.
1. [Deny writes to the `profile` field on user documents.](accounts.html#dont-use-profile)
1. [Use Methods instead of client-side insert/update/remove and allow/deny.](security.html#allow-deny)
1. Use specific selectors and [filter fields](http://guide.meteor.com/security.html#fields) in publications.
1. Don't use [raw HTML inclusion in Blaze](blaze.html#rendering-html) unless you really know what you are doing.
1. [Make sure secret API keys and passwords aren't in your source code.](security.html#api-keys)
1. Secure the data, not the UI - redirecting away from a client-side route does nothing for security, it's just a nice UX feature.
1. [Don't ever trust user IDs passed from the client.](http://guide.meteor.com/security.html#user-id-client) Use `this.userId` inside Methods and publications.
1. Set up [browser policy](https://atmospherejs.com/meteor/browser-policy), but know that not all browsers support it so it just provides an extra layer of security to users with modern browsers.
