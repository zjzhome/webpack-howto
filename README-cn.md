# 开始学习Webpack

## 本文目标

本文讲述如何使用webpack将事情做好，包括了我们在Instagram做的大多数事情，我们没做的当然没有:)

我的建议：以此文作为学习webpack的开始，然后再阅读官方的文档说明。

## 预备知识

* 要了解browserify，RequireJS或者其他类似的东东
* 了解以下事情的价值:
  * 模块划分
  * 异步加载
  * 打包静态资源，比如图片和CSS

## 1、为什么使用webpack

* **和browserify类似**，但可以将你的app划分为不同的文件。 如果你的单页应用你有多个页面，用户只需下载当前页的代码，如果他们离开此页浏览其他页面，不用再次下载那些通用的代码。

* **通常情况下可以替代grunt和gulp**，因为webpack可以构建和打包CSS，预处理CSS、可以编译为JS的语言和图片等等。

webpack支持AMD和CommonJS以及其他的模块系统（Angular、ES6）。如果你不知道用哪种，使用CommonJS。

## 2、对于使用Browserify的童鞋

下面的两个命令是等价的：

```js
browserify main.js > bundle.js
```

```js
webpack main.js bundle.js
```

不过，webpack比browserify更加给力，一般情况下，我们会创建`webpack.config.js`来保证条理性。

```js
// webpack.config.js
module.exports = {
  entry: './main.js',
  output: {
    filename: 'bundle.js'       
  }
};
```

这完全是我们熟悉的JS，所以随意将挥洒你的代码。

## 3、如果调用webpack

切换到包含`webpack.config.js`的目录, 运行：

* `webpack` 构建一次，开发环境下使用
* `webpack -p` 构建一次，生产环境使用（压缩）
* `webpack --watch` 持续构建生产环境下使用（构建速度快）
* `webpack -d` 生成sourcemap

## 4、可编译为JS的语言

webpack中和browserify转换器以及RequireJS中的插件等价的是`loader`。下面展示了如何让webpack支持加载CoffeeScript和Facebook JSX+ES6（你需要首先`npm install babel-loader coffee-loader`）):

```js
// webpack.config.js
module.exports = {
  entry: './main.js',
  output: {
    filename: 'bundle.js'       
  },
  module: {
    loaders: [
      { test: /\.coffee$/, loader: 'coffee-loader' },
      { test: /\.js$/, loader: 'babel-loader' }
    ]
  }
};
```
为了在加载文件的时候不必生命文件扩展名，你必须加上`resolve.extensions`这个参数来告诉webpack该寻找哪些文件.

```js
// webpack.config.js
module.exports = {
  entry: './main.js',
  output: {
    filename: 'bundle.js'       
  },
  module: {
    loaders: [
      { test: /\.coffee$/, loader: 'coffee-loader' },
      { test: /\.js$/, loader: 'babel-loader' }
    ]
  },
  resolve: {
    // you can now require('file') instead of require('file.coffee')
    extensions: ['', '.js', '.json', '.coffee'] 
  }
};
```

## 5、样式和图片

首先，在你的代码中使用`require()`来引入你的静态资源。

```js
require('./bootstrap.css');
require('./myapp.less');

var img = document.createElement('img');
img.src = require('./glyph.png');
```

当你引入CSS（或者less等），webpack会将CSS作为字符串打包到JS中，然后`require()`会在页面中插入`<style>`标签。当引入图片资源的时候，webpack会将图片地址打包进JS，`require()`返回图片地址。

但是你要告诉webpack怎么做：

```js
// webpack.config.js
module.exports = {
  entry: './main.js',
  output: {
    path: './build', // This is where images AND js will go
    publicPath: 'http://mycdn.com/', // This is used to generate URLs to e.g. images
    filename: 'bundle.js'
  },
  module: {
    loaders: [
      { test: /\.less$/, loader: 'style-loader!css-loader!less-loader' }, // use ! to chain loaders
      { test: /\.css$/, loader: 'style-loader!css-loader' },
      {test: /\.(png|jpg)$/, loader: 'url-loader?limit=8192'} // inline base64 URLs for <=8k images, direct URLs for the rest
    ]
  }
};
```

