# node-esm-ts

如果我们想用 TypeScript 进行开发 Node.js 应用，并且希望最终使用 ES6 模块语法。那么我们就会遇到这些问题：

1. `package.json` 中必须要指定 `"type": "module"` 来启用 ES6 模块语法。
2. TS 编译之后无法生成 `.js` 文件扩展名。
3. 一旦使用启用 ES6 模块语法，就必须指定文件的扩展名,比如 `.js`。

## ESM 与 CJS

ESM 会有一些与 CJS 不同：

- 只有 ESM 才能使用 top-level await

- ESM 中无法使用 `__dirname/__filename` 这类 Node.js 的全局变量。[alternative-for-dirname-in-node-js-when-using-es6-modules](https://stackoverflow.com/questions/46745014/alternative-for-dirname-in-node-js-when-using-es6-modules)

- ESM 是可以向下兼容 CJS 的。但是要 CJS 向上兼容，导入 ESM 模块，就会很麻烦，必须要使用 `dynamic import()`。

  ```ts
  // main.cjs
  import("./fileA.mjs").then((ctx) => {
    console.log(ctx.name); // "mjs"
  });
  ```

 在 NodeJS 中想要同步和动态导入 ES6 模块，可以考虑：[import-sync](https://github.com/nktnet1/import-sync)

### `.cts` 和 `.mts` 文件
随着 Node.js 12 的引入和 ES 模块支持的增加，TypeScript 引入了新的文件扩展名，以更清楚地区分 CommonJS 和 ES 模块：

- `.cts`：这是一个被视为 CommonJS 模块的 TypeScript 文件。它相当于使用 `--module commonjs` 编译选项。
- `.mts`：这是一个被视为 ES 模块的 TypeScript 文件。它相当于使用 `--module esnext` 编译选项。

### `require` 与 `import`

1. `require` 调用同步 CJS 模块加载器
2. `require` 在 CJS 模块中使用，在 ES 模块中可以使用 [`module.createRequire()`](https://nodejs.org/api/module.html#modulecreaterequirefilename) 构造 `require` 函数。
3. `require` 只能引用 CJS 模块，引用 ES 模块将抛出 `ERR_REQUIRE_ESM`，因为不可能从同步模块调用异步模块加载器。[node.js 22](https://nodejs.org/en/blog/announcements/v22-release-announce) adds `require()` support for synchronous ESM graphs under the flag `--experimental-require-module`。[pr](https://github.com/nodejs/node/pull/51977))
4. `import` 调用异步 ES 模块加载器
5. `import` 只能在 ES 模块调用
6. `import` 可以引用 ES 和 CJS 模块。

## JSON 模块

```js
import packageConfig from "./package.json" with { type: "json" };
```

必须使用 `with` 语法，会导致 node 不能识别。所以不要使用这种语法。

## `require('xxx').default`

有时，node 环境下 require 一个包时，会发现需要这么引入：`const axios = require('axios').default`。

其实很简单，axios 这个模块是一个 ES6 模块，是用 ESModule 的标准写的。其内部是这样导出的：`export default xxx`，相当于 `export {xxx as default}`。对外把变量命名为了 `default`，ES6 `default` 语法糖让我们在 import 时，不需要写 `default` 了。

而 require 在 commonJS 中对应 `module.exports` 的部分。当 require 去处理 ES6 的导出时，对它而言，ES6 的代码就会被转化成这样：

```js
module.exports = {
  default: xxx,
};
```

于是，在 ES6 语法糖 `default` 的影响下，require 导出的不是 `xxx` 这个对象，`xxx` 是作为 `default` 的值，被包在一个更大的对象里。

## 库规范

有依赖 commonjs 包的库，考虑输出多包；
同时面对多种环境，优先考虑输出多包。

1. 导出 `cjs\mjs` 两种文件，方便用户使用。
2. 通过 `package.json` 的 `exports` 同时，指定 `require、import、types` 文件导出
3. 使用 ts 编译，生成 `.d.ts` 类型定义文件

### 相关构建工具
- [unbuild](https://github.com/unjs/unbuild)
- [tshy](https://github.com/isaacs/tshy)


## Pure ESM package
> 轮子哥 [sindresorhus](https://github.com/sindresorhus) 的 [Pure ESM package](https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c)

## 参考

- [nonara/ts-patch](https://github.com/nonara/ts-patch)

- [Modules: ECMAScript modules](https://nodejs.org/api/esm.html#modules-ecmascript-modules)

- [ts-bridge/ts-bridge](https://github.com/ts-bridge/ts-bridge)

- [GervinFung/ts-add-js-extension](https://github.com/GervinFung/ts-add-js-extension)

- [tsc-esm-fix](https://www.npmjs.com/package/tsc-esm-fix)

- [Node.js + Typescript with ESM in 2023](https://medium.com/codememo/node-js-typescript-with-esm-in-2023-6b87e6f8e737)
