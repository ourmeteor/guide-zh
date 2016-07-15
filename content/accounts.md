---
title: Users and Accounts
order: 13
description: 如何在 Meteor 应用中建立用户登录系统。用户可以通过密码，Facebook, Google, GitHub 等登录。
discourseTopicId: 19664
---

阅读完本文，你将能够：

1. 应用的用户账号有哪些核心功能
1. 如何使用 accounts-ui 快速建立原型
1. 如何使用 useraccounts 众多的包建立登录 UI.
1. 如何建立一个全功能的秘密登录系统
1. 如何通过 OAuth 供应商如 Facebook 实现登录
1. 如何添加定制化数据到 Meeteor 用户集
1. 如何管理用户角色和权限

<h2 id="core-meteor">Meteor 核心特点</h2>

在外面了解 Meteor 面向用户的账户功能之前，我们先来了解一下 Meteor DDP 协议内置功能和  `accounts-base` 包。如果你的 Meteor 应用中有用户账号，这就是你应该关注的，大部分都是通过添加/删除包就行操作的。

<h3 id="userid-ddp">DDP 中的 userId</h3>

DDP 是 Meteor 的内置 pub/sub 和 RPC 协议。你可以从 [数据加载](data-loading.html) 和 [Methods](methods.html)中学习这方面的知识。除了 数据加载和 Method 的概念，DDP 还有一项内置功能——`userId`属性。`userId` 跟踪登录状态，而使用的用户 UI 包和登录服务无关。

这项内置功能意味着在 Methods 和 Publications 中我们总是可以获取 `this.userId`，在客户端也可以获取用户 ID. 了解这一点对于建立自定义的用户系统非常重要，大多数开发者并不需要担心机制问题，因为大部分时间是在跟 `accounts-base` 实现交互。

<h3 id="accounts-base">`accounts-base`</h3>

这个包是 Meteor 面向开发者的用户账号功能的核心，包含：

