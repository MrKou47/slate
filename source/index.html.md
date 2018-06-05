---
title: understand_webpack

language_tabs: # must be one of https://git.io/vQNgJ
  - javascript

toc_footers:
  - <a href='#'>Learn webpack with source code</a>
  - <a href='https://github.com/lord/slate'>Powered by Slate</a>

search: true
---

# 前言

**注意!** 本文并不是 webpack 的官方文档。官方文档请移步 ➡ [webpack](https://webpack.js.org)

本文档期望，经过对 webpack 源码的阅读，来更深入的了解这个工具。也方便日后在处理 webpack 的 error 时能够更快速的定位和解决。
阅读本文档之前，需要你已经能够熟练使用 webpack ，并且详细阅读过 webpack 的官方文档。

<aside class="notice">
版本:   
webpack: 3.12.0  
tapable: 0.2.8  
</aside>


# 从 tapable 开始..

我们都知道 webpack 采用插件的机制，使用 tapable 来完成事件的注册与调用。tapable 实际上就是一个 [Pub/Sub](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) 模式的实现。



> tapable 核心实现

```js
function Tapable() {
	this._plugins = {};
}
module.exports = Tapable;

// 注册事件
Tapable.prototype.plugin = function plugin(name, fn) {
	if(Array.isArray(name)) {
		name.forEach(function(name) {
			this.plugin(name, fn);
		}, this);
		return;
	}
	if(!this._plugins[name]) this._plugins[name] = [fn];
	else this._plugins[name].push(fn);
};

// 触发事件
Tapable.prototype.applyPlugins = function applyPlugins(name) {
	if(!this._plugins[name]) return;
	var args = Array.prototype.slice.call(arguments, 1);
	var plugins = this._plugins[name];
	for(var i = 0; i < plugins.length; i++)
		plugins[i].apply(this, args);
};

Tapable.prototype.apply = function apply() {
	for(var i = 0; i < arguments.length; i++) {
		arguments[i].apply(this);
	}
};
```

具体包含

- 注册事件
- 触发事件
- 退订事件

tapable 提供了多种对于回调函数的调用方式，webpack 也主要通过使用这些方法来控制整体构建流程。

## apply

将tapable代码精简后，可以看到 tapable 并没有退订的实现，同时增加了一个 `apply` 的实现。

我们知道书写插件的时候插件需要提供一个 apply 方法。我们定义的这个 `apply` 方法就是在此处调用的。具体调用方式为

```js
compilation.apply(new CustomerPlugin()); // then called (new CustomerPlugin()).apply(this) modify context 
```

## applyPlugins

普通的事件触发方式。通过使用 `Array.prototype.slice.call(arguments, 1)` 来获取事件需要的参数 `tapable.plugin('eventName', function(params){})`。

```js
Tapable.prototype.applyPlugins = function applyPlugins(name) {
	if(!this._plugins[name]) return;
	var args = Array.prototype.slice.call(arguments, 1);
	var plugins = this._plugins[name];
	for(var i = 0; i < plugins.length; i++)
		plugins[i].apply(this, args);
};
```

## applyPluginsBailResult
 
和 `applyPlugins` 类似。此方法提供了返回值。当在执行事件处理函数时，如果函数有返回值，则直接返回该值，同时跳出循环。

```js
Tapable.prototype.applyPluginsBailResult = function applyPluginsBailResult(name) {
	if(!this._plugins[name]) return;
	var args = Array.prototype.slice.call(arguments, 1);
	var plugins = this._plugins[name];
	for(var i = 0; i < plugins.length; i++) {
		var result = plugins[i].apply(this, args);
		if(typeof result !== "undefined") {
			return result;
		}
	}
};
```

同时提供了 `applyPluginsBailResult0`, `applyPluginsBailResult1`, `applyPluginsBailResult2`, `applyPluginsBailResult3`, `applyPluginsBailResult4`, `applyPluginsBailResult5`

需要注意的是，tapable 会通过定义不同的方法来指定事件需要的参数(如 `applyPlugins1(name, param)`, `applyPlugins2(name, param1, param2)`，
从而省去 `Array.prototype.slice.call(arguments, 1)` 这个步骤。

## applyPluginsWaterfall

`applyPluginsWaterfall(name: string, init: any): any`  
定义的事件处理函数将顺序执行，同时每一个事件处理函数接受的值为上一个处理函数执行的结果。可通过设置 `init` 参数来配置初始值。
此方法带有返回值，值为最后一个事件处理函数的返回值。

```js
Tapable.prototype.applyPluginsWaterfall = function applyPluginsWaterfall(name, init) {
	if(!this._plugins[name]) return init;
	var args = Array.prototype.slice.call(arguments, 1);
	var plugins = this._plugins[name];
	var current = init;
	for(var i = 0; i < plugins.length; i++) {
		args[0] = current;
		current = plugins[i].apply(this, args);
	}
	return current;
};
```

同时提供 `applyPluginsWaterfall0`, `applyPluginsWaterfall1`, `applyPluginsWaterfall2`

## applyPluginsAsyncWaterfall

异步的 `applyPluginsWaterfall` 的实现，用于顺序执行事件处理函数。流程控制。

```js
Tapable.prototype.applyPluginsAsyncWaterfall =
  function applyPluginsAsyncWaterfall(name, init, callback) {
    if(!this._plugins[name] || this._plugins[name].length === 0) return callback(null, init);
    var plugins = this._plugins[name];
    var i = 0;
    var _this = this;
    var next = copyProperties(callback, function(err, value) {
      if(err) return callback(err);
      i++;
      if(i >= plugins.length) {
        return callback(null, value);
      }
      plugins[i].call(_this, value, next);
    });
    plugins[0].call(this, init, next);
  };
```

## applyPluginsAsync/applyPluginsAsyncSeries

```js
Tapable.prototype.applyPluginsAsyncSeries =
Tapable.prototype.applyPluginsAsync =
  function applyPluginsAsyncSeries(name) {
    var args = Array.prototype.slice.call(arguments, 1);
    var callback = args.pop();
    var plugins = this._plugins[name];
    if(!plugins || plugins.length === 0) return callback();
    var i = 0;
    var _this = this;
    args.push(copyProperties(callback, function next(err) { // 一个经过包装后的 callback 函数。之后的事件处理函数接受的 callback 为此函数
      if(err) return callback(err);
      i++;
      if(i >= plugins.length) {
        return callback();
      }
      plugins[i].apply(_this, args);
    }));
    plugins[0].apply(this, args);
  };
```

顺序执行所有注册事件 注册的函数通过 callback 调用来继续执行下一个函数。可用于异步流程控制。
提供了 `applyPluginsAsyncSeries1`

## applyPluginsParallel

并行执行事件处理函数。一旦事件处理出错。则不再处理其他函数，同时触发 `callback(err)`;

```js
Tapable.prototype.applyPluginsParallel =
  function applyPluginsParallel(name) {
    var args = Array.prototype.slice.call(arguments, 1);
    var callback = args.pop();
    if(!this._plugins[name] || this._plugins[name].length === 0) return callback();
    var plugins = this._plugins[name];
    var remaining = plugins.length;
    args.push(copyProperties(callback, function(err) {
      if(remaining < 0) return; // ignore
      if(err) {
        remaining = -1;
        return callback(err);
      } 
      remaining--;
      if(remaining === 0) {
        return callback();
      }
    }));
    for(var i = 0; i < plugins.length; i++) {
      plugins[i].apply(this, args);
      if(remaining < 0) return;
    }
  };
```

## 总结

tapable 共提供了 6 种流程控制方法。其中：

- applyPlugins 循环调用 不能保证完成顺序
- applyPluginsWaterfall 循环调用 可用于值的传递 拥有同步返回值
- applyPluginsAsyncWaterfall 在保证完成顺序的情况下做值的传递 可异步获取返回值
- applyPluginsAsync 在保证完成顺序的情况下调用事件处理函数
- applyPluginsParallel 并行执行事件处理函数

理解这几个方法对于理解 webpack 工作流程非常重要。

# webpack

让我们从运行 webpack 开始。平时我们可能从 *shell* 运行 `$ webpack`, 也可能调用 `webpack(webpackConfig).run()` 来执行 webpack 方法。这一切的开始都包含了对于 webpack 配置的处理。
这是一系列普通但是繁琐的过程。 包含了默认值的定义，配置的 `merge`，配置可用性的判断等等。我们先不考虑这一部分的功能，而是从 `webpack(webpackConfig).run()` 入手，使用一个最简单的配置来了解 webpack 的整体流程。

当然，在此之前我们可以再看一下 [compiler-hooks](https://webpack.js.org/api/compiler-hooks) 与 [compilation-hooks](https://webpack.js.org/api/compilation-hooks)。我们可以理解为，在创造 webpack 的时候，
人们预先约定了这些 hooks，用于分阶段的处理我们的代码，从而更好的对整体流程进行控制，也便于第三方插件的引入。学习 webpack 工作的机制，也就是学习 hooks 的工作流程。当我们弄清了这些 hook 在什么时候挂载、什么时候执行的时候，基本上
也就弄清了 webpack 的整个流程。

## 入口

我们使用 [commonjs](https://github.com/webpack/webpack/tree/master/examples/commonjs) 作为例子。

> debug.js

```js
const webpack = require('../../lib/webpack');
const path = require('path');

webpack({
  entry: './example.js',
  output: {
    path: path.join(__dirname, 'dist'),
    filename: 'bundle.js'
  }
}, (err) => {
  console.log(err);
})
```

> webpack.js

```js
function webpack(options, callback) {
	const webpackOptionsValidationErrors = validateSchema(webpackOptionsSchema, options);
	if(webpackOptionsValidationErrors.length) {
		throw new WebpackOptionsValidationError(webpackOptionsValidationErrors);
	}
	let compiler;
	if(Array.isArray(options)) {
		compiler = new MultiCompiler(options.map(options => webpack(options)));
	} else if(typeof options === "object") {
		// TODO webpack 4: process returns options
		new WebpackOptionsDefaulter().process(options);

		compiler = new Compiler();
		compiler.context = options.context;
		compiler.options = options;
		new NodeEnvironmentPlugin().apply(compiler);
		if(options.plugins && Array.isArray(options.plugins)) {
			compiler.apply.apply(compiler, options.plugins);
		}
		compiler.applyPlugins("environment");
		compiler.applyPlugins("after-environment");
		compiler.options = new WebpackOptionsApply().process(options, compiler);
	} else {
		throw new Error("Invalid argument: options");
	}
	if(callback) {
		if(typeof callback !== "function") throw new Error("Invalid argument: callback");
		if(options.watch === true || (Array.isArray(options) && options.some(o => o.watch))) {
			const watchOptions = Array.isArray(options) ? options.map(o => o.watchOptions || {}) : (options.watchOptions || {});
			return compiler.watch(watchOptions, callback);
		}
		compiler.run(callback);
	}
	return compiler;
}
```

配置很简单，只有 entry，output， 基本的流程也很简单

- options 验证
- 根据 options 的类型, 创建 `MultiCompiler` 或 `Compiler`
- 创建 `Compiler`
  - `new NodeEnvironmentPlugin().apply(compiler)` 为 compiler 增加 `before-run` hook 
  - 调用 `environment` 与 `after-environment` hook
  - `new WebpackOptionsApply().process(options, compiler)` 为 compiler 添加 hook
- 判断是否有 callback 参数来执行 webpack

## WebpackOptionsApply

`WebpackOptionsApply.js` 里引入了所有的插件，并根据传入的 option 的不同来应用到 compiler 对象上。
主要有针对 `target` 的处理， `output.library`, `output.libraryTarget` 的处理、`externals` 的处理

> 这些插件能够处理所有的模块系统格式

```js
const LoaderPlugin = require("./dependencies/LoaderPlugin");
const CommonJsPlugin = require("./dependencies/CommonJsPlugin");
const HarmonyModulesPlugin = require("./dependencies/HarmonyModulesPlugin");
const SystemPlugin = require("./dependencies/SystemPlugin");
const ImportPlugin = require("./dependencies/ImportPlugin");
const AMDPlugin = require("./dependencies/AMDPlugin");
const RequireContextPlugin = require("./dependencies/RequireContextPlugin");
const RequireEnsurePlugin = require("./dependencies/RequireEnsurePlugin");
const RequireIncludePlugin = require("./dependencies/RequireIncludePlugin");
```

> 比较重要的 target="web" target="node"

```js
switch(options.target) {
  case "web":
    JsonpTemplatePlugin = require("./JsonpTemplatePlugin");
    NodeSourcePlugin = require("./node/NodeSourcePlugin");
    compiler.apply(
      new JsonpTemplatePlugin(options.output),
      new FunctionModulePlugin(options.output),
      new NodeSourcePlugin(options.node),
      new LoaderTargetPlugin(options.target)
    );
    break;
  case "node":
  case "async-node":
    NodeTemplatePlugin = require("./node/NodeTemplatePlugin");
    NodeTargetPlugin = require("./node/NodeTargetPlugin");
    compiler.apply(
      new NodeTemplatePlugin({
        asyncChunkLoading: options.target === "async-node"
      }),
      new FunctionModulePlugin(options.output),
      new NodeTargetPlugin(),
      new LoaderTargetPlugin("node")
    );
    break;
  default:
    throw new Error("Unsupported target '" + options.target + "'.");
}
```

基本流程可以概括为

- options.target
   - web
   - webworker
   - node/async-node
   - node-webkit
   - atom/electron/electron-main
   - electron-renderer
- library, libraryTarget
  - LibraryTemplatePlugin
- externals
  - ExternalsPlugin
- devtool
- 增加 `entry-option` hook
- 触发 `entry-option` (applyPluginsBailResult)
- 应用插件
  - CompatibilityPlugin
  - HarmonyModulesPlugin
  - AMDPlugin
  - CommonJsPlugin
  - LoaderPlugin
  - NodeStuffPlugin
  - RequireJsStuffPlugin
  - APIPlugin
  - ConstPlugin
  - UseStrictPlugin
  - RequireIncludePlugin
  - RequireEnsurePlugin
  - RequireContextPlugin
  - ImportPlugin
  - SystemPlugin
  - EnsureChunkConditionsPlugin
  - RemoveParentModulesPlugin
  - RemoveEmptyChunksPlugin
  - MergeDuplicateChunksPlugin
  - FlagIncludedChunksPlugin
  - OccurrenceOrderPlugin(true
  - FlagDependencyExportsPlugin
  - FlagDependencyUsagePlugin
- options.performance
  - SizeLimitsPlugin
- 应用插件
  - TemplatedPathPlugin
  - RecordIdsPlugin
  - WarnCaseSensitiveModulesPlugin
- options.cache
  - CachePlugin
- 触发 `after-plugin` 钩子(applyPlugins)
- compiler.resolvers 增加解析属性
- 触发 `after-resolvers` 钩子(applyPlugins)

