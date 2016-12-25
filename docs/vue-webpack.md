# Vue + Webpack 开发实践

虽然接触前段时间不长，不过大大小小的框架也碰过一些，现在最爱的就是 Vue，不仅是前期学习成本低，更重要的很喜欢组件化的方式以及简洁的方法。无论简单还是复杂的项目，都能有一致且清晰的架构。在好几个项目上都有过尝试，下面就整理一套自己的实践方式。

# 技术栈选型

## 多单页需要后端模板渲染的项目

这也是我接触过最多的项目了。前端仅是信息展示，简单的业务逻辑处理。路由和数据渲染都是后端来负责，数据也是直接渲染在模板上，甚至连 CDN 的 path 都是通过模板变量渲染的。这就会存在一个问题，在资源打包的时候，不指定绝对路径那资源肯定就是当前 path 下找资源了，也就用不到 CDN 了，但是如果把 CDN 写死，当然不符合最开始的设定啦，CDN 路径的控制权要交给后端嘛。技术栈如下：

- [vue](https://cinwell.com/post/vue-webpack/vuejs.org)
- [webpack](https://webpack.github.io/)
- [vue-loader](https://github.com/vuejs/vue-loader)

选择用 webpack 作为模块管理以及打包工具的理由是

- 打包资源后可以把 publicPath 当作一个变量传入
- chunks
- hot reload

前面说过有的项目资源路径是不确定的，而是后端模板渲染的一个变量。例如有个 django 模板如下

```
<html>
  <body>
    <script src="{{ STATIC_URL }}/static/entry.js"></script>
  </body>
</html>
```

那么如果某个 CSS 文件引用了图片资源，没办法知道 `STATIC_URL` 是什么的情况下就没法用到指定路径下的资源了。webpack 在打包资源的时候可以设置一个参数是 `publicPath`，神奇的是这个 publicPath 可以是一个变量，在你的入口文件给一个变量 `__webpack_public_path__` 传值，也就相当于指定了资源路径。但是这样的弊端就是不能用 `ExtractTextPlugin` 提出 CSS 文件了 :(

- entry.js 入口文件

```
__webpack_public_path__ = window.PUBLICPATH;
```

- django 模板

```
<html>
  <body>
    <script>
      window.PUBLICPATH = '{{ STATIC_URL }}';
    </script>
    <script src="{{ STATIC_URL }}/static/entry.js"></script>
  </body>
</html>
```

其次 webpack 的 chunks 机制对于组件化后的文件也非常实用。将一些体积较大的组件以 chunk 的方式引入，分散脚本文件体积。 [官方例子如下](http://cn.vuejs.org/guide/components.html#u5F02_u6B65_u7EC4_u4EF6)

- vue component

```
Vue.component('async-webpack-example', function (resolve) {
  // 这个特殊的 require 语法告诉 webpack
  // 自动将编译后的代码分割成不同的块，
  // 这些块将通过 ajax 请求自动下载。
  require(['./my-async-component'], resolve)
})
```

最后的 hot reloader 就不用介绍了，webpack 的特点之一，确实提高了不少开发效率。搭配 vue-loader 将组件拆分成独立的文件，更加利于团队协作。

### 样式写在哪

按照组件化的思路，确实应该讲样式写到每个组件内，互不影响。但是就存在一个问题，样式可服用的几率更大，如果全部拆分到每个组件内，团队协作无法形成统一规范。但是如果全部写到外部文件也不利于组件的开发，找到组件后还要去找对应的样式文件太麻烦，而且团队协作没有命名规范就很容易冲突。

所以最终采用的方式是**基本的样式写到外部，每个组件特有的样式写到组件内**，所谓的基本样式就好比`bootstrap`之类的样式文件，起到提供基本布局样式、风格以及标准的作用。组件的特定的样式都以 `scoped` 的方式写到 vue 文件内。

## SPA

单页应用的约束就少很多了，路由和模板都是由前端来决定。技术栈上增加

- [vue-router](https://github.com/vuejs/vue-router)

# 目录结构

关于基本项目的搭建，我采用官方提供的 [vue-cli](https://github.com/vuejs/vue-cli) 基本上能满足我的基本需求，所以目录结构也是基于该结构上增加的

```
|- build/ # 基本配置
  |- webpack.base.js
  |- webpack.prod.js
  |- webpack.dev.js
  |- webpack.test.js
  |- setting.js # 公共的配置信息，例如一些开发时候的 HOST 或者 PORT 或者 CDN 什么的
  |- templates
    |- scripts.html
    |- styles.html
|- dist/
|- src/
  |- assets/
    |- images/
    |- fonts/
    |- scss/
      |- static.scss
  |- components/
  |- directives/
  |- plugins/
  |- filters/
  |- libs/ # 非 vue 的插件
  |- entry.js
|- test/
```

如果是多页（多入口）项目，就把 entry.js 换成 `entry` 文件夹，里面放各种入口文件。

## 后端模板的项目如何引用 build 后的前端资源文件

如果直接写死在后端模板，如果资源被浏览器或者 CDN 服务器缓存，也就永远无法加载到最新的资源。例如

```
<html>
  <head>
    <link rel="stylesheet" href="{{ STATIC_URL }}dist/styles.css">
  </head>
  <body>
    <script src="{{ STATIC_URL }}dist/scripts.js"></script>
  </body>
</html>
```

当然也可以在后面加 tag，用 `dist/scripts.js?tag=` 但是依旧没办法解决 CDN 缓存的问题。所以通常的解决方法是硬编码资源名字，用时间戳或者 hash 生成如
`20160205.scripts.js` 之类的文件，保证每次 build 后的文件名都不同，也就不会被缓存了。但是这样带来的问题就是没办法动态更新模板文件。

虽然有 gulp 或 grunt 替换模板内容的插件，但是效率太低，我的做法是直接用 [HtmlWebpackPlugin](https://github.com/ampedandwired/html-webpack-plugin) 生成 html 文件，而且能替换 script 和 style 内容。我的做法是新建两个模板，`styles.html` 和 `scripts.html`

- styles.html 额这里是 1.x 的写法

  ```
  {% for (var css in o.htmlWebpackPlugin.files.css) { %}
      <link href="{{ STATIC_URL }}dist{%=o.htmlWebpackPlugin.files.css[css] %}" rel="stylesheet">{% } %}
  ```

- scripts.html

  ```
  {% for (var chunk in o.htmlWebpackPlugin.files.chunks) { %}
      <script src="{{ STATIC_URL }}dist{%=o.htmlWebpackPlugin.files.chunks[chunk].entry %}"></script>{% } %}
  ```

然后生成到后端模板的目录下，在 base.html 文件里 include 进这两个文件，也就实现了前端资源和后端模板分离。