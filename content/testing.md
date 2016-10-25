---
title: "测试"
order: 14
description: 如何测试你的 meteor 应用
discourseTopicId: 20191
---

<h2 id="introduction">介绍</h2>

测试可以使你确保你的应用程序如你预期那样工作，尤其是随着时间推移，你的基本代码也逐渐发生更改时。良好的测试可以使你可以放心地重构或改写你的代码。测试也是为应用的预期行为编制的最为具体的文档形式，因为其他开发人员可以通过阅读测试代码来了解如何使用你的代码。

自动化测试至关重要，相较于手动测试，自动化测试能够以更高的频度来运行更大的测试用例集，以帮助你快速捕获回归错误。

<h3 id="testing-types">测试的分类</h3>

很多书籍都会将测试主题纳入其中，因此在这里，我们将只会简单地提及一些测试基础。书写测试代码的重点在于：你准备将其用于应用中哪个部分的测试，以及你如何验证（应用的）行为是否为正常工作。

- **单元测试**：如果你正在（编写代码）测试你应用中的一个小模块，那么这意味着你编写的是单元测试。你需要 *"stub"* 并 *"mock"* 经常被该模块所影响的其他模块，以便于 *隔离* 各个测试。你通常还需要 *监视* 该模块的动作以验证它们已经发生。

- **集成测试**：如果你正在（编写代码）测试多个模块在协同任务中的行为的正确性，那么你编写的是集成测试。这类测试要复杂得多，而且可能需要同时运行客户端和服务端的代码，以便校验两者之间的通信是否如预期一样工作。通常地，集成测试仍然需要将应用中的一部分从整体中隔离出来，并且直接在测试代码中验证运行结果。

- **验收试验**：如果你想编写一个测试，用于在任何你当前正在运行的 app 版本，并在浏览器级别验证你的操作是否触发与预期一致的行为，那么你编写的是验收测试（有时也被称为『端到端测试』）。这类测试通常尽可能不深入应用，除此之外，你可能需要预置测试运行所需的正确数据。

- **负载测试**：在最后，你可能希望测试你的应用通常能在多大负荷下工作，或者能处理多少负荷而不挂掉。这被称为负载测试或压力测试。这类测试的创建工作是具有挑战性的，并且该类测试通常不会频繁运行，但对于一个大型产品发布前夕增强信心这一点而言是非常重要的。

<h3 id="challenges-with-meteor">在 Meteor 中测试所要面临的挑战</h3>

在很多方面，测试一个 Meteor 应用与测试其他全栈的 JavaScript 应用并无不同。但是，相较于更传统的聚焦于后端或前端中一方的框架而言，有两个因素会让测试工作更有挑战性：

