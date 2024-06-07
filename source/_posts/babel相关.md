---
title: babel相关
toc: true
date: 2024-06-07 19:51:38
tags:
---

# babel

## 流程

| 编译                                     | 转换                      | 输出              |
| ---------------------------------------- | ------------------------- | ----------------- |
| parser                                   | transform                 | generator         |
| @babel/parser                            | @babel/traverse           | @babel/generator  |
| 将代码进行词法分析、语法分析后转换成 AST | 遍历 AST，对 AST 进行修改 | 将 AST 转换成代码 |

## 名词介绍

`Plugins` 在 `Presets` 前运行

### plugins：插件

```js
{
  "plugins": ["plugin1", "plugin2"],
}
```

先执行`plugin1`后执行`plugin2`

Babel 推崇功能的单一性，就是每个插件的功能尽可能的单一。比如想使用 `ES6` 的箭头函数，需要插件`@babel/plugin-transform-arrow-functions`，但是`ES6`、`ES7`...那么多新语法难道要一个个设置吗？这时候就需要`presets`了

- 插件名可以缩写，把 `plugin` 省略：`@babel/plugin-transform-runtime`等价于`@babel/transform-runtime`

#### @babel/plugin-transform-runtime

例如：

```js
["@babel/plugin-transform-runtime", { corejs: 3 }];
```

默认 `corejs:fase`

它可以将产生的`helper`函数，统一集中起来，在需要的地方`require`，例如`class`语法，只会引入`_createClass`用来生成 `class`
但是`@babel/plugin-transform-runtime` 通常仅在开发时使用，但是运行时最终代码需要依赖` @babel/runtime`，所以 `@babel/runtime` 必须要作为生产依赖被安装
请注意， `corejs: 2` 仅支持全局变量（例如 `Promise` ）和静态属性（例如 `Array.from`），同时 `corejs: 3` 还支持`实例属性`（例如`[].includes`）

综上所述，它提供了几个功能：

- 对辅助函数的复用，解决代码冗余问题
- 解决全局变量污染问题
- 提供 polyfill，并且是根据代码使用情况导入+目标浏览器来提供，相当于@babel/preset-env 中的"useBuiltIns": "usage"

### presets：预设

> presets 就是 plugins 的集合

```js
{
  "presets": ["preset1", "preset2", "preset3"]
}
```

先执行`preset3`后执行`preset2`后执行`preset1`

#### @babel/preset-env

