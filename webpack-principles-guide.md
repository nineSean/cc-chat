# Webpack 原理详解指南

## 目录

1. [Webpack 核心概念](#1-webpack-核心概念)
2. [Webpack 编译流程](#2-webpack-编译流程)
3. [模块系统与依赖分析](#3-模块系统与依赖分析)
4. [Loader 机制深入](#4-loader-机制深入)
5. [Plugin 插件系统](#5-plugin-插件系统)
6. [代码分割与懒加载](#6-代码分割与懒加载)
7. [优化策略与性能调优](#7-优化策略与性能调优)
8. [热更新机制 (HMR)](#8-热更新机制-hmr)
9. [实战案例与最佳实践](#9-实战案例与最佳实践)
10. [Webpack 5 新特性](#10-webpack-5-新特性)

---

## 1. Webpack 核心概念

### 1.1 什么是 Webpack

Webpack 是一个现代化的模块打包工具，它将项目中的各种资源（JavaScript、CSS、图片等）视为模块，通过分析模块间的依赖关系，构建出一个或多个 bundle 文件。

**核心思想：**
- **万物皆模块**：将所有静态资源都视为模块
- **依赖驱动**：通过依赖关系构建模块图
- **配置驱动**：通过配置文件定义打包行为

### 1.2 五大核心概念

#### Entry（入口）
```javascript
module.exports = {
  entry: {
    main: './src/index.js',
    vendor: './src/vendor.js'
  }
};
```

入口定义了 Webpack 开始构建依赖图的起点。Webpack 会从入口文件开始，递归地构建整个应用的依赖图。

**思考要点：**
- 单入口 vs 多入口的选择
- 入口文件的职责划分
- 动态入口的实现方式

#### Output（输出）
```javascript
module.exports = {
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].[contenthash].js',
    chunkFilename: '[id].[contenthash].chunk.js',
    publicPath: '/assets/'
  }
};
```

输出定义了打包后文件的输出位置和命名规则。

**关键配置项解析：**
- `path`: 输出目录的绝对路径
- `filename`: 入口文件的输出文件名
- `chunkFilename`: 非入口 chunk 的文件名
- `publicPath`: 运行时的公共路径

#### Loader（加载器）
```javascript
module.exports = {
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: ['@babel/preset-env']
          }
        }
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader']
      }
    ]
  }
};
```

Loader 使 Webpack 能够处理非 JavaScript 文件，将它们转换为有效的模块。

**Loader 执行特点：**
- **链式调用**：从右到左，从下到上执行
- **单一职责**：每个 loader 只做一件事
- **可组合性**：可以通过组合实现复杂功能

#### Plugin（插件）
```javascript
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

module.exports = {
  plugins: [
    new HtmlWebpackPlugin({
      template: './src/index.html'
    }),
    new MiniCssExtractPlugin({
      filename: '[name].[contenthash].css'
    })
  ]
};
```

Plugin 扩展了 Webpack 的功能，可以执行更广泛的任务，如打包优化、资源管理等。

#### Mode（模式）
```javascript
module.exports = {
  mode: 'production' // 'development' | 'production' | 'none'
};
```

模式定义了 Webpack 的优化级别和默认配置。

---

## 2. Webpack 编译流程

### 2.1 编译流程概览

Webpack 的编译流程可以分为以下几个阶段：

```
初始化 → 编译 → 输出
   ↓
参数合并 → 创建Compiler → 开始编译 → 确定入口 → 编译模块 → 完成模块编译 → 输出资源 → 输出完成
```

### 2.2 详细编译步骤

#### 第一阶段：初始化

1. **参数合并**
   - 合并配置文件和命令行参数
   - 应用默认配置
   - 创建 options 对象

2. **创建 Compiler 对象**
   ```javascript
   // 简化的 Compiler 创建过程
   const compiler = new Compiler(options.context);
   compiler.options = options;
   ```

3. **注册插件**
   ```javascript
   // 插件注册过程
   options.plugins.forEach(plugin => {
     plugin.apply(compiler);
   });
   ```

#### 第二阶段：编译

1. **开始编译**
   - 触发 `beforeRun` 和 `run` 钩子
   - 创建 Compilation 对象

2. **确定入口**
   ```javascript
   // 入口处理
   for (const [name, entry] of Object.entries(options.entry)) {
     compilation.addEntry(context, entry, name, callback);
   }
   ```

3. **编译模块**
   - 调用对应的 Loader 处理模块
   - 使用 Parser 解析模块
   - 查找模块依赖
   - 递归编译依赖模块

4. **完成模块编译**
   - 构建模块依赖图
   - 确定 chunk 分组

#### 第三阶段：输出

1. **输出资源**
   - 根据入口和模块的依赖关系，组装成 chunk
   - 将 chunk 转换成文件
   - 输出到文件系统

2. **输出完成**
   - 触发 `done` 钩子
   - 报告编译结果

### 2.3 核心对象解析

#### Compiler 对象
```javascript
class Compiler extends Tapable {
  constructor(context) {
    super();
    this.hooks = {
      beforeRun: new AsyncSeriesHook(['compiler']),
      run: new AsyncSeriesHook(['compiler']),
      compile: new SyncHook(['params']),
      compilation: new SyncHook(['compilation', 'params']),
      done: new AsyncSeriesHook(['stats'])
    };
    // ...其他属性
  }
}
```

Compiler 是 Webpack 的核心引擎，负责：
- 管理整个编译流程
- 维护配置信息
- 提供插件系统的钩子

#### Compilation 对象
```javascript
class Compilation extends Tapable {
  constructor(compiler) {
    super();
    this.hooks = {
      buildModule: new SyncHook(['module']),
      succeedModule: new SyncHook(['module']),
      finishModules: new AsyncSeriesHook(['modules']),
      seal: new SyncHook([]),
      optimize: new SyncHook([])
    };
    // ...其他属性
  }
}
```

Compilation 对象代表一次编译过程，负责：
- 管理模块编译
- 维护模块依赖图
- 生成输出文件

---

## 3. 模块系统与依赖分析

### 3.1 模块类型

Webpack 支持多种模块规范：

#### ES6 模块
```javascript
// 导出
export const name = 'webpack';
export default function() {
  console.log('Hello Webpack');
}

// 导入
import defaultExport, { name } from './module';
```

#### CommonJS 模块
```javascript
// 导出
module.exports = {
  name: 'webpack'
};

// 导入
const { name } = require('./module');
```

#### AMD 模块
```javascript
// 定义模块
define(['dependency'], function(dep) {
  return {
    name: 'webpack'
  };
});

// 使用模块
require(['module'], function(module) {
  console.log(module.name);
});
```

### 3.2 依赖分析机制

#### 依赖收集过程
```javascript
// Webpack 如何分析依赖
class Parser {
  parse(source, options) {
    const ast = acorn.parse(source, {
      sourceType: 'module',
      ecmaVersion: 2020
    });
    
    this.walkStatements(ast.body);
    return ast;
  }
  
  walkImportDeclaration(statement) {
    const source = statement.source.value;
    // 收集依赖
    this.dependencies.push({
      type: 'import',
      request: source,
      loc: statement.loc
    });
  }
}
```

#### 模块解析过程
```javascript
// 模块解析策略
const resolve = {
  modules: ['node_modules'],
  extensions: ['.js', '.json', '.jsx'],
  alias: {
    '@': path.resolve(__dirname, 'src'),
    'utils': path.resolve(__dirname, 'src/utils')
  },
  mainFields: ['browser', 'module', 'main']
};
```

### 3.3 模块图构建

#### 依赖图数据结构
```javascript
// 简化的模块图结构
class ModuleGraph {
  constructor() {
    this.modules = new Map(); // 模块集合
    this.dependencies = new Map(); // 依赖关系
  }
  
  addModule(module) {
    this.modules.set(module.identifier(), module);
  }
  
  addDependency(dependency) {
    this.dependencies.set(dependency, module);
  }
  
  getModule(dependency) {
    return this.dependencies.get(dependency);
  }
}
```

#### 循环依赖处理
```javascript
// 循环依赖检测
function detectCircularDependency(moduleGraph) {
  const visited = new Set();
  const stack = new Set();
  
  function dfs(module) {
    if (stack.has(module)) {
      // 发现循环依赖
      return true;
    }
    
    if (visited.has(module)) {
      return false;
    }
    
    visited.add(module);
    stack.add(module);
    
    const dependencies = moduleGraph.getDependencies(module);
    for (const dep of dependencies) {
      if (dfs(dep)) {
        return true;
      }
    }
    
    stack.delete(module);
    return false;
  }
}
```

---

## 4. Loader 机制深入

### 4.1 Loader 本质

Loader 本质上是一个导出函数的 Node.js 模块：

```javascript
// 一个简单的 loader
module.exports = function(source) {
  // source 是模块的源代码
  const result = transformSource(source);
  return result;
};
```

### 4.2 Loader 执行机制

#### 链式调用
```javascript
// CSS 处理链
module: {
  rules: [
    {
      test: /\.css$/,
      use: [
        'style-loader',    // 3. 将 CSS 插入到页面
        'css-loader',      // 2. 解析 CSS
        'postcss-loader'   // 1. 首先执行，处理 CSS 兼容性
      ]
    }
  ]
}
```

#### Loader 上下文
```javascript
module.exports = function(source) {
  // this 指向 loader 上下文
  const options = this.getOptions(); // 获取配置选项
  const callback = this.async();     // 异步回调
  const emitFile = this.emitFile;    // 输出文件
  
  // 处理逻辑
  processSource(source, options)
    .then(result => callback(null, result))
    .catch(err => callback(err));
};
```

### 4.3 常用 Loader 原理

#### babel-loader
```javascript
const babel = require('@babel/core');

module.exports = function(source) {
  const options = this.getOptions();
  const callback = this.async();
  
  babel.transform(source, options, (err, result) => {
    if (err) return callback(err);
    callback(null, result.code, result.map);
  });
};
```

#### css-loader
```javascript
const postcss = require('postcss');
const icss = require('icss-utils');

module.exports = function(source) {
  const callback = this.async();
  const options = this.getOptions();
  
  postcss([
    // CSS 模块化处理
    require('postcss-modules-extract-imports'),
    require('postcss-modules-local-by-default'),
    require('postcss-modules-scope')
  ])
  .process(source, { from: this.resourcePath })
  .then(result => {
    const exports = icss.extractICSS(result.root);
    const js = `module.exports = ${JSON.stringify(exports.icssExports)};`;
    callback(null, js);
  })
  .catch(callback);
};
```

### 4.4 自定义 Loader 开发

#### 基础模板
```javascript
const { validate } = require('schema-utils');

// 配置项校验 schema
const schema = {
  type: 'object',
  properties: {
    option1: { type: 'string' },
    option2: { type: 'boolean' }
  }
};

module.exports = function(source) {
  const options = this.getOptions();
  
  // 验证配置项
  validate(schema, options, {
    name: 'My Loader',
    baseDataPath: 'options'
  });
  
  // 处理源代码
  return transformSource(source, options);
};
```

#### 异步 Loader
```javascript
module.exports = function(source) {
  const callback = this.async();
  const options = this.getOptions();
  
  // 异步处理
  processAsync(source, options)
    .then(result => {
      // 返回处理结果
      callback(null, result.code, result.map, result.meta);
    })
    .catch(error => {
      callback(error);
    });
};
```

#### Raw Loader
```javascript
// 处理二进制文件
module.exports = function(source) {
  // source 是 Buffer 对象
  const result = processBuffer(source);
  return result;
};

// 标记为 raw loader
module.exports.raw = true;
```

---

## 5. Plugin 插件系统

### 5.1 Plugin 架构原理

#### Tapable 事件系统
```javascript
const { SyncHook, AsyncSeriesHook } = require('tapable');

class MyCompiler {
  constructor() {
    this.hooks = {
      compile: new SyncHook(['params']),
      emit: new AsyncSeriesHook(['compilation'])
    };
  }
  
  run() {
    // 触发同步钩子
    this.hooks.compile.call(compilationParams);
    
    // 触发异步钩子
    this.hooks.emit.callAsync(compilation, (err) => {
      if (err) console.error(err);
    });
  }
}
```

#### 钩子类型
```javascript
// 同步钩子
const syncHook = new SyncHook(['arg1', 'arg2']);
syncHook.tap('PluginName', (arg1, arg2) => {
  // 同步处理
});

// 异步串行钩子
const asyncSeriesHook = new AsyncSeriesHook(['arg1']);
asyncSeriesHook.tapAsync('PluginName', (arg1, callback) => {
  // 异步处理
  setTimeout(() => callback(), 1000);
});

// 异步并行钩子
const asyncParallelHook = new AsyncParallelHook(['arg1']);
asyncParallelHook.tapPromise('PluginName', async (arg1) => {
  // 返回 Promise
  return await processAsync(arg1);
});
```

### 5.2 Plugin 生命周期

#### Compiler 钩子
```javascript
class MyPlugin {
  apply(compiler) {
    // 编译开始前
    compiler.hooks.beforeRun.tapAsync('MyPlugin', (compiler, callback) => {
      console.log('编译即将开始');
      callback();
    });
    
    // 编译开始
    compiler.hooks.run.tapAsync('MyPlugin', (compiler, callback) => {
      console.log('编译开始');
      callback();
    });
    
    // 编译完成
    compiler.hooks.done.tap('MyPlugin', (stats) => {
      console.log('编译完成');
    });
  }
}
```

#### Compilation 钩子
```javascript
class MyPlugin {
  apply(compiler) {
    compiler.hooks.compilation.tap('MyPlugin', (compilation) => {
      // 模块构建开始
      compilation.hooks.buildModule.tap('MyPlugin', (module) => {
        console.log('开始构建模块:', module.resource);
      });
      
      // 模块构建完成
      compilation.hooks.succeedModule.tap('MyPlugin', (module) => {
        console.log('模块构建完成:', module.resource);
      });
      
      // 优化阶段
      compilation.hooks.optimize.tap('MyPlugin', () => {
        console.log('开始优化');
      });
      
      // 资源生成
      compilation.hooks.emit.tapAsync('MyPlugin', (compilation, callback) => {
        // 可以修改输出资源
        Object.keys(compilation.assets).forEach(filename => {
          const asset = compilation.assets[filename];
          console.log('输出文件:', filename, '大小:', asset.size());
        });
        callback();
      });
    });
  }
}
```

### 5.3 常用 Plugin 原理

#### HtmlWebpackPlugin
```javascript
class HtmlWebpackPlugin {
  constructor(options = {}) {
    this.options = {
      template: 'src/index.html',
      filename: 'index.html',
      inject: true,
      ...options
    };
  }
  
  apply(compiler) {
    compiler.hooks.emit.tapAsync('HtmlWebpackPlugin', (compilation, callback) => {
      // 获取入口文件
      const entryNames = Array.from(compilation.entrypoints.keys());
      const assets = this.getAssets(compilation, entryNames);
      
      // 生成 HTML
      const html = this.generateHtml(assets);
      
      // 添加到输出资源
      compilation.assets[this.options.filename] = {
        source: () => html,
        size: () => html.length
      };
      
      callback();
    });
  }
  
  generateHtml(assets) {
    return `
<!DOCTYPE html>
<html>
<head>
  <title>Webpack App</title>
  ${assets.css.map(css => `<link rel="stylesheet" href="${css}">`).join('\n')}
</head>
<body>
  <div id="root"></div>
  ${assets.js.map(js => `<script src="${js}"></script>`).join('\n')}
</body>
</html>
    `;
  }
}
```

#### MiniCssExtractPlugin
```javascript
class MiniCssExtractPlugin {
  constructor(options = {}) {
    this.options = {
      filename: '[name].css',
      chunkFilename: '[id].css',
      ...options
    };
  }
  
  apply(compiler) {
    compiler.hooks.compilation.tap('MiniCssExtractPlugin', (compilation) => {
      // 注册 loader
      compilation.hooks.normalModuleLoader.tap('MiniCssExtractPlugin', (loaderContext, module) => {
        loaderContext[NS] = {
          filename: this.options.filename,
          publicPath: compilation.outputOptions.publicPath
        };
      });
      
      // 渲染 manifest
      compilation.hooks.renderManifest.tap('MiniCssExtractPlugin', (result, options) => {
        const { chunk } = options;
        const renderedModules = Array.from(chunk.modulesIterable).filter(module => 
          module.type === 'css/mini-extract'
        );
        
        if (renderedModules.length > 0) {
          result.push({
            render: () => this.renderContentAsset(renderedModules),
            filenameTemplate: this.options.filename,
            pathOptions: {
              chunk,
              contentHashType: 'css/mini-extract'
            },
            identifier: `css-${chunk.id}`,
            hash: chunk.contentHash['css/mini-extract']
          });
        }
      });
    });
  }
}
```

### 5.4 自定义 Plugin 开发

#### 基础插件模板
```javascript
class MyCustomPlugin {
  constructor(options = {}) {
    this.options = options;
  }
  
  apply(compiler) {
    const pluginName = 'MyCustomPlugin';
    
    compiler.hooks.emit.tapAsync(pluginName, (compilation, callback) => {
      // 插件逻辑
      this.processAssets(compilation);
      callback();
    });
  }
  
  processAssets(compilation) {
    // 处理输出资源
    Object.keys(compilation.assets).forEach(filename => {
      if (filename.endsWith('.js')) {
        const asset = compilation.assets[filename];
        const source = asset.source();
        const processedSource = this.processJavaScript(source);
        
        compilation.assets[filename] = {
          source: () => processedSource,
          size: () => processedSource.length
        };
      }
    });
  }
  
  processJavaScript(source) {
    // 自定义 JS 处理逻辑
    return source.replace(/console\.log/g, 'console.debug');
  }
}

module.exports = MyCustomPlugin;
```

#### 资源分析插件
```javascript
class BundleAnalyzerPlugin {
  constructor(options = {}) {
    this.options = {
      outputPath: './bundle-analysis.json',
      ...options
    };
  }
  
  apply(compiler) {
    compiler.hooks.emit.tapAsync('BundleAnalyzerPlugin', (compilation, callback) => {
      const stats = compilation.getStats().toJson();
      const analysis = this.analyzeBundle(stats);
      
      const analysisJson = JSON.stringify(analysis, null, 2);
      compilation.assets[this.options.outputPath] = {
        source: () => analysisJson,
        size: () => analysisJson.length
      };
      
      callback();
    });
  }
  
  analyzeBundle(stats) {
    return {
      totalSize: stats.assets.reduce((total, asset) => total + asset.size, 0),
      assets: stats.assets.map(asset => ({
        name: asset.name,
        size: asset.size,
        chunks: asset.chunks
      })),
      modules: stats.modules.map(module => ({
        name: module.name,
        size: module.size,
        reasons: module.reasons
      }))
    };
  }
}
```

---

## 6. 代码分割与懒加载

### 6.1 代码分割策略

#### 入口分割
```javascript
module.exports = {
  entry: {
    main: './src/index.js',
    vendor: './src/vendor.js',
    admin: './src/admin.js'
  },
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10,
          chunks: 'all'
        },
        common: {
          name: 'common',
          minChunks: 2,
          priority: 5,
          chunks: 'all',
          enforce: true
        }
      }
    }
  }
};
```

#### 动态导入
```javascript
// 异步加载模块
const loadModule = async () => {
  const { default: moduleA } = await import(
    /* webpackChunkName: "module-a" */ './moduleA'
  );
  return moduleA;
};

// 条件加载
if (shouldLoadFeature) {
  import(/* webpackChunkName: "feature" */ './feature')
    .then(module => {
      module.default.init();
    });
}

// 魔法注释
import(
  /* webpackChunkName: "my-chunk-name" */
  /* webpackMode: "lazy" */
  /* webpackExports: ["default", "named"] */
  './my-module'
);
```

### 6.2 SplitChunks 配置详解

#### 默认配置
```javascript
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'async',        // 只对异步模块进行分割
      minSize: 20000,         // 最小尺寸 20KB
      minRemainingSize: 0,    // 确保拆分后剩余的最小尺寸
      minChunks: 1,           // 最少被引用次数
      maxAsyncRequests: 30,   // 最大异步请求数
      maxInitialRequests: 30, // 最大初始请求数
      enforceSizeThreshold: 50000, // 强制执行拆分的尺寸阈值
      cacheGroups: {
        defaultVendors: {
          test: /[\\/]node_modules[\\/]/,
          priority: -10,
          reuseExistingChunk: true
        },
        default: {
          minChunks: 2,
          priority: -20,
          reuseExistingChunk: true
        }
      }
    }
  }
};
```

#### 高级配置策略
```javascript
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        // 第三方库
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10,
          chunks: 'all'
        },
        // React 相关
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
          name: 'react',
          priority: 20,
          chunks: 'all'
        },
        // UI 组件库
        antd: {
          test: /[\\/]node_modules[\\/]antd[\\/]/,
          name: 'antd',
          priority: 15,
          chunks: 'all'
        },
        // 工具库
        utils: {
          test: /[\\/]node_modules[\\/](lodash|moment|axios)[\\/]/,
          name: 'utils',
          priority: 15,
          chunks: 'all'
        },
        // 公共代码
        common: {
          name: 'common',
          minChunks: 2,
          priority: 5,
          chunks: 'all',
          enforce: true
        }
      }
    }
  }
};
```

### 6.3 懒加载实现机制

#### Webpack 运行时
```javascript
// __webpack_require__.e 实现
__webpack_require__.e = function(chunkId) {
  var promises = [];
  
  // 检查是否已安装
  var installedChunkData = installedChunks[chunkId];
  if(installedChunkData !== 0) {
    if(installedChunkData) {
      promises.push(installedChunkData[2]);
    } else {
      // 创建 Promise 并缓存
      var promise = new Promise(function(resolve, reject) {
        installedChunkData = installedChunks[chunkId] = [resolve, reject];
      });
      promises.push(installedChunkData[2] = promise);
      
      // 创建 script 标签加载 chunk
      var script = document.createElement('script');
      script.charset = 'utf-8';
      script.timeout = 120;
      script.src = __webpack_require__.p + chunkId + '.js';
      
      // 错误处理
      var onScriptComplete = function(event) {
        var chunk = installedChunks[chunkId];
        if(chunk !== 0) {
          if(chunk) {
            var errorType = event && (event.type === 'load' ? 'missing' : event.type);
            var realSrc = event && event.target && event.target.src;
            var error = new Error('Loading chunk ' + chunkId + ' failed.\n(' + errorType + ': ' + realSrc + ')');
            chunk[1](error);
          }
          installedChunks[chunkId] = undefined;
        }
      };
      
      script.onerror = script.onload = onScriptComplete;
      document.head.appendChild(script);
    }
  }
  
  return Promise.all(promises);
};
```

#### 路由级别的代码分割
```javascript
// React Router 懒加载
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

const Home = lazy(() => import('./pages/Home'));
const About = lazy(() => import('./pages/About'));
const Contact = lazy(() => import('./pages/Contact'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<div>Loading...</div>}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/about" element={<About />} />
          <Route path="/contact" element={<Contact />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

#### 组件级别的懒加载
```javascript
// 高阶组件实现懒加载
function withLazyLoading(importFunc, fallback = <div>Loading...</div>) {
  const LazyComponent = lazy(importFunc);
  
  return function WrappedComponent(props) {
    return (
      <Suspense fallback={fallback}>
        <LazyComponent {...props} />
      </Suspense>
    );
  };
}

// 使用示例
const LazyChart = withLazyLoading(
  () => import('./Chart'),
  <div>Loading chart...</div>
);
```

### 6.4 预加载和预获取

#### 预加载 (Preload)
```javascript
// 预加载关键资源
import(
  /* webpackPreload: true */
  './critical-module'
);

// 生成的 HTML
// <link rel="preload" href="critical-module.js" as="script">
```

#### 预获取 (Prefetch)
```javascript
// 预获取可能需要的资源
import(
  /* webpackPrefetch: true */
  './optional-module'
);

// 生成的 HTML
// <link rel="prefetch" href="optional-module.js">
```

#### 智能预加载策略
```javascript
class IntelligentPreloader {
  constructor() {
    this.userBehavior = new Map();
    this.preloadThreshold = 0.7; // 70% 概率预加载
  }
  
  trackUserBehavior(route, action) {
    const key = `${route}-${action}`;
    const count = this.userBehavior.get(key) || 0;
    this.userBehavior.set(key, count + 1);
  }
  
  shouldPreload(route, action) {
    const totalVisits = Array.from(this.userBehavior.values())
      .reduce((sum, count) => sum + count, 0);
    const actionCount = this.userBehavior.get(`${route}-${action}`) || 0;
    
    return (actionCount / totalVisits) > this.preloadThreshold;
  }
  
  preloadRoute(route) {
    if (this.shouldPreload(window.location.pathname, 'navigate-to-' + route)) {
      import(/* webpackChunkName: "[request]" */ `./pages/${route}`)
        .then(() => console.log(`Preloaded ${route}`))
        .catch(err => console.warn(`Failed to preload ${route}`, err));
    }
  }
}
```

---

## 7. 优化策略与性能调优

### 7.1 构建性能优化

#### 缓存策略
```javascript
module.exports = {
  cache: {
    type: 'filesystem',
    cacheDirectory: path.resolve(__dirname, '.webpack-cache'),
    buildDependencies: {
      config: [__filename]
    },
    managedPaths: [path.resolve(__dirname, 'node_modules')],
    profile: false,
    maxAge: 5184000000, // 60 天
    allowCollectingMemory: true
  }
};
```

#### 多线程构建
```javascript
const os = require('os');

module.exports = {
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: [
          {
            loader: 'thread-loader',
            options: {
              workers: os.cpus().length - 1,
              workerParallelJobs: 50,
              workerNodeArgs: ['--max-old-space-size=1024'],
              poolTimeout: 2000,
              poolParallelJobs: 50
            }
          },
          'babel-loader'
        ]
      }
    ]
  }
};
```

#### 构建分析
```javascript
// webpack-bundle-analyzer 配置
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      openAnalyzer: false,
      reportFilename: 'bundle-report.html'
    })
  ]
};

// 速度分析
const SpeedMeasurePlugin = require('speed-measure-webpack-plugin');
const smp = new SpeedMeasurePlugin();

module.exports = smp.wrap({
  // webpack 配置
});
```

### 7.2 运行时性能优化

#### Tree Shaking
```javascript
// package.json
{
  "sideEffects": false // 或指定文件数组
}

// webpack.config.js
module.exports = {
  mode: 'production',
  optimization: {
    usedExports: true,
    sideEffects: false
  }
};

// 代码示例
// utils.js
export function usedFunction() {
  return 'used';
}

export function unusedFunction() {
  return 'unused'; // 这个函数会被 tree shaking 移除
}

// main.js
import { usedFunction } from './utils';
console.log(usedFunction());
```

#### 模块压缩
```javascript
const TerserPlugin = require('terser-webpack-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');

module.exports = {
  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          parse: {
            ecma: 8
          },
          compress: {
            ecma: 5,
            warnings: false,
            comparisons: false,
            inline: 2,
            drop_console: true, // 移除 console
            drop_debugger: true, // 移除 debugger
            pure_funcs: ['console.log'] // 移除指定函数调用
          },
          mangle: {
            safari10: true
          },
          output: {
            ecma: 5,
            comments: false,
            ascii_only: true
          }
        },
        parallel: true,
        extractComments: false
      }),
      new CssMinimizerPlugin({
        minimizerOptions: {
          preset: [
            'default',
            {
              discardComments: { removeAll: true }
            }
          ]
        }
      })
    ]
  }
};
```

#### 资源优化
```javascript
module.exports = {
  module: {
    rules: [
      // 图片优化
      {
        test: /\.(png|jpe?g|gif|svg)$/i,
        type: 'asset',
        parser: {
          dataUrlCondition: {
            maxSize: 8 * 1024 // 8KB 以下转为 base64
          }
        },
        generator: {
          filename: 'images/[name].[contenthash:8][ext]'
        }
      },
      // 字体优化
      {
        test: /\.(woff|woff2|eot|ttf|otf)$/i,
        type: 'asset/resource',
        generator: {
          filename: 'fonts/[name].[contenthash:8][ext]'
        }
      }
    ]
  },
  plugins: [
    // 图片压缩
    new ImageMinimizerPlugin({
      minimizer: {
        implementation: ImageMinimizerPlugin.imageminMinify,
        options: {
          plugins: [
            ['imagemin-mozjpeg', { quality: 80 }],
            ['imagemin-pngquant', { quality: [0.6, 0.8] }],
            ['imagemin-svgo', {
              plugins: [
                { name: 'preset-default', params: { overrides: { removeViewBox: false } } }
              ]
            }]
          ]
        }
      }
    })
  ]
};
```

### 7.3 长期缓存策略

#### 文件名哈希
```javascript
module.exports = {
  output: {
    filename: '[name].[contenthash:8].js',
    chunkFilename: '[id].[contenthash:8].chunk.js'
  },
  plugins: [
    new MiniCssExtractPlugin({
      filename: '[name].[contenthash:8].css',
      chunkFilename: '[id].[contenthash:8].chunk.css'
    })
  ]
};
```

#### Runtime Chunk 提取
```javascript
module.exports = {
  optimization: {
    runtimeChunk: 'single', // 或 { name: 'runtime' }
    splitChunks: {
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all'
        }
      }
    }
  }
};
```

#### Module ID 稳定化
```javascript
module.exports = {
  optimization: {
    moduleIds: 'deterministic',
    chunkIds: 'deterministic'
  }
};
```

### 7.4 开发体验优化

#### 热更新配置
```javascript
module.exports = {
  devServer: {
    hot: true,
    liveReload: false,
    client: {
      overlay: {
        errors: true,
        warnings: false
      }
    }
  },
  plugins: [
    new webpack.HotModuleReplacementPlugin()
  ]
};
```

#### 开发环境优化
```javascript
module.exports = {
  mode: 'development',
  devtool: 'eval-cheap-module-source-map',
  cache: {
    type: 'memory'
  },
  optimization: {
    removeAvailableModules: false,
    removeEmptyChunks: false,
    splitChunks: false
  },
  output: {
    pathinfo: false
  },
  experiments: {
    lazyCompilation: {
      imports: true,
      entries: false
    }
  }
};
```

---

## 8. 热更新机制 (HMR)

### 8.1 HMR 工作原理

#### 整体架构
```
Browser ←→ WebSocket ←→ Webpack Dev Server ←→ Webpack Compiler
   ↓                                              ↓
