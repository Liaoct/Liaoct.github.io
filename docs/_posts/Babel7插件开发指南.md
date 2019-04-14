---
category: Tutorial
tags:
  - babel
title: Babel 7插件开发指南
vssue-title: Babel 7插件开发指南
---

Babel的三个主要处理步骤分别是：解析（parse）、转换（transform）、生成（generate）。

**解析**

解析步骤主要是接受源代码并输出抽象语法树（AST）。此步骤主要由`@babel/parser`（原`Babylon`）负责解析和理解js代码，输出对应的AST。

**转换**

转换步骤主要是接受AST，并对其进行遍历，在此过程中会进行分析和修改AST，这也是Babel插件主要工作的地方。此步骤主要用到`@babel/traverse`和`@babel/types`两个包。

**生成**

生成步骤主要是将（经过一系列转换之后的）AST再转换为正常的字符串代码。此步骤主要由`@babel/generator`深度优先遍历整个AST，然后构建可以表示转换后代码的字符串。

# 抽象语法树（AST）

学过《编译原理》的童鞋应该都知道AST，即使不知道也没关系，我们可以通过`astexplorer`在线查看。

![astexplorer](https://user-images.githubusercontent.com/11884369/55702696-f13ca600-5a09-11e9-9cff-adec9a70821a.png)

如上所示。

```
function square(n) {
  return n * n;
}
```

这段代码可以表示成如下所示的一棵树：

```
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

> 可以使用json对象表示AST，出于简化的目的，上面移除了部分属性

这个AST中的每一层结构叫做`节点(node)`，一个AST可以由单一的节点或是成百上千个节点构成。 它们组合在一起可以描述用于静态分析的程序语法。

每一个节点都有如下所示的接口（Interface）：

```
interface Node {
  type: string;
}
```

字符串形式的 type 字段表示节点的类型（如： "FunctionDeclaration"，"Identifier"，或 "BinaryExpression"）。 每一种类型的节点定义了一些附加属性用来进一步描述该节点类型。

Babel插件就是对这些节点进行添加、更新和删除。

# 路径（Path）

AST能够表示语法的结构，但是我们对节点进行操作时，更多的是希望获得节点之间的联系。

**Path**是表示两个节点之间连接的对象。

例如，如果有下面这样一个节点及其子节点︰

```
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  ...
}
```

将子节点`Identifier`表示为一个路径（Path）的话，看起来是这样的：

```
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

同时它还包含关于该路径的其他元数据：

```
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

在某种意义上，路径是一个节点在树中的位置以及关于该节点各种信息的响应式`Reactive`表示。路径对象还包含添加、更新、移动和删除节点有关的其他很多方法，当你调用一个修改树的方法后，路径信息也会被更新。

`@babel/traverse`这个独立的包对AST进行遍历，解析出整个树的`path`，并更新节点。

# 访问者（visitor）

`@babel/traverse`遍历AST时，会依次进入每个节点。

假设有如下AST结构：
```
- FunctionDeclaration
  - Identifier (id)
  - Identifier (params[0])
  - BlockStatement (body)
    - ReturnStatement (body)
      - BinaryExpression (argument)
        - Identifier (left)
        - Identifier (right)
