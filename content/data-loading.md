---
title: 发布和数据加载
order: 11
description: Meteor 应用如何使用发布和订阅加载数据，以及在哪里加载数据
discourseTopicId: 19661
---

阅读完本文，你将能够：

1. Meteor 平台中订阅和发布指的是什么。
2. 如何在服务器端定义一个发布。
3. 客户端如何订阅数据，以及使用哪个模板。
4. 订阅管理的有效模式
5. 如何响应式发布相关数据。
6. 如何在响应式改变中确保发布是安全的。
7. 如何使用低级发布 API  接口发布信息。
8. 当订阅一个发布的时候会发生什么。
8. 如何将第三方 REST 端点转换为一个发布。
10. 如何将你应用中的一个发布转换为 一个 REST 端点。

<h2 id="publications-and-subscriptions">发布和订阅</h2>

在传统的基于 HTTP 的 web 应用程序，客户端和服务器端是通过 “请求 - 响应” 的方式沟通的。经典的做法是客户端向服务器端发出 RESTful HTTP 请求，然后响应接收 HTML 或者 JSON 数据，但是当后端数据更新的时候服务器没有办法将数据推送到客户端。

Meteor 是建立在分布式数据协议(DDP)的基础上，允许前后端的数据交流。构建一个 Meteor 应用并不需要设置 REST 端点用于序列化和发送数据。我们通过构建 *publication* 端点将数据从服务器端推送到客户端。

**publication** 是 Meteor 用于构建可发送到客户端的数据的 API 接口。客户端初始化 **subscription** 用于连接 publication 并获取数据。subscription 初始化时获取了数据，并根据发布的数据随时更新。