HMR Runtime                                  File Watcher
   ↓                                              ↓
Module Update                               Change Detection
```

#### HMR 流程详解
1. **文件监听**: Webpack 监听文件变化
2. **重新编译**: 只编译变化的模块
3. **生成更新**: 生成 JSON manifest 和更新的 JS 文件
4. **通知客户端**: 通过 WebSocket 发送更新通知
5. **模块替换**: 客户端下载并应用更新

### 8.2 HMR Runtime 实现

#### 客户端 HMR 代码
```javascript
// HMR Runtime 简化实现
if (module.hot) {
  const hotAPI = {
    accept: function(deps, callback) {
      // 接受模块更新
      if (typeof deps === 'string') {
        deps = [deps];
      }
      
      deps.forEach(dep => {
        hotModuleData.acceptedDependencies[dep] = callback || true;
      });
    },
    
    dispose: function(callback) {
      // 模块销毁时的回调
      hotModuleData.disposeHandlers.push(callback);
    },
    
    invalidate: function() {
      // 标记模块需要完全重新加载
      hotModuleData.invalidated = true;
    }
  };
  
  module.hot = hotAPI;
}

// 更新检查和应用
function hotCheck() {
  return hotDownloadManifest()
    .then(function(update) {
      if (!update) {
        return null;
      }
      
      return hotDownloadUpdateChunk(update.c)
        .then(function() {
          return hotApply();
        });
    });
}