```

则遍历过程如下：

- 进入 `FunctionDeclaration`
	- 进入 `Identifier (id)`
	- 走到尽头
	- 退出 `dentifier (id)`
	- 进入 `Identifier (params[0])`
	- 走到尽头
	- 退出 `Identifier (params[0])`
	- 进入 `BlockStatement (body)`
	- 进入 `ReturnStatement (body)`
		- 进入 `BinaryExpression (argument)`
		- 进入 `Identifier (left)`
 			- 走到尽头
		- 退出 `Identifier (left)`
		- 进入 `Identifier (right)`
 			- 走到尽头
		- 退出 `Identifier (right)`
		- 退出 `BinaryExpression (argument)`
	- 退出 `ReturnStatement (body)`
	- 退出 `BlockStatement (body)`
- 退出 `FunctionDeclaration`

当我们说进入某一个节点，实际上是说我们在**访问**他们。

访问者简单的说就是一个对象，它定义了在树状机构的遍历中，如何获取节点的方法。

```
const MyVisitor = {
  Identifier() {
    console.log("Called!");
  }
};
```

这是一个简单的访问者，把它用于遍历中时，每当在树中遇见一个`Identifier`的时候会调用`Identifier()`方法。

所以在下面的代码中`Identifier()`方法会被调用四次（包括`square`在内，总共有四个`Identifier`）。

```
function square(n) {
  return n * n;
}
```

```
traverse(ast, MyVisitor);
Called!
Called!
Called!
Called!
```

这些调用都发生在进入节点时，不过有时候我们也可以在退出时调用访问者方法。

```
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

因此，对于一个具体的节点我们有两次访问的机会。

当你有一个`Identifier()`成员方法的访问者时，你实际上是在访问路径而非节点。通过这种方式，你操作的就是节点的响应式表示（译注：即路径）而非节点本身。

```
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;

const code = `function square(n) {
  return n * n;
}`;

const ast = parser.parse(code);

const MyVisitor = {
    Identifier: {
      enter(path) {
        console.log(`${path.node.name} Entered!`);
      },
      exit(path) {
        console.log(`${path.node.name} Exited!`);
      }
    }
  };

traverse(ast, MyVisitor);
```

输出结果如下：

```
Identifier square Entered!
Identifier square Exited!
Identifier n Entered!
Identifier n Exited!
Identifier n Entered!
Identifier n Exited!
Identifier n Entered!
Identifier n Exited!
```

# 初窥插件

从上面的示例我们已经知道如何访问节点，现在我们可以操作节点。

```
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;
const generate = require("@babel/generator").default;

const code = `function square(n) {
  return n * n;
}`;

const ast = parser.parse(code);

const MyVisitor = {
    enter(path) { // enter会在每个节点进入时执行
        if (
            path.node.type === "Identifier" &&
            path.node.name === "n"
        ) {
            path.node.name = "x";
        }
    }
};

traverse(ast, MyVisitor);

const result = generate(ast);

console.log(result.code);
```

输出结果：

```
function square(x) {
  return x * x;
}
```

这便是Babel插件的基本运行原理。

# `@Babel/types`工具库

`@Babel/Types`模块是一个用于 AST 节点的`Lodash`式工具库，它包含了构造、验证以及变换 AST 节点的方法。 该工具库包含考虑周到的工具方法，对编写处理AST逻辑非常有用。

```
const t = require("@babel/types");

...

const MyVisitor = {
    enter(path) {
        if (t.isIdentifier(path.node, { name: "n" })) {
            path.replaceWith(t.identifier('x'))
          }
    }
};

...
```

这里使用`isIdentifier(path.node, { name: "n" })`验证值为`n`的`Identifier`节点。然后使用`identifier('x')`创建一个值为`x`的节点进行替换。

输出结果一致。

`@Babel/Types`还提供了多种节点类型的构造、验证方法（eg：`binaryExpression`、`returnStatement`、`classDeclaration`），详细请查阅文档。

判断节点类型时，在类型名称前加`is`，然后将类型名称第一个字母变成大写，例如`isIdentifier`，该方法还有另外一个版本`assetIdentifier`(抛出异常，而不是返回true 与 false)。

创建一个类型节点时（用于插入AST中，或者替换AST中的节点），直接调用节点类型函数，类型名称第一个字母小写，例如`identifier('x')`。

# 转换操作函数

`Path`对象提供了添加、更新、移动和删除等一系列节点操作方法。

