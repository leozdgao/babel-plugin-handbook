# Babel 插件开发指南

这篇文档涵盖了如何创建 [Babel](https://babeljs.io) [插件](https://babeljs.io/docs/advanced/plugins/)等方面的内容。

[![cc-by-4.0](https://licensebuttons.net/l/by/4.0/80x15.png)](http://creativecommons.org/licenses/by/4.0/)

特别感谢 [@sebmck](https://github.com/sebmck/),
[@hzoo](https://github.com/hzoo),
[@jdalton](https://github.com/jdalton),
[@abraithwaite](https://github.com/abraithwaite),
[@robey](https://github.com/robey)，及其他人为本手册提供的慷慨帮助。

# Node 脚本化版本（Node Packaged Manuscript）

你可以使用 npm 安装这份指南，方法如下：

```sh
$ npm install -g babel-plugin-handbook
```

现在你可以使用 `babel-plugin-handbook` 命令并在 `$PAGER` 中打开这份文档。此外，你也可以继续阅读这份在线的版本。

# 翻译

- [简体中文](/translations/zh-hans.md)
- [繁體中文](/translations/zh-tw.md)
- [Deutsche](/translations/de.md)
- [Español](/translations/es.md)
- [Français](/translations/fr.md)
- [Italiano](/translations/it.md)
- [日本語](/translations/ja.md)
- [한국어](/translations/ko.md)
- [Nederlands](/translations/nl.md)
- [Português](/translations/pl.md)
- [Pусский](/translations/ru.md)

# 目录

> 译注：本指南中有许多标题都是代码中的专有词汇，若是直接按字面翻译成中文反而会破坏阅读时的一致性和流畅性。因此特作如下约定：
> 1. 适合于做字面翻译的专有词汇将原文写在后面用 `（）` 括起以助对照，如 `抽象语法树（ASTs）` 等；
> 2. 不适合做字面翻译的直接保留原文，但将字面翻译写在后面用 `（）` 括起以助理解，如 `Builders（构造器）` 等；

- [介绍](#介绍)
- [基础](#基础)
  - [抽象语法树（ASTs）](#抽象语法树asts)
  - [Babel 的处理步骤](#babel-的处理步骤)
    - [解析](#解析)
      - [词法分析](#词法分析)
      - [语法分析](#语法分析)
    - [转换](#转换)
    - [生成](#生成)
  - [遍历](#遍历)
    - [Visitors（访问者）](#visitors访问者)
    - [Paths（路径）](#paths路径)
      - [Paths in Visitors（存在于访问者中的路径）](#paths-in-visitors存在于访问者中的路径)
    - [State（状态）](#state状态)
    - [Scopes（作用域）](#scopes作用域)
      - [Bindings（绑定）](#bindings绑定)
- [API](#api)
  - [babylon](#babylon)
  - [babel-traverse](#babel-traverse)
  - [babel-types](#babel-types)
    - [Definitions（定义）](#definitions定义)
    - [Builders（构造器）](#builders构造器)
    - [Validators（验证器）](#validators验证器)
    - [Converters（变换器）](#converters变换器)
  - [babel-generator](#babel-generator)
  - [babel-template](#babel-template)
- [编写你的第一个 Babel 插件](#编写你的第一个-babel-插件)
- [转换操作](#转换操作)
  - [访问](#访问)
    - [检查节点是否为某种特定类型](#检查节点是否为某种特定类型)
    - [检查标识符是否正在被引用着](#检查标识符是否正在被引用着)
  - [处理](#处理)
    - [替换节点](#替换节点)
    - [用多个节点替换一个节点](#用多个节点替换一个节点)
    - [用字符串源码替换节点](#用字符串源码替换节点)
    - [插入同级节点](#插入同级节点)
    - [移除节点](#移除节点)
    - [替换父节点](#替换父节点)
    - [移除父节点](#移除父节点)
  - [Scope（作用域）](#scope-作用域)
    - [检查本地变量是否有绑定](#检查本地变量是否有绑定)
    - [生成唯一标识符（UID）](#生成唯一标识符-uid)
    - [提升变量声明至父级作用域](#提升变量声明至父级作用域)
    - [重命名绑定及其引用](#重命名绑定及其引用)
- [插件选项](#插件选项)
- [构建节点](#构建节点)
- [最佳实践](#最佳实践)
  - [尽量避免遍历抽象语法树（AST）](#尽量避免遍历抽象语法树-ast)
    - [及时合并访问者对象](#及时合并访问者对象)
    - [可以手动查找就不要遍历](#可以手动查找就不要遍历)
  - [优化嵌套的访问者对象](#优化嵌套的访问者对象)
  - [留意嵌套结构](#留意嵌套结构)

# 介绍

Babel 是一个通用的多功能的 JavaScript 编译器。此外它还拥有众多模块可用于不同形式的静态分析。

> 静态分析是在不需要执行代码的前提下对代码进行分析的处理过程（执行代码的同时进行代码分析即是动态分析）。
> 静态分析的目的是多种多样的，它可用于语法检查，编译，代码高亮，代码转换，优化，压缩等等场景。

你可以使用 Babel 创建多种类型的工具来帮助你更有效率并且写出更好的程序。

# 基础

Babel 是 JavaScript 编译器，更确切地说是源码到源码的编译器，通常也叫做“转换编译器（transpiler）”。意思是说你为 Babel 提供一些 JavaScript 代码，Babel 更改这些代码，然后返回给你新生成的代码。

## 抽象语法树（ASTs）

这个处理过程中的每一步都涉及到创建或是操作[抽象语法树](https://en.wikipedia.org/wiki/Abstract_syntax_tree)，亦称 AST。

> Babel 使用一个基于 [ESTree](https://github.com/estree/estree) 并修改过的 AST，它的内核说明文档可以在[这里](https://github.com/babel/babel/blob/master/doc/ast/spec.md)找到。

```js
function square(n) {
  return n * n;
}
```

> [AST Explorer](http://astexplorer.net/) 可以让你对 AST 节点有一个更好的感性认识。[这里](http://astexplorer.net/#/Z1exs6BWMq)是上述代码的一个示例链接。

同样的程序可以表述为下面的列表：

```md
- FunctionDeclaration:
  - id:
    - Identifier:
      - name: square
  - params [1]
    - Identifier
      - name: n
  - body:
    - BlockStatement
      - body [1]
        - ReturnStatement
          - argument
            - BinaryExpression
              - operator: *
              - left
                - Identifier
                  - name: n
              - right
                - Identifier
                  - name: n
```

或是如下所示的 JavaScript Object（对象）：

```javascript
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  params: [{
    type: "Identifier",
    name: "n"
  }],
  body: {
    type: "BlockStatement",
    body: [{
      type: "ReturnStatement",
      argument: {
        type: "BinaryExpression",
        operator: "*",
        left: {
          type: "Identifier",
          name: "n"
        },
        right: {
          type: "Identifier",
          name: "n"
        }
      }
    }]
  }
}
```

你会留意到 AST 的每一层都拥有相同的结构：

```javascript
{
  type: "FunctionDeclaration",
  id: {...},
  params: [...],
  body: {...}
}
```

```javascript
{
  type: "Identifier",
  name: ...
}
```

```javascript
{
  type: "BinaryExpression",
  operator: ...,
  left: {...},
  right: {...}
}
```

> 注意：出于简化的目的移除了某些属性

这样的每一层结构也被叫做 **节点（Node）**。一个 AST 可以由单一的节点或是成百上千个节点构成。它们组合在一起可以描述用于静态分析的程序语法。

每一个节点都有如下所示的接口（Interface）：

```typescript
interface Node {
  type: string;
}
```

字符串形式的 `type` 字段表示节点的类型（如：`"FunctionDeclaration"`，`"Identifier"`，或 `"BinaryExpression"`）。每一种类型的节点定义了一些附加属性用来进一步描述该节点类型。

Babel 还为每个节点额外生成了一些属性，用于描述该节点在原始代码中的位置。

```javascript
{
  type: ...,
  start: 0,
  end: 38,
  loc: {
    start: {
      line: 1,
      column: 0
    },
    end: {
      line: 3,
      column: 1
    }
  },
  ...
}
```

每一个节点都会有 `start`，`end`，`loc` 这几个属性。

## Babel 的处理步骤

Babel 的三个主要处理步骤分别是： **解析（parse）**，**转换（transform）**，**生成（generate）**。

### 解析

**解析**步骤接收代码并输出 AST。这个步骤分为两个阶段：[**词法分析（Lexical Analysis）**](https://en.wikipedia.org/wiki/Lexical_analysis) 和 [**语法分析（Syntactic Analysis）**](https://en.wikipedia.org/wiki/Parsing)。

#### 词法分析

词法分析阶段把字符串形式的代码转换为 **令牌（tokens）** 流。

你可以把令牌看作是一个扁平的语法片段数组：

```javascript
n * n;
```

```javascript
[
  { type: { ... }, value: "n", start: 0, end: 1, loc: { ... } },
  { type: { ... }, value: "*", start: 2, end: 3, loc: { ... } },
  { type: { ... }, value: "n", start: 4, end: 5, loc: { ... } },
  ...
]
```

每一个 `type` 有一组属性来描述该令牌：

```javascript
{
  type: {
    label: 'name',
    keyword: undefined,
    beforeExpr: false,
    startsExpr: true,
    rightAssociative: false,
    isLoop: false,
    isAssign: false,
    prefix: false,
    postfix: false,
    binop: null,
    updateContext: null
  },
  ...
}
```

和 AST 节点一样它们也有 `start`，`end`，`loc` 属性。

#### 语法分析

语法分析阶段会把一个令牌流转换成 AST 的形式。这个阶段会使用令牌中的信息把它们转换成一个 AST 的表述结构，这样更易于后续的操作。
                                                                                  v
### 转换

[转换](https://en.wikipedia.org/wiki/Program_transformation)步骤接收 AST 并对其进行遍历，在此过程中对节点进行添加、更新及移除等操作。这是 Babel 或是其他编译器中最复杂的过程同时也是插件将要介入工作的部分，这将是本手册的主要内容，因此让我们慢慢来。

### 生成

[代码生成](https://en.wikipedia.org/wiki/Code_generation_(compiler))步骤把最终（经过一系列转换之后）的 AST转换成字符串形式的代码，同时创建[源码映射（source maps）](http://www.html5rocks.com/en/tutorials/developertools/sourcemaps/)。

代码生成其实很简单：深度优先遍历整个 AST，然后构建可以表示转换后代码的字符串。

## 遍历

想要转换 AST 你需要进行递归的[树形遍历](https://en.wikipedia.org/wiki/Tree_traversal)。

比方说我们有一个 `FunctionDeclaration` 类型。它有几个属性：`id`，`params`，和 `body`，每一个都有一些内嵌节点。

```javascript
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  params: [{
    type: "Identifier",
    name: "n"
  }],
  body: {
    type: "BlockStatement",
    body: [{
      type: "ReturnStatement",
      argument: {
        type: "BinaryExpression",
        operator: "*",
        left: {
          type: "Identifier",
          name: "n"
        },
        right: {
          type: "Identifier",
          name: "n"
        }
      }
    }]
  }
}
```

于是我们从 `FunctionDeclaration` 开始并且我们知道它的内部属性（即：`id`，`params`，`body`），所以我们依次访问每一个属性及它们的子节点。

接着我们来到 `id`，它是一个 `Identifier`。`Identifier` 没有任何子节点属性，所以我们继续。

之后是 `params`，由于它是一个数组节点所以我们访问其中的每一个，它们都是 `Identifier` 类型的单一节点，然后我们继续。

此时我们来到了 `body`，这是一个 `BlockStatement` 并且也有一个 `body`节点，而且也是一个数组节点，我们继续访问其中的每一个。

这里唯一的一个属性是 `ReturnStatement` 节点，它有一个 `argument`，我们访问 `argument` 就找到了 `BinaryExpression`。

`BinaryExpression` 有一个 `operator`，一个 `left`，和一个 `right`。 `operator` 不是一个节点，它只是一个值因此我们不用继续向内遍历，我们只需要访问 `left` 和 `right`。

Babel 的转换步骤全都是这样的遍历过程。

### Visitors（访问者）

When we talk about "going" to a node, we actually mean we are **visiting** them.
The reason we use that term is because there is this concept of a
[**visitor**](https://en.wikipedia.org/wiki/Visitor_pattern).

Visitors are a pattern used in AST traversal across languages. Simply put they
are an object with methods defined for accepting particular node types in a
tree. That's a bit abstract so let's look at an example.

```js
const MyVisitor = {
  Identifier() {
    console.log("Called!");
  }
};
```

> **Note:** `Identifier() { ... }` is shorthand for
> `Identifier: { enter() { ... } }`.

This is a basic visitor that when used during a traversal will call the
`Identifier()` method for every `Identifier` in the tree.

So with this code the `Identifier()` method will be called four times with each
`Identifier` (including `square`).

```js
function square(n) {
  return n * n;
}
```
```js
Called!
Called!
Called!
Called!
```

These calls are all on node **enter**. However there is also the possibility of
calling a visitor method when on **exit**.

Imagine we have this tree structure:

```js
- FunctionDeclaration
  - Identifier (id)
  - Identifier (params[0])
  - BlockStatement (body)
    - ReturnStatement (body)
      - BinaryExpression (argument)
        - Identifier (left)
        - Identifier (right)
```

As we traverse down each branch of the tree we eventually hit dead ends where we
need to traverse back up the tree to get to the next node. Going down the tree
we **enter** each node, then going back up we **exit** each node.

Let's _walk_ through what this process looks like for the above tree.

- Enter `FunctionDeclaration`
  - Enter `Identifier (id)`
    - Hit dead end
  - Exit `Identifier (id)`
  - Enter `Identifier (params[0])`
    - Hit dead end
  - Exit `Identifier (params[0])`
  - Enter `BlockStatement (body)`
    - Enter `ReturnStatement (body)`
      - Enter `BinaryExpression (argument)`
        - Enter `Identifier (left)`
          - Hit dead end
        - Exit `Identifier (left)`
        - Enter `Identifier (right)`
          - Hit dead end
        - Exit `Identifier (right)`
      - Exit `BinaryExpression (argument)`
    - Exit `ReturnStatement (body)`
  - Exit `BlockStatement (body)`
- Exit `FunctionDeclaration`

So when creating a visitor you have two opportunities to visit a node.

```js
const MyVisitor = {
  Identifier: {
    enter() {
      console.log("Entered!");
    },
    exit() {
      console.log("Exited!");
    }
  }
};
```

### Paths（路径）

An AST generally has many Nodes, but how do Nodes relate to one another? We
could have one giant mutable object that you manipulate and have full access to,
or we can simplify this with **Paths**.

A **Path** is an object representation of the link between two nodes.

For example if we take the following node and its child:

```js
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  ...
}
```

And represent the child `Identifier` as a path, it looks something like this:

```js
{
  "parent": {
    "type": "FunctionDeclaration",
    "id": {...},
    ....
  },
  "node": {
    "type": "Identifier",
    "name": "square"
  }
}
```

It also has additional metadata about the path:

```js
{
  "parent": {...},
  "node": {...},
  "hub": {...},
  "contexts": [],
  "data": {},
  "shouldSkip": false,
  "shouldStop": false,
  "removed": false,
  "state": null,
  "opts": null,
  "skipKeys": null,
  "parentPath": null,
  "context": null,
  "container": null,
  "listKey": null,
  "inList": false,
  "parentKey": null,
  "key": null,
  "scope": null,
  "type": null,
  "typeAnnotation": null
}
```

As well as tons and tons of methods related to adding, updating, moving, and
removing nodes, but we'll get into those later.

In a sense, paths are a **reactive** representation of a node's position in the
tree and all sorts of information about the node. Whenever you call a method
that modifies the tree, this information is updated. Babel manages all of this
for you to make working with nodes easy and as stateless as possible.

#### Paths in Visitors（存在于访问者中的路径）

When you have a visitor that has a `Identifier()` method, you're actually
visiting the path instead of the node. This way you are mostly working with the
reactive representation of a node instead of the node itself.

```js
const MyVisitor = {
  Identifier(path) {
    console.log("Visiting: " + path.node.name);
  }
};
```

```js
a + b + c;
```

```js
Visiting: a
Visiting: b
Visiting: c
```

### State（状态）

State is the enemy of AST transformation. State will bite you over and over
again and your assumptions about state will almost always be proven wrong by
some syntax that you didn't consider.

Take the following code:

```js
function square(n) {
  return n * n;
}
```

Let's write a quick hacky visitor that will rename `n` to `x`.

```js
let paramName;

const MyVisitor = {
  FunctionDeclaration(path) {
    const param = path.node.params[0];
    paramName = param.name;
    param.name = "x";
  },

  Identifier(path) {
    if (path.node.name === paramName) {
      path.node.name = "x";
    }
  }
};
```

This might work for the above code, but we can easily break that by doing this:

```js
function square(n) {
  return n * n;
}
n;
```

The better way to deal with this is recursion. So let's make like a Christopher
Nolan film and put a visitor inside of a visitor.

```js
const updateParamNameVisitor = {
  Identifier(path) {
    if (path.node.name === this.paramName) {
      path.node.name = "x";
    }
  }
};

const MyVisitor = {
  FunctionDeclaration(path) {
    const param = path.node.params[0];
    const paramName = param.name;
    param.name = "x";

    path.traverse(updateParamNameVisitor, { paramName });
  }
};
```

Of course, this is a contrived example but it demonstrates how to eliminate
global state from your visitors.

### Scopes（作用域）

Next let's introduce the concept of a
[**scope**](https://en.wikipedia.org/wiki/Scope_(computer_science)). JavaScript
has [lexical scoping](https://en.wikipedia.org/wiki/Scope_(computer_science)#Lexical_scoping_vs.\_dynamic_scoping),
which is a tree structure where blocks create new scope.

```js
// global scope

function scopeOne() {
  // scope 1

  function scopeTwo() {
    // scope 2
  }
}
```

Whenever you create a reference in JavaScript, whether that be by a variable,
function, class, param, import, label, etc., it belongs to the current scope.

```js
var global = "I am in the global scope";

function scopeOne() {
  var one = "I am in the scope created by `scopeOne()`";

  function scopeTwo() {
    var two = "I am in the scope created by `scopeTwo()`";
  }
}
```

Code within a deeper scope may use a reference from a higher scope.

```js
function scopeOne() {
  var one = "I am in the scope created by `scopeOne()`";

  function scopeTwo() {
    one = "I am updating the reference in `scopeOne` inside `scopeTwo`";
  }
}
```

A lower scope might also create a reference of the same name without modifying
it.

```js
function scopeOne() {
  var one = "I am in the scope created by `scopeOne()`";

  function scopeTwo() {
    var one = "I am creating a new `one` but leaving reference in `scopeOne()` alone.";
  }
}
```

When writing a transform, we want to be wary of scope. We need to make sure we
don't break existing code while modifying different parts of it.

We may want to add new references and make sure they don't collide with existing
ones. Or maybe we just want to find where a variable is referenced. We want to
be able to track these references within a given scope.

A scope can be represented as:

```js
{
  path: path,
  block: path.node,
  parentBlock: path.parent,
  parent: parentScope,
  bindings: [...]
}
```

When you create a new scope you do so by giving it a path and a parent scope.
Then during the traversal process it collects all the references ("bindings")
within that scope.

Once that's done, there's all sorts of methods you can use on scopes. We'll get
into those later though.

#### Bindings（绑定）

References all belong to a particular scope; this relationship is known as a
**binding**.

```js
function scopeOnce() {
  var ref = "This is a binding";

  ref; // This is a reference to a binding

  function scopeTwo() {
    ref; // This is a reference to a binding from a lower scope
  }
}
```

A single binding looks like this:

```js
{
  identifier: node,
  scope: scope,
  path: path,
  kind: 'var',

  referenced: true,
  references: 3,
  referencePaths: [path, path, path],

  constant: false,
  constantViolations: [path]
}
```

With this information you can find all the references to a binding, see what
type of binding it is (parameter, declaration, etc.), lookup what scope it
belongs to, or get a copy of its identifier. You can even tell if it's
constant and if not, see what paths are causing it to be non-constant.

Being able to tell if a binding is constant is useful for many purposes, the
largest of which is minification.

```js
function scopeOne() {
  var ref1 = "This is a constant binding";

  becauseNothingEverChangesTheValueOf(ref1);

  function scopeTwo() {
    var ref2 = "This is *not* a constant binding";
    ref2 = "Because this changes the value";
  }
}
```

----

# API

Babel 拥有众多模块，在这个部分我们会过一遍其中的几个主要模块，并解释一下它们的作用以及如何使用它们。

> 注意：这并不是一份具体的API文档，具体的API文档之后会马上发布。

## [`babylon`](https://github.com/babel/babel/tree/master/packages/babylon)

Babylon 是 Babel 的解析器。刚开始的时候它是从 Acorn 那里 fork 而来的，它高效且易用，提供基于插件的架构，从而可以解析非标准的功能（同时也是未来的标准）。

首先，让我们先安装它。

```sh
$ npm install --save babylon
```

我们从解析一段简单的代码开始：

```js
import * as babylon from "babylon";

const code = `function square(n) {
  return n * n;
}`;

babylon.parse(code);
// Node {
//   type: "File",
//   start: 0,
//   end: 38,
//   loc: SourceLocation {...},
//   program: Node {...},
//   comments: [],
//   tokens: [...]
// }
```

我们也可以给 `parse()` 传入一些选项，比如：

```js
babylon.parse(code, {
  sourceType: "module", // default: "script"
  plugins: ["jsx"] // default: []
});
```

`sourceType` 的值可以是 `"module"` 或是 `"script"`，它代表Babylon的解析模式。`"module"` 将在
严格模式（strict mode）下解析并允许模块声明，`"script"` 则不行。

> 注意： `sourceType` 默认值为 `"script"`，如果代码中包含 `import` 或者是 `export` 的时候会报
错。设置 `sourceType: "module"` 就不会有这个错误了。

既然 Babylon 使用了基于插件的架构模式，那么就有 `plugins` 选项来设置启用的内部插件。注意，Babylon
还没有暴露它的API给外部插件，不过未来可能会有的。

想查看完整的插件列表，可以看 Babylon 的
[README文件](https://github.com/babel/babel/blob/master/packages/babylon/README.md#plugins)。

## [`babel-traverse`](https://github.com/babel/babel/tree/master/packages/babel-traverse)

Babel Traverse 模块维护了整颗 AST 的状态，并负责替换、删除和添加节点（ Node ）。

通过运行下面的命令来安装它：

```sh
$ npm install --save babel-traverse
```

我们让它和 Babylon 配合使用，来遍历和更新节点（ nodes ）：

```js
import * as babylon from "babylon";
import traverse from "babel-traverse";

const code = `function square(n) {
  return n * n;
}`;

const ast = babylon.parse(code);

traverse(ast, {
  enter(path) {
    if (
      path.node.type === "Identifier" &&
      path.node.name === "n"
    ) {
      path.node.name = "x";
    }
  }
});
```

## [`babel-types`](https://github.com/babel/babel/tree/master/packages/babel-types)

Babel Types 是一个等同于 Lodash 的工具类，主要针对 AST 节点。它包括构建、验证和转化 AST 节点。
使用这些工具方法对于理清 AST 的操作逻辑是很有帮助的。

通过运行下面的命令来安装它：

```sh
$ npm install --save babel-types
```

接下来我们开始使用它：

```js
import traverse from "babel-traverse";
import * as t from "babel-types";

traverse(ast, {
  enter(path) {
    if (t.isIdentifier(path.node, { name: "n" })) {
      path.node.name = "x";
    }
  }
});
```

### Definitions（定义）

Babel Types 对每一种类型的节点都有一个定义，每个定义都有各自的属性来表示节点信息，比如什么值是合理的、
如何来构建这个节点、节点应该怎么被遍历以及节点的别名。

例如下面这个单个节点的类型定义：

```js
defineType("BinaryExpression", {
  builder: ["operator", "left", "right"],
  fields: {
    operator: {
      validate: assertValueType("string")
    },
    left: {
      validate: assertNodeType("Expression")
    },
    right: {
      validate: assertNodeType("Expression")
    }
  },
  visitor: ["left", "right"],
  aliases: ["Binary", "Expression"]
});
```

### Builders（构造器）

你会发现在上例的定义中 `BinaryExpression` 类型有一个字段 `builder`。

```js
builder: ["operator", "left", "right"]
```

这是因为每一个节点类型都一个构造器方法（ builder method ），在使用的时候就像这样：

```js
t.binaryExpression("*", t.identifier("a"), t.identifier("b"));
```

它创建的 AST 如下：

```js
{
  type: "BinaryExpression",
  operator: "*",
  left: {
    type: "Identifier",
    name: "a"
  },
  right: {
    type: "Identifier",
    name: "b"
  }
}
```

实际输出的代码就是这样：

```js
a * b
```

构造器（ Builders ）会验证它们将要创建的节点，如果对构造器的使用不当，那么会有一个带有详细描述
的错误被抛出。这引出了下面的另一类方法。

### Validators（验证器）

`BinaryExpression` 的定义中包括属性 `fields`，它表示这个定义中的节点信息以及如何验证这些节点。

```js
fields: {
  operator: {
    validate: assertValueType("string")
  },
  left: {
    validate: assertNodeType("Expression")
  },
  right: {
    validate: assertNodeType("Expression")
  }
}
```

有两种用来验证的方法。第一种是 `isX` 形式：

```js
t.isBinaryExpression(maybeBinaryExpressionNode);
```

这个测试是用来确保节点是一个二元表达式，你也可以传递第二个参数，来确保节点需要包含的属性和值：

```js
t.isBinaryExpression(maybeBinaryExpressionNode, { operator: "*" });
```

这些方法还有一种更加断言式的版本，如果不满足断言，则会抛出错误，而不是返回 `true` 或者 `false`。

```js
t.assertBinaryExpression(maybeBinaryExpressionNode);
t.assertBinaryExpression(maybeBinaryExpressionNode, { operator: "*" });
// Error: Expected type "BinaryExpression" with option { "operator": "*" }
```

### Converters（变换器）

> [WIP]

## [`babel-generator`](https://github.com/babel/babel/tree/master/packages/babel-generator)

Babel Generator 是 Babel 的代码生成器，它接受一个 AST，并将它转换成代码，并生成 sourcemaps。

通过运行下面的命令来安装它：

```sh
$ npm install --save babel-generator
```

然后来使用它：

```js
import * as babylon from "babylon";
import generate from "babel-generator";

const code = `function square(n) {
  return n * n;
}`;

const ast = babylon.parse(code);

generate(ast, null, code);
// {
//   code: "...",
//   map: "..."
// }
```

你也可以传递一些选项给 `generate()`。

```js
generate(ast, {
  retainLines: false,
  compact: "auto",
  concise: false,
  quotes: "double",
  // ...
}, code);
```

## [`babel-template`](https://github.com/babel/babel/tree/master/packages/babel-template)

Babel Template 是另一个虽然小但格外有用的模块。它允许你通过占位符（ placeholders ）
的方式生成一段代码，而不需要构建一棵繁琐的 AST。

```sh
$ npm install --save babel-template
```

```js
import template from "babel-template";
import generate from "babel-generator";
import * as t from "babel-types";

const buildRequire = template(`
  var IMPORT_NAME = require(SOURCE);
`);

const ast = buildRequire({
  IMPORT_NAME: t.identifier("myModule"),
  SOURCE: t.stringLiteral("my-module")
});

console.log(generate(ast).code);
```

```js
var myModule = require("my-module");
```

# 编写你的第一个 Babel 插件

Now that you're familiar with all the basics of Babel, let's tie it together
with the plugin API.

Start off with a `function` that gets passed the current `babel` object.

```js
export default function(babel) {
  // plugin contents
}
```

Since you'll be using it so often, you'll likely want to grab just `babel.types`
like so:

```js
export default function({ types: t }) {
  // plugin contents
}
```

Then you return an object with a property `visitor` which is the primary visitor
for the plugin.

```js
export default function({ types: t }) {
  return {
    visitor: {
      // visitor contents
    }
  };
};
```

Let's write a quick plugin to show off how it works. Here's our source code:

```js
foo === bar;
```

Or in AST form:

```js
{
  type: "BinaryExpression",
  operator: "===",
  left: {
    type: "Identifier",
    name: "foo"
  },
  right: {
    type: "Identifier",
    name: "bar"
  }
}
```

We'll start off by adding a `BinaryExpression` visitor method.

```js
export default function({ types: t }) {
  return {
    visitor: {
      BinaryExpression(path) {
        // ...
      }
    }
  };
}
```

Then let's narrow it down to just `BinaryExpression`s that are using the `===`
operator.

```js
visitor: {
  BinaryExpression(path) {
    if (path.node.operator !== "===") {
      return;
    }

    // ...
  }
}
```

Now let's replace the `left` property with a new identifier:

```js
BinaryExpression(path) {
  if (path.node.operator !== "===") {
    return;
  }

  path.node.left = t.identifier("sebmck");
  // ...
}
```

Already if we run this plugin we would get:

```js
sebmck === bar;
```

Now let's just replace the `right` property.

```js
BinaryExpression(path) {
  if (path.node.operator !== "===") {
    return;
  }

  path.node.left = t.identifier("sebmck");
  path.node.right = t.identifier("dork");
}
```

And now for our final result:

```js
sebmck === dork;
```

Awesome! Our very first Babel plugin.

----

# 转换操作

## 访问

### 检查节点是否为某种特定类型

If you want to check what the type of a node is, the preferred way to do so is:

```js
BinaryExpression(path) {
  if (t.isIdentifier(path.node.left)) {
    // ...
  }
}
```

You can also do a shallow check for properties on that node:

```js
BinaryExpression(path) {
  if (t.isIdentifier(path.node.left, { name: "n" })) {
    // ...
  }
}
```

This is functionally equivalent to:

```js
BinaryExpression(path) {
  if (
    path.node.left != null &&
    path.node.left.type === "Identifier" &&
    path.node.left.name === "n"
  ) {
    // ...
  }
}
```

### 检查标识符是否正在被引用着

```js
Identifier(path) {
  if (path.isReferencedIdentifier()) {
    // ...
  }
}
```

Alternatively:

```js
Identifier(path) {
  if (t.isReferenced(path.node, path.parent)) {
    // ...
  }
}
```

## 处理

### 替换节点

```js
BinaryExpression(path) {
  path.replaceWith(
    t.binaryExpression("**", path.node.left, t.numberLiteral(2))
  );
}
```

```diff
  function square(n) {
-   return n * n;
+   return n ** 2;
  }
```

### 用多个节点替换一个节点

```js
ReturnStatement(path) {
  path.replaceWithMultiple([
    t.expressionStatement(t.stringLiteral("Is this the real life?")),
    t.expressionStatement(t.stringLiteral("Is this just fantasy?")),
    t.expressionStatement(t.stringLiteral("(Enjoy singing the rest of the song in your head)")),
  ]);
}
```

```diff
  function square(n) {
-   return n * n;
+   "Is this the real life?";
+   "Is this just fantasy?";
+   "(Enjoy singing the rest of the song in your head)";
  }
```

> **Note:** When replacing an expression with multiple nodes, they must be
> statements. This is because Babel uses heuristics extensively when replacing
> nodes which means that you can do some pretty crazy transformations that would
> be extremely verbose otherwise.

### 用字符串源码替换节点

```js
FunctionDeclaration(path) {
  path.replaceWithSourceString(`function add(a, b) {
    return a + b;
  }`);
}
```

```diff
- function square(n) {
-   return n * n;
+ function add(a, b) {
+   return a + b;
  }
```

> **Note:** It's not recommended to use this API unless you're dealing with
> dynamic source strings, otherwise it's more efficient to parse the code
> outside of the visitor.

### 插入同级节点

```js
FunctionDeclaration(path) {
  path.insertBefore(t.expressionStatement(t.stringLiteral("Because I'm easy come, easy go.")));
  path.insertAfter(t.expressionStatement(t.stringLiteral("A little high, little low.")));
}
```

```diff
+ "Because I'm easy come, easy go.";
  function square(n) {
    return n * n;
  }
+ "A little high, little low.";
```

> **Note:** This should always be a statement or an array of statements. This
> uses the same heuristics mentioned in
> [Replacing a node with multiple nodes](#replacing-a-node-with-multiple-nodes).

### 移除节点

```js
FunctionDeclaration(path) {
  path.remove();
}
```

```diff
- function square(n) {
-   return n * n;
- }
```

### 替换父节点

```js
BinaryExpression(path) {
  path.parentPath.replaceWith(
    t.expressionStatement(t.stringLiteral("Anyway the wind blows, doesn't really matter to me, to me."))
  );
}
```

```diff
  function square(n) {
-   return n * n;
+   "Anyway the wind blows, doesn't really matter to me, to me.";
  }
```

### 移除父节点

```js
BinaryExpression(path) {
  path.parentPath.remove();
}
```

```diff
  function square(n) {
-   return n * n;
  }
```

## Scope（作用域）

### 检查本地变量是否有绑定

```js
FunctionDeclaration(path) {
  if (path.scope.hasBinding("n")) {
    // ...
  }
}
```

This will walk up the scope tree and check for that particular binding.

You can also check if a scope has its **own** binding:

```js
FunctionDeclaration(path) {
  if (path.scope.hasOwnBinding("n")) {
    // ...
  }
}
```

### 生成唯一标识符（UID）

This will generate an identifier that doesn't collide with any locally defined
variables.

```js
FunctionDeclaration(path) {
  path.scope.generateUidIdentifier("uid");
  // Node { type: "Identifier", name: "_uid" }
  path.scope.generateUidIdentifier("uid");
  // Node { type: "Identifier", name: "_uid2" }
}
```

### 提升变量声明至父级作用域

Sometimes you may want to push a `VariableDeclaration` so you can assign to it.

```js
FunctionDeclaration(path) {
  const id = path.scope.generateUidIdentifierBasedOnNode(path.node.id);
  path.remove();
  path.scope.parent.push({ id, init: path.node });
}
```

```diff
- function square(n) {
+ var _square = function square(n) {
    return n * n;
- }
+ };
```

### 重命名绑定及其引用

```js
FunctionDeclaration(path) {
  path.scope.rename("n", "x");
}
```

```diff
- function square(n) {
-   return n * n;
+ function square(x) {
+   return x * x;
  }
```

Alternatively, you can rename a binding to a generated unique identifier:

```js
FunctionDeclaration(path) {
  path.scope.rename("n");
}
```

```diff
- function square(n) {
-   return n * n;
+ function square(_n) {
+   return _n * _n;
  }
```

----

# 插件选项

If you would like to let your users customize the behavior of your Babel plugin
you can accept plugin specific options which users can specify like this:

```js
{
  plugins: [
    ["my-plugin", {
      "option1": true,
      "option2": false
    }]
  ]
}
```

These options then get passed into plugin visitors through the `state` object:

```js
export default function({ types: t }) {
  return {
    visitor: {
      FunctionDeclaration(path, state) {
        console.log(state.opts);
        // { option1: true, option2: false }
      }
    }
  }
}
```

These options are plugin-specific and you cannot access options from other
plugins.

----

# 构建节点

When writing transformations you'll often want to build up some nodes to insert
into the AST. As mentioned previously, you can do this using the
[builder](#builder) methods in the [`babel-types`](#babel-types) package.

The method name for a builder is simply the name of the node type you want to
build except with the first letter lowercased. For example if you wanted to
build a `MemberExpression` you would use `t.memberExpression(...)`.

The arguments of these builders are decided by the node definition. There's some
work that's being done to generate easy-to-read documentation on the
definitions, but for now they can all be found
[here](https://github.com/babel/babel/tree/master/packages/babel-types/src/definitions).

A node definition looks like the following:

```js
defineType("MemberExpression", {
  builder: ["object", "property", "computed"],
  visitor: ["object", "property"],
  aliases: ["Expression", "LVal"],
  fields: {
    object: {
      validate: assertNodeType("Expression")
    },
    property: {
      validate(node, key, val) {
        let expectedType = node.computed ? "Expression" : "Identifier";
        assertNodeType(expectedType)(node, key, val);
      }
    },
    computed: {
      default: false
    }
  }
});
```

Here you can see all the information about this particular node type, including
how to build it, traverse it, and validate it.

By looking at the `builder` property, you can see the 3 arguments that will be
needed to call the builder method (`t.memberExpression`).

```js
builder: ["object", "property", "computed"],
```

> Note that sometimes there are more properties that you can customize on the
> node than the `builder` array contains. This is to keep the builder from
> having too many arguments. In these cases you need to set the properties
> manually. An example of this is
> [`ClassMethod`](https://github.com/babel/babel/blob/bbd14f88c4eea88fa584dd877759dd6b900bf35e/packages/babel-types/src/definitions/es2015.js#L238-L276).

You can see the validation for the builder arguments with the `fields` object.

```js
fields: {
  object: {
    validate: assertNodeType("Expression")
  },
  property: {
    validate(node, key, val) {
      let expectedType = node.computed ? "Expression" : "Identifier";
      assertNodeType(expectedType)(node, key, val);
    }
  },
  computed: {
    default: false
  }
}
```

You can see that `object` needs to be an `Expression`, `property` either needs
to be an `Expression` or an `Identifier` depending on if the member expression
is `computed` or not and `computed` is simply a boolean that defaults to
`false`.

So we can construct a `MemberExpression` by doing the following:

```js
t.memberExpression(
  t.identifier('object'),
  t.identifier('property')
  // `computed` is optional
);
```

Which will result in:

```js
object.property
```

However, we said that `object` needed to be an `Expression` so why is
`Identifier` valid?

Well if we look at the definition of `Identifier` we can see that it has an
`aliases` property which states that it is also an expression.

```js
aliases: ["Expression", "LVal"],
```

So since `MemberExpression` is a type of `Expression`, we could set it as the
`object` of another `MemberExpression`:

```js
t.memberExpression(
  t.memberExpression(
    t.identifier('member'),
    t.identifier('expression')
  ),
  t.identifier('property')
)
```

Which will result in:

```js
member.expression.property
```

It's very unlikely that you will ever memorize the builder method signatures for
every node type. So you should take some time and understand how they are
generated from the node definitions.

You can find all of the actual
[definitions here](https://github.com/babel/babel/tree/master/packages/babel-types/src/definitions)
and you can see them
[documented here](https://github.com/babel/babel/blob/master/doc/ast/spec.md)

----

# 最佳实践

> I'll be working on this section over the coming weeks.

## 尽量避免遍历抽象语法树（AST）

Traversing the AST is expensive, and it's easy to accidentally traverse the AST
more than necessary. This could be thousands if not tens of thousands of
extra operations.

Babel optimizes this as much as possible, merging visitors together if it can in
order to do everything in a single traversal.

### 及时合并访问者对象

When writing visitors, it may be tempting to call `path.traverse` in multiple
places where they are logically necessary.

```js
path.traverse({
  Identifier(path) {
    // ...
  }
});

path.traverse({
  BinaryExpression(path) {
    // ...
  }
});
```

However, it is far better to write these as a single visitor that only gets run
once. Otherwise you are traversing the same tree multiple times for no reason.

```js
path.traverse({
  Identifier(path) {
    // ...
  },
  BinaryExpression(path) {
    // ...
  }
});
```

### 可以手动查找就不要遍历

It may also be tempting to call `path.traverse` when looking for a particular
node type.

```js
const visitorOne = {
  Identifier(path) {
    // ...
  }
};

const MyVisitor = {
  FunctionDeclaration(path) {
    path.get('params').traverse(visitorOne);
  }
};
```

However, if you are looking for something specific and shallow, there is a good
chance you can manually lookup the nodes you need without performing a costly
traversal.

```js
const MyVisitor = {
  FunctionDeclaration(path) {
    path.node.params.forEach(function() {
      // ...
    });
  }
};
```

## 优化嵌套的访问者对象

When you are nesting visitors, it might make sense to write them nested in your
code.

```js
const MyVisitor = {
  FunctionDeclaration(path) {
    path.traverse({
      Identifier(path) {
        // ...
      }
    });
  }
};
```

However, this creates a new visitor object everytime `FunctionDeclaration()` is
called above, which Babel then needs to explode and validate every single time.
This can be costly, so it is better to hoist the visitor up.

```js
const visitorOne = {
  Identifier(path) {
    // ...
  }
};

const MyVisitor = {
  FunctionDeclaration(path) {
    path.traverse(visitorOne);
  }
};
```

If you need some state within the nested visitor, like so:

```js
const MyVisitor = {
  FunctionDeclaration(path) {
    var exampleState = path.node.params[0].name;

    path.traverse({
      Identifier(path) {
        if (path.node.name === exampleState) {
          // ...
        }
      }
    });
  }
};
```

You can pass it in as state to the `traverse()` method and have access to it on
`this` in the visitor.

```js
const visitorOne = {
  Identifier(path) {
    if (path.node.name === this.exampleState) {
      // ...
    }
  }
};

const MyVisitor = {
  FunctionDeclaration(path) {
    var exampleState = path.node.params[0].name;
    path.traverse(visitorOne, { exampleState });
  }
};
```

## 留意嵌套结构

Sometimes when thinking about a given transform, you might forget that the given
structure can be nested.

For example, imagine we want to lookup the `constructor` `ClassMethod` from the
`Foo` `ClassDeclaration`.

```js
class Foo {
  constructor() {
    // ...
  }
}
```

```js
const constructorVisitor = {
  ClassMethod(path) {
    if (path.node.name === 'constructor') {
      // ...
    }
  }
}

const MyVisitor = {
  ClassDeclaration(path) {
    if (path.node.id.name === 'Foo') {
      path.traverse(constructorVisitor);
    }
  }
}
```

We are ignoring the fact that classes can be nested and using the traversal
above we will hit a nested `constructor` as well:

```js
class Foo {
  constructor() {
    class Bar {
      constructor() {
        // ...
      }
    }
  }
}
```