## 特征标识

有些代码我们只想用在开发环境（比如日志）或者内部的服务器（比如处于内测的未发布产品特性）。在你的代码里，使用以下几个变量：

```js
if (__DEV__) {
  console.warn('Extra logging');
}
// ...
if (__PRERELEASE__) {
  showSecretFeature();
}
```

然后告诉webpack这些全局变量：

```js
// webpack.config.js

// definePlugin takes raw strings and inserts them, so you can put strings of JS if you want.
var definePlugin = new webpack.DefinePlugin({
  __DEV__: JSON.stringify(JSON.parse(process.env.BUILD_DEV || 'true')),
  __PRERELEASE__: JSON.stringify(JSON.parse(process.env.BUILD_PRERELEASE || 'false'))
});

module.exports = {
  entry: './main.js',
  output: {
    filename: 'bundle.js'       
  },
  plugins: [definePlugin]
};
```

然后你可以在命令行里使用`BUILD_DEV=1 BUILD_PRERELEASE=1 webpack`命令构建，不过注意因为`webpack -p`命令会去清除无用的代码，所有抱在这些代码块里的代码都会被删除，所以不要泄露私密特性或字符串。

## 7、多入口

假设你有一个用户资料页面和一个订阅页面，用户在浏览资料页面的你不想用户下载订阅页面的代码。所以要打多个包，为每一个页面创建一个“主要模块”（也叫作入口）：

```js
// webpack.config.js
module.exports = {
  entry: {
    Profile: './profile.js',
    Feed: './feed.js'
  },
  output: {
    path: 'build',
    filename: '[name].js' // Template based on keys in entry above
  }
};
```

对于资料页，在页面插入`<script src="build/Profile.js"></script>`，订阅页面也是一样。

## 8、优化通用代码

订阅页面和资料页面共用许多代码（像React以及其他的样式和组件等）。webpack可以抽出他们共用的代码，然后打一个通用包在各个页面缓存。

```js
// webpack.config.js

var webpack = require('webpack');

var commonsPlugin =
  new webpack.optimize.CommonsChunkPlugin('common.js');

module.exports = {
  entry: {
    Profile: './profile.js',
    Feed: './feed.js'
  },
  output: {
    path: 'build',
    filename: '[name].js' // Template based on keys in entry above
  },
  plugins: [commonsPlugin]
};
```

在上一步的script标签之前加上`<script src="build/common.js"></script>`，然后尽情享受免费的高速缓存吧。

## 9、异步加载

CommonJS是同步的，但是Webpack提供一个方法来异步的指定依赖。这在客户端路由中很有用，每个页面都需要路由，你不想直到你真正需要他们的时候才下载相关功能。

指定你想异步加载的**分割点**，比如：

```js
if (window.location.pathname === '/feed') {
  showLoadingState();
  require.ensure([], function() { // this syntax is weird but it works
    hideLoadingState();
    require('./feed').show(); // when this function is called, the module is guaranteed to be synchronously available.
  });
} else if (window.location.pathname === '/profile') {
  showLoadingState();
  require.ensure([], function() {
    hideLoadingState();
    require('./profile').show();
  });
}
```

webpack会做接下来的工作，并且产生额外的块文件，然后加载他们。

当你加载这些文件到html的script标签，webpack假定他们在根目录，你可以通过`output.publicPath`来定义:

```js
// webpack.config.js
output: {
    path: "/home/proj/public/assets", //path to where webpack will build your stuff
    publicPath: "/assets/" //path that will be considered when requiring your files
}
```

## 其他资源

看一下真实的案例：一个成功的团队是如何使用webpack的。这是Pete Hunt在全球开源大会上关于Intagram使用的webpack的讨论：[http://youtu.be/VkTCL6Nqm6Y](http://youtu.be/VkTCL6Nqm6Y)。

# FAQ

## webpack看起来不是模块化的

webpack是**严格**模块化的。和browserify和requirejs等替代选择工具相比，webpack的伟大之处在于在构建过程中他让插件将自己注入到尽可能多的地方。webpack很多看似内建的功能只是默认加载的插件，他们是可以重写的（比如CommonJS的require()语法分析器）。