*  `get`：获取子节点属性
*  `isReferencedIdentifier`：检查标示符（Identifier）是否被引用
*  `findParent`：向父路径搜索节点
*  `find`：向父节点搜索节点，并且搜索本节点
*  `getFunctionParent`：查找最接近的父函数或程序
*  `getStatementParent`：向上遍历语法树，直到找到在列表中的父节点路径
*  `inList`：判断路径是否有同级节点
*  `getSibling`：获得同级路径
*  `key`：获取路径所在容器的索引
*  `container`：获取路径的容器（包含所有同级节点的数组）
*  `skip`：停止子节点的遍历
*  `stop`：停止整个路径的遍历
*  `replaceWith`：替换节点
*  `replaceWithMultiple`：用多节点替换单节点
*  `replaceWithSourceString`：用字符串源码替换节点
*  `insertBefore`：在当前节点之前插入兄弟节点
*  `insertAfter`：在当前节点之前插入兄弟节点
*  `unshiftContainer`：在节点容器的开始插入节点
*  `pushContainer`：在节点容器的结尾插入节点
*  `remove`：删除节点（自身）

...

更多函数与使用方法参考`babel-handbook`。

# 第一个Babel插件

**插件分析**

假设我想把所有的如下代码：

```
var hasBarProperty = foo.hasOwnProperty("bar");
var isPrototypeOfBar = foo.isPrototypeOf(bar);
var barIsEnumerable = foo.propertyIsEnumerable("bar");
```

转换成这种写法：

```
var hasBarProperty = Object.prototype.hasOwnProperty.call(foo, "bar");
var isPrototypeOfBar = Object.prototype.isPrototypeOf.call(foo, bar);
var barIsEnumerable = Object.prototype.propertyIsEnumerable.call(foo, "bar");
```

这里主要目的是将`Object.prototype`上的方法调用，改成`Object.prototype.propertyIsEnumerable.call()`形式。

这里以转换`hasOwnProperty`为例，先在`astexplorer`观察待处理AST(`foo.hasOwnProperty("bar")`)与目标AST(`Object.prototype.hasOwnProperty.call(foo, "bar")`)结构。

待处理的`foo.hasOwnProperty("bar")`的AST大致如下：

```
{
    "type": "CallExpression",
    "loc": {...},
    "callee": {
        "type": "MemberExpression",
        "loc": {...},
        "object": {
            "type": "Identifier",
            "loc": {...},
            "name": "foo"
        },
        "property": {
            "type": "Identifier",
            "loc": {...},
            "name": "hasOwnProperty"
        },
        "computed": false
    },
    "arguments": [
        {
            "type": "StringLiteral",
            "loc": {...},
            "value": "bar"
        }
    ]
}
```

期望转换为`Object.prototype.hasOwnProperty.call(foo, "bar")`的目标AST大致如下：

```

{
    "type": "CallExpression",
    "loc": {...},
    "callee": {
        "type": "MemberExpression",
        "loc": {...},
        "object": {
            "type": "MemberExpression",
            "loc": {...},
            "object": {
                "type": "MemberExpression",
                "loc": {...},
                "object": {
                    "type": "Identifier",
                    "loc": {...},
                    "name": "Object"
                },
                "property": {
                    "type": "Identifier",,
                    "loc": {...},
                    "name": "prototype"
                },
                "computed": false
            },
            "property": {
                "type": "Identifier",
                "loc": {...},
                "name": "hasOwnProperty"
            },
            "computed": false
        },
        "property": {
            "type": "Identifier",
            "loc": {...},
            "name": "call"
        },
        "computed": false
    },
    "arguments": [
        {
            "type": "Identifier",
            "loc": {...},
            "name": "foo"
        },
        {
            "type": "StringLiteral",
            "loc": {... },
            "value": "bar"
        }
    ]
}
```
根据如上两个AST，转换的大致思路为：

