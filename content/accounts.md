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

T这个包是 Meteor 面向开发者的用户账号功能的核心，包含：

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

<h2 id="useraccounts">Customizable UI: useraccounts</h2>

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

Now that you have the access token, you need to actually make a request to the appropriate API. Here you have two options:

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

Meteor comes with a default MongoDB collection for user data. It's stored in the database under the name `users`, and is accessible in your code through `Meteor.users`. The schema of a user document in this collection will depend on which login service was used to create the account. Here's an example of a user that created their account with `accounts-password`:

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

Here's what the same user would look like if they instead logged in with Facebook:

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

Note that the schema is different when users register with different login services. There are a few things to be aware of when dealing with this collection:

1. User documents in the database have secret data like access keys and hashed passwords. When [publishing user data to the client](#publish-custom-data), be extra careful not to include anything that client shouldn't be able to see.
2. DDP, Meteor's data publication protocol, only knows how to resolve conflicts in top-level fields. This means that you can't have one publication send `services.facebook.first_name` and another send `services.facebook.locale` - one of them will win, and only one of the fields will actually be available on the client. The best way to fix this is to denormalize the data you want onto custom top-level fields, as described in the section about [custom user data](#custom-user-data).
3. The OAuth login service packages populate `profile.name`. We don't recommend using this but, if you plan to, make sure to deny client-side writes to `profile`. See the section about the [`profile` field on users](dont-use-profile).
4. When finding users by email or username, make sure to use the case-insensitive functions provided by `accounts-password`. See the [section about case-sensitivity](#case-sensitivity) for more details.

<h2 id="custom-user-data">Custom data about users</h2>

As your app gets more complex, you will invariably need to store some data about individual users, and the most natural place to put that data is in additional fields on the `Meteor.users` collection described above. In a more normalized data situation it would be a good idea to keep Meteor's user data and yours in two separate tables, but since MongoDB doesn't deal well with data associations it makes sense to just use one collection.

<h3 id="top-level-fields">Add top-level fields onto the user document</h3>

The best way to store your custom data onto the `Meteor.users` collection is to add a new uniquely-named top-level field on the user document. For example, if you wanted to add a mailing address to a user, you could do it like this:

```js
// Using address schema from schema.org
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

<h3 id="adding-fields-on-registration">Adding fields on user registration</h3>

The code above is just code that you could run on the server inside a Meteor Method to set someone's mailing address. Sometimes, you want to set a field when the user first creates their account, for example to initialize a default value or compute something from their social data. You can do this using [`Accounts.onCreateUser`](http://docs.meteor.com/#/full/accounts_oncreateuser):

```js
// Generate user initials after Facebook login
Accounts.onCreateUser((options, user) => {
  if (! user.services.facebook) {
    throw new Error('Expected login with Facebook only.');
  }

  const { first_name, last_name } = user.services.facebook;
  user.initials = first_name[0].toUpperCase() + last_name[0].toUpperCase();

  // Don't forget to return the new user object at the end!
  return user;
});
```

Note that the `user` object provided doesn't have an `_id` field yet. If you need to do something with the new user's ID inside this function, a useful trick can be to generate the ID yourself:

```js
// Generate a todo list for each new user
Accounts.onCreateUser((options, user) => {
  // Generate a user ID ourselves
  user._id = Random.id(); // Need to add the `random` package

  // Use the user ID we generated
  Lists.createListForUser(user._id);

  // Don't forget to return the new user object at the end!
  return user;
});
```

<h3 id="dont-use-profile">Don't use profile</h3>

There's a tempting existing field called `profile` that is added by default when a new user registers. This field was historically intended to be used as a scratch pad for user-specific data - maybe their image avatar, name, intro text, etc. Because of this, **the `profile` field on every user is automatically writeable by that user from the client**. It's also automatically published to the client for that particular user.

It turns out that having a field writeable by default without making that super obvious might not be the best idea. There are many stories of new Meteor developers storing fields such as `isAdmin` on `profile`... and then a malicious user can easily set that to true whenever they want, making themselves an admin. Even if you aren't concerned about this, it isn't a good idea to let malicious users store arbitrary amounts of data in your database.

Rather than dealing with the specifics of this field, it can be helpful to just ignore its existence entirely. You can safely do that as long as you deny all writes from the client:

```js
// Deny all client-side updates to user documents
Meteor.users.deny({
  update() { return true; }
});
```

Even ignoring the security implications of `profile`, it isn't a good idea to put all of your app's custom data onto one field. As discussed in the [Collections article](collections.html#schema-design), Meteor's data transfer protocol doesn't do deeply nested diffing of fields, so it's a good idea to flatten out your objects into many top-level fields on the document.

<h3 id="publish-custom-data">Publishing custom data</h3>

If you want to access the custom data you've added to the `Meteor.users` collection in your UI, you'll need to publish it to the client. Mostly, you can just follow the advice in the [Data Loading](data-loading.html#publications) and [Security](security.html#publications) articles.

The most important thing to keep in mind is that user documents are certain to contain private data about your users. In particular, the user document includes hashed password data and access keys for external APIs. This means it's critically important to [filter the fields](http://guide.meteor.com/security.html#fields) of the user document that you send to any client.

Note that in Meteor's publication and subscription system, it's totally fine to publish the same document multiple times with different fields - they will get merged internally and the client will see a consistent document with all of the fields together. So if you just added one custom field, you should just write a publication with that one field. Let's look at an example of how we might publish the `initials` field from above:

```js
Meteor.publish('Meteor.users.initials', function ({ userIds }) {
  // Validate the arguments to be what we expect
  new SimpleSchema({
    userIds: { type: [String] }
  }).validate({ userIds });

  // Select only the users that match the array of IDs passed in
  const selector = {
    _id: { $in: userIds }
  };

  // Only return one field, `initials`
  const options = {
    fields: { initials: 1 }
  };

  return Meteor.users.find(selector, options);
});
```

This publication will let the client pass an array of user IDs it's interested in, and get the initials for all of those users.

<h2 id="roles-and-permissions">Roles and permissions</h2>

One of the main reasons you might want to add a login system to your app is to have permissions for your data. For example, if you were running a forum, you would want administrators or moderators to be able to delete any post, but normal users can only delete their own. This uncovers two different types of permissions:

1. Role-based permissions
2. Per-document permissions

<h3 id="alanning-roles">alanning:roles</h3>

The most popular package for role-based permissions in Meteor is [`alanning:roles`](https://atmospherejs.com/alanning/roles). For example, here is how you would make a user into an administrator, or a moderator:

```js
// Give Alice the 'admin' role
Roles.addUsersToRoles(aliceUserId, 'admin', Roles.GLOBAL_GROUP);

// Give Bob the 'moderator' role for a particular category
Roles.addUsersToRoles(bobsUserId, 'moderator', categoryId);
```

Now, let's say you wanted to check if someone was allowed to delete a particular forum post:

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

Note that we can check for multiple roles at once, and if someone has a role in the `GLOBAL_GROUP`, they are considered as having that role in every group. In this case, the groups were by category ID, but you could use any unique identifier to make a group.

Read more in the [`alanning:roles` package documentation](https://atmospherejs.com/alanning/roles).

<h3 id="per-document-permissions">Per-document permissions</h3>

Sometimes, it doesn't make sense to abstract permissions into "groups" - you just want documents to have owners and that's it. In this case, you can use a simpler strategy using collection helpers.

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

Now, we can call this simple function to determine if a particular user is allowed to edit this list:

```js
const list = Lists.findOne(listId);

if (! list.editableBy(userId)) {
  throw new Meteor.Error('unauthorized',
    'Only list owners can edit private lists.');
}
```

Learn more about how to use collection helpers in the [Collections article](collections.html#collection-helpers).
