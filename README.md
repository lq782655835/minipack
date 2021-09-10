# minipack源码解析

## 整体流程

先递归找到依赖项 & 编译每个文件 -> 代码拼接方式生成最终代码。

流程细节：入口entry文件 -> 生成ATS -> ATS中找到依赖项 & babel编译代码（createAsset） -> 依赖项递归生成队列queue（createGraph） -> 代码拼接（build，基于CommonJS规范）

``` js
const graph = createGraph('./example/entry.js'); // ATS分析图
const result = bundle(graph); // 技巧拼接字符串
```

核心代码分析如下：

## 1. babel编译代码

babel编译代码（babylon.parse -> AST -> transformFromAst -> code

``` js
const babylon = require('babylon');
const {transformFromAst} = require('babel-core');

// 生成ATS
const ast = babylon.parse(content, {
    sourceType: 'module',
  });

// 最终code代码
const {code} = transformFromAst(ast, null, { // springleo: babel编译代码（babylon.parse -> AST -> transformFromAst -> code）
    presets: ['env'],
  });
```

## 2. ATS可以获取依赖项

``` js
const traverse = require('babel-traverse').default;

const dependencies = [];

  traverse(ast, {
    // Every time we see an import statement we can just count its value as a
    // dependency.
    ImportDeclaration: ({node}) => {
      dependencies.push(node.source.value); // 通过ATS找到依赖项目
    },
  });
```

## 拼接代码(核心)

技巧性利用CommonJS规范

``` js
function bundle(graph) {
  let modules = '';

  // 代码拼接
  graph.forEach(mod => {
    modules += `${mod.id}: [
      function (require, module, exports) {
        ${mod.code}
      },
      ${JSON.stringify(mod.mapping)},
    ],`;
  });

  // 入口0开始寻找require(1)
  const result = `
    (function(modules) {
      function require(id) {
        const [fn, mapping] = modules[id];

        // 自己实现的require本地包
        function localRequire(name) {
          return require(mapping[name]); // springleo: 从依赖包的名字，找到包编译的序列号ID
        }

        const module = { exports : {} };

        fn(localRequire, module, module.exports);

        return module.exports;
      }

      require(0);
    })({${modules}})
  `;

  return result;
}
```

graph是解析的图数据结构：

![image](https://user-images.githubusercontent.com/6310131/132790449-d87054f7-3eed-4005-8969-3c8c4ee2bfcb.png)