*  编写一个`CallExpression`访问者
*  获取`CallExpression.callee`，该节点为`MemberExpression`类型，代表属性调用表达式（即`foo.hasOwnProperty`部分）。
*  获取`CallExpression.arguments`，该节点是一个数组，代表参数部分（即`"bar"`部分）。
*  当`callee.object`为`Identifier`类型节点，并且`callee.property`为值为`hasOwnProperty`的`Identifier`节点时，则进行转换。
*  构造一个`Object.prototype.hasOwnProperty.call`形式的`MemberExpression`嵌套节点。
*  构造时，先构造最下面一层的`MemberExpression`节点，即`Object.prototype`部分，以此类推，构造完整的`Object.prototype.hasOwnProperty.call`。
*  然后再构造一个新的`CallExpression`节点，其中新的参数为原`CallExpression.callee.object`的值，与原参数组成的数组。
*  用新的`CallExpression`节点替换原来的节点。

**编写插件**

新建如下结构项目：

```
--plugins
  |--babel-plugin-transform-object-prototype-methods.js
--index.js
--.babelrc
--package.json
```

分别有如下代码：

```
// index.js
var hasBarProperty = foo.hasOwnProperty("bar");
```

```
// plugins/babel-plugin-transform-object-prototype-methods.js
module.exports = function (babel) {
    const { types: t } = babel;
    return {
        name: "ast-transform", // not required
        visitor: {
          CallExpression(path) {
              const memberExp = path.get("callee");
              const arg = path.get("arguments.0");
              const memberProperty = memberExp.get("property");
              const memberObject = memberExp.get("object");
              if (t.isIdentifier(memberProperty) && memberProperty.node.name === "hasOwnProperty") { // 对`hasOwnProperty`方法调用的节点进行转换
                const objectMemberExp = t.MemberExpression(t.Identifier("Object"), t.Identifier("prototype")); // 构造`Object.prototype`的`MemberExpression`节点
                const prototypeMemberExp = t.MemberExpression(objectMemberExp, t.Identifier("hasOwnProperty")); // 构造`Object.prototype.hasOwnProperty`的`MemberExpression`节点
                const hasOwnPropertyMemberExp = t.MemberExpression(prototypeMemberExp, t.Identifier("call")); // 构造`Object.prototype.hasOwnProperty.call`的`MemberExpression`节点
                const newCallExpression = t.callExpression(hasOwnPropertyMemberExp, [memberObject.node, arg.node]); // 构造`Object.prototype.hasOwnProperty.call(p, "bar")`的`callExpression`节点
                path.replaceWith(newCallExpression); // 使用新节点替换
              }
            }
        }
    };
}
```

```
// .babelrc
{
    "plugins": [
        "./plugins/babel-plugin-transform-object-prototype-methods"
    ]
}
```

在使用插件之前需要先安装依赖：

```
yarn add @babel/core @babel/cli -D
```

执行如下命令，使用`babel-plugin-transform-object-prototype-methods`插件转换代码：

```
npx babel index.js -o output.js
```

查看输出文件`output.js`：

```
var hasBarProperty = Object.prototype.hasOwnProperty.call(foo, "bar");
```

源代码已经正确转换。

从上可知：
*  babel插件是一个函数。
*  该函数接受一个`babel`对象作为参数。
*  `babel.types`可直接使用`@babel/types`模块，在插件中不用单独引入。
*  在特定的类型访问者中，可以获取节点，以及添加、删除、替换节点等操作。
*  babel插件函数可以被`.babelrc`配置文件，或者命令行`--plugins`参数，或者`babel.transform`的`plugins`选项加载。

**完善插件功能**

下面来完善插件功能，让该插件可以处理`hasOwnProperty`、`isPrototypeOf`、`propertyIsEnumerable`三种类型。