1. 有着标准架构的用户集，用户集通过 [`Meteor.users`](http://docs.meteor.com/#/full/meteor_users) 获取，[`Meteor.userId()`](http://docs.meteor.com/#/full/meteor_userid) 和 [`Meteor.user()`](http://docs.meteor.com/#/full/meteor_user) 代表客户端的用户的登录状态。
2. 有很多有用且通用的 Methods 可以跟踪登录状态，退出登录状态，用户验证等，访问 [Accounts section of the docs](http://docs.meteor.com/#/full/accounts_api) 获取完整列表。
3. .一个用于注册新登录处理器的 API 接口，用于集成其他账户包和账户系统，关于这个 API 接口没有正式的文件说明，可以通过[访问 MeteorHacks blog](https://meteorhacks.com/extending-meteor-accounts)获取更多信息。

通常情况下不需要添加 `accounts-base` 因为当添加类似 `accounts-password` 的包时就已经顺便添加了。但你应该知道有这回事。

<h2 id="accounts-ui">使用 `accounts-ui` 快速构建原型</h2>

通常情况下，开始设计应用的第一件事并不是构建一个相当复杂的用户系统，所以我们需要知道有哪些方法可以快速上手。这就是我们为什么要介绍 `accounts-ui`——只需要在你的应用中加一行代码，就可以建立一个用户系统。

```js
meteor add accounts-ui
```

包含于 Blaze 模板中：

```html
{{> loginButtons}}
```

然后选择一个登录提供商；它们会自动集成 `accounts-ui`：

```sh
# pick one or more of the below
meteor add accounts-password
meteor add accounts-facebook
meteor add accounts-google
meteor add accounts-github
meteor add accounts-twitter
meteor add accounts-meetup
meteor add accounts-meteor-developer
```

现在只需要打开应用，跟着配置流程走，就 OK 了——如果你完成了 [Meteor tutorial](https://www.meteor.com/tutorials/blaze/adding-user-accounts), 那你应该记得这一步。当然，在生产应用中，我们可能会需要定制化的用户界面和用户体验，你可以在接下来的教程中学到这些东西。

这是几张 `accounts-ui` 截图可以帮助理解：

<img src="images/accounts-ui.png">

<h2 id="useraccounts">自定义 UI: useraccounts</h2>

一旦建立了运行 `accounts-ui` 的应用原型，你就会想要实现更强大和更配置化的功能，这样就可以更好地把登录流程整合到应用中。[`useraccounts` 家庭包](http://useraccounts.meteor.com/)是 Meteor 最强大的一组账户管理 UI 控制包。

<h3 id="useraccounts-flexibility">使用任何路由或 UI 框架</h3>

了解 `useraccounts` 的第一步就是认识到核心账户管理逻辑跟 HTML 模板和路由包是相互独立的。这意味着你可以使用 [`useraccounts:core`](https://atmospherejs.com/useraccounts/core) 自定义登录模板。通常来说，你会需要一个登录模板包和一个登录路由包。可供选择的模板：

—— [`useraccounts:unstyled`](https://atmospherejs.com/useraccounts/unstyled)可以自定义 CSS;我们在 Todos 应用中使用这个包将登录 UI 无缝融合到应用中。
—— [Bootstrap, Semantic UI, Materialize, and more](http://useraccounts.meteor.com/)预建立的模板。这些模板不自带 CSS 框架，所以可以选择自己喜欢的 Bootstrap 包。

虽然基础的应用可能不需要路由也一样可以工作，但是在应用中集成路由是一种好习惯：

- [Flow Router](https://atmospherejs.com/useraccounts/flow-routing), [本教程中推荐的](routing.html)路由。
- [Iron Router](https://atmospherejs.com/useraccounts/iron-routing), Meteor 社区中另外一个很受欢迎的路由。

在 Todos 应用中我们成功使用  Flow Router. 接下来的一些章节将介绍如何自定义路线和模板，以更好地满足您的应用程序。

<h3 id="useraccounts-drop-in">不带路由的插入式 UI</h3>

如果你不想为登录流程配置路由，可以使用自我管理账户界面。当想要呈现账户 UI 模板时，只需要引入 `atForm` 模板即可，就像这样：

```html
{{> atForm}}
```

如果你根据[下面的章节](#useraccounts-customizing-routes)配置路由，就可以移除上面的 `atForm` 模板。

<h3 id="useraccounts-customizing-templates">自定义模板</h3>

对于一些应用来说，`useraccounts` UI 包提供的登录界面就已经足够用了，但对于大多数应用来说，自定义是必须滴。有一个简单的办法就是使用 `aldeed:template-extension` 包来实现替换模板的功能。

首先，通过查看包的源代码找出你想要替换的模板。例如，`useraccounts:unstyled` 包含的模板[在 GitHub 目录中](https://github.com/meteor-useraccounts/unstyled/tree/master/lib). 通过查找文件名和 HTML 字符串，我们可以按照自定义的目标替换 `atPwdFormBtn` 模板。我们先来看一下原始模板：

```html
<template name="atPwdFormBtn">
  <button type="submit" class="at-btn submit {{submitDisabled}}" id="at-btn">
    {{buttonText}}
  </button>
</template>
```

一旦确定了要替换哪个模板，就需要自定义一个新的模板。在这个例子中，我们要修改按钮的类定义的 CSS, 在覆盖原模板的时候需要注意以下几点：

1. 以前模板中 render helpers 的方式保持不变， 在这个例子中我们使用 `buttonText`.
2. 保留任何 `id` 属性，如 `at-btn`, 因为这些属性在事件处理中有用到。

我们新的覆盖模板长这样：

```html
<template name="override-atPwdFormBtn">
  <button type="submit" class="btn-primary" id="at-btn">
    {{buttonText}}
  </button>
</template>
```

然后，在模板之间通过 `replaces` 函数替换 `useraccounts` 中存在的模板：

```js
Template['override-atPwdFormBtn'].replaces('atPwdFormBtn');
```

<h3 id="useraccounts-customizing-routes">自定义路径</h3>

除了可以控制模板，你还可以通过控制路由和 URLs访问 `useraccounts` 的不同界面。因为  Flow Router 是 Meteor 官方推荐的，所以在这个例子中我们也会使用  Flow Router.

首先，我们需要确定渲染账户模板时所配置的布局：

```js
AccountsTemplates.configure({
  defaultTemplate: 'Auth_page',
  defaultLayout: 'App_body',
  defaultContentRegion: 'main',
  defaultLayoutRegions: {}
});
```

在这个例子中，我们想要在所有的账户相关页面中使用 `App_body` 布局模板。这个模板有一个命名为 `main` 的内容区域。现在我们来配置一些路径：

```js
// Define these routes in a file loaded on both client and server
AccountsTemplates.configureRoute('signIn', {
  name: 'signin',
  path: '/signin'
});

AccountsTemplates.configureRoute('signUp', {
  name: 'join',
  path: '/join'
});

AccountsTemplates.configureRoute('forgotPwd');

AccountsTemplates.configureRoute('resetPwd', {
  name: 'resetPwd',
  path: '/reset-password'
});
```

现在我们可以很轻松地将页面链接到我们的登录界面：

```html
<div class="btns-group">
  <a href="{{pathFor 'signin'}}" class="btn-secondary">Sign In</a>
  <a href="{{pathFor 'join'}}" class="btn-secondary">Join</a>
</div>
```

注意到我们已经指定了密码重设路径。通常来说，我们需要配置 Meteor 的账户系统，在密码重置邮件中发送该路径，但是 `useraccounts:flow-routing` 已经帮我们做了这项工作。[关于如何配置电子邮件我们将在下文提到](#email-flows).

你可以在[`useraccounts:flow-routing` 文档](https://github.com/meteor-useraccounts/flow-routing#routes)获取一个完整的可使用路径列表。

<h3 id="useraccounts-further-customization">更高级的定制化</h3>

`useraccounts` 还提供除模板和路由之外的其他定制功能。阅读[`useraccounts` 教程](https://github.com/meteor-useraccounts/core/blob/master/Guide.md)了解更多的功能。

<h2 id="accounts-password">密码登录</h2>

Meteor 配有安全和全功能的密码登录系统，需要使用的话要添加包：

```sh
meteor add accounts-password
```

要知道使用这个包可以做什么，请详细阅读[`accounts-password` API 接口文档](http://docs.meteor.com/#/full/accounts_passwords).

<h3 id="requiring-username-email">需要用户名或电子邮件</h3>

> 注意：如果你正在使用 `useraccounts` 的话可以省略这一步。这一步是禁用通常的 Meteor 客户端账号创建功能，并自定义有效性。

默认情况下，`accounts-password` 提供的 `Accounts.createUser` 功能允许你使用用户名，邮件，或者两者创建账户。大多数应用都需要这两者才能创建应用，所以你肯定会希望能够检验新用户创建的有效性：

```js
// 验证每个用户都提供邮件的代码应该放在服务器端
Accounts.validateNewUser((user) => {
  new SimpleSchema({
    _id: { type: String },
    emails: { type: Array },
    'emails.$': { type: Object },
    'emails.$.address': { type: String },
    'emails.$.verified': { type: Boolean },
    createdAt: { type: Date },
    services: { type: Object, blackbox: true }
  }).validate(user);

  // 返回 true 以允许创建新用户
  return true;
});
```

<h3 id="multiple-emails">多个邮件地址</h3>

大多数情况下，用户可能会需要关联多个邮件地址到同一个账户。`accounts-password` 通过将邮件地址储存在数组中来解决这个问题。然后用一些方便的 API 接口来处理[添加](http://docs.meteor.com/#/full/Accounts-addEmail), [移除](http://docs.meteor.com/#/full/Accounts-removeEmail), 和[验证](http://docs.meteor.com/#/full/accounts_verifyemail)邮件地址。

应用中一个有用的功能就是主邮件地址的概念。如果用户添加了多个邮件地址，可以确定将验证邮件发送到主邮件地址。

<h3 id="case-sensitivity">区分大小写</h3>

Meteor 1.2 之前，数据库中所有的邮件地址和用户名都是区分大小写的。也就是说，如果你使用 `AdaLovelace@example.com` 注册一个用户，然后使用 `adalovelace@example.com` 登录，会出现该用户不存在的错误信息。这看起来很怪，所以 Meteor 在 @1.2 版本做了改进。但事情远没有想象中简单，因为 MongoDB 没有区分大小写索引的概念，在数据库层面保证邮件地址的唯一性是不可能的。因为这个原因，我们就在应用层面设置一些 API 接口，用于查询和更新用户信息，并用于解决区分大小写的问题。

<h4 id="case-sensitivity-in-my-app">这对我的应用来说意味着什么？</h4>

只需要遵循一个简单的原则：不要直接通过 `username` 和 `email` 在数据库中查询信息。而应该使用 Meteor 提供的 [`Accounts.findUserByUsername`](http://docs.meteor.com/#/full/Accounts-findUserByUsername) 和 [`Accounts.findUserByEmail`](http://docs.meteor.com/#/full/Accounts-findUserByEmail). 这样的查询就是区分大小写的，我们才可以确保所查找用户的准确性。

<h3 id="email-flows">电子邮件流量</h3>

当你应用中的用户登录系统基于电子邮件，就有可能产生基于电子邮件的流量。这些工作流的共同点就是发送一个唯一的链接到用户的邮件地址，当点击链接时会触发不同事件。我们来看一下 Meteor 的 `accounts-password` 包所支持的一些事件：

1. **密码重置** 当用户点击邮箱中的这个链接时，会跳转到一个输入账户新密码的页面。
1. **用户注册** 新用户由管理员创建，但未设置密码。当用户点击邮箱中的这个链接时，会跳转到一个设置账户新密码的页面，跟密码重置非常相似。
1. **邮箱验证** 当用户点击邮箱中的这个链接时，应用会记录到这个邮箱属于正确的用户所有。

我们将讲解一下如何从头到尾手动管理全过程。

<h4 id="default-email-flow">电子邮件在外部跟账户 UI 包一起使用</h4>

进一步开发，`accounts-ui` 和 `useraccounts` 可以为你做一些基础工作。如果要自定义邮件流量的话请紧跟以下流程操作。

<h4 id="sending-email">发送电子邮件</h4>

`accounts-password` 自带可以在服务器端调用的用于发送邮件的函数。按照实际功能命名：

1. [`Accounts.sendResetPasswordEmail`](http://docs.meteor.com/#/full/accounts_sendresetpasswordemail)
2. [`Accounts.sendEnrollmentEmail`](http://docs.meteor.com/#/full/accounts_sendenrollmentemail)
3. [`Accounts.sendVerificationEmail`](http://docs.meteor.com/#/full/accounts_sendverificationemail)

邮件通过邮件模板[Accounts.emailTemplates](http://docs.meteor.com/#/full/accounts_emailtemplates)生成，包含 `Accounts.urls` 生成的链接。我们接下来会讲解更多自定义右键模板和链接的内容。

<h4 id="identifying-link-click">识别链接被点击了</h4>

当用户收到邮件并且点击了链接，浏览器会把他们带到应用的界面。现在我们需要识别这些特殊的链接并链接到特定页面。如果还没有自定义链接，你可以使用一些内置的回调来确定邮件流量可以接入你的应用。

通常情况下，当 Meteor 的客户端连接到服务器端时做的第一件事就是传递恢复登录界面的口令，以便重建之前的登录界面。但是，当这些邮件流量中的回调被触发时，不会马上传递恢复口令，要等到传递给注册回调的 `done` 函数被调用，才会传递恢复口令。这意味着如果你之前以用户 A 的身份登陆，然后你点击了用户 B 的密码重置链接，然后你又调用 `done()` 来取消密码重置流程，客户端会保持以用户 A 身份登陆。

1. [`Accounts.onResetPasswordLink`](http://docs.meteor.com/#/full/Accounts-onResetPasswordLink)
2. [`Accounts.onEnrollmentLink`](http://docs.meteor.com/#/full/Accounts-onEnrollmentLink)
3. [`Accounts.onEmailVerificationLink`](http://docs.meteor.com/#/full/Accounts-onEmailVerificationLink)

在这里我们可以看一下如何使用这些函数：

```js
Accounts.onResetPasswordLink((token, done) => {
  // 显示密码重置界面，获得新密码...

  Accounts.resetPassword(token, newPassword, (err) => {
    if (err) {
      // Display error
    } else {
      // Resume normal operation
      done();
    }
  });
})
```

如果想要一个不同的 URL 来链接到密码重置页面，可以通过 `Accounts.urls` 选项自定义：

```js
Accounts.urls.resetPassword = (token) => {
  return Meteor.absoluteUrl(`reset-password/${token}`);
};
```

如果你自定义了 URL, 需要在路由中添加路径处理该 URL, `Accounts.onResetPasswordLink` 就退役啦。

<h4 id="completing-email-flow">显示相应的用户界面，并完成该进程</h4>

现在你知道用户试图重置密码，或者设置初始密码，或者验证邮箱，应该有一个友好的界面方便用户这样操作。例如，可以设置一个页面表单用户可以输入密码。

当用户提交表单后，你需要调用合适的函数将改变提交到数据库。这里的每个函数需要用到新值以及上一步获得的口令。

1. [`Accounts.resetPasswords`] —— 该函数应该被用于重置密码和用户注册；两种口令都可以接收。
2. [`Accounts.verifyEmail`](http://docs.meteor.com/#/full/accounts_verifyemail)

当你调用了上面两个函数中的其中一个或者用户取消了进程，在链接的回调中调用 `done` 函数。也就是说，当你试图重置密码，或新用户注册，或验证邮箱时，调用 `done` 函数都会通知 Meteor 会从目前的状态跳出来。

<h3 id="customizing-emails">自定义账户邮件</h3>

你可能会想要自定义 `accounts-password` 发出的邮件模板。这可以通过[`Accounts.emailTemplates` API](http://docs.meteor.com/#/full/accounts_emailtemplates)轻松做到。下面是 Todos 应用的参考代码：

```js
Accounts.emailTemplates.siteName = "Meteor Guide Todos Example";
Accounts.emailTemplates.from = "Meteor Todos Accounts <accounts@example.com>";

Accounts.emailTemplates.resetPassword = {
  subject(user) {
    return "Reset your password on Meteor Todos";
  },
  text(user, url) {
    return `Hello!
Click the link below to reset your password on Meteor Todos.
${url}
If you didn't request this email, please ignore it.
Thanks,
The Meteor Todos team
`
  },
  html(user, url) {
    // 这里包含 HTML 邮件内容
    // 下面章节会讲到 HTML 邮件
  }
};
```

正如你所看到的，我们可以使用 ES2015 模板字符串的功能，生成一个多行字符串，其中包括密码重置 URL。我们还可以设置自定义 `from` 地址和电子邮件的主题。

<h4 id="html-emails">HTML 邮件</h4>

如果你曾经需要通过应用发送漂亮的 HTML 邮件，你应该知道这是怎样的一场噩梦。流行的电子邮件客户端对基本的 HTML 功能如 CSS 的兼容性是出了名的参差不齐，所以创作出在所有客户端都兼容的 HTML 邮件是很难的。从[响应电子邮件模板](https://github.com/leemunroe/responsive-html-email-template) 或 [框架](http://foundation.zurb.com/emails/email-templates.html)开始，然后使用工具将邮件转换为可以兼容所有邮件客户端。[Mailgun 发表的一篇文章讲解了一些 HTML 邮件的主要知识](http://blog.mailgun.com/transactional-html-email-templates/)。理论上，一个社区包可以扩展 Meteor 的构建系统并为你做邮件的编译，但在写这些教程的时候我们还没发现社区上有这种包。

<h2 id="oauth">授权登录</h2>

在很久很久以前，让用户通过 Facebook 或者 Google 登录应用是一件很令人头疼的事。幸运的是，目前大多数登录供应商提供标准的登录[授权](https://en.wikipedia.org/wiki/OAuth) API 接口，Meteor 也支持一些最流行的登录方式接口。

<h3 id="supported-login-services">Facebook, Google, and more</h3>

下面是 Meteor 精心维护的登录供应商的完整列表：

1. Facebook with `accounts-facebook`
2. Google with `accounts-google`
3. GitHub with `accounts-github`
4. Twitter with `accounts-twitter`
5. Meetup with `accounts-meetup`
6. Meteor Developer Accounts with `accounts-meteor-developer`

有一个包是专门用于连接微博登陆接口的，但现在已经没怎么有人维护了。

<h3 id="oauth-logging-in">登录</h3>

如果你使用  `accounts-ui` 或者 `useraccounts` 等现成的登录 UI 系统，则在添加好上述包后不再需要写任何代码。如果你从头开始自定义一个登录界面，可以通过[`Meteor.loginWith<Service>`](http://docs.meteor.com/#/full/meteor_loginwithexternalservice)函数实现登录功能，代码如下：

```js
Meteor.loginWithFacebook({
  requestPermissions: ['user_friends', 'public_profile', 'email']
}, (err) => {
  if (err) {
    // handle error
  } else {
    // successful login!
  }
});
```

<h3 id="oauth-configuration">配置授权</h3>

配置授权登录需要知道以下几点：

1. **客户端 ID 和密钥** 最好的方法是将授权密钥保存在源代码之外，然后通过 Meteor.settings 传递。了解如何操作请阅读[安全性章节](security.html#api-keys-oauth).
2. **URL 重定向** 对于授权提供方来说，你需要提供重定向的 URL。URL 大概长这样 `https://www.example.com/_oauth/facebook`. 将 `facebook` 换成你所使用的服务提供商的名字。注意到你需要配置两个 URL —— 一个用于你的应用，一个用于开发环境，开发环境的 URL 大概长这样 `http://localhost:3000/_oauth/facebook`.
3. **权限** 每个授权提供商都应该有文档说明提供哪些权限接口。例如，[这是 Facebook 的文档](https://developers.facebook.com/docs/facebook-login/permissions).如果你还需要所登录用户的额外信息，将 `requestPermissions` 选项中提供的某些字符串传递给 `Meteor.loginWithFacebook` 或者[`Accounts.ui.config`](http://docs.meteor.com/#/full/accounts_ui_config)。在下一章节我们会讲解如何获取额外信息。

<h3 id="oauth-calling-api">调用服务 API 获得更多数据</h3>

如果你的应用支持或者要求用户使用 Facebook 登录，我们通过 Facebook API 获取用户额外的信息，例如用户的照片列表就变得顺理成章了。

首先，需要在用户登录时请求相关权限。查看[上文](#oauth-configuration)如何传递这些选项。

然后，需要获得用户的访问口令。可以在 `Meteor.users` 数据集中的 `services` 属性找到该口令。例如，如果想要获得某一用户的 Facebook 访问口令：

```js
// 给定一个 userId，获得用户的 Facebook 访问口令
const user = Meteor.users.findOne(userId);
const fbAccessToken = user.services.facebook.accessToken;
```

下面我们会细讲用户数据库中存储的数据，以及如何获取用户数据。

现在你已经获取了访问口令，还需要向对于的 API 接口发送一个请求。有两种选择：

1. 使用[`http` 包](http://docs.meteor.com/#/full/http)直接获取服务 API 接口。你可能需要在刚开始时就传递上面的访问口令。要详细了解请阅读相关服务的 API 接口文档。 
2. 使用 tmosphere 或者 npm 上面的包，这些包会将 API 接口封装成非常漂亮的 JavaScript 界面。例如，如果需要加载来自 Facebook 的数据可以使用[fbgraph](https://www.npmjs.com/package/fbgraph) npm 包。了解如何在你的应用中使用 npm 包请阅读[系统构建章节](build-tool.html#npm)。

<h2 id="displaying-user-data">加载和显示用户数据</h2>

.Meteor 的账户系统，跟我们在 `accounts-base` 中展示的一样，同样包括数据库数据集和获取用户数据的函数功能。

<h3 id="current-user">当前登录的用户</h3>

一旦用户通过上面的任何一种方法登录到你的应用，识别是哪一个用户登录，并在注册的时候获取用户信息。

<h4 id="current-user-client">客户端：Meteor.userId()</h4>

在客户端上运行的代码，全局 `Meteor.userId()` 函数可以提供目前登录用户的 ID。

除了 `Meteor.userId()` 这种核心 API, 还有一些同样容易记住的 helpers: `Meteor.user()`，等价于 `Meteor.users.findOne(Meteor.userId())`，另外，`{% raw %}{{currentUser}}{% endraw %}` Blaze helper 返回 `Meteor.user()` 的值。

请注意，适当在应用中限制使用 “获得当前用户” 的功能可以使得一样更加模块化，测试起来更容易。要了解更多相关信息请阅读[UI 章节](ui-ux.html#global-stores)。

<h4 id="current-user-server">服务器端：this.userId</h4>

在客户端，每个连接所对应的用户都不一样，所以没有全局登录用户的概念。但是因为 Meteor 会在每次调用 Method 后跟踪环境，所以还是可以使用 全局 `Meteor.userId()`，根据调用方法的不同返回不同的值，但在使用异步代码时可能会遇到边缘情况。另外，`Meteor.userId()` 也不可以在 publications 中使用。

我们建议在 Method 和 publication 中使用 `this.userId`，并将其作为函数参数传递。

```js
// 在一个 publication 中获取 this.userId
Meteor.publish('lists.private', function() {
  if (!this.userId) {
    return this.ready();
  }

  return Lists.find({
    userId: this.userId
  }, {
    fields: Lists.publicFields
  });
});
```

```js
// 在一个 Method 中获取 this.userId
Meteor.methods({
  'todos.updateText'({ todoId, newText }) {
    new SimpleSchema({
      todoId: { type: String },
      newText: { type: String }
    }).validate({ todoId, newText }),

    const todo = Todos.findOne(todoId);

    if (!todo.editableBy(this.userId)) {
      throw new Meteor.Error('todos.updateText.unauthorized',
        'Cannot edit todos in a private list that is not yours');
    }

    Todos.update(todoId, {
      $set: { text: newText }
    });
  }
});
```

<h3 id="meteor-users-collection">Meteor.users 数据集</h3>

Meteor 带有一个默认的 MongoDB 数据库用于储存用户数据。用户数据储存在数据库 `users` 数据集中，通过 `Meteor.users` 可以获取。一个用户的信息在数据集中的结构跟注册时所使用的服务有关。下面是一个通过 `accounts-password` 创建的用户数据结构：

```js
{
  "_id": "DQnDpEag2kPevSdJY",
  "createdAt": "2015-12-10T22:34:17.610Z",
  "services": {
    "password": {
      "bcrypt": "XXX"
    },
    "resume": {
      "loginTokens": [
        {
          "when": "2015-12-10T22:34:17.615Z",
          "hashedToken": "XXX"
        }
      ]
    }
  },
  "emails": [
    {
      "address": "ada@lovelace.com",
      "verified": false
    }
  ]
}
```

下面是一个通过 Facebook 创建的用户数据结构：

```js
{
  "_id": "Ap85ac4r6Xe3paeAh",
  "createdAt": "2015-12-10T22:29:46.854Z",
  "services": {
    "facebook": {
      "accessToken": "XXX",
      "expiresAt": 1454970581716,
      "id": "XXX",
      "email": "ada@lovelace.com",
      "name": "Ada Lovelace",
      "first_name": "Ada",
      "last_name": "Lovelace",
      "link": "https://www.facebook.com/app_scoped_user_id/XXX/",
      "gender": "female",
      "locale": "en_US",
      "age_range": {
        "min": 21
      }
    },
    "resume": {
      "loginTokens": [
        {
          "when": "2015-12-10T22:29:46.858Z",
          "hashedToken": "XXX"
        }
      ]
    }
  },
  "profile": {
    "name": "Sashko Stubailo"
  }
}
```

请注意，当用户使用不同的登录服务注册时所得到用户数据结构是不一样的。当处理用户数据集的时候应该注意以下几点：

1. 数据库中的用户文件包含访问密钥和哈希密码等秘密数据。当[将数据发布到客户端时](#publish-custom-data)，要注意哪些数据是不能发布到客户端的。
2. DDP 是 Meteor 的数据发布协议，该协议只知道如何处理顶级域的冲突。这意味着你不能同时使用不同的 publication 发送 `services.facebook.first_name` 和 `services.facebook.locale` —— 只能执行其中一个 publication，客户端也只能显示一个域。解决这个问题最好的办法是 将要放到自定义顶级域的数据非标准化，如我们下文[自定义用户数据](#custom-user-data)所讲的一样。
3. 授权登录服务包会填充 `profile.name`，我们并不支持使用 `profile`，但是你一定要用，请注意一定要禁止客户端对 `profile` 的写入。了解更多请查看[用户信息中的 `profile` 域](dont-use-profile)。
4. 要通过电子邮件或者用户名找到用户，请记住使用 `accounts-password` 提供的区分大小写的查找函数。要了解更多请阅读[区分大小写的章节](#case-sensitivity)。

<h2 id="custom-user-data">自定义用户数据</h2>

当你的应用越来越复杂的时候，不可避免的需要存储单个用户的数据，最理想的存储位置就是上面所讲到的 `Meteor.users` 的其中一个域。如果要让数据更标准，最好是将 Meteor 的用户数据和应用的数据分别放在不同的表，但是因为 MongoDB 不支持表格关联，所以只能使用一个表。

<h3 id="top-level-fields">添加顶级域到用户文件</h3>

将自定义的数据存储到 `Meteor.users` 最好的办法就是在用户文件中添加一个新的顶级域，如果要给用户添加一个电子邮件地址，可以这样做：

```js
// 使用 schema.org 提供的地址结构
// https://schema.org/PostalAddress
const newMailingAddress = {
  addressCountry: 'US',
  addressLocality: 'Seattle',
  addressRegion: 'WA',
  postalCode: '98052',
  streetAddress: "20341 Whitworth Institute 405 N. Whitworth"
};

Meteor.users.update(userId, {
  $set: {
    mailingAddress: newMailingAddress
  }
});
```

<h3 id="adding-fields-on-registration">在用户注册添加新字段</h3>

上面的代码只可以在服务器端 Meteor Method 中运行，用于设置用户的 邮箱地址。当用户新建账户时，有时我们可能会需要设置一个新的域，例如初始化默认值或者计算用户的社交数据。要这样做可以采用[`Accounts.onCreateUser`](http://docs.meteor.com/#/full/accounts_oncreateuser):

```js
// 使用 Facebook 登录后初始化用户
Accounts.onCreateUser((options, user) => {
  if (! user.services.facebook) {
    throw new Error('Expected login with Facebook only.');
  }

  const { first_name, last_name } = user.services.facebook;
  user.initials = first_name[0].toUpperCase() + last_name[0].toUpperCase();

  // 最后不要忘记返回新的用户对象！
  return user;
});
```

注意到上面提到的 `user` 还没有 `_id` 域。如果在该函数中我们需要使用用户 ID，那么我们使用的小技能就是自定义一个用户 ID.

```js
// 为每一位新用户生成 todo 列表
Accounts.onCreateUser((options, user) => {
  // 自定义用户 ID
  user._id = Random.id(); // 需要添加 `random` 包

  // 使用我们自定义的用户 ID
  Lists.createListForUser(user._id);

  // 最后不要忘记返回新的用户对象！
  return user;
});
```

<h3 id="dont-use-profile">不要使用 profile</h3>

默认情况下新用户注册时会生成一个 `profile` 临时域。该域原先是为了存储用户特定的数据 —— 例如用户的图像，名字，简介等。因为这个原因，**每个用户的 `profile` 域都可以通过客户端实现更新**。该数据也会自动发布到该用户客户端。

事实证明，数据集中有一个可写入的域，但却有很多人不知道，显然不太说得过去。之前有很多 Meteor 新开发者将类似 `isAdmin` 判断条件存储在 `profile`，然后恶意用户可以将判断条件设置为 true，这样他们就是管理员了。即使这不是你的关注点，但我想你应该也不希望恶意用户将杂七杂八的数据存储到你的数据库吧。

与其花费精力处理这个域，不如直接忽略掉它。最安全的做法就是对改域禁止任何来自客户端的更新：

```js
// 禁止任何客户端更新用户文档
Meteor.users.deny({
  update() { return true; }
});
```

即使忽略 `profile` 的安全性使用，将应用中所有自定义的数据都放在一个域里面也不是一个好主意。正如我们在[数据集文章](collections.html#schema-design)中所讨论的，Meteor 的数据传输协议不做域的深度嵌套区别，所以最好将数据对象扁平化然后作为顶级域插入数据集。

<h3 id="publish-custom-data">发布自定义数据</h3>

如果要在客户端获取你添加到 `Meteor.users` 数据集的自定义数据，首先需要将数据发布到客户端。我们在文章[数据加载](data-loading.html#publications) 和 [安全性](security.html#publications)中有讲到相关知识和建议，请自行阅读。

要记住的一件至关重要的事情是用户文件肯定会包含用户的私人信息。特别是用户文件中包含哈希密码和外部 API 接口的访问密钥。这说明[过滤掉相关域](http://guide.meteor.com/security.html#fields)后再将数据发布到客户端是至关重要的一步。

在 Meteor 的 publication 和 subscription 系统中，是可以在不同的域中多次发布同一个文件的 —— 这些文件内部会进行合并，用户在客户端看到的是不同域合并在一起的最后文件。所以如果只自定义了一个域，那么就只需要为这个域写一个发布。下面我们看一下如何发布上面提到的 `initials` 域。

```js
Meteor.publish('Meteor.users.initials', function ({ userIds }) {
  // 按我们的要求验证参数有效性
  new SimpleSchema({
    userIds: { type: [String] }
  }).validate({ userIds });

  // 只选中匹配该 ID 数组的用户
  const selector = {
    _id: { $in: userIds }
  };

  // 只返回一个域 `initials`
  const options = {
    fields: { initials: 1 }
  };

  return Meteor.users.find(selector, options);
});
```

这个 publication 会让客户端获取它所感兴趣的 ID 数组内的用户，并将初始值设置为 1。

<h2 id="roles-and-permissions">角色和权限</h2>

在应用中添加登录系统的一个主要原因就是设置获取数据的权限。例如，如果你运营一个论坛，那么可能需要有管理员或主席的权限让你可以删除其他人的发布，但通常情况下用户只可以删除自己的发布。这里涉及到两种不同的权限：

1. 角色权限
2. 单个文件权限

<h3 id="alanning-roles">alanning:roles</h3>

Meteor 最热门的角色分配权限包是[`alanning:roles`](https://atmospherejs.com/alanning/roles)。下面是如何赋予一个用户管理员或主席权限的例子：

```js
//赋予 Alice 管理员的角色 
Roles.addUsersToRoles(aliceUserId, 'admin', Roles.GLOBAL_GROUP);

// 赋予 Bob 在特定类别的主席角色
Roles.addUsersToRoles(bobsUserId, 'moderator', categoryId);
```

现在可以判断某一用户是否有权限删除特定的论坛信息：

```js
const forumPost = Posts.findOne(postId);

const canDelete = Roles.userIsInRole(userId,
  ['admin', 'moderator'], forumPost.categoryId);

if (! canDelete) {
  throw new Meteor.Error('unauthorized',
    'Only admins and moderators can delete posts.');
}

Posts.remove(postId);
```

注意到现在我们可以同时判断多个角色，如果用户是 `GLOBAL_GROUP` 中的一员，那么该用户就拥有所有组的权限。在这个例子中，组是通过类别 ID 识别的，但你可以使用任何唯一的标识符划分组。

阅读[`alanning:roles` 文档](https://atmospherejs.com/alanning/roles)了解更多。

<h3 id="per-document-permissions">单个文件权限</h3>

有的时候，将权限划分到组并不合理 —— 可能只需要确保文件所有者对文件有操作功能就好了。在这种情况下，我们使用数据集 helper 就可以解决了。

```js
Lists.helpers({
  // ...
  editableBy(userId) {
    if (!this.userId) {
      return true;
    }

    return this.userId === userId;
  },
  // ...
});
```

现在我们可以调用这个简单的函数来决定一个特定的用户是否有编辑权限：

```js
const list = Lists.findOne(listId);

if (! list.editableBy(userId)) {
  throw new Meteor.Error('unauthorized',
    'Only list owners can edit private lists.');
}
```

要学习更多关于数据集 helper 的知识请阅读[数据集章节](collections.html#collection-helpers)。