function hotApply() {
  const outdatedModules = [];
  const newModuleIds = Object.keys(hotUpdate);
  
  // 查找过期模块
  newModuleIds.forEach(function(moduleId) {
    const module = installedModules[moduleId];
    if (module && module.hot._selfInvalidated) {
      outdatedModules.push(moduleId);
    }
  });
  
  // 应用更新
  outdatedModules.forEach(function(moduleId) {
    delete installedModules[moduleId];
    hotUpdate[moduleId].call(modules, moduleId);
  });
}
```

### 8.3 框架集成

#### React Hot Reload
```javascript
// React Fast Refresh 配置
module.exports = {
  module: {
    rules: [
      {
        test: /\.jsx?$/,
        exclude: /node_modules/,
        use: [
          {
            loader: 'babel-loader',
            options: {
              presets: ['@babel/preset-react'],
              plugins: [
                'react-refresh/babel'
              ]
            }
          }
        ]
      }
    ]
  },
  plugins: [
    new ReactRefreshWebpackPlugin()
  ]
};

// React 组件 HMR
function MyComponent() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <h1>Count: {count}</h1>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  );
}

// HMR 接受
if (module.hot) {
  module.hot.accept('./MyComponent', () => {
    // 组件会自动重新渲染，状态保持
  });
}
```

#### Vue Hot Reload
```javascript
// Vue Loader 自动处理 HMR
module.exports = {
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader'
      }
    ]
  },
  plugins: [
    new VueLoaderPlugin()
  ]
};