所以订阅可以看作一组随时变化的数据。订阅架起了[服务器端 MongoDB 数据集](/collections.html#server-collections) 和 [客户端 Minimongo 缓存](collections.html#client-collections) 之间的桥梁。你可以把订阅想象成一条连接 “实际” 数据集子集和客户端 “缓存” 数据集的管道，并根据服务器端的数据不间断保持更新。

<h2 id="publications">定义一个发布</h2>

发布只能在服务器端文件定义。例如，在 Todos 应用中，我们对所有用户发布一系列的公开列表：

```js
Meteor.publish('lists.public', function() {
  return Lists.find({
    userId: {$exists: false}
  }, {
    fields: Lists.publicFields
  });
});
```

理解这个代码块有几个关键点。首先，我们将该发布命名为 `lists.public`，可以从客户端通过该名字获取该发布。然后，简单地从发布函数返回一个 Mongo *cursor*。注意到 cursor 通过过滤只返回数据集中指定的域，了解更多请阅读[安全性章节](security.html#fields)。

这意味着发布可以简单地确保将匹配查询的数据提供给订阅的客户端。在这个例子中，匹配查询的数据是指不带 `userId` 的列表。所以当开放订阅时，客户端名为 `Lists` 的数据集会存储所有服务器端名为 `Lists` 的数据集的公开列表。在 Todos 应用中，在应用开启时初始化订阅，并一直开放该订阅，在后面的章节我们会讨论[订阅的生命周期](data-loading.html#patterns)。

每个发布带两个变量参数：

1. `this`，包含目前 DDP 连接的信息。例如，可以通过 `this.userId` 获取当前用户的 `_id`。
2. 发布函数的参数，该参数可以在调用 `Meteor.subscribe` 时传递。

> 注意：因为我们要使用 `this` 所以应该使用 `function() {}` 而不是使用 ES2015 `() => {}`。可以通过 `eslint-disable prefer-arrow-callback` 在发布文件中禁用箭头函数。未来版本的发布 API 接口可以更好地跟 ES2015 兼容。

在这个加载私人列表的发布中，我们需要使用 `this.userId` 来获取只属于该用户的 todo 列表。

```js
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

感谢 DDP 和 Meteor 的账户系统所提供的可靠保障，上面的发布可以确保获取只属于该用户的 todo 列表。注意到用户退出登录后发布会重新加载，这意味着发布的私人 todo 列表会根据用户的改变而改变。

对于未登录的用户，我们明确调用 `this.ready()`，对订阅说我们已经将所有应该发送的数据都给你了（在这个例子中没有数据）。如果在发布中没有返回 cursor 或者没有调用 `this.ready()`，用户的订阅就会一直进行中，用户可能会一直看到一个加载中的状态。

下面是发布带有参数的例子。注意到首先检查参数的类型是否跟我们所期望的相同。

```js
Meteor.publish('todos.inList', function(listId) {
  // 检查 `listId` 看是否是我们期望的类型
  new SimpleSchema({
    listId: {type: String}
  }).validate({ listId });

  // ...
});
```

当在客户端订阅这个发布，可以在调用 `Meteor.subscribe()` 时提供 list._id：

```js
Meteor.subscribe('todos.inList', list._id);
```

<h3 id="organization-publications">组织一个发布</h3>

把发布跟功能模块放在一起是有道理的。例如，有的时候发布提供了非常具体的数据，但这些数据只对所在的开发视图有用。在这个例子中，把发布跟视图代码放在同一个模块或目录下非常有意义。

通常情况下发布应该更常见。在 Todos 应用中，我们创建了一个名为 `todos.inList` 的发布，用于发布列表中的所有 todos。尽管在应用中我们只在一个地方使用该发布（在 `Lists_show` 模板），在一个大型的应用，有很大可能我们需要在不同的地方获取列表中的所有 todos。所以把发布放在一个 `todos` 包是明智的做法。

<h2 id="subscriptions">订阅数据</h2>

要使用发布，需要在客户端创建对应的订阅。也即调用 `Meteor.subscribe()` 和发布的名字。当这样做的时候，会打开对该发布的订阅，然后服务器开始发送数据，确保客户端 “缓存” 数据集包含最新的发布数据。

`Meteor.subscribe()` 同时返回一个带有 `.ready()` 属性的 "subscription handle"。这是一个响应函数，当发布被标记为 "ready" 时（无论是你明确调用 `this.ready()` 函数，还是 cursor 的初始内容已被发送）会返回 `true`。

```js
const handle = Meteor.subscribe('lists.public');
```

<h3 id="stopping-subscriptions">停止订阅</h3>

订阅处理器还有另外一个重要的属性，就是 `.stop` 方法。当执行订阅的时候，订阅完成后要记住调用 `.stop()`。这可以确保通过订阅发送的文件从本地 Minimongo 缓存清除，服务器也会停止对该订阅的响应。如果忘记调用停止函数，会在客户端和服务器端消耗不必要的资源。

*但是*，当在响应环境（）中调用 `Meteor.subscribe()` 或者在 Blaze 组件中使用 `this.subscribe()`，Meteor 响应系统会根据情况自动调用 `this.stop()`。

<h3 id="organizing-subscriptions">UI 组件中的订阅</h3>

最好是将订阅代码尽可能跟需要这些订阅数据的模块放在一起。这可以降低 “超距作用” 和使得应用的数据流动更清晰。如果订阅和获取数据是分开的，那么当订阅改变时（例如改变函数），我们对于 cursor 内容会随着改变的认识可能不会那么清晰。

在实践中这意味着你需要将订阅的调用代码放在 *components*。在 Blaze 组件中最好在 `onCreated()` 回调中调用订阅函数：

```js
Template.Lists_show_page.onCreated(function() {
  this.getListId = () => FlowRouter.getParam('_id');

  this.autorun(() => {
    this.subscribe('todos.inList', this.getListId());
  });
});
```

在这个代码片段中我们可以看到 Blaze 模板中调用订阅函数的两个技巧：

1. 调用 `this.subscribe()` (而不是 `Meteor.subscribe`)，该函数附加一个特殊的 `subscriptionsReady()` 函数到模板实例，当模板中的订阅都准备好的时候 `subscriptionsReady()` 的值是 true。

2. 调用 `this.autorun` 设置一个响应式背景，当响应式函数 `this.getListedId()` 函数改变的时候可以重新初始化订阅。

了解更多关于 Blaze 订阅的内容请阅读[Blaze 章节](blaze.html#subscribing)，了解更多关于跟踪 UI 组件中加载状态的请阅读[UI 章节](ui-ux.html#subscription-readiness)。

<h3 id="fetching">获取数据</h3>

订阅数据并放在客户端数据集。要在用户界面使用数据，需要查询客户端的数据集。查询数据时有几点需要注意。

<h4 id="specific-queries">总是使用特定查询获取数据</h4>

如果要发布数据的一个子集，比较有诱惑力的做法通常是查询服务器端数据集的所有数据（例如 `Lists.find()`），然后在客户端查询获取我们要的那个子集，而不需要在发布数据的时候就使用 MongoDB 选择器选定该子集。

但如果这样做，当其他订阅的操作改变了同一个数据集的时候可能会发生错误，因为通过 `Lists.find()` 返回的数据可能会改变。在一个不断开发的应用中，要预料到未来的变化是比较难的，这对修复 bug 来说是一个很大的挑战。

另外，当切换订阅的时候，有一个短暂的时期会加载这两个订阅（参考下文的[改变参数后的发布行为](#publication-behavior-with-arguments)），所以当在应用中执行分页功能时，有很大可能会出现这种情况。

<h4 id="fetch-near-subscribe">在订阅的附近获取数据</h4>

原因跟我们将订阅代码尽可能跟需要这些订阅数据的模块放在一起是一样的 —— 这可以避免 “超距作用” 和使得应用的数据流动更清晰。一个常见的模式是在父级模板获取数据，然后传递个一个 “纯” 子集组件，如我们在[UI 章节](ui-ux.html#components)中看到的一样。

请注意，该准则也是有例外的。常见的就是 `Meteor.user()` —— 虽然严格来说这也是一个订阅（自动的），但是要将其作为参数通过组件层级传递给各个组件还是过于复杂了。记住不要很多地方使用因为做起组件测试会非常困难。

<h3 id="global-subscriptions">全局订阅</h3>

如果在应用中很多地方我们 *总是* 需要获取某些数据，那就不应该只在局部组件中定义一个订阅。例如，在你的应用中的很多地方都需要用户对象额外域的数据，这时就需要全局订阅。

当然，比较好的方法是使用一个布局组件（封装应用中所有组件）订阅该订阅。这种做法应该保持一致，并且将系统做得灵活一点，这样应用中也可以有组件 *不需要* 这些全局订阅的数据。 

<h2 id="patterns">数据加载模式</h2>

纵观 Meteor 应用，我们有必要知道客户端的数据加载和管理的常见模式。我们将在[UI/UX 章节](ui-ux.html)详细讲解。

<h3 id="readiness">订阅就绪</h3>

关键要明白订阅不会立即提供其订阅的数据。从客户端订阅数据到数据从服务器端的发布传递过来，期间会有延迟。你也应该意识到生产环境中的延迟可能比开发环境的更长。

虽然跟踪系统可以使得我们在构建应用时不 *需要* 过多考虑这个问题，但想要让用户体验最佳，需要知道数据说明时候准备好。

要知道数据说明时候准备好，`Meteor.subscribe()` 和 ( Blaze 组件中的 `this.subscribe()`)返回一个 包含响应式数据源 `.ready()` 的 "subscription handle"：

```js
const handle = Meteor.subscribe('lists.public');
Tracker.autorun(() => {
  const isReady = handle.ready();
  console.log(`Handle is ${isReady ? 'ready' : 'not ready'}`);  
});
```

当我们尝试展示数据给用户或者展示一个加载中的画面时，我们可以使用这些信息。

<h3 id="changing-arguments">响应式变更订阅参数</h3>

当订阅的参数改变时，我们使用 `autorun` 再次订阅，相关的例子我们也看过了。但详细的内容我们放在这里讲：

```js
Template.Lists_show_page.onCreated(function() {
  this.getListId = () => FlowRouter.getParam('_id');

  this.autorun(() => {
    this.subscribe('todos.inList', this.getListId());
  });
});
```

在这个例子中，当 `this.getListId()` 改变时 `autorun` 都会重新运行，（其实是因为 `FlowRouter.getParam('_id')` 改变了），其他常见的响应式数据源有：

1. 模板的数据背景（可以通过 `Template.currentData()` 响应式获取）。
2. 当前用户状态（`Meteor.user()` 和 `Meteor.loggingIn()`）。
3. 其他针对应用的客户端数据存储内容。

技术上来说，当这些响应式数据源其中一项改变时，会有以下变化：

1. 响应式数据源使得 autorun *失效*（给它做一个标记在下一个跟踪器冲洗周期重新运行）。
2. 订阅检测到以上变化时，下一个计算运行后可以有多种结果，因此订阅把自己标记为可重构的。
3. 计算重新运行，重新调用 `.subscibe()`，可以带相同或不同的参数。
4. 如果订阅运行使用 *相同的参数*，那么 “新” 的订阅会发现之前 “把自己标记为可重构” 的订阅，相同的数据已经准备好了，拿来用就可以。
5. 如果订阅运行使用 *不同的参数*，会生成一个新的订阅，链接到服务器端的发布。
6. 在冲洗周期结束后（例如重新计算结束后），旧的订阅会检测自己是否有被重新使用，没有的话发送信息关闭该订阅的信息到服务器。

上面的步骤 4 是一个重要的细节 —— 系统清楚地知道如果参数相同的话不需要重新订阅。这对模板结构中其他新订阅来说也是一样的。例如，如果用户在两个有相同订阅的页面之间切换，相同的机制将会发生，因此没有必要重新订阅。

<h3 id="publication-behavior-with-arguments">参数改变后的发布行为</h3>

当开始新的订阅和停止旧的订阅时服务器上发生了什么，这也是很值得了解的。

服务器 *明确地* 等到所有数据都被发送（新的订阅准备好了）之后再从旧的订阅中删除数据。目的是防止闪烁 —— 当然你可以一直显示旧的订阅获取的数据直到新的数据准备好，然后立即切换到新订阅的完整数据集。

想说明的是，通常情况下，当改变订阅的时候，有一段时间会 *过度获取* 数据，客户端的数据比我们查询的多。这也是为什么我们应该获取和订阅相同的数据（而不是 “过度获取”）。

<h3 id="pagination">订阅分页</h3>

一个非常常见的数据访问方式是分页。这指的是在一个页面获取有序列表的数据 —— 通常是一定数量的物品，假设 20 件。

There are two styles of pagination that are commonly used, a "page-by-page" style---where you show only one page of results at a time, starting at some offset (which the user can control), and "infinite-scroll" style, where you show an increasing number of pages of items, as the user moves through the list (this is the typical "feed" style user interface).

In this section, we'll consider a publication/subscription technique for the second, infinite-scroll style pagination. The page-by-page technique is a little tricker to handle in Meteor, due to it being difficult to calculate the offset on the client. If you need to do so, you can follow many of the same techniques that we use here and use the [`percolate:find-from-publication`](https://atmospherejs.com/percolate/find-from-publication) package to keep track of which records have come from your publication.

在一个可以无限滚动下拉的页面，我们只需要简单在订阅中添加一个新参数控制加载的物品数量。例如我们要在 Todos 应用中实现分页。

```js
const MAX_TODOS = 1000;

Meteor.publish('todos.inList', function(listId, limit) {
  new SimpleSchema({
    listId: { type: String },
    limit: { type: Number }
  }).validate({ listId, limit });

  const options = {
    sort: {createdAt: -1},
    limit: Math.min(limit, MAX_TODOS)
  };

  // ...
});
```

It's important that we set a `sort` parameter on our query (to ensure a repeatable order of list items as more pages are requested), and that we set an absolute maximum on the number of items a user can request (at least in the case where lists can grow without bound).

Then on the client side, we'd set some kind of reactive state variable to control how many items to request:

```js
Template.Lists_show_page.onCreated(function() {
  this.getListId = () => FlowRouter.getParam('_id');

  this.autorun(() => {
    this.subscribe('todos.inList',
      this.getListId(), this.state.get('requestedTodos'));
  });
});
```

We'd increment that `requestedTodos` variable when the user clicks "load more" (or perhaps just when they scroll to the bottom of the page).

One piece of information that's very useful to know when paginating data is the *total number of items* that you could see. The [`tmeasday:publish-counts`](https://atmospherejs.com/tmeasday/publish-counts) package can be useful to publish this. We could add a `Lists.todoCount` publication like so

```js
Meteor.publish('Lists.todoCount', function({ listId }) {
  new SimpleSchema({
    listId: {type: String}
  }).validate({ listId });

  Counts.publish(this, `Lists.todoCount.${listId}`, Todos.find({listId}));
});
```

Then on the client, after subscribing to that publication, we can access the count with

```js
Counts.get(`Lists.todoCount.${listId}`)
```

<h2 id="stores">客户端数据和响应式存储</h2>

In Meteor, persistent or shared data comes over the wire on publications. However, there are some types of data which doesn't need to be persistent or shared between users. For instance, the "logged-in-ness" of the current user, or the route they are currently viewing.

Although client-side state is often best contained as state of an individual template (and passed down the template hierarchy as arguments where necessary), sometimes you have a need for "global" state that is shared between unrelated sections of the template hierarchy.

Usually such state is stored in a *global singleton* object which we can call a store. A singleton is a data structure of which only a single copy logically exists. The current user and the router from above are typical examples of such global singletons.

<h3 id="store-types">存储的类型</h3>

In Meteor, it's best to make stores *reactive data* sources, as that way they tie most naturally into the rest of the ecosystem. There are a few different packages you can use for stores.

If the store is single-dimensional, you can probably use a `ReactiveVar` to store it (provided by the [`reactive-var`](https://atmospherejs.com/meteor/reactive-var) package). A `ReactiveVar` has two properties, `get()` and `set()`:

```js
DocumentHidden = new ReactiveVar(document.hidden);
$(window).on('visibilitychange', (event) => {
  DocumentHidden.set(document.hidden);
});
```

If the store is multi-dimensional, you may want to use a `ReactiveDict` (from the [`reactive-dict`](https://atmospherejs.com/meteor/reactive-dict) package):

```js
const $window = $(window);
function getDimensions() {
  return {
    width: $window.width(),
    height: $window.height()
  };
};

WindowSize = new ReactiveDict();
WindowSize.set(getDimensions());
$window.on('resize', () => {
  WindowSize.set(getDimensions());
});
```

The advantage of a `ReactiveDict` is you can access each property individually (`WindowSize.get('width')`), and the dict will diff the field and track changes on it individually (so your template will re-render less often for instance).

If you need to query the store, or store many related items, it's probably a good idea to use a Local Collection (see the [Collections Article](collections.html#local-collections)).

<h3 id="accessing-stores">访问存储</h3>

You should access stores in the same way you'd access other reactive data in your templates---that means centralizing your store access, much like you centralize your subscribing and data fetch. For a Blaze template, that's either in a helper, or from within a `this.autorun()` inside an `onCreated()` callback.

This way you get the full reactive power of the store.

<h3 id="updating-stores">上传存储</h3>

If you need to update a store as a result of user action, you'd update the store from an event handler, just like you call [Methods](methods.html).

If you need to perform complex logic in the update (e.g. not just call `.set()` etc), it's a good idea to define a mutator on the store. As the store is a singleton, you can just attach a function to the object directly:

```js
WindowSize.simulateMobile = (device) => {
  if (device === 'iphone6s') {
    this.set({width: 750, height: 1334});
  }
}
```

<h2 id="advanced-publications">更高级的发布</h2>

Sometimes, the simple mechanism of returning a query from a publication function won't cover your needs. In those situations, there are some more powerful publication patterns that you can use.

<h3 id="publishing-relations">发布关系型数据</h3>

It's common to need related sets of data from multiple collections on a given page. For instance, in the Todos app, when we render a todo list, we want the list itself, as well as the set of todos that belong to that list.

One way you might do this is to return more than one cursor from your publication function:

```js
Meteor.publish('todos.inList', function(listId) {
  new SimpleSchema({
    listId: {type: String}
  }).validate({ listId });

  const list = Lists.findOne(listId);

  if (list && (!list.userId || list.userId === this.userId)) {
    return [
      Lists.find(listId),
      Todos.find({listId})
    ];
  } else {
    // The list doesn't exist, or the user isn't allowed to see it.
    // In either case, make it appear like there is no list.
    return this.ready();
  }
});
```

However, this example will not work as you might expect. The reason is that reactivity doesn't work in the same way on the server as it does on the client. On the client, if *anything* in a reactive function changes, the whole function will re-run, and the results are fairly intuitive.

On the server however, the reactivity is limited to the behavior of the cursors you return from your publish functions. You'll see any changes to the data that matches their queries, but *their queries will never change*.

So in the case above, if a user subscribes to a list that is later made private by another user, although the `list.userId` will change to a value that no longer passes the condition, the body of the publication will not re-run, and so the query to the `Todos` collection (`{listId}`) will not change. So the first user will continue to see items they shouldn't.

However, we can write publications that are properly reactive to changes across collections. To do this, we use the [`reywood:publish-composite`](https://atmospherejs.com/reywood/publish-composite) package.

The way this package works is to first establish a cursor on one collection, and then explicitly set up a second level of cursors on a second collection with the results of the first cursor. The package uses a query observer behind the scenes to trigger the subscription to change and queries to re-run whenever the source data changes.

```js
Meteor.publishComposite('todos.inList', function(listId) {
  new SimpleSchema({
    listId: {type: String}
  }).validate({ listId });

  const userId = this.userId;

  return {
    find() {
      const query = {
        _id: listId,
        $or: [{userId: {$exists: false}}, {userId}]
      };

      // We only need the _id field in this query, since it's only
      // used to drive the child queries to get the todos
      const options = {
        fields: { _id: 1 }
      };

      return Lists.find(query, options);
    },

    children: [{
      find(list) {
        return Todos.find({ listId: list._id }, { fields: Todos.publicFields });
      }
    }]
  };
});
```

In this example, we write a complicated query to make sure that we only ever find a list if we are allowed to see it, then, once per list we find (which can be one or zero times depending on access), we publish the todos for that list. Publish Composite takes care of stopping and starting the dependent cursors if the list stops matching the original query or otherwise.

<h3 id="complex-auth">复杂的授权</h3>

We can also use `publish-composite` to perform complex authorization in publications. For instance, consider if we had a `Todos.admin.inList` publication that allowed an admin to bypass default publication's security for users with an `admin` flag set.

We might want to write:

```js
Meteor.publish('Todos.admin.inList', function({ listId }) {
  new SimpleSchema({
    listId: {type: String}
  }).validate({ listId });

  const user = Meteor.users.findOne(this.userId);

  if (user && user.admin) {
    // We don't need to worry about the list.userId changing this time
    return [
      Lists.find(listId),
      Todos.find({listId})
    ];
  } else {
    return this.ready();
  }
});
```

However, due to the same reasons discussed above, the publication *will not re-run* if the user's `admin` status changes. If this is something that is likely to happen and reactive changes are needed, then we'll need to make the publication reactive. We can do this via the same technique as above however:

```js
Meteor.publishComposite('Todos.admin.inList', function(listId) {
  new SimpleSchema({
    listId: {type: String}
  }).validate({ listId });

  const userId = this.userId;
  return {
    find() {
      return Meteor.users.find({userId, admin: true});
    },
    children: [{
      find() {
        // We don't need to worry about the list.userId changing this time
        return [
          Lists.find(listId),
          Todos.find({listId})
        ];
      }  
    }]
  };
});
```

<h3 id="custom-publication">使用低级 API 接口自定义发布</h3>

In all of our examples so far (outside of using`Meteor.publishComposite()`) we've returned a cursor from our `Meteor.publish()` handlers. Doing this ensures Meteor takes care of the job of keeping the contents of that cursor in sync between the server and the client. However, there's another API you can use for publish functions which is closer to the way the underlying Distributed Data Protocol (DDP) works.

DDP uses three main messages to communicate changes in the data for a publication: the `added`, `changed` and `removed` messages. So, we can similarly do the same for a publication:

```js
Meteor.publish('custom-publication', function() {
  // We can add documents one at a time
  this.added('collection-name', 'id', {field: 'values'});

  // We can call ready to indicate to the client that the initial document sent has been sent
  this.ready();

  // We may respond to some 3rd party event and want to send notifications
  Meteor.setTimeout(() => {
    // If we want to modify a document that we've already added
    this.changed('collection-name', 'id', {field: 'new-value'});

    // Or if we don't want the client to see it any more
    this.removed('collection-name', 'id');
  });

  // It's very important to clean up things in the subscription's onStop handler
  this.onStop(() => {
    // Perhaps kill the connection with the 3rd party server
  });
});
```

From the client's perspective, data published like this doesn't look any different---there's actually no way for the client to know the difference as the DDP messages are the same. So even if you are connecting to, and mirroring, some esoteric data source, on the client it'll appear like any other Mongo collection.

One point to be aware of is that if you allow the user to *modify* data in the "pseudo-collection" you are publishing in this fashion, you'll want to be sure to re-publish the modifications to them via the publication, to achieve an optimistic user experience.

<h3 id="lifecycle">订阅的生命周期</h3>

Although you can use publications and subscriptions in Meteor via an intuitive understanding, sometimes it's useful to know exactly what happens under the hood when you subscribe to data.

Suppose you have a simple publication of the following form:

```js
Meteor.publish('Posts.all', function() {
  return Posts.find({}, {limit: 10});
});
```

Then when a client calls `Meteor.subscribe('Posts.all')` the following things happen inside Meteor:

1. The client sends a `sub` message with the name of the subscription over DDP.

2. The server starts up the subscription by running the publication handler function.

3. The publication handler identifies that the return value is a cursor. This enables a convenient mode for publishing cursors.

4. The server sets up a query observer on that cursor, unless such an observer already exists on the server (for any user), in which case that observer is re-used.

5. The observer fetches the current set of documents matching the cursor, and passes them back to the subscription (via the `this.added()` callback).

6. The subscription passes the added documents to the subscribing client's connection *mergebox*, which is an on-server cache of the documents that have been published to this particular client. Each document is merged with any existing version of the document that the client knows about, and an `added` (if the document is new to the client) or `changed` (if it is known but this subscription is adding or changing fields) DDP message is sent.

  Note that the mergebox operates at the level of top-level fields, so if two subscriptions publish nested fields (e.g. sub1 publishes `doc.a.b = 7` and sub2 publishes `doc.a.c = 8`), then the "merged" document might not look as you expect (in this case `doc.a = {c: 8}`, if sub2 happens second).

7. The publication calls the `.ready()` callback, which sends the DDP `ready` message to the client. The subscription handle on the client is marked as ready.

8. The observer observes the query. Typically, it [uses MongoDB's Oplog](https://github.com/meteor/meteor/wiki/Oplog-Observe-Driver) to notice changes that affect the query. If it sees a relevant change, like a new matching document or a change in a field on a matching document, it calls into the subscription (via `.added()`, `.changed()` or `.removed()`), which again sends the changes to the mergebox, and then to the client via DDP.

This continues until the client [stops](#stopping-subscriptions) the subscription, triggering the following behavior:

1. The client sends the `unsub` DDP message.

2. The server stops its internal subscription object, triggering the following effects:

3. Any `this.onStop()` callbacks setup by the publish handler run. In this case, it is a single automatic callback setup when returning a cursor from the handler, which stops the query observer and cleans it up if necessary.

4. All documents tracked by this subscription are removed from the mergebox, which may or may not mean they are also removed from the client.

5. The `nosub` message is sent to the client to indicate that the subscription has stopped.

<h2 id="rest-interop">使用 REST API 接口</h2>

Publications and subscriptions are the primary way of dealing with data in Meteor's DDP protocol, but lots of data sources use the popular REST protocol for their API. It's useful to be able to convert between the two.

<h3 id="loading-from-rest">使用发布从 REST 端点加载数据</h3>

As a concrete example of using the [low-level API](#custom-publication), consider the situation where you have some 3rd party REST endpoint which provides a changing set of data that's valuable to your users. How do you make that data available?

One option would be to provide a Method that simply proxies through to the endpoint, for which it's the client's responsibility to poll and deal with the changing data as it comes in. So then it's the clients problem to deal with keeping a local data cache of the data, updating the UI when changes happen, etc. Although this is possible (you could use a Local Collection to store the polled data, for instance), it's simpler, and more natural to create a publication that does this polling for the client.

A pattern for turning a polled REST endpoint looks something like this:

```js
const POLL_INTERVAL = 5000;

Meteor.publish('polled-publication', function() {
  const publishedKeys = {};

  const poll = () => {
    // Let's assume the data comes back as an array of JSON documents, with an _id field, for simplicity
    const data = HTTP.get(REST_URL, REST_OPTIONS);

    data.forEach((doc) => {
      if (publishedKeys[doc._id]) {
        this.changed(COLLECTION_NAME, doc._id, doc);
      } else {
        publishedKeys[doc._id] = true;
        this.added(COLLECTION_NAME, doc._id, doc);
      }
    });
  };

  poll();
  this.ready();

  const interval = Meteor.setInterval(poll, POLL_INTERVAL);

  this.onStop(() => {
    Meteor.clearInterval(interval);
  });
});
```

Things can get more complicated; for instance you may want to deal with documents being removed, or share the work of polling between multiple users (in a case where the data being polled isn't private to that user), rather than doing the exact same poll for each interested user.

<h3 id="publications-as-rest">以 REST 端点身份访问一个发布</h3>

The opposite scenario occurs when you want to publish data to be consumed by a 3rd party, typically over REST. If the data we want to publish is the same as what we already publish via a publication, then we can use the [simple:rest](https://atmospherejs.com/simple/rest) package to do this really easily.

In the Todos example app, we have done this, and you can now access our publications over HTTP:

```bash
$ curl localhost:3000/publications/lists.public
{
  "Lists": [
    {
      "_id": "rBt5iZQnDpRxypu68",
      "name": "Meteor Principles",
      "incompleteCount": 7
    },
    {
      "_id": "Qzc2FjjcfzDy3GdsG",
      "name": "Languages",
      "incompleteCount": 9
    },
    {
      "_id": "TXfWkSkoMy6NByGNL",
      "name": "Favorite Scientists",
      "incompleteCount": 6
    }
  ]
}
```

You can also access authenticated publications (such as `lists.private`). Suppose we've signed up (via the web UI) as `user@example.com`, with the password `password`, and created a private list. Then we can access it as follows:

```bash
# First, we need to "login" on the commandline to get an access token
$ curl localhost:3000/users/login  -H "Content-Type: application/json" --data '{"email": "user@example.com", "password": "password"}'
{
  "id": "wq5oLMLi2KMHy5rR6",
  "token": "6PN4EIlwxuVua9PFoaImEP9qzysY64zM6AfpBJCE6bs",
  "tokenExpires": "2016-02-21T02:27:19.425Z"
}

# Then, we can make an authenticated API call
$ curl localhost:3000/publications/lists.private -H "Authorization: Bearer 6PN4EIlwxuVua9PFoaImEP9qzysY64zM6AfpBJCE6bs"
{
  "Lists": [
    {
      "_id": "92XAn3rWhjmPEga4P",
      "name": "My Private List",
      "incompleteCount": 5,
      "userId": "wq5oLMLi2KMHy5rR6"
    }
  ]
}
```
