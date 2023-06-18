# 【前端工程化】webpack5从零搭建完整的vue3+ts开发和打包环境 

## 目录

1.  前言
2.  初始化项目
3.  配置基础版**vue3**+**ts**环境
4.  常用功能配置
5.  优化构建速度
6.  优化构建结果文件
7.  总结

**全文概览**

![xmind.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3b88fa105a2f4548bd0fd683aad9d208~tplv-k3u1fbpfcp-watermark.image?)

## 一. 前言

本文是专栏[《前端工程化》](https://juejin.cn/column/7092293219064479752 "https://juejin.cn/column/7092293219064479752")第8篇文章，会持续更新前端工程化方面详细高质量的详细教程

从**2020**年**10**月**10**日，**webpack** 升级至 5 版本到现在已经快两年，**webpack5**版本优化了很多原有的功能比如**tree-shakin**g优化，也新增了很多新特性，比如联邦模块，具体变动可以看这篇文章[阔别两年，webpack 5 正式发布了！](https://juejin.cn/post/6882663278712094727)。

本文将使用最新的**webpack5**一步一步从零搭建一个完整的**vue3+ts**开发和打包环境，配置完善的模块热替换以及**构建速度**和**构建结果**的优化，完整代码已上传到[webpack5-vue3-ts](https://github.com/guojiongwei/webpack5-vue3-ts.git)。本文只是配置**webpack5**的，配置**代码规范和工程化**会在下一篇文章发布。

## 二. 初始化项目

在开始**webpack**配置之前，先手动初始化一个基本的**vue3**+**ts**项目，新建项目文件夹**webpack5-vue3-18**, 在项目下执行

```bash
npm init -y
```

初始化好**package.json**后,在项目下新增以下所示目录结构和文件

```yaml
├── build
|   ├── webpack.base.js # 公共配置
|   ├── webpack.dev.js  # 开发环境配置
|   └── webpack.prod.js # 打包环境配置
├── public
│   └── index.html # html模板
├── src
|   ├── App.vue 
│   └── index.ts # vue3应用入口页面
├── tsconfig.json  # ts配置
└── package.json
```

安装**webpack**依赖

```sh
npm i webpack@5.85.1 webpack-cli@5.1.3 -D
```

安装**vue3**依赖

```sh
npm i vue@^3.3.4 -S
```

添加**public/index.html**内容

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>webpack5-vue3-ts</title>
</head>
<body>
  <!-- 容器节点 -->
  <div id="root"></div>
</body>
</html>
```

添加**tsconfig.json**内容

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "module": "ESNext",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "preserve",

    /* Linting */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["src/**/*.ts", "src/**/*.d.ts", "src/**/*.vue"],
}
```

添加**src/App.vue**内容

```vue
<template>
  <h2>webpack5-vue3-ts</h2>
</template>

<script setup lang="ts">
</script>
<style scoped>
</style>
```

添加**src/index.ts**内容

```ts
import { createApp } from 'vue'
import App from './App.vue'

createApp(App).mount('#root')
```

现在项目业务代码已经添加好了,接下来可以配置**webpack**的代码了。

## 三. 配置基础版vue3+ts环境

### 2.1. webpack公共配置

修改**webpack.base.js**

**1. 配置入口文件**

```js
// webpack.base.js
const path = require('path')

module.exports = {
  entry: path.join(__dirname, '../src/index.ts'), // 入口文件
}
```

**2. 配置出口文件**

```js
// webpack.base.js
const path = require('path')

module.exports = {
  // ...
  // 打包文件出口
  output: {
    filename: 'static/js/[name].js', // 每个输出js的名称
    path: path.join(__dirname, '../dist'), // 打包结果输出路径
    clean: true, // webpack4需要配置clean-webpack-plugin来删除dist文件,webpack5内置了
    publicPath: '/' // 打包后文件的公共前缀路径
  },
}
```

**3. 配置loader解析ts和vue**

由于**webpack**默认只能识别**js**文件,不能识别**vue**语法,需要配置**loader**的预设预设 [**@babel/preset-typescript**](https://www.npmjs.com/package/vue-loader) 来先**ts**语法转换为 **js** 语法。再借助 [**vue-loader**](https://www.npmjs.com/package/vue-loader) 来识别**vue**语法。

**安装babel核心模块和babel预设以及vue-loader**

```sh
npm i babel-loader@^9.1.2 @babel/core@^7.22.1 @babel/preset-typescript@^7.21.5 vue-loader@^17.2.2 -D
```

在**webpack.base.js**添加**module.rules**配置和**vue-loader**对应插件配置

```js
// webpack.base.js
const { VueLoaderPlugin } = require('vue-loader')
module.exports = {
  // ...
  module: {
    rules: [
      {
      	test: /\.vue$/, // 匹配.vue文件
      	use: 'vue-loader', // 用vue-loader去解析vue文件
      },
      {
        test: /\.ts$/, // 匹配.ts文件
        use: {
          loader: 'babel-loader',
          options: {
            presets: [
              [
                "@babel/preset-typescript",
                {
                  allExtensions: true, //支持所有文件扩展名(重要)
                },
              ],
            ]
          }
        }
      }
    ]
  },
  plugins: [
    // ...
    new VueLoaderPlugin(), // vue-loader插件
  ]
}
```

单文件.vue文件被 vue-loader 解析三个部分，script 部分会由 ts 来处理，但是ts不会处理 .vue 结尾的文件，所以要在预设里面加上 allExtensions: true配置项来支持所有文件扩展名。

**4. 配置extensions**

**extensions**是**webpack**的**resolve**解析配置下的选项，在引入模块时不带文件后缀时，会来该配置数组里面依次添加后缀查找文件，因为**ts**不支持引入以 **.ts**, **.vue**为后缀的文件，所以要在**extensions**中配置，而第三方库里面很多引入**js**文件没有带后缀，所以也要配置下**js**

修改**webpack.base.js**，注意把高频出现的文件后缀放在前面

```js
// webpack.base.js
module.exports = {
  // ...
  resolve: {
    extensions: ['.vue', '.ts', '.js', '.json'],
  }
}
```

这里只配置**js**, **vue**和**ts和json**，其他文件引入都要求带后缀，可以提升构建速度。

**4. 添加html-webpack-plugin插件**

**webpack**需要把最终构建好的静态资源都引入到一个**html**文件中,这样才能在浏览器中运行,[html-webpack-plugin](https://www.npmjs.com/package/html-webpack-plugin)就是来做这件事情的,安装依赖：

```sh
npm i html-webpack-plugin -D
```

因为该插件在开发和构建打包模式都会用到,所以还是放在公共配置**webpack.base.js**里面

```js
// webpack.base.js
const path = require('path')
const HtmlWebpackPlugin = require('html-webpack-plugin')

module.exports = {
  // ...
  plugins: [
    new HtmlWebpackPlugin({
      template: path.resolve(__dirname, '../public/index.html'), // 模板取定义root节点的模板
      inject: true, // 自动注入静态资源
    })
  ]
}
```

到这里一个最基础的**vue3**基本公共配置就已经配置好了,需要在此基础上分别配置开发环境和打包环境了。

### 2.2. webpack开发环境配置

**1. 安装 webpack-dev-server**

开发环境配置代码在**webpack.dev.js**中,需要借助 [webpack-dev-server](https://www.npmjs.com/package/webpack-dev-server)在开发环境启动服务器来辅助开发,还需要依赖[webpack-merge](https://www.npmjs.com/package/webpack-merge)来合并基本配置,安装依赖:

```sh
npm i webpack-dev-server webpack-merge -D
```

修改**webpack.dev.js**代码, 合并公共配置，并添加开发模式配置

```js
// webpack.dev.js
const path = require('path')
const { merge } = require('webpack-merge')
const baseConfig = require('./webpack.base.js')

// 合并公共配置,并添加开发环境配置
module.exports = merge(baseConfig, {
  mode: 'development', // 开发模式,打包更加快速,省了代码优化步骤
  devtool: 'eval-cheap-module-source-map', // 源码调试模式,后面会讲
  devServer: {
    port: 3000, // 服务端口号
    compress: false, // gzip压缩,开发环境不开启,提升热更新速度
    hot: true, // 开启热更新，后面会讲vue3模块热替换具体配置
    historyApiFallback: true, // 解决history路由404问题
    static: {
      directory: path.join(__dirname, "../public"), //托管静态资源public文件夹
    }
  }
})
```

**2. package.json添加dev脚本**

在**package.json**的**scripts**中添加

```js
// package.json
"scripts": {
  "dev": "webpack-dev-server -c build/webpack.dev.js"
},
```

**-c 是 --config的缩写，是指定webpack-dev-server的配置文件。**

执行**npm run dev**,就能看到项目已经启动起来了,访问<http://localhost:3000/>,就可以看到项目界面,具体完善的**vue3**模块热替换在下面会讲到。

### 2.3. webpack打包环境配置

**1. 修改webpack.prod.js代码**

```js
// webpack.prod.js
const { merge } = require('webpack-merge')
const baseConfig = require('./webpack.base.js')
module.exports = merge(baseConfig, {
  mode: 'production', // 生产模式,会开启tree-shaking和压缩代码,以及其他优化
})
```

**2. package.json添加build打包命令脚本**

在**package.json**的**scripts**中添加**build**打包命令

```js
"scripts": {
    "dev": "webpack-dev-server -c build/webpack.dev.js",
    "build": "webpack -c build/webpack.prod.js"
},
```

执行**npm run build**,最终打包在**dist**文件中, 打包结果:

```sh
dist                    
├── static
|   ├── js
|     ├── main.js
├── index.html
```

**3. 浏览器查看打包结果**

打包后的**dist**文件可以在本地借助**node**服务器**serve**打开,全局安装**serve**

```sh
npm i serve -g
```

然后在项目根目录命令行执行**serve -s dist**,就可以启动打包后的项目了。

到现在一个基础的支持**vue3**和**ts**的**webpack5**就配置好了,但只有这些功能是远远不够的,还需要继续添加其他配置。

## 四. 基础功能配置

### 4.1 配置环境变量

环境变量按作用来分分两种

1.  区分是开发模式还是打包构建模式
2.  区分项目业务环境,开发/测试/预测/正式环境

区分开发模式还是打包构建模式可以用**process.env.NODE\_ENV**,因为很多第三方包里面判断都是采用的这个环境变量。

区分项目接口环境可以自定义一个环境变量**process.env.BASE\_ENV**,设置环境变量可以借助[cross-env](https://www.npmjs.com/package/cross-env)和[webpack.DefinePlugin](https://www.webpackjs.com/plugins/define-plugin/)来设置。

*   **cross-env**：兼容各系统的设置环境变量的包
*   **webpack.DefinePlugin**：**webpack**内置的插件,可以为业务代码注入环境变量

安装**cross-env**

```sh
npm i cross-env -D
```

修改**package.json**的**scripts**脚本字段,删除原先的**dev**和**build**,改为

```js
"scripts": {
    "dev:dev": "cross-env NODE_ENV=development BASE_ENV=development webpack-dev-server -c build/webpack.dev.js",
    "dev:test": "cross-env NODE_ENV=development BASE_ENV=test webpack-dev-server -c build/webpack.dev.js",
    "dev:pre": "cross-env NODE_ENV=development BASE_ENV=pre webpack-dev-server -c build/webpack.dev.js",
    "dev:prod": "cross-env NODE_ENV=development BASE_ENV=production webpack-dev-server -c build/webpack.dev.js",
    
    "build:dev": "cross-env NODE_ENV=production BASE_ENV=development webpack -c build/webpack.prod.js",
    "build:test": "cross-env NODE_ENV=production BASE_ENV=test webpack -c build/webpack.prod.js",
    "build:pre": "cross-env NODE_ENV=production BASE_ENV=pre webpack -c build/webpack.prod.js",
    "build:prod": "cross-env NODE_ENV=production BASE_ENV=production webpack -c build/webpack.prod.js"
  }
```

**dev**开头是开发模式,**build**开头是打包模式,冒号后面对应的**dev**/**test**/**pre**/**prod**是对应的业务环境的**开发**/**测试**/**预测**/**正式**环境。

**process.env.NODE\_ENV**环境变量**webpack**会自动根据设置的**mode**字段来给业务代码注入对应的**development**和**prodction**,这里在命令中再次设置环境变量**NODE\_ENV**是为了在**webpack**和**babel**的配置文件中访问到。

在**webpack.base.js**中打印一下设置的环境变量

```js
// webpack.base.js
// ...
console.log('NODE_ENV', process.env.NODE_ENV)
console.log('BASE_ENV', process.env.BASE_ENV)
```

执行**npm run build:dev**,可以看到打印的信息

```js
// NODE_ENV production
// BASE_ENV development
```

当前是打包模式,业务环境是开发环境,这里需要把**process.env.BASE\_ENV**注入到业务代码里面,就可以通过该环境变量设置对应环境的接口地址和其他数据,要借助**webpack.DefinePlugin**插件。

修改**webpack.base.js**

```js
// webpack.base.js
// ...
const webpack = require('webpack')
module.export = {
  // ...
  plugins: [
    // ...
    new webpack.DefinePlugin({
      'process.env.BASE_ENV': JSON.stringify(process.env.BASE_ENV),
      'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV)
    })
  ]
}
```

配置后会把值注入到业务代码里面去,**webpack**解析代码匹配到**process.env.BASE\_ENV**,就会设置到对应的值。测试一下，在**src/index.vue**打印一下两个环境变量

```vue
// src/index.vue
// ...
console.log('NODE_ENV', process.env.NODE_ENV)
console.log('BASE_ENV', process.env.BASE_ENV)
```

执行**npm run dev:test**,可以在浏览器控制台看到打印的信息

```js
// NODE_ENV development
// BASE_ENV test
```

当前是开发模式,业务环境是测试环境。

### 4.2 处理css和less文件

在**src**下新增**app.css**

```css
h2 {
    color: red;
    transform: translateY(100px);
}
```

在**src/App.vue**中引入**app.css**

```vue
<template>
  <h2>webpack5-vue3-ts</h2>
</template>

<script setup lang="ts">
  import './app.css'
</script>

<style scoped>
</style>
```

执行打包命令**npm run build:dev**,会发现有报错, 因为**webpack**默认只认识**js**,是不识别**css**文件的,需要使用**loader**来解析**css**, 安装依赖

```sh
npm i style-loader css-loader -D
```

*   **style-loader**: 把解析后的**css**代码从**js**中抽离,放到头部的**style**标签中(在运行时做的)
*   **css-loader:** 解析**css**文件代码

因为解析**css**的配置开发和打包环境都会用到,所以加在公共配置**webpack.base.js**中

```js
// webpack.base.js
// ...
module.exports = {
  // ...
  module: { 
    rules: [
      // ...
      {
        test: /\.css$/, //匹配 css 文件
        use: ['style-loader','css-loader']
      }
    ]
  },
  // ...
}
```

上面提到过,**loader**执行顺序是从右往左,从下往上的,匹配到**css**文件后先用**css-loader**解析**css**, 最后借助**style-loader**把**css**插入到头部**style**标签中。

配置完成后再**npm run build:dev**打包,借助**serve -s dist**启动后在浏览器查看,可以看到样式生效了。

![1.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6a1fa1f5a4854af1a9e511685075516a~tplv-k3u1fbpfcp-watermark.image?)

### 4.3 支持less或scss

项目开发中,为了更好的提升开发体验,一般会使用**css**超集**less**或者**scss**,对于这些超集也需要对应的**loader**来识别解析。以**less**为例,需要安装依赖:

```sh
npm i less-loader less -D
```

*   **less-loader**: 解析**less**文件代码,把**less**编译为**css**
*   **less**: **less**核心

实现支持**less**也很简单,只需要在**rules**中添加**less**文件解析,遇到**less**文件,使用**less-loader**解析为**css**,再进行**css**解析流程,修改**webpack.base.js**：

```js
// webpack.base.js
module.exports = {
  // ...
  module: {
    // ...
    rules: [
      // ...
      {
        test: /.(css|less)$/, //匹配 css和less 文件
        use: ['style-loader','css-loader', 'less-loader']
      }
    ]
  },
  // ...
}
```

测试一下,新增**src/app.less**

```less
#root {
  h2 {
    font-size: 20px;
  }
}
```

在**App.vue**中引入**app.less**,执行**npm run build:dev**打包,借助**serve -s dist**启动项目,可以看到**less**文件编写的样式编译**css**后也插入到**style**标签了了。

![2.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b66c45a5b0fd45fd9ffa219a965f8ce5~tplv-k3u1fbpfcp-watermark.image?)

### 4.4 处理css3前缀兼容(可省略)

使用了**vue3**了基本上就不用考虑低版本浏览器了，但有时候需要给**css3**加前缀，可以学一下，可以借助插件来自动加前缀, [postcss-loader](https://link.juejin.cn/?target=https%3A%2F%2Fwebpack.docschina.org%2Floaders%2Fpostcss-loader%2F)就是来给**css3**加浏览器前缀的,安装依赖：

```sh
npm i postcss-loader autoprefixer -D
```

*   **postcss-loader**：处理**css**时自动加前缀
*   **autoprefixer**：决定添加哪些浏览器前缀到**css**中

修改**webpack.base.js**, 在解析**css**和**less**的规则中添加配置

```js
module.exports = {
  // ...
  module: { 
    rules: [
      // ...
      {
        test: /.(css|less)$/, //匹配 css和less 文件
        use: [
          'style-loader',
          'css-loader',
          // 新增
          {
            loader: 'postcss-loader',
            options: {
              postcssOptions: {
                plugins: ['autoprefixer']
              }
            }
          },
          'less-loader'
        ]
      }
    ]
  },
  // ...
}
```

配置完成后,需要有一份要兼容浏览器的清单,让**postcss-loader**知道要加哪些浏览器的前缀,在根目录创建 **.browserslistrc**文件

```sh
IE 9 # 兼容IE 9
chrome 35 # 兼容chrome 35
```

以兼容到**ie9**和**chrome35**版本为例,配置好后,执行**npm run build:dev**打包,可以看到打包后的**css**文件已经加上了**ie**和谷歌内核的前缀

![3.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/303cf8ce1a5145a29502ab715b3055a9~tplv-k3u1fbpfcp-watermark.image?)

上面可以看到解析**css**和**less**有很多重复配置,可以进行提取**postcss-loader**配置优化一下

**postcss.config.js**是**postcss-loader**的配置文件,会自动读取配置,根目录新建**postcss.config.js**：

```js
module.exports = {
  plugins: ['autoprefixer']
}
```

修改**webpack.base.js**, 取消**postcss-loader**的**options**配置

```js
// webpack.base.js
// ...
module.exports = {
  // ...
  module: { 
    rules: [
      // ...
      {
        test: /\.(css|less)$/, //匹配 css和less 文件
        use: [
          'style-loader',
          'css-loader',
          'postcss-loader',
          'less-loader'
        ]
      },
    ]
  },
  // ...
}
```

提取**postcss-loader**配置后,再次打包,可以看到依然可以解析**css**, **less**文件, **css3**对应前缀依然存在。

### 4.5 babel预设处理js兼容

现在**js**不断新增很多方便好用的标准语法来方便开发,甚至还有非标准语法比如装饰器,都极大的提升了代码可读性和开发效率。但前者标准语法很多低版本浏览器不支持,后者非标准语法所有的浏览器都不支持。需要把最新的标准语法转换为低版本语法,把非标准语法转换为标准语法才能让浏览器识别解析,而**babel**就是来做这件事的,这里只讲配置,更详细的可以看[Babel 那些事儿](https://juejin.cn/post/6992371845349507108)。

安装依赖

```sh
npm i babel-loader @babel/core @babel/preset-env core-js -D
```

*   babel-loader: 使用 **babel** 加载最新js代码并将其转换为 **ES5**（上面已经安装过）
*   @babel/corer: **babel** 编译的核心包
*   @babel/preset-env: **babel** 编译的预设,可以转换目前最新的**js**标准语法
*   core-js: 使用低版本**js**语法模拟高版本的库,也就是垫片

修改**webpack.base.js**

```js
// webpack.base.js
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.ts$/,
        use: {
          loader: 'babel-loader',
          options: {
            // 执行顺序由右往左,所以先处理ts,再处理jsx,最后再试一下babel转换为低版本语法
            presets: [
              [
                "@babel/preset-env",
                {
                  // 设置兼容目标浏览器版本,这里可以不写,babel-loader会自动寻找上面配置好的文件.browserslistrc
                  // "targets": {
                  //  "chrome": 35,
                  //  "ie": 9
                  // },
                   "useBuiltIns": "usage", // 根据配置的浏览器兼容,以及代码中使用到的api进行引入polyfill按需添加
                   "corejs": 3, // 配置使用core-js低版本
                  }
                ],
              [
                '@babel/preset-typescript',
                {
                  allExtensions: true, //支持所有文件扩展名，很关键
                },
              ]
            ]
          }
        }
      }
    ]
  }
}
```

此时再打包就会把语法转换为对应浏览器兼容的语法了。

为了避免**webpack**配置文件过于庞大,可以把**babel-loader**的配置抽离出来, 新建**babel.config.js**文件,使用**js**作为配置文件,是因为可以访问到**process.env.NODE\_ENV**环境变量来区分是开发还是打包模式。

```js
// babel.config.js
module.exports = {
  // 执行顺序由右往左,所以先处理ts,再处理jsx,最后再试一下babel转换为低版本语法
  "presets": [
    [
      "@babel/preset-env",
      {
        // 设置兼容目标浏览器版本,这里可以不写,babel-loader会自动寻找上面配置好的文件.browserslistrc
        // "targets": {
        //  "chrome": 35,
        //  "ie": 9
        // },
        "useBuiltIns": "usage", // 根据配置的浏览器兼容,以及代码中使用到的api进行引入polyfill按需添加
        "corejs": 3 // 配置使用core-js使用的版本
      }
    ],
    [
      "@babel/preset-typescript",
      {
        allExtensions: true, //支持所有文件扩展名，很关键
      },
    ],
  ]
}
```

移除**webpack.base.js**中**babel-loader**的**options**配置

```js
// webpack.base.js
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.ts$/,
        use: 'babel-loader'
      },
      // 如果使用到了js，可以把js文件配置加上
      // {
      //  test: /.(js)$/,
      //  use: 'babel-loader'
      // }
      // ...
    ]
  }
}
```

### 4.8 复制public文件夹

一般**public**文件夹都会放一些静态资源,可以直接根据绝对路径引入,比如**图片**,**css**,**js**文件等,不需要**webpack**进行解析,只需要打包的时候把**public**下内容复制到构建出口文件夹中,可以借助[copy-webpack-plugin](https://www.npmjs.com/package/copy-webpack-plugin)插件,安装依赖

```sh
npm i copy-webpack-plugin -D
```

开发环境已经在**devServer**中配置了**static**托管了**public**文件夹,在开发环境使用绝对路径可以访问到**public**下的文件,但打包构建时不做处理会访问不到,所以现在需要在打包配置文件**webpack.prod.js**中新增**copy**插件配置。

```js
// webpack.prod.js
// ..
const path = require('path')
const CopyPlugin = require('copy-webpack-plugin');
module.exports = merge(baseConfig, {
  mode: 'production',
  plugins: [
    // 复制文件插件
    new CopyPlugin({
      patterns: [
        {
          from: path.resolve(__dirname, '../public'), // 复制public下文件
          to: path.resolve(__dirname, '../dist'), // 复制到dist目录中
          filter: source => {
            return !source.includes('index.html') // 忽略index.html
          }
        },
      ],
    }),
  ]
})
```

在上面的配置中,忽略了**index.html**,因为**html-webpack-plugin**会以**public**下的**index.html**为模板生成一个**index.html**到**dist**文件下,所以不需要再复制该文件了。

测试一下,在**public**中新增一个[**favicon.ico**](https://guojiongwei.top/favicon.ico)图标文件,在**index.html**中引入

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <!-- 绝对路径引入图标文件 -->
  <link data-n-head="ssr" rel="icon" type="image/x-icon" href="/favicon.ico">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>webpack5-vue3-ts</title>
</head>
<body>
  <!-- 容器节点 -->
  <div id="root"></div>
</body>
</html>
```

再执行**npm run build:dev**打包,就可以看到**public**下的**favicon.ico**图标文件被复制到**dist**文件中了。

![4.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2fc4609967d242fb9606004fd47bd971~tplv-k3u1fbpfcp-watermark.image?)

### 4.7 处理图片文件

对于图片文件,**webpack4**使用**file-loader**和**url-loader**来处理的,但**webpack5**不使用这两个**loader**了,而是采用自带的[**asset-module**](https://webpack.js.org/guides/asset-modules/#root)来处理

修改**webpack.base.js**,添加图片解析配置

```js
module.exports = {
  module: {
    rules: [
      // ...
      {
        test:/.(png|jpg|jpeg|gif|svg)$/, // 匹配图片文件
        type: "asset", // type选择asset
        parser: {
          dataUrlCondition: {
            maxSize: 10 * 1024, // 小于10kb转base64位
          }
        },
        generator:{ 
          filename:'static/images/[name][ext]', // 文件输出目录和命名
        },
      },
    ]
  }
}
```

测试一下,准备一张小于[10kb](https://github.com/guojiongwei/webpack5-vue3-ts/blob/main/src/assets/imgs/5kb.png)的图片和大于[10kb](https://github.com/guojiongwei/webpack5-vue3-ts/blob/main/src/assets/imgs/22kb.png)的图片,放在**src/assets/imgs**目录下, 修改**App.vue**:

```vue
<template>
  <img :src="smallImg" alt="小于10kb的图片" />
  <img :src="bigImg" alt="大于于10kb的图片" />
</template>

<script setup lang="ts">
  import smallImg from './assets/imgs/5kb.png'
  import bigImg from './assets/imgs/22kb.png'
  import './app.css'
  import './app.less'
</script>

<style scoped>
</style>
```

> 这个时候在引入图片的地方会报：**找不到模块“./assets/imgs/22kb.png”或其相应的类型声明**，需要添加一个图片的声明文件

新增**src/images.d.ts**文件，添加内容

```js
declare module '*.svg'
declare module '*.png'
declare module '*.jpg'
declare module '*.jpeg'
declare module '*.gif'
declare module '*.bmp'
declare module '*.tiff'
declare module '*.less'
declare module '*.css'
```

添加图片声明文件后,就可以正常引入图片了, 然后执行**npm run build:dev**打包,借助**serve -s dist**查看效果,可以看到可以正常解析图片了,并且小于**10kb**的图片被转成了**base64**位格式的。

![5.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bf479c32d9954bdfbeb70efbe40ad771~tplv-k3u1fbpfcp-watermark.image?)

**css**中的背景图片一样也可以解析,修改**app.vue**。

```vue
<template>
  <img :src="smallImg" alt="小于10kb的图片" />
  <img :src="bigImg" alt="大于于10kb的图片" />
  <!-- 小图片背景容器 -->
  <div className='smallImg'></div>
  <!-- 大图片背景容器 -->
  <div className='bigImg'></div>
</template>

<script setup lang="ts">
  import smallImg from './assets/imgs/5kb.png'
  import bigImg from './assets/imgs/22kb.png'
  import './app.css'
  import './app.less'
</script>

<style scoped>
</style>
```

修改**app.less**

```less
// app.less
#root {
  .smallImg {
    width: 69px;
    height: 75px;
    background: url('./assets/imgs/5kb.png') no-repeat;
  }
  .bigImg {
    width: 232px;
    height: 154px;
    background: url('./assets/imgs/22kb.png') no-repeat;
  }
}
```

可以看到背景图片也一样可以识别,小于**10kb**转为**base64**位。

![6.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cf92a2d51bb1455d834bc42de187dea9~tplv-k3u1fbpfcp-watermark.image?)

### 4.8 处理字体和媒体文件

字体文件和媒体文件这两种资源处理方式和处理图片是一样的,只需要把匹配的路径和打包后放置的路径修改一下就可以了。修改**webpack.base.js**文件：

```js
// webpack.base.js
module.exports = {
  module: {
    rules: [
      // ...
      {
        test:/.(woff2?|eot|ttf|otf)$/, // 匹配字体图标文件
        type: "asset", // type选择asset
        parser: {
          dataUrlCondition: {
            maxSize: 10 * 1024, // 小于10kb转base64位
          }
        },
        generator:{ 
          filename:'static/fonts/[name][ext]', // 文件输出目录和命名
        },
      },
      {
        test:/.(mp4|webm|ogg|mp3|wav|flac|aac)$/, // 匹配媒体文件
        type: "asset", // type选择asset
        parser: {
          dataUrlCondition: {
            maxSize: 10 * 1024, // 小于10kb转base64位
          }
        },
        generator:{ 
          filename:'static/media/[name][ext]', // 文件输出目录和命名
        },
      },
    ]
  }
}
```

## 五. 优化构建速度

### 5.1 构建耗时分析

当进行优化的时候,肯定要先知道时间都花费在哪些步骤上了,而[speed-measure-webpack-plugin](https://link.juejin.cn/?target=https%3A%2F%2Fwww.npmjs.com%2Fpackage%2Fspeed-measure-webpack-plugin)插件可以帮我们做到,安装依赖：

```sh
npm i speed-measure-webpack-plugin -D
```

使用的时候为了不影响到正常的开发/打包模式,我们选择新建一个配置文件,新增**webpack**构建分析配置文件**build/webpack.analy.js**

```js
const prodConfig = require('./webpack.prod.js') // 引入打包配置
const SpeedMeasurePlugin = require('speed-measure-webpack-plugin'); // 引入webpack打包速度分析插件
const smp = new SpeedMeasurePlugin(); // 实例化分析插件
const { merge } = require('webpack-merge') // 引入合并webpack配置方法

// 使用smp.wrap方法,把生产环境配置传进去,由于后面可能会加分析配置,所以先留出合并空位
module.exports = smp.wrap(merge(prodConfig, {

}))
```

修改**package.json**添加启动**webpack**打包分析脚本命令,在**scripts**新增：

```js
{
  // ...
  "scripts": {
    // ...
    "build:analy": "cross-env NODE_ENV=production BASE_ENV=production webpack -c build/webpack.analy.js"
  }
  // ...
}
```

执行**npm run build:analy**命令

![6.1.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1afc736a148a49a99d854a723c54148a~tplv-k3u1fbpfcp-watermark.image?)

可以在图中看到各**plugin**和**loader**的耗时时间,现在因为项目内容比较少,所以耗时都比较少,在真正的项目中可以通过这个来分析打包时间花费在什么地方,然后来针对性的优化。

### 5.2 开启持久化存储缓存

在**webpack5**之前做缓存是使用**babel-loader**缓存解决**js**的解析结果,**cache-loader**缓存**css**等资源的解析结果,还有模块缓存插件**hard-source-webpack-plugin**,配置好缓存后第二次打包,通过对文件做哈希对比来验证文件前后是否一致,如果一致则采用上一次的缓存,可以极大地节省时间。

**webpack5** 较于 **webpack4**,新增了持久化缓存、改进缓存算法等优化,通过配置 [webpack 持久化缓存](https%3A%2F%2Fwebpack.docschina.org%2Fconfiguration%2Fcache%2F%23root),来缓存生成的 **webpack** 模块和 **chunk**,改善下一次打包的构建速度,可提速 **90%** 左右,配置也简单，修改**webpack.base.js**

```js
// webpack.base.js
// ...
module.exports = {
  // ...
  cache: {
    type: 'filesystem', // 使用文件缓存
  },
}
```

当前文章代码的测试结果

| 模式     | 第一次耗时  | 第二次耗时 |
| ------ | ------ | ----- |
| 启动开发模式 | 2869毫秒 | 687毫秒 |
| 启动打包模式 | 5455毫秒 | 552毫秒 |

通过开启**webpack5**持久化存储缓存,再次打包的时间提升了**90%**。

![7.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6365790dba09490cbf5aa48cc8df4704~tplv-k3u1fbpfcp-watermark.image?)

缓存的存储位置在**node\_modules/.cache/webpack**,里面又区分了**development**和**production**缓存

![8转存失败，建议直接上传图片文件](/Users/guojiongwei/Desktop/md文章/webpack5-vue3-ts/8.png)

### 5.3 开启多线程loader

**webpack**的**loader**默认在单线程执行,现代电脑一般都有多核**cpu**,可以借助多核**cpu**开启多线程**loader**解析,可以极大地提升**loader**解析的速度,[thread-loader](https://link.juejin.cn/?target=https%3A%2F%2Fwebpack.docschina.org%2Floaders%2Fthread-loader%2F%23root)就是用来开启多进程解析**loader**的,安装依赖

```sh
npm i thread-loader -D
```

使用时,需将此 **loader** 放置在其他 **loader** 之前。放置在此 **loader** 之后的 **loader** 会在一个独立的 **worker** 池中运行。

修改**webpack.base.js**

```js
// webpack.base.js
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.vue/,
        use: ['thread-loader', 'vue-loader']
      },
      {
        test: /\.ts$/,
        use: ['thread-loader', 'babel-loader']
      }
    ]
  }
}
```

由于**thread-loader**不支持抽离css插件**MiniCssExtractPlugin.loader**(下面会讲),所以这里只配置了多进程解析**js**,开启多线程也是需要启动时间,大约**500ms**左右,所以适合规模比较大的项目。

### 5.4 配置alias别名

**webpack**支持设置别名**alias**,设置别名可以让后续引用的地方减少路径的复杂度。

修改**webpack.base.js**

```js
module.export = {
  // ...
   resolve: {
    // ...
    alias: {
      '@': path.join(__dirname, '../src')
    }
  }
}
```

修改**tsconfig.json**,添加**baseUrl**和**paths**

```js
{
  "compilerOptions": {
    // ...
    "baseUrl": ".",
    "paths": {
      "@/*": [
        "src/*"
      ]
    }
  }
}
```

配置修改完成后,在项目中使用 **@/xxx.xx**,就会指向项目中**src/xxx.xx,**在**js/ts**文件和**vue**文件中都可以用。

**src/App.vue**可以修改为

```vue
<template>
  <img :src="smallImg" alt="小于10kb的图片" />
  <img :src="bigImg" alt="大于于10kb的图片" />
  <!-- 小图片背景容器 -->
  <div className='smallImg'></div>
  <!-- 大图片背景容器 -->
  <div className='bigImg'></div>
</template>

<script setup lang="ts">
  import smallImg from '@/assets/imgs/5kb.png'
  import bigImg from '@/assets/imgs/22kb.png'
  import '@/app.css'
  import '@/app.less'
</script>

<style scoped>
</style>
```

**src/app.less**可以修改为

```less
// app.less
#root {
  .smallImg {
    width: 69px;
    height: 75px;
    background: url('@/assets/imgs/5kb.png') no-repeat;
  }
  .bigImg {
    width: 232px;
    height: 154px;
    background: url('@/assets/imgs/22kb.png') no-repeat;
  }
}
```

### 5.5 缩小loader作用范围

一般第三库都是已经处理好的,不需要再次使用**loader**去解析,可以按照实际情况合理配置**loader**的作用范围,来减少不必要的**loader**解析,节省时间,通过使用 **include**和**exclude** 两个配置项,可以实现这个功能,常见的例如：

*   **include**：只解析该选项配置的模块
*   **exclude**：不解该选项配置的模块,优先级更高

修改**webpack.base.js**

```js
// webpack.base.js
const path = require('path')
module.exports = {
  // ...
  module: {
    rules: [
      {
        include: [path.resolve(__dirname, '../src')], 只对项目src文件的vue进行loader解析
        test: /\.vue$/,
        use: ['thread-loader', 'vue-loader']
      },
      {
  			include: [path.resolve(__dirname, '../src')], 只对项目src文件的ts进行loader解析
        test: /\.ts/,
        use: ['thread-loader', 'babel-loader']
      },
    ]
  }
}
```

其他**loader**也是相同的配置方式,如果除**src**文件外也还有需要解析的,就把对应的目录地址加上就可以了,比如需要引入**antd**的**css**,可以把**antd**的文件目录路径添加解析**css**规则到**include**里面。

### 5.6 精确使用loader

**loader**在**webpack**构建过程中使用的位置是在**webpack**构建模块依赖关系引入新文件时，会根据文件后缀来倒序遍历**rules**数组，如果文件后缀和**test**正则匹配到了，就会使用该**rule**中配置的**loader**依次对文件源代码进行处理，最终拿到处理后的**sourceCode**结果，可以通过避免使用无用的**loader**解析来提升构建速度，比如使用**less-loader**解析**css**文件。

可以拆分上面配置的**less**和**css**, 避免让**less-loader**再去解析**css**文件

```js
// webpack.base.js
// ...
module.exports = {
  module: {
    // ...
    rules: [
      // ...
      {
        test: /\.css$/, //匹配所有的 css 文件
        include: [path.resolve(__dirname, '../src')],
        use: [
          'style-loader',
          'css-loader',
          'postcss-loader'
        ]
      },
      {
        test: /\.less$/, //匹配所有的 less 文件
        include: [path.resolve(__dirname, '../src')],
        use: [
          'style-loader',
          'css-loader',
          'postcss-loader',
          'less-loader'
        ]
      },
    ]
  }
}
```

### 5.7 缩小模块搜索范围

**node**里面模块有三种

*   **node**核心模块
*   **node\_modules**模块
*   自定义文件模块

使用**require**和**import**引入模块时如果有准确的相对或者绝对路径,就会去按路径查询,如果引入的模块没有路径,会优先查询**node**核心模块,如果没有找到会去当前目录下**node\_modules**中寻找,如果没有找到会查从父级文件夹查找**node\_modules**,一直查到系统**node**全局模块。

这样会有两个问题,一个是当前项目没有安装某个依赖,但是上一级目录下**node\_modules**或者全局模块有安装,就也会引入成功,但是部署到服务器时可能就会找不到造成报错,另一个问题就是一级一级查询比较消耗时间。可以告诉**webpack**搜索目录范围,来规避这两个问题。

修改**webpack.base.js**

```js
// webpack.base.js
const path = require('path')
module.exports = {
  // ...
  resolve: {
     // ...
     // 如果用的是pnpm 就暂时不要配置这个，会有幽灵依赖的问题，访问不到很多模块。
     modules: [path.resolve(__dirname, '../node_modules')], // 查找第三方模块只在本项目的node_modules中查找
  },
}
```

### 5.8 devtool 配置

开发过程中或者打包后的代码都是**webpack**处理后的代码,如果进行调试肯定希望看到源代码,而不是编译后的代码, [source map](http://blog.teamtreehouse.com/introduction-source-maps)就是用来做源码映射的,不同的映射模式会明显影响到构建和重新构建的速度, [**devtool**](https://webpack.js.org/configuration/devtool/)选项就是**webpack**提供的选择源码映射方式的配置。

**devtool**的命名规则为 **^(inline-|hidden-|eval-)?(nosources-)?(cheap-(module-)?)?source-map\$**

| 关键字       | 描述                                           |
| --------- | -------------------------------------------- |
| inline    | 代码内通过 dataUrl 形式引入 SourceMap                 |
| hidden    | 生成 SourceMap 文件,但不使用                         |
| eval      | `eval(...)` 形式执行代码,通过 dataUrl 形式引入 SourceMap |
| nosources | 不生成 SourceMap                                |
| cheap     | 只需要定位到行信息,不需要列信息                             |
| module    | 展示源代码中的错误位置                                  |

开发环境推荐：**eval-cheap-module-source-map**

*   本地开发首次打包慢点没关系,因为 **eval** 缓存的原因,  热更新会很快
*   开发中,我们每行代码不会写的太长,只需要定位到行就行,所以加上 **cheap**
*   我们希望能够找到源代码的错误,而不是打包后的,所以需要加上 **module**

修改**webpack.dev.js**

```js
// webpack.dev.js
module.exports = {
  // ...
  devtool: 'eval-cheap-module-source-map'
}
```

打包环境推荐：**none**(就是不配置**devtool**选项了，不是配置**devtool**: '**none**')

```js
// webpack.prod.js
module.exports = {
  // ...
  // devtool: '', // 不用配置devtool此项
}
```

*   **none**话调试只能看到编译后的代码,也不会泄露源代码,打包速度也会比较快。
*   只是不方便线上排查问题, 但一般都可以根据报错信息在本地环境很快找出问题所在。

### 5.9 其他优化配置

除了上面的配置外，**webpack**还提供了其他的一些优化方式,本次搭建没有使用到，所以只简单罗列下

*   [**externals**](https://www.webpackjs.com/configuration/externals/): 外包拓展，打包时会忽略配置的依赖，会从上下文中寻找对应变量
*   [**module.noParse**](https://www.webpackjs.com/configuration/module/#module-noparse): 匹配到设置的模块,将不进行依赖解析，适合**jquery**,**boostrap**这类不依赖外部模块的包
*   [**ignorePlugin**](https://webpack.js.org/plugins/ignore-plugin/#root): 可以使用正则忽略一部分文件，常在使用多语言的包时可以把非中文语言包过滤掉

## 六. 优化构建结果文件

### 6.1 webpack包分析工具

[webpack-bundle-analyzer](https://www.npmjs.com/package/webpack-bundle-analyzer)是分析**webpack**打包后文件的插件,使用交互式可缩放树形图可视化 **webpack** 输出文件的大小。通过该插件可以对打包后的文件进行观察和分析,可以方便我们对不完美的地方针对性的优化,安装依赖：

```sh
npm install webpack-bundle-analyzer -D
```

修改**webpack.analy.js**

```js
// webpack.analy.js
const prodConfig = require('./webpack.prod.js')
const SpeedMeasurePlugin = require('speed-measure-webpack-plugin');
const smp = new SpeedMeasurePlugin();
const { merge } = require('webpack-merge')
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer') // 引入分析打包结果插件
module.exports = smp.wrap(merge(prodConfig, {
  plugins: [
    new BundleAnalyzerPlugin() // 配置分析打包结果插件
  ]
}))
```

配置好后,执行**npm run build:analy**命令,打包完成后浏览器会自动打开窗口,可以看到打包文件的分析结果页面,可以看到各个文件所占的资源大小。

![9.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b013a30d57d4b09ba2ddbc3fd30a9eb~tplv-k3u1fbpfcp-watermark.image?)

### 6.2 抽取css样式文件

在开发环境我们希望**css**嵌入在**style**标签里面,方便样式热替换,但打包时我们希望把**css**单独抽离出来,方便配置缓存策略。而插件[mini-css-extract-plugin](https://github.com/webpack-contrib/mini-css-extract-plugin)就是来帮我们做这件事的,安装依赖：

```sh
npm i mini-css-extract-plugin -D
```

修改**webpack.base.js**, 根据环境变量设置开发环境使用**style-looader**,打包模式抽离**css**

```js
// webpack.base.js
// ...
const MiniCssExtractPlugin = require('mini-css-extract-plugin')
const isDev = process.env.NODE_ENV === 'development' // 是否是开发模式
module.exports = {
  // ...
  module: { 
    rules: [
      // ...
      {
        test: /.css$/, //匹配所有的 css 文件
        include: [path.resolve(__dirname, '../src')],
        use: [
          // 开发环境使用style-looader,打包模式抽离css
          isDev ? 'style-loader' : MiniCssExtractPlugin.loader,
          'css-loader',
          'postcss-loader'
        ]
      },
      {
        test: /.less$/, //匹配所有的 less 文件
        include: [path.resolve(__dirname, '../src')],
        use: [
          // 开发环境使用style-looader,打包模式抽离css
          isDev ? 'style-loader' : MiniCssExtractPlugin.loader,
          'css-loader',
          'postcss-loader',
          'less-loader'
        ]
      },
    ]
  },
  // ...
}
```

再修改**webpack.prod.js**, 打包时添加抽离css插件

```js
// webpack.prod.js
// ...
const MiniCssExtractPlugin = require('mini-css-extract-plugin')
module.exports = merge(baseConfig, {
  mode: 'production',
  plugins: [
    // ...
    // 抽离css插件
    new MiniCssExtractPlugin({
      filename: 'static/css/[name].css' // 抽离css的输出目录和名称
    }),
  ]
})
```

配置完成后,在开发模式**css**会嵌入到**style**标签里面,方便样式热替换,打包时会把**css**抽离成单独的**css**文件。

### 6.3 压缩css文件

上面配置了打包时把**css**抽离为单独**css**文件的配置,打开打包后的文件查看,可以看到默认**css**是没有压缩的,需要手动配置一下压缩**css**的插件。

![10.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2beb58e8f23f4468a6b9287b72bed0c7~tplv-k3u1fbpfcp-watermark.image?)

可以借助[css-minimizer-webpack-plugin](https://link.juejin.cn/?target=https%3A%2F%2Fwww.npmjs.com%2Fpackage%2Fcss-minimizer-webpack-plugin)来压缩css,安装依赖

```sh
npm i css-minimizer-webpack-plugin -D
```

修改**webpack.prod.js**文件， 需要在优化项[optimization](https://webpack.js.org/configuration/optimization/)下的[minimizer](https://webpack.js.org/configuration/optimization/#optimizationminimizer)属性中配置

```js
// webpack.prod.js
// ...
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin')
module.exports = {
  // ...
  optimization: {
    minimizer: [
      new CssMinimizerPlugin(), // 压缩css
    ],
  },
}
```

再次执行打包就可以看到**css**已经被压缩了。

### 6.4 压缩js文件

设置**mode**为**production**时,**webpack**会使用内置插件[terser-webpack-plugin](https://www.npmjs.com/package/terser-webpack-plugin)压缩**js**文件,该插件默认支持多线程压缩,但是上面配置**optimization.minimizer**压缩**css**后,**js**压缩就失效了,需要手动再添加一下,**webpack**内部安装了该插件,由于**pnpm**解决了幽灵依赖问题,如果用的**pnpm**的话,需要手动再安装一下依赖。

```sh
npm i terser-webpack-plugin -D
```

修改**webpack.prod.js**文件

```js
// ...
const TerserPlugin = require('terser-webpack-plugin')
module.exports = {
  // ...
  optimization: {
    minimizer: [
      // ...
      new TerserPlugin({ // 压缩js
        parallel: true, // 开启多线程压缩
        terserOptions: {
          compress: {
            pure_funcs: ["console.log"] // 删除console.log
          }
        }
      }),
    ],
  },
}
```

配置完成后再打包,**css**和**js**就都可以被压缩了。

### 6.5 合理配置打包文件hash

项目维护的时候,一般只会修改一部分代码,可以合理配置文件缓存,来提升前端加载页面速度和减少服务器压力,而**hash**就是浏览器缓存策略很重要的一部分。**webpack**打包的**hash**分三种：

*   **hash**：跟整个项目的构建相关,只要项目里有文件更改,整个项目构建的**hash**值都会更改,并且全部文件都共用相同的**hash**值
*   **chunkhash**：不同的入口文件进行依赖文件解析、构建对应的**chunk**,生成对应的哈希值,文件本身修改或者依赖文件修改,**chunkhash**值会变化
*   **contenthash**：每个文件自己单独的 **hash** 值,文件的改动只会影响自身的 **hash** 值

**hash**是在输出文件时配置的,格式是**filename: "\[name].\[chunkhash:8]\[ext]"**,**\[xx]** 格式是**webpack**提供的占位符, **:8**是生成**hash**的长度。

| 占位符         | 解释                 |
| ----------- | ------------------ |
| ext         | 文件后缀名              |
| name        | 文件名                |
| path        | 文件相对路径             |
| folder      | 文件所在文件夹            |
| hash        | 每次构建生成的唯一 hash 值   |
| chunkhash   | 根据 chunk 生成 hash 值 |
| contenthash | 根据文件内容生成hash 值     |

因为**js**我们在生产环境里会把一些公共库和程序入口文件区分开,单独打包构建,采用**chunkhash**的方式生成哈希值,那么只要我们不改动公共库的代码,就可以保证其哈希值不会受影响,可以继续使用浏览器缓存,所以**js**适合使用**chunkhash**。

**css**和图片资源媒体资源一般都是单独存在的,可以采用**contenthash**,只有文件本身变化后会生成新**hash**值。

修改**webpack.base.js**,把**js**输出的文件名称格式加上**chunkhash**,把**css**和图片媒体资源输出格式加上**contenthash**

```js
// webpack.base.js
// ...
module.exports = {
  // 打包文件出口
  output: {
    filename: 'static/js/[name].[chunkhash:8].js', // // 加上[chunkhash:8]
    // ...
  },
  module: {
    rules: [
      {
        test:/.(png|jpg|jpeg|gif|svg)$/, // 匹配图片文件
        // ...
        generator:{ 
          filename:'static/images/[name].[contenthash:8][ext]' // 加上[contenthash:8]
        },
      },
      {
        test:/.(woff2?|eot|ttf|otf)$/, // 匹配字体文件
        // ...
        generator:{ 
          filename:'static/fonts/[name].[contenthash:8][ext]', // 加上[contenthash:8]
        },
      },
      {
        test:/.(mp4|webm|ogg|mp3|wav|flac|aac)$/, // 匹配媒体文件
        // ...
        generator:{ 
          filename:'static/media/[name].[contenthash:8][ext]', // 加上[contenthash:8]
        },
      },
    ]
  },
  // ...
}
```

再修改**webpack.prod.js**,修改抽离**css**文件名称格式

```js
// webpack.prod.js
// ...
const MiniCssExtractPlugin = require('mini-css-extract-plugin')
module.exports = merge(baseConfig, {
  mode: 'production',
  plugins: [
    // 抽离css插件
    new MiniCssExtractPlugin({
      filename: 'static/css/[name].[contenthash:8].css' // 加上[contenthash:8]
    }),
    // ...
  ],
  // ...
})
```

再次打包就可以看到文件后面的**hash**了

### 6.6 代码分割第三方包和公共模块

一般第三方包的代码变化频率比较小,可以单独把**node\_modules**中的代码单独打包, 当第三包代码没变化时,对应**chunkhash**值也不会变化,可以有效利用浏览器缓存，还有公共的模块也可以提取出来,避免重复打包加大代码整体体积, **webpack**提供了代码分隔功能, 需要我们手动在优化项[optimization](https://webpack.js.org/configuration/optimization/)中手动配置下代码分隔[splitChunks](https://webpack.js.org/configuration/optimization/#optimizationsplitchunks)规则。

修改**webpack.prod.js**

```js
module.exports = {
  // ...
  optimization: {
    // ...
    splitChunks: { // 分隔代码
      cacheGroups: {
        vendors: { // 提取node_modules代码
          test: /node_modules/, // 只匹配node_modules里面的模块
          name: 'vendors', // 提取文件命名为vendors,js后缀和chunkhash会自动加
          minChunks: 1, // 只要使用一次就提取出来
          chunks: 'initial', // 只提取初始化就能获取到的模块,不管异步的
          minSize: 0, // 提取代码体积大于0就提取出来
          priority: 1, // 提取优先级为1
        },
        commons: { // 提取页面公共代码
          name: 'commons', // 提取文件命名为commons
          minChunks: 2, // 只要使用两次就提取出来
          chunks: 'initial', // 只提取初始化就能获取到的模块,不管异步的
          minSize: 0, // 提取代码体积大于0就提取出来
        }
      }
    }
  }
}
```

配置完成后执行打包,可以看到**node\_modules**里面的模块被抽离到**vendors.6aeb3c4c.js**中,业务代码在**main.f70933df.js**中。

![11.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1e9e46701db34ddf986c33f6581c4c77~tplv-k3u1fbpfcp-watermark.image?)

测试一下,此时**verdors.js**的**chunkhash**是**6aeb3c4c.js**,**main.js**文件的chunkhash是**f70933df**,改动一下**App.vue**,再次打包,可以看到下图**main.js**的chunkhash值变化了,但是**vendors.js**的chunkhash还是原先的,这样发版后,浏览器就可以继续使用缓存中的**verdors.ec725ef1.js**,只需要重新请求**main.js**就可以了。

![12.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d1da22ad5a5049dca5cd1936fb33e254~tplv-k3u1fbpfcp-watermark.image?)

### 6.7 tree-shaking清理未引用js

[Tree Shaking](https://link.juejin.cn/?target=https%3A%2F%2Fwebpack.docschina.org%2Fguides%2Ftree-shaking%2F)的意思就是摇树,伴随着摇树这个动作,树上的枯叶都会被摇晃下来,这里的**tree-shaking**在代码中摇掉的是未使用到的代码,也就是未引用的代码,最早是在**rollup**库中出现的,**webpack**在**2**版本之后也开始支持。模式**mode**为**production**时就会默认开启**tree-shaking**功能以此来标记未引入代码然后移除掉,测试一下。

在**src/components**目录下新增**Demo1**,**Demo2**两个组件

```vue
// src/components/Demo1.vue
<template>
  我是Demo1组件
</template>

// src/components/Demo2.vue
<template>
  我是Demo2组件
</template>
```

再在**src/components**目录下新增**index.ts**, 把**Demo1**和**Demo2**组件引入进来再暴露出去

```ts
// src/components/index.ts
export { default as Demo1 } from './Demo1'
export { default as Demo2 } from './Demo2'
```

在**App.vue**中引入两个组件,但只使用**Demo1**组件

```vue
<template>
  <img :src="smallImg" alt="小于10kb的图片" />
  <img :src="bigImg" alt="大于于10kb的图片" />
  <!-- 小图片背景容器 -->
  <div className='smallImg'></div>
  <!-- 大图片背景容器 -->
  <div className='bigImg'></div>
  <div> 修改App.vue</div>
  <!-- 使用Demo1组件 -->
  <Demo1 />
</template>

<script setup lang="ts">
  import smallImg from './assets/imgs/5kb.png'
  import bigImg from './assets/imgs/22kb.png'
  import './app.css'
  import './app.less'
  import { Demo1, Demo2 } from '@/components' //引入Demo1和Demo2组件
</script>

<style scoped>
</style>
```

再次执行**npm run build:dev**打包,可以看到在**main.js**中搜索**Demo**,只搜索到了**Demo1**, 代表**Demo2**组件被**tree-shaking**移除掉了。

![13.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b30871d4fb944d4aa0f6a0dda554dafb~tplv-k3u1fbpfcp-watermark.image?)

### 6.8 tree-shaking清理未使用css

**js**中会有未使用到的代码,**css**中也会有未被页面使用到的样式,可以通过[purgecss-webpack-plugin](https://www.npmjs.com/package/purgecss-webpack-plugin)插件打包的时候移除未使用到的**css**样式,这个插件是和[mini-css-extract-plugin](https://www.npmjs.com/package/mini-css-extract-plugin)插件配合使用的,在上面已经安装过,还需要[**glob-all**](https://www.npmjs.com/package/glob-all)来选择要检测哪些文件里面的类名和**id**还有标签名称, 安装依赖:

```sh
npm i purgecss-webpack-plugin glob-all -D
```

修改**webpack.prod.js**

```js
// webpack.prod.js
// ...
const MiniCssExtractPlugin = require('mini-css-extract-plugin')
const globAll = require('glob-all')
const { PurgeCSSPlugin } = require('purgecss-webpack-plugin')
module.exports = {
  // ...
  plugins: [
    // 抽离css插件
    new MiniCssExtractPlugin({
      filename: 'static/css/[name].[contenthash:8].css'
    }),
    // 清理无用css
    new PurgeCSSPlugin({
      // 检测src下所有vue文件和public下index.html中使用的类名和id和标签名称
      // 只打包这些文件中用到的样式
      paths: globAll.sync([
        `${path.join(__dirname, '../src')}/**/*.vue`,
        path.join(__dirname, '../public/index.html')
      ]),
    }),
  ]
}
```

测试一下, 现在**App.vue**中有两个**div**，类名分别是**smallImg**和**bigImg**,当前**app.less**代码为

```less
#root {
  .smallImg {
    width: 69px;
    height: 75px;
    background: url('./assets/imgs/5kb.png') no-repeat;
  }
  .bigImg {
    width: 232px;
    height: 154px;
    background: url('./assets/imgs/22kb.png') no-repeat;
  }
}
```

此时先执行一下打包,查看**main.css**

![14.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7838458f7e824fe2b34e2eddbe51d3dd~tplv-k3u1fbpfcp-watermark.image?)

因为页面中有 **smallImg**和**bigImg**类名，所以打包后的**css**也有，此时修改一下**app.less**中的 **.smallImg**为 **.smallImg1**，后面加一个1，这样 **.smallImg1**就是无用样式了,因为没有页面没有类名为 **.smallImg1**的节点,再打包后查看 **main.css**

![微信截图\_20220617141901.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fbcd882dc21944d58a3e30bf1816efea~tplv-k3u1fbpfcp-watermark.image?)

可以看到**main.css**已经没有 **.smallImg1**类名的样式了,做到了删除无用**css**的功能。

但是**purgecss-webpack-plugin**插件不是全能的,由于项目业务代码的复杂,插件不能百分百识别哪些样式用到了,哪些没用到,所以请不要寄希望于它能够百分百完美解决你的问题,这个是不现实的。

插件本身也提供了一些白名单**safelist**属性,符合配置规则选择器都不会被删除掉,比如使用了组件库[element-plus](https://element-plus.org/zh-CN/), **purgecss-webpack-plugin**插件检测**src**文件下**vue**文件中使用的类名和**id**时,是检测不到在**src**中使用**element-plus**组件的类名的,打包的时候就会把**element-plus**的类名都给过滤掉,可以配置一下安全选择列表,避免删除**element-plus**组件库的前缀**el-**。

```js
new PurgeCSSPlugin({
  // ...
  safelist: {
    standard: [/^el-/], // 过滤以el-开头的类名，哪怕没用到也不删除
  }
})
```

### 6.9 打包时生成gzip文件

前端代码在浏览器运行,需要从服务器把**html**,**css**,**js**资源下载执行,下载的资源体积越小,页面加载速度就会越快。一般会采用**gzip**压缩,现在大部分浏览器和服务器都支持**gzip**,可以有效减少静态资源文件大小,压缩率在 **70%** 左右。

**nginx**可以配置**gzip: on**来开启压缩,但是只在**nginx**层面开启,会在每次请求资源时都对资源进行压缩,压缩文件会需要时间和占用服务器**cpu**资源，更好的方式是前端在打包的时候直接生成**gzip**资源,服务器接收到请求,可以直接把对应压缩好的**gzip**文件返回给浏览器,节省时间和**cpu**。

**webpack**可以借助[compression-webpack-plugin](https://www.npmjs.com/package/compression-webpack-plugin) 插件在打包时生成 **gzip** 文章,安装依赖

```sh
npm i compression-webpack-plugin -D
```

添加配置,修改**webpack.prod.js**

```js
// ...
const CompressionPlugin  = require('compression-webpack-plugin')
module.exports = {
  // ...
  plugins: [
     // ...
     new CompressionPlugin({
      test: /.(js|css)$/, // 只生成css,js压缩文件
      filename: '[path][base].gz', // 文件命名
      algorithm: 'gzip', // 压缩格式,默认是gzip
      test: /.(js|css)$/, // 只生成css,js压缩文件
      threshold: 10240, // 只有大小大于该值的资源会被处理。默认值是 10k
      minRatio: 0.8 // 压缩率,默认值是 0.8
    })
  ]
}
```

配置完成后再打包,可以看到打包后js的目录下多了一个 **.gz** 结尾的文件

![15.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ce0dd7ab80d14f2190740c8b231344c0~tplv-k3u1fbpfcp-watermark.image?)

因为只有**verdors.js**的大小超过了**10k**, 所以只有它生成了**gzip**压缩文件,借助**serve -s dist**启动**dist**,查看**verdors.js**加载情况

![16.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/17a7f30cb3574f4dbc8971ac64035e29~tplv-k3u1fbpfcp-watermark.image?)

可以看到**verdors.js**的原始大小是**61kb**, 使用**gzip**压缩后的文件只剩下了**23kb**,减少了\*\*70%\*\*左右 的大小,可以极大提升页面初始加载速度，也能减轻服务器压力。

## 七. 总结

到目前为止已经使用**webpack5**把**vue3+ts**的基本构建环境配置完成，并且配置比较常见的**优化构建速度**和**构建结果**的配置，完整代码已上传到[webpack5-vue3-ts](https://github.com/guojiongwei/webpack5-vue3-ts.git)
。还有细节需要优化，比如把容易改变的配置单独写个**config.js**来配置，输出文件路径封装。这篇文章只是配置，如果想学好**webpack**，还需要学习**webpack**的构建原理以及**loader**和**plugin**的实现机制。

本文是总结自己在工作中使用**webpack5**搭建**vue3+ts**构建环境中使用到的配置, 肯定也很多没有做好的地方，后续有好的使用技巧和配置也会继续更新记录。

附上上面安装依赖的版本

```json
"dependencies": {
    "vue": "^3.3.4"
  },
  "devDependencies": {
    "@babel/core": "^7.22.1",
    "@babel/preset-typescript": "^7.21.5",
    "autoprefixer": "^10.4.14",
    "babel-loader": "^9.1.2",
    "compression-webpack-plugin": "^10.0.0",
    "copy-webpack-plugin": "^11.0.0",
    "cross-env": "^7.0.3",
    "css-loader": "^6.8.1",
    "css-minimizer-webpack-plugin": "^5.0.1",
    "glob-all": "^3.3.1",
    "html-webpack-plugin": "^5.5.1",
    "less": "^4.1.3",
    "less-loader": "^11.1.2",
    "mini-css-extract-plugin": "^2.7.6",
    "postcss-loader": "^7.3.2",
    "purgecss-webpack-plugin": "^5.0.0",
    "speed-measure-webpack-plugin": "^1.5.0",
    "style-loader": "^3.3.3",
    "terser-webpack-plugin": "^5.3.9",
    "thread-loader": "^4.0.2",
    "vue-loader": "^17.2.2",
    "webpack": "^5.85.1",
    "webpack-bundle-analyzer": "^4.9.0",
    "webpack-cli": "^5.1.3",
    "webpack-dev-server": "^4.15.0",
    "webpack-merge": "^5.9.0"
  }
```

## 参考

1.  [webpack官网](https://www.webpackjs.com/)
2.  [babel官网](https://www.babeljs.cn/)
3.  [【万字】透过分析 webpack 面试题，构建 webpack5.x 知识体系](https://juejin.cn/post/7023242274876162084)
4.  [Babel 那些事儿](https://juejin.cn/post/6992371845349507108)
5.  [阔别两年，webpack 5 正式发布了！](https://juejin.cn/post/6882663278712094727)