// Vue 组件自动支持 HMR
<template>
  <div>
    <h1>{{ message }}</h1>
    <button @click="increment">Count: {{ count }}</button>
  </div>
</template>

<script>
export default {
  data() {
    return {
      message: 'Hello Vue!',
      count: 0
    };
  },
  methods: {
    increment() {
      this.count++;
    }
  }
};
</script>
```

### 8.4 自定义 HMR 处理

#### CSS HMR
```javascript
// CSS 模块 HMR
if (module.hot) {
  module.hot.accept('./style.css', function() {
    // 更新样式
    const link = document.querySelector('link[href*="style.css"]');
    if (link) {
      const newLink = document.createElement('link');
      newLink.rel = 'stylesheet';
      newLink.href = link.href.split('?')[0] + '?' + Date.now();
      newLink.onload = () => link.remove();
      document.head.appendChild(newLink);
    }
  });
}
```

#### 状态保持 HMR
```javascript
// 数据存储模块
let state = {
  user: null,
  preferences: {}
};

export function getState() {
  return state;
}

export function setState(newState) {
  state = { ...state, ...newState };
}

// HMR 状态保持
if (module.hot) {
  if (module.hot.data) {
    // 恢复之前的状态
    state = module.hot.data.state;
  }
  
  module.hot.dispose((data) => {
    // 保存当前状态
    data.state = state;
  });
  
  module.hot.accept();
}
```

#### API 模块 HMR
```javascript
// API 模块
class ApiClient {
  constructor(baseURL) {
    this.baseURL = baseURL;
    this.cache = new Map();
  }
  