```
// plugins/babel-plugin-transform-object-prototype-methods.js
module.exports = function (babel) {
  const { types: t } = babel;
  const PropertySchema = {
    hasOwnProperty: {
      type: "boolean",
      default: true
    },
    isPrototypeOf: {
      type: "boolean",
      default: true
    },
    propertyIsEnumerable: {
      type: "boolean",
      default: true
    }
  };

  return {
      name: "transform-object-prototype-methods", // not required
      visitor: {
        CallExpression(path) {
            const memberExp = path.get("callee");
            const arg = path.get("arguments.0");
            const memberProperty = memberExp.get("property");
            const memberObject = memberExp.get("object");
            if (t.isIdentifier(memberProperty) && PropertySchema[memberProperty.node.name]) {
              const objectMemberExp = t.MemberExpression(t.Identifier("Object"), t.Identifier("prototype"));
              const prototypeMemberExp = t.MemberExpression(objectMemberExp, t.Identifier(memberProperty.node.name));
              const callMemberExp = t.MemberExpression(prototypeMemberExp, t.Identifier("call"));
              const newCallExpression = t.callExpression(callMemberExp, [memberObject.node, arg.node]);
              path.replaceWith(newCallExpression);
            }
          }
      }
  };
}
```

现在以下代码均可正确转换：

```
var hasBarProperty = foo.hasOwnProperty("bar");
var isPrototypeOfBar = foo.isPrototypeOf(bar);
var barIsEnumerable = foo.propertyIsEnumerable("bar");

if(foo.hasOwnProperty("bar")) {}
if(foo.isPrototypeOf(bar)) {}
if(foo.propertyIsEnumerable("bar")) {}
```

**插件选项**

下面我们让该插件可以接受插件选项，并根据选项开启或禁用转换：

```
{
    "plugins": [
        ["./plugins/babel-plugin-transform-object-prototype-methods", {
            "hasOwnProperty": true,
            "isPrototypeOf": false
        }]
    ]
}
```

修改插件：

```
module.exports = function (babel) {
  const { types: t } = babel;
  const PropertySchema = {
    hasOwnProperty: {
      type: "boolean",
      default: true
    },
    isPrototypeOf: {
      type: "boolean",
      default: true
    },
    propertyIsEnumerable: {
      type: "boolean",
      default: true
    }
  };

  // 默认插件选项
  const defaultOpts = {
    hasOwnProperty: PropertySchema.hasOwnProperty.default,
    isPrototypeOf: PropertySchema.isPrototypeOf.default,
    propertyIsEnumerable: PropertySchema.propertyIsEnumerable.default
  };

  return {
      name: "ast-transform", // not required
      visitor: {
        CallExpression(path, state) {
            const memberExp = path.get("callee");
            const arg = path.get("arguments.0");
            const memberProperty = memberExp.get("property");
            const memberObject = memberExp.get("object");
            // 合并选项
            const options = Object.assign({}, defaultOpts, state.opts);
            const propertyName = memberProperty.node.name;
            // 只有规定的`Object.prototype`原型方法，并且只有启用转换该原型方法时，才会被转换
            if (t.isIdentifier(memberProperty) && PropertySchema[propertyName] && options[propertyName]) {
              const objectMemberExp = t.MemberExpression(t.Identifier("Object"), t.Identifier("prototype"));
              const prototypeMemberExp = t.MemberExpression(objectMemberExp, t.Identifier(propertyName));
              const callMemberExp = t.MemberExpression(prototypeMemberExp, t.Identifier("call"));
              const newCallExpression = t.callExpression(callMemberExp, [memberObject.node, arg.node]);
              path.replaceWith(newCallExpression);
            }
          }
      }
  };
}
```

现在插件选项已经可以工作了，转换结果如下：

```
var hasBarProperty = Object.prototype.hasOwnProperty.call(foo, "bar");
var isPrototypeOfBar = foo.isPrototypeOf(bar);
var barIsEnumerable = Object.prototype.propertyIsEnumerable.call(foo, "bar");

if (Object.prototype.hasOwnProperty.call(foo, "bar")) {}

if (foo.isPrototypeOf(bar)) {}

if (Object.prototype.propertyIsEnumerable.call(foo, "bar")) {}
```

**恭喜你，你已经开始你的大佬（装逼）之路了。**