- **客户端/服务端数据**：Meteor 的数据系统可以很方便地弥合客户端 - 服务器间的间隙，往往可以让你不用思考如何围绕数据构建应用程序。测试你的代码在跨越两端的情况下是否仍然工作正常，变得至关重要。在传统框架中，你通常可以通过隔离客户端和服务端分别测试的方式，来避免对客户端与服务端间接口的测试的思考上花费大量时间，但 Meteor 的[全应用测试模式](#test-modes)使得编写覆盖全栈的[集成测试](#full-app-integration-test)变得简单。这里的另一个挑战，是要在客户端环境中构造测试数据，我们将会在下文的[生成测试数据部分](#generating-test-data)中对此进行讨论。

- **实时响应**：Meteor 的实时响应系统是『最终一致性』的，这意味着当你更改了一个具有响应性的输入至系统中后，在一段时间后，你将会看到用户界面发生相应的更改。这会是测试工作中的一个挑战，但也有一些方式可以等到这些更改发生之后再验证测试结果，例如 `Tracker.afterFlush()` 。

<h2 id="test-modes">'meteor test' 命令</h2>

在 Meteor 中，测试应用的主要途径是执行 `meteor test` 命令。

这一命令会使用一个特殊的『测试模式』来加载你的应用，它会做以下几件事：

1. *不会*直接加载任何 Meteor 在正常运行时会直接加载的*任何*东西。
2. *会*立即加载*任何*名称满足 `*.test[s].*` 或 `*.spec[s].*` 匹配规则的文件（包括在 `imports/` 目录中的文件）。
3. 设置 `Meteor.isTest` 为 true。
4. 需要启动测试驱动包（[见下文](#driver-packages)）。

> [Meteor 构建工具](build-tool.html#what-it-does)和 `meteor test` 命令会忽略任何路径下的 `tests/` 目录下的任何文件。这一特性允许你在该目录中放置一些测试用例，以便于你在 Meteor 内建的测试工具以外的其他测试工具中运行，同时也可以避免这些文件被加载到你的应用程序之中。见 Meteor 的[默认文件加载顺序](structure.html#load-order)规则

以上内容意味着，你可以在以约定的命名方式命名的文件中编写测试代码，而这些文件将不会在你正常构建应用时被包含其中。而当你在测试模式中运行你的应用时，这些文件将会被直接加载（而其他文件不会），它们可以导入你想要测试的模块。如我们所理解的那样，这对于[单元测试](#unit-testing)和[简单集成测试](#simple-integration-test)非常理想。

另外，Meteor 还提供了一个『全应用』测试模式，你可以通过 `meteor test --full-app` 来运行该模式。

该模式类似于测试模式，但稍有不同：

1. 该模式加载满足 `*.app-test[s].*` 或 `*.app-spec[s].*` 命名规则的文件。
2. 该模式**会**像 Meteor 正常运行时一样立即加载应用程序代码。
3. 设置 `Meteor.isAppTest` 为 true（而不是设置 `Meteor.isTest`）。

这意味着，您的应用程序整体（例如包括 Web 服务器和客户端的路由器）会像在普通模式下一样正常加载和运行。这使你能够编写更复杂的[集成测试](#full-app-integration-test)，同时装入其他文件进行[验收测试](#acceptance-testing)。

注意，在 Meteor 工具中还有一条测试命令：`meteor test-packages`，用于测试 Atmosphere 包，我们将在[如何编写 packages](writing-packages.html#testing) 中进行讨论。

<h3 id="driver-packages">测试驱动包</h3>

当你执行 `meteor test` 命令时，你必须提供一个 `--driver-package` 参数。一个测试驱动就是一个小型应用程序，它将代替你的应用本身运行，并且运行你定义的一系列测试，同时在某种形式的用户界面中报告这些测试的运行结果。

测试驱动包主要分为两种类型：

- **Web 界面报告类型**：你可以在 Meteor 应用中一个特殊的用于展示测试报告的 web 界面中观察测试结果。

<img src="images/mocha-test-results.png">

- **控制台报告类型**：该类型的测试驱动完全在命令行中运行，主要在自动化测试中使用，例如[持续集成](#ci)（如我们所知，通常 PhantomJS 就是用来驱动这类测试的）。

<h3 id="mocha">推荐：Mocha</h3>

在这篇文章中，我们会使用流行的 [Mocha](https://mochajs.org) 测试框架，搭配 [Chai](http://chaijs.com) 断言库来测试我们的应用。为了使用 Mocha 来编写和运行测试代码，我们需要添加相应的测试驱动包。

我们有如下几个选项，你可以结合你应用的实际情况进行选择。你可能需要依赖于一个以上的包，并在不同场景下设置不同的测试命令。

* [practicalmeteor:mocha](https://atmospherejs.com/practicalmeteor/mocha) 可以运行客户端和服务端的 package 或应用中的测试代码，并且将全部结果展示到浏览器中。使用 [spacejam](https://www.npmjs.com/package/spacejam) 可以提供命令行 / CI 支持。

* [dispatch:mocha-phantomjs](https://atmospherejs.com/dispatch/mocha-phantomjs) 使用 PhantomJS 来运行客户端和服务端的 Meteor 包或应用中的测试代码，并且将全部结果展示到服务端的控制台中。可以在 CI 服务器中运行测试代码，具备观察模式。
* [dispatch:mocha-browser](https://atmospherejs.com/dispatch/mocha-browser) 使用 Mocha 运行客户端和服务端的 package 或应用中的测试代码，并且将全部结果同时展示于浏览器和服务端控制台。具备观察模式。
* [dispatch:mocha](https://atmospherejs.com/dispatch/mocha) 使用 mocha 运行仅限于服务端的 Meteor 包或应用中的测试代码，并将全部结果展示于服务端控制台中。可以在 CI 服务器中运行测试代码，具备观察模式。

在开发模式或生产模式下，这些包不会做任何事。它们被声明为 `testOnly`，因此它们在测试模式以外的环境中甚至都不会被加载。但是当我们的应用运行于[测试模式](#test-modes)时，这些测试驱动包将会接管和执行客户端和服务端的测试代码，并且将测试结果渲染至浏览器。

这里我们可以添加 [`practicalmeteor:mocha`](https://atmospherejs.com/practicalmeteor/mocha) 这个包到我们的应用中。

```bash
meteor add practicalmeteor:mocha
```

<h2 id="test-files">测试文件</h2>

测试文件（例如 `todos-item.test.js` 或 `routing.app-specs.coffee` 文件）会将它们自身注册于测试驱动，以备测试库使用其通常的方式来执行。对于 Mocha 而言，这一行为是通过 `describe` 和 `it` 实现：

```js
describe('my module', function () {
  it('does something that should be tested', function () {
    // This code will be executed by the test driver when the app is started
    // in the correct mode
  })
})
```

注意，箭头函数的使用在 Mocha 中是[不被推荐的](http://mochajs.org/#arrow-functions)。

<h2 id="test-data">测试数据</h2>

当你的应用在测试模式下运行时，应用会同时初始化一个无数据的测试数据库。

如果你需要运行一个依赖于数据库的测试，而且需要数据库中具备专门的数据，你必须首先在你的测试中执行一些*设置*步骤，以确保数据库处于你所预期的状态。你可以使用一些工具来帮助你实现这个步骤。

[`xolvio:cleaner`](https://atmospherejs.com/xolvio/cleaner) 这个包非常有用，它可以帮助你确保你的数据库已经被清理干净，你可以在一个 `beforeEach` 块中使用这个包来重置数据库：

```js
import { resetDatabase } from 'meteor/xolvio:cleaner';

describe('my module', function () {
  beforeEach(function () {
    resetDatabase();
  });
});
```

这项技术只会在服务端生效。如果你需要在客户端完成该操作，你可以使用一个 method 来完成：

```js
import { resetDatabase } from 'meteor/xolvio:cleaner';

// 注意：在编写此类 method 时，你应当再次检查该文件的确仅会在测试模式中被加载！！
Meteor.methods({
  'test.resetDatabase': () => resetDatabase(),
});

describe('my module', function (done) {
  beforeEach(function (done) {
    // 在我们继续执行测试代码前，我们需要等待 method 执行完成，因此我们需要使用 Mocha 的异步
	// 机制（执行一次 done 回调）
    Meteor.call('test.resetDatabase', done);
  });
});
```

当我们把上面这些代码存放在测试文件中时，这些代码*不会*在普通的开发模式和生产模式中加载（如果这些代码被错误地加载的话，那将绝对是一件非常糟糕的事！）。如果你创建了一个使用了类似特性的 Atmosphere 包，你应当把它标记为 `testOnly` ，类似地，这些特性也将只会在测试模式中加载。

<h3 id="generating-test-data">生成测试代码</h3>

通常来讲，专门为测试运行创建一个数据集的行为是非常明智的。你可以调用集合实例的标准方法 `insert()` 来完成此事，不过你也可以创建*工厂*来帮助你生成随机的测试数据，而且这往往会更加轻松。有一个非常棒的，叫做 [`dburles:factory`](https://atmospherejs.com/dburles/factory) 包可以派上用场。

在 [Todos](https://github.com/meteor/todos) 这个示例应用中，我们使用 [`faker`](https://www.npmjs.com/package/faker) 这个 npm 包定义了一个工厂，专门用来描述如何创建测试用的待办事项：

```js
import faker from 'faker';

Factory.define('todo', Todos, {
  listId: () => Factory.get('list'),
  text: () => faker.lorem.sentence(),
  createdAt: () => new Date(),
});
```

如果要在测试中使用这个工厂，我们要做的仅仅只是调用 `Factory.create`：

```js

// 这行代码会在数据库中创建一个 todo 和一个 list，并返回 todo。
const todo = Factory.create('todo');

// 如果我们已经有了一个 list 实例，那么我们将它的 id 可以传递给工厂，以避免重复创建：
const list = Factory.create('list');
const todoInList = Factory.create('todo', { listId: list._id });
```

<h3 id="mocking-the-database">对数据库的模拟</h3>

`Factory.create` 直接将文档插入到传递至 `Factory.define` 函数中的集合，而如果在客户端也这么做显然是有问题的。尽管如此，仍然有一个简洁的隔离技巧，你可以将由服务端支撑的 `Todos` [客户端数据集](collections.html#client-collections)替换为一个模拟的[本地数据集](#collections.html#local-collections)。而 [`hwillson:stub-collections`](https://atmospherejs.com/hwillson/stub-collections) 这个包已经包含了完成这些任务的编码。

```js
import StubCollections from 'meteor/hwillson:stub-collections';
import { Todos } from 'path/to/todos.js';

StubCollections.stub(Todos);

// 现在 Todos 已经被存根至一个简单的本地的模拟集合中，
// 我们可以在客户端这样做：
Todos.insert({ a: 'document' });

// 还原 `Todos` 集合
StubCollections.restore();
```

在 Mocha 的测试的 `beforeEach`/`afterEach` 块中是非常适合使用 `stub-collections` 的。

<h2 id="unit-testing">单元测试</h2>

单元测试是这样一个过程：隔离一段代码，然后测试这段代码内部是否像你预期那样工作。在[使用 ES2015 modules 来拆分我们的代码](structure.html)这一前提下，在每次测试中分别对这些模块中的一个进行测试是一件自然而然的事情。

通过隔离一个模块并简单地测试其内部功能的方式，我们可以写出*快速*而*准确*的测试——这些测试能够很快地告诉你应用程序中的问题存在于何处。尽管如此，仍然需要注意的是，不完整的单元测试常常会掩饰一些 bug ，这是由其模拟依赖的方式所决定的。因此，将单元测试与速度更慢（并且也许运行频度更低）的集成测试和验收测试结合起来是非常有用的。

<h3 id="simple-blaze-unit-test">一个简单的 Blaze 单元测试</h3>

在[Todos](https://github.com/meteor/todos)这个示例应用中，多亏了这样一个事实：我们将用户界面拆分为[智能组件和可重用组件](ui-ux.html#components)，因此我们会自然而然地想到要对这些可重用组件中的一部分进行单元测试（在下文中，我们将会了解到如何对这些智能的组件进行[集成测试](#simple-integration-test)）。

为了完成这个测试，我们会使用一个简单的测试 helper 构件，它会使用一个给定数据的上下文来渲染一个屏幕中不可见的 Blaze 组件。我们会把它放置于 `imports` 目录中，它并不会在普通模式下被加载到我们的应用中（因为在此模式中，没有任何导入它的代码）。

[`imports/ui/test-helpers.js`](https://github.com/meteor/todos/blob/master/imports/ui/test-helpers.js):

```js
import { _ } from 'meteor/underscore';
import { Template } from 'meteor/templating';
import { Blaze } from 'meteor/blaze';
import { Tracker } from 'meteor/tracker';

const withDiv = function withDiv(callback) {
  const el = document.createElement('div');
  document.body.appendChild(el);
  try {
    callback(el);
  } finally {
    document.body.removeChild(el);
  }
};

export const withRenderedTemplate = function withRenderedTemplate(template, data, callback) {
  withDiv((el) => {
    const ourTemplate = _.isString(template) ? Template[template] : template;
    Blaze.renderWithData(ourTemplate, data, el);
    Tracker.flush();
    callback(el);
  });
};
```

[`Todos_item`](https://github.com/meteor/todos/blob/master/imports/ui/components/todos-item.html) 模板是一个可供测试的简单的可复用组件样例。这里的代码展示了一个单元测试的大致是什么样子（你也可以看看[这个应用程序库中的其他示范](https://github.com/meteor/todos/blob/master/imports/ui/components/client)）。

[`imports/ui/components/client/todos-item.tests.js`](https://github.com/meteor/todos/blob/master/imports/ui/components/client/todos-item.tests.js):

```js
/* eslint-env mocha */
/* eslint-disable func-names, prefer-arrow-callback */

import { Factory } from 'meteor/factory';
import { chai } from 'meteor/practicalmeteor:chai';
import { Template } from 'meteor/templating';
import { $ } from 'meteor/jquery';


import { withRenderedTemplate } from '../../test-helpers.js';
import '../todos-item.js';

describe('Todos_item', function () {
  beforeEach(function () {
    Template.registerHelper('_', key => key);
  });

  afterEach(function () {
    Template.deregisterHelper('_');
  });

  it('renders correctly with simple data', function () {
    const todo = Factory.build('todo', { checked: false });
    const data = {
      todo,
      onEditingChange: () => 0,
    };

    withRenderedTemplate('Todos_item', data, el => {
      chai.assert.equal($(el).find('input[type=text]').val(), todo.text);
      chai.assert.equal($(el).find('.list-item.checked').length, 0);
      chai.assert.equal($(el).find('.list-item.editing').length, 0);
    });
  });
});
```

下面的内容需要特别关注：

<h4 id="unit-test-importing">Importing</h4>

当我们在测试模式下运行我们的应用时，只有我们的测试文件会被立即加载。这意味着：为了使用我们的模板，我们必须专门导入这些模板文件！在这次测试中，我们导入 `todos-item.js`，而这个文件本身会导入 `todos.html`（没错，你的确需要在你的 Blaze 模板中导入 HTML 文件）。

<h4 id="unit-test-stubbing">Stubbing</h4>

为了完成单元测试，我们必须模拟并代替这个模块的依赖关系。在这个案例中，多亏了我们已经将我们的代码隔离到可重用组件中这一方式，我们不需要做太多事；我们需要做的主要是去模拟由 [`tap:i18n`](ui-ux.html#i18n) 系统创建的 `{% raw %}{{_}}{% endraw %}` helper 。注意，我们可以在 `beforeEach` 块中完成模拟（和代替），并在 `afterEach` 块中还原（这些更改）。

如果你正在测试使用了全局变量的代码，那么你需要导入这些全局变量。例如，如果你的文件中使用了一个全局的 `Todos` 集合，而你正在测试这个文件：

```js
// logging.js
export function logTodos() {
  console.log(Todos.findOne());
}
```

那么你需要同时在这个文件以及其测试文件中导入 `Todos`：

```js
// logging.js
import { Todos } from './todos.js'
export function logTodos() {
  console.log(Todos.findOne());
}
```

```js
// logging.test.js
import { Todos } from './todos.js'
Todos.findOne = () => {
  return {text: "write a guide"}
}

import { logTodos } from './logging.js'
// then test logTodos
...
```

<h4 id="unit-test-data">创建数据</h4>

我们可以使用 [Factory package's](#test-data) 的 `.build()` API 来创建一条用于测试的数据文档，而不必将其插入到任何的数据库集合中。鉴于我们一直小心翼翼地避免在可复用组件中直接地调用任何集合，我们可以将这个预先使用 factory 创建好的 `todo` 文档作为替代直接传递给该模板。

<h3 id="simple-react-unit-test">一个简单的 React 单元测试</h3>

我们可以将相同的结构应用于 React 组件的测试工作中，同时我们推荐使用 [Enzyme](https://github.com/airbnb/enzyme) 这个包，这个包伪装成 React 组件的环境，并且允许你使用 CSS 选择器来进行查询。而 [react branch of the Todos app](https://github.com/meteor/todos/tree/react) 这个包支持大型的测试套件，但现在，我们还是先看一个简单的示例吧：

```js
import { Factory } from 'meteor/factory';
import React from 'react';
import { shallow } from 'enzyme';
import { chai } from 'meteor/practicalmeteor:chai';
import TodoItem from './TodoItem.jsx';

describe('TodoItem', () => {
  it('should render', () => {
    const todo = Factory.build('todo', { text: 'testing', checked: false });
    const item = shallow(<TodoItem todo={todo} />);
    chai.assert(item.hasClass('list-item'));
    chai.assert(!item.hasClass('checked'));
    chai.assert.equal(item.find('.editing').length, 0);
    chai.assert.equal(item.find('input[type="text"]').prop('defaultValue'), 'testing');
  });
});
```

这个测试比上面的 Blaze 版本稍微简单一些，因为这个 React 示例没有支持国际化。除此之外，两者在概念上是完全一致的。我们使用 Enzyme 的 `shallow` 函数来渲染 `TodoItem` 组件，并使用其返回的结果对象来查询文档，并且也用它来模拟用户与 UI 的互动。下面是一个模拟用户勾选待办事项的示例：

```js
import { Factory } from 'meteor/factory';
import React from 'react';
import { shallow } from 'enzyme';
import { sinon } from 'meteor/practicalmeteor:sinon';
import TodoItem from './TodoItem.jsx';
import { setCheckedStatus } from '../../api/todos/methods.js';

describe('TodoItem', () => {
  it('should update status when checked', () => {
    sinon.stub(setCheckedStatus, 'call');
    const todo = Factory.create('todo', { checked: false });
    const item = shallow(<TodoItem todo={todo} />);

    item.find('input[type="checkbox"]').simulate('change', {
      target: { checked: true },
    });

    sinon.assert.calledWith(setCheckedStatus.call, {
      todoId: todo._id,
      newCheckedStatus: true,
    });

    setCheckedStatus.call.restore();
  });
});
```

在该实例中，当用户点击时，`TodoItem` 组件调用了一个 [Meteor Method](/methods.html) `setCheckedStatus`。但由于这是一个单元测试，并没有服务器在运行，因此我们使用 [Sinon](http://sinonjs.org) 这个包来模拟它。在我们模拟的点击发生之后，我们会检查这个 stub 是否在参数正确的情况下被调用。最后，我们清除这个 stub 并且恢复被 stub 的源方法的行为。

<h3 id="running-unit-tests">运行单元测试</h3>

要运行我们在应用中定义的测试，我们需要在 [test mode](#test-modes) 中运行我们的应用：

```txt
meteor test --driver-package practicalmeteor:mocha
```

一旦我们定义了一个测试文件 (`imports/todos/todos.tests.js`)，那么这个文件将会被立即加载，并将其中的 `'builds correctly from factory'` 这个测试用例注册至 Mocha 中。
> 原文：As we've defined a test file (`imports/todos/todos.tests.js`), what this means is that the file above will be eagerly loaded, adding the `'builds correctly from factory'` test to the Mocha registry.

在浏览器中访问 http://localhost:3000 以便运行测试。上述的命令中使用了 `practicalmeteor:mocha` 这个包来驱动测试，这使得你的测试可以同时在浏览器和服务端运行。它会将测试结果按照 Mocha 的测试报告的形式反馈至浏览器中：

<img src="images/mocha-test-results.png">

通常地，当我们正在开发一个应用程序时，我们需要在另一个端口（例如 `3100`）运行 `meteor test` 命令，这样做更加合理，同时，你也可以在一个隔离的进程中运行你的主应用。

```bash
# in one terminal window
meteor

# in another
meteor test --driver-package practicalmeteor:mocha --port 3100
```

然后，你可以打开两个浏览器窗口，你会发现，当你执行更改时，你的应用会正确响应，而同时，你也不会因此打断任何测试内容。

<h3 id="isolation-techniques">隔离技术</h3>

在[上述的单元测试](#simple-blaze-unit-test)中，我们通过一个非常受限的例子来了解了如何从一个庞大的应用中将单个模块隔离开来。这一点对于恰当的单元测试而言非常重要。这里还有一些实用程序和技术可供参考：

  - [`velocity:meteor-stubs`](https://atmospherejs.com/velocity/meteor-stubs)包，它可以为 Meteor 的大部分核心对象建立简单的 stubs。

  - 或者你可以使用诸如[Sinon](http://sinonjs.org)一类的工具来直接 stub 你所需要的东西，正如我们在[simple integration test](#simple-integration-test)这个例子中所看到那样。

  - 还有我们在[上面](#mocking-the-database)提到过的[`hwillson:stub-collections`](https://atmospherejs.com/hwillson/stub-collections)这个包。

如何更好地实施隔离并选择与测试相应的使用工具，这其中有着很大的发挥空间。
> 原文：There's a lot of scope for better isolation and testing utilities.

<h4 id="testing-publications">测试发布</h4>

使用[`johanbrook:publication-collector`](https://atmospherejs.com/johanbrook/publication-collector)这个包，你可以单独测试发布的输出，而不用按照惯例为之创建一个订阅：

```js
describe('lists.public', function () {
  it('sends all public lists', function (done) {
    // 设置用户 id，这个 id 将作为 `this.userId` 被提供给 publish 函数，
	// 以便于你测试用户认证。
    const collector = new PublicationCollector({userId: 'some-id'});

    // 收集从 `lists.public` 发布所发送的数据。
    collector.collect('lists.public', (collections) => {
	  // `collections` 是一个以集合名作为键名的字典，
	  // 而被发布的文档以数组的形式作为字典中的键值。
	  // 这里发布的是集合 'Lists' 中的文档。
      chai.assert.typeOf(collections.Lists, 'array');
      chai.assert.equal(collections.Lists.length, 3);
      done();
    });
  });
});
```

注意，用户文档——通常使用 `Meteor.users.find()` 进行查询的那部分——在执行 `PublicationCollector.collect()` 调用时可用，这些文档会被包含在本次调用的字典中，其键名为 `users`。你可以查看这个包中的[相关测试](https://github.com/johanbrook/meteor-publication-collector/blob/master/tests/publication-collector.test.js)以获取更多信息。

<h2 id="integration-testing">集成测试</h2>

集成测试是跨越模块边界的。在这个极其简单的例子中，这仅仅意味着一些与单元测试相似的东西：你所做的只是隔离其中的数个模块，创建一个并不独特的『系统测试』。

尽管在概念上不同于单元测试，然而这类测试往往不需要运行任何与单元测试不同的代码，并且，像在单元测试中一样，我们也同样可以使用[`meteor test` 模式](#running-unit-tests)和[隔离技术](#isolation-techniques)。

不过，在 Meteor 应用中（如果被测试的模块正好跨越了客户端 - 服务端边界），跨越了客户端 - 服务端边界的集成测试，需要一个独特的底层技术的支持，即 Meteor 的 "full app" 测试模式。

让我们看看这两种测试的例子。

<h3 id="simple-integration-test">简单的集成测试</h3>

我们的可复用组件非常自然地适应于单元测试；相似地，由于我们的智能组件的工作是搜集数据并将其提供给可复用组件，对它们测试工作往往需要集成测试才能真正地被正确实施。

在[Todos](https://github.com/meteor/todos)示例应用中，我们为 `Lists_show_page` 这个智能组件写过一个集成测试。这个测试仅仅只是为了确保当数据库中的数据正确时，被测试的模板能够正确地渲染——正如我们所预期那样，这个组件收集了正确的数据。它从 Meteor 栈中更复杂的数据订阅部分中，将渲染树隔离开来。如果我们打算从订阅方面测试其与智能组件的协作，那么我们需要写一个[full app integration test](#full-app-integration-test)。

[`imports/ui/components/client/todos-item.tests.js`](https://github.com/meteor/todos/blob/master/imports/ui/components/client/todos-item.tests.js):

```js
/* eslint-env mocha */
/* eslint-disable func-names, prefer-arrow-callback */

import { Meteor } from 'meteor/meteor';
import { Factory } from 'meteor/factory';
import { Random } from 'meteor/random';
import { chai } from 'meteor/practicalmeteor:chai';
import StubCollections from 'meteor/hwillson:stub-collections';
import { Template } from 'meteor/templating';
import { _ } from 'meteor/underscore';
import { $ } from 'meteor/jquery';
import { FlowRouter } from 'meteor/kadira:flow-router';
import { sinon } from 'meteor/practicalmeteor:sinon';


import { withRenderedTemplate } from '../../test-helpers.js';
import '../lists-show-page.js';

import { Todos } from '../../../api/todos/todos.js';
import { Lists } from '../../../api/lists/lists.js';

describe('Lists_show_page', function () {
  const listId = Random.id();

  beforeEach(function () {
    StubCollections.stub([Todos, Lists]);
    Template.registerHelper('_', key => key);
    sinon.stub(FlowRouter, 'getParam', () => listId);
    sinon.stub(Meteor, 'subscribe', () => ({
      subscriptionId: 0,
      ready: () => true,
    }));
  });

  afterEach(function () {
    StubCollections.restore();
    Template.deregisterHelper('_');
    FlowRouter.getParam.restore();
    Meteor.subscribe.restore();
  });

  it('renders correctly with simple data', function () {
    Factory.create('list', { _id: listId });
    const timestamp = new Date();
    const todos = _.times(3, i => Factory.create('todo', {
      listId,
      createdAt: new Date(timestamp - (3 - i)),
    }));

    withRenderedTemplate('Lists_show_page', {}, el => {
      const todosText = todos.map(t => t.text).reverse();
      const renderedText = $(el).find('.list-items input[type=text]')
        .map((i, e) => $(e).val())
        .toArray();
      chai.assert.deepEqual(renderedText, todosText);
    });
  });
});
```

在该测试中，我们需要特别注意以下几点：

<h4 id="simple-integration-test-importing">Importing</h4>

由于我们会像在单元测试中那样，使用相同方式来运行该测试。因此我们也需要像在[单元测试](#simple-integration-test-importing)中那样，在测试代码中 `import` 相互关联紧密的模块。

<h4 id="simple-integration-test-stubbing">Stubbing</h4>

由于集成测试中的系统拥有更大的表层区域，我们需要 stub 应用栈在集成过程中需要但在测试中需要模拟的其余的更多要点。这里需要特别注意，我们使用了[`hwillson:stub-collections`](#mocking-the-database)和[Sinon](http://sinonjs.org)两个包来 stub 了 Flow Router 和 Subscription 部分。

<h4 id="simple-integration-test-data">创建数据</h4>

在这次测试中，我们使用了[Factory package](#test-data)的 `.create()` API，它会将数据插入到真实的数据库集合中。然而，由于我们将所有 `Todos` 和 `Lists` 集合的方法代理至一个本地集合上（这也是 `hwillson:stub-collections` 所做的事），我们不会因为尝试从客户端执行数据插入而造成任何麻烦。

就像我们上面的[单元测试](#running-unit-tests)那样，本次集成测试能够以与其完全一致的方式来运行。

<h3 id="full-app-integration-test">全应用的集成测试</h3>

在[Todos](https://github.com/meteor/todos)这个示例应用程序中，我们有一个测试，是用来保证当我们路由至某个地址时，能够看到一个列表的完整内容。这个测试展示了集成测试中的一些技巧。

[`imports/startup/client/routes.app-test.js`](https://github.com/meteor/todos/blob/master/imports/startup/client/routes.app-test.js):

```js
/* eslint-env mocha */
/* eslint-disable func-names, prefer-arrow-callback */

import { Meteor } from 'meteor/meteor';
import { Tracker } from 'meteor/tracker';
import { DDP } from 'meteor/ddp-client';
import { FlowRouter } from 'meteor/kadira:flow-router';
import { assert } from 'meteor/practicalmeteor:chai';
import { Promise } from 'meteor/promise';
import { $ } from 'meteor/jquery';

import { denodeify } from '../../utils/denodeify';
import { generateData } from './../../api/generate-data.app-tests.js';
import { Lists } from '../../api/lists/lists.js';
import { Todos } from '../../api/todos/todos.js';


// Utility -- 返回一个 promise 对象，该对象在所有订阅完成时转变为 resolve 状态
const waitForSubscriptions = () => new Promise(resolve => {
  const poll = Meteor.setInterval(() => {
    if (DDP._allSubscriptionsReady()) {
      Meteor.clearInterval(poll);
      resolve();
    }
  }, 200);
});

// Tracker.afterFlush 会等待所有依赖于 tracker 的变更（例如 route 的变化）发生并获得
// 结果之后才执行其内部代码。本例中将会把这个方法 Promise 化。
const afterFlushPromise = denodeify(Tracker.afterFlush);

if (Meteor.isClient) {
  describe('data available when routed', () => {
    // 首先，确保我们所期望的数据已经在服务端加载。
	// 然后，路由至应用首页
    beforeEach(() => generateData()
      .then(() => FlowRouter.go('/'))
      .then(waitForSubscriptions)
    );

    describe('when logged out', () => {
      it('has all public lists at homepage', () => {
        assert.equal(Lists.find().count(), 3);
      });

      it('renders the correct list when routed to', () => {
        const list = Lists.findOne();
        FlowRouter.go('Lists.show', { _id: list._id });

        return afterFlushPromise()
          .then(waitForSubscriptions)
          .then(() => {
            assert.equal($('.title-wrapper').html(), list.name);
            assert.equal(Todos.find({ listId: list._id }).count(), 3);
          });
      });
    });
  });
}
```

这里需要注意：

 - 在运行之前，每个测试都会使用 `generateData` helper 来设置它们各自所需要的数据（参见[为集成测试创建数据]这一部分来获得更多信息），然后跳转到首页。

 - 尽管 Flow Router 不使用完成后的回调，我们仍然可以使用 `Tracker.afterFlush` 来等待它全部的响应式结果完成。

 - 这里我们写了一些小工具（可以被抽象为一个普通的包），用于等待所有由路由创建的订阅（比如本例中的 `todos.inList` 订阅）完成，以避免过早地检查它们的数据。

<h3 id="running-full-app-tests">运行全应用测试</h3>

要在应用程序中运行[全应用测试](#test-modes)，我们需要运行如下命令：

```txt
meteor test --full-app --driver-package practicalmeteor:mocha
```

当我们在浏览器中连接到一个测试实例中，我们会希望渲染一个测试 UI 而不是应用中的 UI，因此 `mocha-web-reporter` 包会隐藏任何来自于我们应用程序中的 UI，并使用其自有的 UI 来覆盖应用的 UI。尽管如此，应用却仍然能够像平常一样运行，因此我们可以路由至应用中的各处，并检查正确的数据是否被加载。

<h3 id="creating-integration-test-data">创建数据</h3>

要在全应用测试模式中创建测试数据，通常比较有意义的做法是创建一些特殊的，在客户端调用的测试方法。通常在测试一个完整的应用时，我们希望确保各个发布所发送的都是正确的数据（就像我们在这次测试中所做的那样），这样并不足以构成模拟数据集合并将模拟数据放入其中的理由。相反，我们更愿意在服务端真实地创建数据并将其发布。

就像我们在前面的[测试数据](#test-data)这一部分中，在 `beforeEach` 中使用一个方法来清除数据库那样，我们也能在现在的测试执行前调用一个方法（来生成数据）。在这个针对路由的测试案例中，我们使用的[`imports/api/generate-data.app-tests.js`](https://github.com/meteor/todos/blob/master/imports/api/generate-data.app-tests.js)这个文件，其中便包含了这个方法的定义（它仅会在全应用测试模式中被加载，因此平常是不可用的）：

```js
// 这个文件将被自动导入至应用测试的上下文中，
// 这确保了其中定义的方法始终可用

import { Meteor } from 'meteor/meteor';
import { Factory } from 'meteor/factory';
import { resetDatabase } from 'meteor/xolvio:cleaner';
import { Random } from 'meteor/random';
import { _ } from 'meteor/underscore';

import { denodeify } from '../utils/denodeify';

const createList = (userId) => {
  const list = Factory.create('list', { userId });
  _.times(3, () => Factory.create('todo', { listId: list._id }));
  return list;
};

// 记住，在添加这样一个方法前，你应当再次确认当前文件仅是测试可用的！
Meteor.methods({
  generateFixtures() {
    resetDatabase();

    // 创建3个公有的列表
    _.times(3, () => createList());

    // 创建3个私有的列表
    _.times(3, () => createList(Random.id()));
  },
});

let generateData;
if (Meteor.isClient) {
  // 单独创建一个到服务器的连接，用于调用测试数据的创建方法。
  // 我们这样做的目的是防止当前测试用户所在的连接上出现竞态。
  const testConnection = Meteor.connect(Meteor.absoluteUrl());

  generateData = denodeify((cb) => {
    testConnection.call('generateFixtures', cb);
  });
}

export { generateData };
```

注意，我们导出了一个客户端标记 `generateData`，它是一个 promise 化版本的方法调用，这样做可以使它在测试的连续调用场景中更加易用。

另外还需要注意我们使用这个连接到服务端，专用于发送测试『控制』方法调用的单独连接的方式。

<h2 id="acceptance-testing">验收测试</h2>

验收测试是一个利用我们应用的未修改版本，从『外』对其进行测试以保证其像我们所期望那样工作的过程。通常来讲，如果一个应用通过了验收测试，那么从产品角度来讲，我们已经彻底地完成了我们的工作。

由于验收测试是在一个完整的浏览器上下文中，以正常使用的方式来测试应用程序的行为，因此有着非常多的工具可用于描述和运行这些测试。在本指南中，我们将会演示使用[Chimp](https://chimp.readme.io)，这个工具具备了一些简洁的、 Meteor 独有的特性，使得它非常易用。

Chimp 需要的 node 版本为4或5。你可以通过运行如下命令来检查你的 node 版本：

```sh
node -v
```

你可以从[nodejs.org](https://nodejs.org/en/download/)下载安装第4版，或者使用 `brew install node` 来安装第5版。然后使用如下的命令来全局安装 Chimp：

```sh
npm install --global chimp
```

> 注意，你也可以将 Chimp 安装为 `package.json` 文件中的 `devDependency`，但由于其本身包含了一些二进制依赖，你可能会在部署过程中遇到一些麻烦。要避免这一问题，你可以在部署前运行 `meteor npm prune` 命令来移除非生产环境的依赖。

Chimp 拥有大量可选的设置选项，不过我们可以添加一些 npm 脚本，这些脚本会在我们定义在 Chimp 中的两种主要模式下运行当前的测试。我们可以将这些脚本添加到 `package.json` 文件中：

```json
{
  "scripts": {
    "chimp-watch": "chimp --ddp=http://localhost:3000 --watch --mocha --path=tests",
    "chimp-test": "chimp --mocha --path=tests"
  }
}
```

Chimp 现在会从 `tests/` 目录（本来是被 Meteor 工具所忽略的目录）中查找你所定义的验收测试。在[Todos](https://github.com/meteor/todos)这个示例应用中，我们定义了一个简单的测试，用于确保我们能够点击 "create list" 这个按钮。

[`tests/lists.js`](https://github.com/meteor/todos/blob/master/tests/lists.js):

```js
/* eslint-env mocha */
/* eslint-disable func-names, prefer-arrow-callback */

// 这是 Chimp 的全局环境
/* globals browser assert server */

function countLists() {
  browser.waitForExist('.list-todo');
  const elements = browser.elements('.list-todo');
  return elements.value.length;
};

describe('list ui', function () {
  beforeEach(function () {
    browser.url('http://localhost:3000');
    server.call('generateFixtures');
  });

  it('can create a list @watch', function () {
    const initialCount = countLists();

    browser.click('.js-new-list');

    assert.equal(countLists(), initialCount + 1);
  });
});
```

<h3 id="running-acceptance-tests">运行验收测试</h3>

要运行验收测试，我们仅需要像平常一样启动 Meteor 应用，然后运行 Chimp。

在第一个终端中，我们可以：

```bash
meteor
```

另一个终端：

```bash
meteor npm run chimp-watch
```

然后，`chimp-watch` 命令会在浏览器中运行这一指定的测试，并且在我们变更了该测试的代码或者应用程序代码后持续地重新运行（注意，本次测试假定我们在 `3000` 这个端口号上运行应用）。


因此，这是一个开发测试代码的好方法——这也是为什么 chimp 拥有这样一个特性：在我们需要进行相关工作的测试的名称上加上 `@watch` 标记以便调起这些测试（在一个大型应用程序中，运行全部验收测试套件会非常耗时）。

`chimp-test` 命令将*只运行所有的测试一次*，便于执行通过我们的测试套件传入的测试，可以作为一个手动的步骤，或是[持续集成](#ci)过程的一部分。


<h3 id="creating-acceptance-test-data">创建数据</h3>

尽管我们可以像前面所做的那样，对『纯』Meteor 应用运行验收测试，但使用一个特定的测试驱动 `tmeasday:acceptance-test-driver`（你需要使用 `meteor add` 命令来添加这个包），来启动一个 meteor 服务器通常是有意义的：

```txt
meteor test --full-app --driver-package tmeasday:acceptance-test-driver
```

运行我们的验收测试套件的优势主要集中在应用在全应用测试模式下运行时，我们所创建的[数据生成方法](#creating-integration-test-data)仍然是有效的。除此之外，`acceptance-test-driver` 不会做任何其他事。

在 Chimp 测试中，你可以使用 `server` 变量来获得连接至服务端的 DDP 连接。因此你可以使用 `server.call()`（在 Chimp 测试中已经被包裹为同步方法）来调用这些方法。这是一条在验收测试和集成测试之间共享数据预制代码的捷径。


<h2 id="ci">持续集成</h2>

持续集成测试是一个为你的项目中的每一次代码提交运行测试用例的过程。

有两个主要的方式可以用于执行持续集成测试：在每一次提交的代码允许被推送到主要的代码库之前，在开发人员的机器上，和一台专用的 CI 服务器上同时执行。两种技巧都非常有用，并且都需要在仅限命令行的模式下运行测试。

<h3 id="command-line">命令行</h3>

我们已经了解过一个通过命令行运行测试的例子，这个例子使用了 `meteor npm run chimp-test` 这一模式。

我们可以使用为 Mocha 开发的命令行驱动[`dispatch:mocha-phantomjs`](http://atmospherejs.com/dispatch/mocha-phantomjs)，在命令行中运行我们的标准测试。

添加和使用这个包的方式简单易懂：

```bash
meteor add dispatch:mocha-phantomjs
meteor test --once --driver-package dispatch:mocha-phantomjs
```

（`--once` 参数确保当前的 Meteor 进程会在测试完成后立即停止）。

我们也可以将该命令作为 `test` 脚本添加至 `package.json` 文件中：

```json
{
  "scripts": {
    "test": "meteor test --once --driver-package dispatch:mocha-phantomjs"
  }
}
```

现在我们可以通过命令 `meteor npm test` 来运行以上测试。

<h3 id="using-circle-ci">CircleCI</h3>

[CircleCI](https://circleci.com)是一个极好的持续集成服务，它允许我们在每一次有代码推送到像 GitHub 这样的代码库中时运行（可能非常耗时的）测试。要使用它来完成我们在前面所定义的命令行测试，我们可以参照他们的标准[入门教程](https://circleci.com/docs/getting-started)，并按照下面的示范完成一个类似的 `circle.yml` 文件：

```
machine:
  node:
    version: 0.10.43
dependencies:
  override:
    - curl https://install.meteor.com | /bin/sh
    - npm install
checkout:
  post:
    - git submodule update --init
```