  async get(url) {
    if (this.cache.has(url)) {
      return this.cache.get(url);
    }
    
    const response = await fetch(this.baseURL + url);
    const data = await response.json();
    this.cache.set(url, data);
    return data;
  }
  
  clearCache() {
    this.cache.clear();
  }
}

let apiClient = new ApiClient('/api');

export default apiClient;

// HMR 处理
if (module.hot) {
  if (module.hot.data) {
    // 恢复缓存
    apiClient.cache = module.hot.data.cache;
  }
  
  module.hot.dispose((data) => {
    // 保存缓存
    data.cache = apiClient.cache;
  });
  
  module.hot.accept((err) => {
    if (err) {
      console.error('Cannot apply HMR update:', err);
    }
  });
}
```

---

## 9. 实战案例与最佳实践

### 9.1 大型 SPA 应用配置

#### 项目结构
```
project/
├── src/
│   ├── components/
│   ├── pages/
│   ├── services/
│   ├── utils/
│   ├── assets/
│   └── index.js
├── public/
├── webpack/
│   ├── webpack.common.js
│   ├── webpack.dev.js
│   └── webpack.prod.js
└── package.json
```

#### 通用配置 (webpack.common.js)
```javascript
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const { CleanWebpackPlugin } = require('clean-webpack-plugin');