例如：[@babel/preset-env](https://babel.nodejs.cn/docs/babel-preset-env/)
![alt text](image-1.png)

如果只配置`presets`，如下，就会产生一大堆 `helper` 函数，例如兼容`class语法`，会产生`_createClass`、`_definProperties`等等。试想一下，如果不同的 js 文件里都要用到 class 语法，那么每个文件都会产生这些`helper`函数，造成体积太大。

```json
{
  "presets": [
    [
      "@babel/preset-env",
      {
        // 实现按需加载
        "useBuiltIns": "usage",
        "corejs": {
          "version": 3
        }
      }
    ]
  ],
  "plugins": []
}
```

有没有办法能解决呢，有～，那就是`@babel/plugin-transform-runtime`

##### useBuiltIns

> 此选项标识如何处理 `polyfill`，会将 core-js 作为裸模块导入，意味着在装包的时候需要将`core-js`作为生产环境依赖

- usage：按需加载，为每个文件导入所需要的`polyfill`,需要指定 `corejs` 版本
- entry：手动写上` import "core-js"``，babel ` 会识别这句，然后替换，会根据目标浏览器（`browserslist`），将不支持的功能通过导入 plyfill`兼容`,需要指定 `corejs` 版本
- false(默认)：不要自动为每个文件添加 polyfill，也不转换`import "core-js"`为 `polyfill`

##### corejs

> `core-js` 版本，与`useBuiltIns:usage | entry`一起使用才有效
> `string` 或 `{ version: string, proposals: boolean }`，默认版本为 `2.0`  
> `proposals`表示是否使用正在提案的特性

##### targets

> `string` | `Array<string>` | `{ [string]: string }`
> 默认值： `{}`。未指定目标时：`Babel`会假设你的目标是尽可能的最老的浏览器

例如

```js
 "targets": ["> 0.25%","not dead"]
```

标识基于全球使用率大于`0.25%`的浏览器版本范围，2 年内仍有官方支持或更新的浏览器（兼容 IE 就把 `not dead` 去掉）

### polyfill：垫片

> polyfill 就是一些兼容性代码，比如`Promise`、`Array.prototype.includes`等,例如 core-js

`@babel/polyfill`基本已经弃用了，因为引入了下面两个库，一股脑将所有的兼容性代码引入，导致打包体积太大，也不能按需加载。

```js
import "core-js/stable";
import "regenerator-runtime/runtime";
```

### 总结

现在大多使用框架，例如 React\Vue，官方都有预设好的 babel 配置。看下官方文档直接用就行

## 开发

> [参考——babel-handbook](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/zh-Hans/plugin-handbook.md#toc-parse) > [相关属性文档](https://evilrecluse.top/Babel-traverse-api-doc/#/?id=babeltraverse)

### @babel/parser

> 是基于 acorn 实现的，扩展了很多语法，可以支持 esnext、jsx、flow、typescript 等语法的解析，其中 jsx、flow、typescript 这些非标准的语法的解析需要指定语法插件。
> [文档](https://babeljs.io/docs/babel-parser)

```js
require("@babel/parser").parse("code", {
  // module | script
  sourceType: "module",
  plugins: ["jsx", "typescript"],
});
```

`sourceType:module | script`，它表示应该用哪种模式来解析，`module` 将会在严格模式下解析并且允许模块定义，`script` 则不会。 `sourceType` 的默认值是 `script` 并且在发现 `import` 或 `export` 时产生错误。 使用 `scourceType: "module"` 来避免这些错误。

### @babel/traverse

> 模块维护整体树状态，并负责替换、删除和添加节点。

```js
// 进入 FunctionDeclaration 节点时调用
traverse(ast, {
  FunctionDeclaration: {
    enter(path, state) {},
  },
});

// 默认是进入节点时调用，和上面等价
traverse(ast, {
  FunctionDeclaration(path, state) {},
});

// 进入 FunctionDeclaration 和 VariableDeclaration 节点时调用
traverse(ast, {
  "FunctionDeclaration|VariableDeclaration"(path, state) {},
});

// 通过别名指定离开各种 Declaration 节点时调用
traverse(ast, {
  Declaration: {
    exit(path, state) {},
  },
});
```

```js
import parser from "@babel/parser";
import traverse from "@babel/traverse";

const code = `function square(n) {
  return n * n;
}`;

const ast = parser.parse(code);

traverse(ast, {
  enter(path) {
    if (path.node.type === "Identifier" && path.node.name === "n") {
      path.node.name = "x";
    }
  },
});
```

`path` 是遍历过程中的路径，会保留上下文信息，有很多属性和方法，比如:

- 获取当前节点以及它的关联节点:
  - `path.node` 指向当前 `AST` 节点
  - `path.parent` 指向父级 `AST` 节点
  - `path.getSibling、path.getNextSibling、path.getPrevSibling` 获取兄弟节点
  - `path.get、path.set` 获取和设置当前节点属性的 `path`
- `path.scope` 获取当前节点的作用域信息
- 用于判断 `AST` 类型:
  - `path.isXxx` 判断当前节点是不是 `Xxx` 类型
  - `path.assertXxx` 判断当前节点是不是 `Xxx` 类型，不是则抛出异常
- 对 AST 进行增删改:
  - `path.insertBefore`、`path.insertAfter` 插入节点
  - `path.replaceWith`、` path.replaceWithMultiple``、replaceWithSourceString ` 替换节点
  - `path.remove` 删除节点

### @babel/types

> 用于 AST 节点的 `Lodash` 式实用程序库。它包含用于生成、验证和转换 `AST` 节点的方法。

```js
// 判断
import traverse from "@babel/traverse";
import * as t from "@babel/types";

traverse(ast, {
  BinaryExpression(path) {
  if (t.isIdentifier(path.node.left, { name: "n" })) {
  }
}
});
// 功能上等价⬇️
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

```js
// 创建
t.binaryExpression("*", t.identifier("a"), t.identifier("b")); // a*b
// ⬇️
// {
//   type: "BinaryExpression",
//   operator: "*",
//   left: {
//     type: "Identifier",
//     name: "a"
//   },
//   right: {
//     type: "Identifier",
//     name: "b"
//   }
// }
```

```js
// 断言
// 如果不是，则会抛出异常
t.assertBinaryExpression(maybeBinaryExpressionNode);
t.assertBinaryExpression(maybeBinaryExpressionNode, { operator: "*" });
// Error: Expected type "BinaryExpression" with option { "operator": "*" }
```

### @babel/template

> 通过 `@babel/types` 创建 AST 还是比较麻烦的，要一个个的创建然后组装，如果 `AST` 节点比较多的话需要写很多代码，这时候就可以使用 `@babel/template` 包来批量创建。
> [文档](https://babeljs.io/docs/babel-template)

语法占位符：

```js
import template from "@babel/template";
import generate from "@babel/generator";
import * as t from "@babel/types";

const buildRequire = template(`
  var %%importName%% = require(%%source%%);
`);

const ast = buildRequire({
  importName: t.identifier("myModule"),
  source: t.stringLiteral("my-module"),
});

console.log(generate(ast).code);
```

标识符占位符：

```js
const buildRequire = template(`
  var IMPORT_NAME = require(SOURCE);
`);

const ast = buildRequire({
  IMPORT_NAME: t.identifier("myModule"),
  SOURCE: t.stringLiteral("my-module"),
});
```

如果不使用占位符，可以直接使用`ast`生成

```js
const ast = template.ast(`
  var myModule = require("my-module");
`);
```

更多用法查看文档

### @babel/generator

> 将 `AST` 转换成代码。

单个 AST

```js
import { parse } from "@babel/parser";
import generate from "@babel/generator";

const code = "class Example {}";
const ast = parse(code);

const output = generate(
  ast,
  {
    /* options */
    sourceMaps: true,
  },
  code
);
```

多个 AST 可以合并成一个，生成代码时传入 sourceFilename 和 sourceMaps 参数

```js
import { parse } from "@babel/parser";
import generate from "@babel/generator";

const a = "var a = 1;";
const b = "var b = 2;";
const astA = parse(a, { sourceFilename: "a.js" });
const astB = parse(b, { sourceFilename: "b.js" });
const ast = {
  type: "Program",
  body: [].concat(astA.program.body, astB.program.body),
};

const { code, map } = generate(
  ast,
  { sourceMaps: true },
  {
    "a.js": a,
    "b.js": b,
  }
);
```

### @babel/code-frame

> 生成代码帧，用于显示错误位置。
> [文档](https://babeljs.io/docs/babel-code-frame)

```js
import { codeFrameColumns } from "@babel/code-frame";

const rawLines = `class Foo {
  constructor()
}`;
const location = { start: { line: 2, column: 16 } };

const result = codeFrameColumns(rawLines, location, {
  /* options */
});

console.log(result);
```

```
  1 | class Foo {
> 2 |   constructor()
    |                ^
  3 | }
```

```js
const { codeFrameColumns } = require("@babel/code-frame");

try {
  throw new Error("xxx 错误");
} catch (err) {
  console.error(
    codeFrameColumns(
      `console.log(123)`,
      {
        start: { line: 1, column: 14 },
      },
      {
        highlightCode: true,
        message: err.message,
      }
    )
  );
}
```

### @babel/core

> Babel 的核心模块，提供了 `babel.transform` 方法，用于将代码编译成 `AST`，以及 `babel.transformFromAst` 方法，用于将 `AST` 编译成代码。相当于上面的包的集合
> [文档](https://babeljs.io/docs/babel-core)

```js
transformSync(code, options); // => { code, map, ast }
transformFileSync(filename, options); // => { code, map, ast }
transformFromAstSync(parsedAst, sourceCode, options); // => { code, map, ast }

transformAsync("code();", options).then((result) => {});
transformFileAsync("filename.js", options).then((result) => {});
transformFromAstAsync(parsedAst, sourceCode, options).then((result) => {});
```

### @babel/helper-module-imports

> 用于在 AST 中插入 import 语句
> [文档](https://babeljs.io/docs/babel-helper-module-imports)