module.exports = {
  entry: {
    app: './src/index.js'
  },
  output: {
    path: path.resolve(__dirname, '../dist'),
    publicPath: '/',
    assetModuleFilename: 'assets/[name].[contenthash:8][ext]'
  },
  module: {
    rules: [
      {
        test: /\.jsx?$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: [
              ['@babel/preset-env', { useBuiltIns: 'usage', corejs: 3 }],
              '@babel/preset-react'
            ],
            plugins: [
              '@babel/plugin-proposal-class-properties',
              ['import', { libraryName: 'antd', style: true }]
            ]
          }
        }
      },
      {
        test: /\.tsx?$/,
        exclude: /node_modules/,
        use: [
          {
            loader: 'babel-loader',
            options: {
              presets: ['@babel/preset-typescript']
            }
          }
        ]
      },
      {
        test: /\.(png|jpe?g|gif|svg)$/i,
        type: 'asset',
        parser: {
          dataUrlCondition: {
            maxSize: 8 * 1024
          }
        }
      },
      {
        test: /\.(woff|woff2|eot|ttf|otf)$/i,
        type: 'asset/resource'
      }
    ]
  },
  plugins: [
    new CleanWebpackPlugin(),
    new HtmlWebpackPlugin({
      template: './public/index.html',
      favicon: './public/favicon.ico'
    })
  ],
  resolve: {
    extensions: ['.js', '.jsx', '.ts', '.tsx'],
    alias: {
      '@': path.resolve(__dirname, '../src'),
      '@components': path.resolve(__dirname, '../src/components'),
      '@pages': path.resolve(__dirname, '../src/pages'),
      '@utils': path.resolve(__dirname, '../src/utils'),
      '@services': path.resolve(__dirname, '../src/services')
    }
  },
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10
        },
        antd: {
          test: /[\\/]node_modules[\\/]antd[\\/]/,
          name: 'antd',
          priority: 15
        },
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom|react-router)[\\/]/,
          name: 'react',
          priority: 20
        },
        utils: {
          test: /[\\/]src[\\/]utils[\\/]/,
          name: 'utils',
          minChunks: 2,
          priority: 5
        }
      }
    }
  }
};
```

#### 开发环境配置 (webpack.dev.js)
```javascript
const { merge } = require('webpack-merge');
const common = require('./webpack.common.js');
const webpack = require('webpack');

module.exports = merge(common, {
  mode: 'development',
  devtool: 'eval-cheap-module-source-map',
  
  module: {
    rules: [
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader', 'postcss-loader']
      },
      {
        test: /\.less$/,
        use: [
          'style-loader',
          'css-loader',
          'postcss-loader',
          {
            loader: 'less-loader',
            options: {
              lessOptions: {
                modifyVars: {
                  '@primary-color': '#1890ff'
                },
                javascriptEnabled: true
              }
            }
          }
        ]
      }
    ]
  },
  
  plugins: [
    new webpack.HotModuleReplacementPlugin()
  ],
  
  devServer: {
    contentBase: './dist',
    hot: true,
    open: true,
    port: 3000,
    historyApiFallback: true,
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
        pathRewrite: {
          '^/api': ''
        }
      }
    }
  },
  
  cache: {
    type: 'memory'
  }
});
```

#### 生产环境配置 (webpack.prod.js)
```javascript
const { merge } = require('webpack-merge');
const common = require('./webpack.common.js');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
const TerserPlugin = require('terser-webpack-plugin');
const CompressionPlugin = require('compression-webpack-plugin');
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');

module.exports = merge(common, {
  mode: 'production',
  devtool: 'source-map',
  
  output: {
    filename: '[name].[contenthash:8].js',
    chunkFilename: '[id].[contenthash:8].chunk.js'
  },
  
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          MiniCssExtractPlugin.loader,
          'css-loader',
          'postcss-loader'
        ]
      },
      {
        test: /\.less$/,
        use: [
          MiniCssExtractPlugin.loader,
          'css-loader',
          'postcss-loader',
          {
            loader: 'less-loader',
            options: {
              lessOptions: {
                modifyVars: {
                  '@primary-color': '#1890ff'
                },
                javascriptEnabled: true
              }
            }
          }
        ]
      }
    ]
  },
  
  plugins: [
    new MiniCssExtractPlugin({
      filename: '[name].[contenthash:8].css',
      chunkFilename: '[id].[contenthash:8].chunk.css'
    }),
    new CompressionPlugin({
      algorithm: 'gzip',
      test: /\.(js|css|html|svg)$/,
      threshold: 8192,
      minRatio: 0.8
    }),
    process.env.ANALYZE && new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      openAnalyzer: false
    })
  ].filter(Boolean),
  
  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: true,
            drop_debugger: true
          }
        }
      }),
      new CssMinimizerPlugin()
    ],
    runtimeChunk: 'single',
    moduleIds: 'deterministic'
  },
  
  cache: {
    type: 'filesystem',
    buildDependencies: {
      config: [__filename]
    }
  }
});
```

### 9.2 微前端架构配置

#### 主应用配置
```javascript
const ModuleFederationPlugin = require('@module-federation/webpack');

module.exports = {
  mode: 'development',
  entry: './src/index.js',
  
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        'header-app': 'headerApp@http://localhost:3001/remoteEntry.js',
        'sidebar-app': 'sidebarApp@http://localhost:3002/remoteEntry.js',
        'content-app': 'contentApp@http://localhost:3003/remoteEntry.js'
      },
      shared: {
        react: { 
          singleton: true,
          requiredVersion: '^17.0.0'
        },
        'react-dom': { 
          singleton: true,
          requiredVersion: '^17.0.0'
        }
      }
    })
  ]
};
```

#### 子应用配置
```javascript
const ModuleFederationPlugin = require('@module-federation/webpack');

module.exports = {
  mode: 'development',
  entry: './src/index.js',
  
  plugins: [
    new ModuleFederationPlugin({
      name: 'headerApp',
      filename: 'remoteEntry.js',
      exposes: {
        './Header': './src/Header',
        './Navigation': './src/Navigation'
      },
      shared: {
        react: { 
          singleton: true,
          requiredVersion: '^17.0.0'
        },
        'react-dom': { 
          singleton: true,
          requiredVersion: '^17.0.0'
        }
      }
    })
  ]
};
```

### 9.3 多页面应用配置

```javascript
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const glob = require('glob');

// 动态生成入口
function getEntries() {
  const entries = {};
  const entryFiles = glob.sync('./src/pages/*/index.js');
  
  entryFiles.forEach(file => {
    const match = file.match(/\/pages\/(.+)\/index\.js$/);
    if (match) {
      entries[match[1]] = file;
    }
  });
  
  return entries;
}

// 动态生成 HTML 插件
function getHtmlPlugins() {
  const entries = getEntries();
  return Object.keys(entries).map(name => 
    new HtmlWebpackPlugin({
      template: `./src/pages/${name}/index.html`,
      filename: `${name}.html`,
      chunks: ['vendors', 'common', name],
      minify: true
    })
  );
}

module.exports = {
  entry: getEntries(),
  
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].[contenthash:8].js'
  },
  
  plugins: [
    ...getHtmlPlugins()
  ],
  
  optimization: {
    splitChunks: {
      cacheGroups: {
        vendors: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
          priority: 10
        },
        common: {
          name: 'common',
          minChunks: 2,
          chunks: 'all',
          priority: 5
        }
      }
    }
  }
};
```

### 9.4 性能监控配置

```javascript
// 性能监控插件
class PerformanceMonitorPlugin {
  constructor(options = {}) {
    this.options = {
      outputPath: 'performance-report.json',
      threshold: {
        assets: 244 * 1024, // 244KB
        chunks: 244 * 1024,
        total: 2 * 1024 * 1024 // 2MB
      },
      ...options
    };
  }
  
  apply(compiler) {
    compiler.hooks.emit.tapAsync('PerformanceMonitorPlugin', (compilation, callback) => {
      const stats = compilation.getStats().toJson();
      const report = this.generateReport(stats);
      
      // 输出报告
      const reportJson = JSON.stringify(report, null, 2);
      compilation.assets[this.options.outputPath] = {
        source: () => reportJson,
        size: () => reportJson.length
      };
      
      // 性能警告
      this.checkPerformance(report);
      
      callback();
    });
  }
  
  generateReport(stats) {
    const totalSize = stats.assets.reduce((sum, asset) => sum + asset.size, 0);
    
    return {
      timestamp: new Date().toISOString(),
      buildTime: stats.time,
      totalSize,
      assets: stats.assets.map(asset => ({
        name: asset.name,
        size: asset.size,
        chunks: asset.chunks
      })),
      chunks: stats.chunks.map(chunk => ({
        id: chunk.id,
        names: chunk.names,
        size: chunk.size,
        modules: chunk.modules?.length || 0
      })),
      warnings: stats.warnings,
      errors: stats.errors
    };
  }
  
  checkPerformance(report) {
    const { threshold } = this.options;
    
    // 检查总体积
    if (report.totalSize > threshold.total) {
      console.warn(`⚠️ Total bundle size (${formatSize(report.totalSize)}) exceeds threshold (${formatSize(threshold.total)})`);
    }
    
    // 检查单个资源
    report.assets.forEach(asset => {
      if (asset.size > threshold.assets) {
        console.warn(`⚠️ Asset "${asset.name}" (${formatSize(asset.size)}) exceeds threshold (${formatSize(threshold.assets)})`);
      }
    });
    
    // 检查 chunk 大小
    report.chunks.forEach(chunk => {
      if (chunk.size > threshold.chunks) {
        console.warn(`⚠️ Chunk "${chunk.names.join(',')}" (${formatSize(chunk.size)}) exceeds threshold (${formatSize(threshold.chunks)})`);
      }
    });
  }
}

function formatSize(bytes) {
  const sizes = ['Bytes', 'KB', 'MB', 'GB'];
  if (bytes === 0) return '0 Bytes';
  const i = Math.floor(Math.log(bytes) / Math.log(1024));
  return Math.round(bytes / Math.pow(1024, i) * 100) / 100 + ' ' + sizes[i];
}

module.exports = PerformanceMonitorPlugin;
```

---

## 10. Webpack 5 新特性

### 10.1 Module Federation

#### 基本概念
Module Federation 允许多个 Webpack 构建一起工作，实现真正的微前端架构。

```javascript
// 主应用 (Host)
const ModuleFederationPlugin = require('@module-federation/webpack');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host_app',
      remotes: {
        remote_app: 'remote_app@http://localhost:3001/remoteEntry.js'
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true }
      }
    })
  ]
};

// 远程应用 (Remote)
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'remote_app',
      filename: 'remoteEntry.js',
      exposes: {
        './Button': './src/Button',
        './utils': './src/utils'
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true }
      }
    })
  ]
};
```

#### 高级用法
```javascript
// 动态远程加载
const RemoteButton = React.lazy(() => import('remote_app/Button'));

// 版本控制
shared: {
  react: {
    singleton: true,
    requiredVersion: '^17.0.0',
    strictVersion: true
  }
}

// 运行时远程
remotes: {
  dynamic_remote: 'promise new Promise(resolve => {/*dynamic logic*/})'
}
```

### 10.2 Asset Modules

#### 内置资源处理
```javascript
module.exports = {
  module: {
    rules: [
      {
        test: /\.png$/,
        type: 'asset/resource', // 生成文件
        generator: {
          filename: 'images/[hash][ext][query]'
        }
      },
      {
        test: /\.svg$/,
        type: 'asset/inline' // 内联 base64
      },
      {
        test: /\.txt$/,
        type: 'asset/source' // 导出源代码
      },
      {
        test: /\.jpg$/,
        type: 'asset', // 自动选择
        parser: {
          dataUrlCondition: {
            maxSize: 4 * 1024 // 4kb
          }
        }
      }
    ]
  }
};
```

### 10.3 Web Workers 支持

```javascript
// 语法糖支持
const worker = new Worker(new URL('./worker.js', import.meta.url));

// 配置
module.exports = {
  output: {
    workerChunkFilename: '[name].[contenthash].worker.js'
  }
};
```

### 10.4 顶级 await

```javascript
// 直接在模块顶层使用 await
const data = await fetch('/api/data').then(r => r.json());

export default data;

// 配置支持
module.exports = {
  experiments: {
    topLevelAwait: true
  }
};
```

### 10.5 改进的缓存机制

```javascript
module.exports = {
  cache: {
    type: 'filesystem',
    version: '1.0', // 缓存版本
    cacheDirectory: path.resolve(__dirname, '.webpack-cache'),
    store: 'pack', // 存储格式
    buildDependencies: {
      config: [__filename], // 配置文件依赖
      tsconfig: [path.resolve(__dirname, 'tsconfig.json')]
    },
    managedPaths: [path.resolve(__dirname, 'node_modules')],
    profile: false,
    maxAge: 5184000000, // 60天
    allowCollectingMemory: true,
    compression: 'gzip'
  }
};
```

### 10.6 Tree Shaking 增强

#### 嵌套的 tree shaking
```javascript
// utils/index.js
export { default as formatDate } from './formatDate';
export { default as formatCurrency } from './formatCurrency';

// main.js
import { formatDate } from './utils'; // 只有 formatDate 被打包
```

#### sideEffects 字段增强
```javascript
// package.json
{
  "sideEffects": [
    "*.css",
    "*.less",
    "./src/polyfills.js"
  ]
}
```

### 10.7 实验性功能

#### CSS 作为模块
```javascript
module.exports = {
  experiments: {
    css: true
  }
};

// 可以直接导入 CSS
import styles from './styles.css';
```

#### 懒编译
```javascript
module.exports = {
  experiments: {
    lazyCompilation: {
      imports: true,
      entries: false,
      backend: {
        client: 'webpack-dev-server/client/index.js?hot=true&live-reload=true',
        server: {
          listen: 3001
        }
      }
    }
  }
};
```

---

## 总结

Webpack 作为现代前端工程化的核心工具，其原理的深入理解对于提高开发效率和项目性能至关重要。通过本指南的学习，你应该能够：

1. **掌握核心概念**: 理解 Entry、Output、Loader、Plugin、Mode 的作用和配置
2. **理解编译流程**: 了解从源码到最终产物的完整构建过程
3. **掌握模块机制**: 理解不同模块规范的处理和依赖分析
4. **熟练使用 Loader**: 能够配置和开发自定义 Loader
5. **熟练使用 Plugin**: 能够配置和开发自定义 Plugin
6. **优化构建性能**: 掌握各种优化策略和最佳实践
7. **实现代码分割**: 合理拆分代码，提高加载性能
8. **使用热更新**: 提高开发体验
9. **应用最新特性**: 利用 Webpack 5 的新功能

### 实践建议

1. **渐进式学习**: 从基础配置开始，逐步深入高级特性
2. **项目实践**: 在实际项目中应用所学知识
3. **性能监控**: 建立完整的构建性能监控体系
4. **持续优化**: 根据项目需求持续优化配置
5. **关注更新**: 跟进 Webpack 社区的最新发展

### 进一步学习资源

- [Webpack 官方文档](https://webpack.js.org/)
- [Webpack 源码解析](https://github.com/webpack/webpack)
- [Tapable 事件系统](https://github.com/webpack/tapable)
- [Module Federation 指南](https://module-federation.github.io/)

通过系统性的学习和实践，你将能够充分发挥 Webpack 的强大功能，构建高效、现代化的前端应用。