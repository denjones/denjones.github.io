---
title: 为什么 React 体积那么大
date: 2016-03-21 17:36:32
comments: true
categories:
 - 技术
tags:
 - Mithril
 - React
---

React 文件体积为何如此庞大？首先，为什么会得出 React 大的结论，对比几个前端框架的 min 文件：
 - Mithril 0.2.3 19K (8K gzipped)
 - Angular 1.2.16 102K (38K gzipped)
 - Vue 1.0.8 73K (24K gzipped)
 - React 0.14.7 133K (38K gzipped)

React 作为一个 View-ViewModel 库，相比于 Mithril，Vue 这些目的大致相同的库，文件显得尤为庞大，甚至比 Angular 这种全能 MVVM 框架还大。但是这就能说 React 大吗，我认为是的，Mithril 与 React 都是基于 Virtual DOM 的实现，我觉得这很有可比性。虽然 Mithril 的真实性能大致为 React 的一半，但是代码量却是 React 的不到 1/10（实际上 Mithril 只有大概 2000 行代码），Mithril 在使用了一些小技巧之后甚至性能飙升至 React 的数十倍。对 Mithril 感兴趣可以看下 {% post_link Mithril-Rendering 我对 Mithril 渲染性能的分析 %}。

<!--more-->

通过分析 React 的源码及其打包过程，可以得出 React 的成分构成：
 - React: 687KB 100%
   - renderers 这部分代表了 Virtual DOM 的渲染部分
     - ReactDOM 316KB 55% 主要作用是将 Virtual DOM 渲染成真实 DOM，并进行关联以及之后的 DOM 操作，DOM 事件处理框架
     - ReactDOMServer 9KB 1% 这部分代码很大程度上重用 ReactDOM 所以代码量不多
     - ReactReconciler 106KB 15% 这部分代表了 React 的差异比较以及做出 DOM 操作的调和算法
     - ReactEvent 63KB 9%
   - ReactIsomorphic 103KB 15% 这部分是 Virtual DOM 的结构代码，包括 ReactComponent，ReactClass，ReactElement 等的实现
   - 辅助测试代码 48KB 7% 可能这里有一些没有打包的模块，但是确实有一些如 ReactPerf，ReactDefaultPerf 等模块是打包进去了
   - 其他依赖库 38KB 6%


这里分析的百分比是打包进 react.js（压缩前）的各部分源码文件大小（及其依赖文件）占所有打包文件大小（不包括addons以及fbjs）的百分比，跟压缩合并后的百分比有一定差别，但是可以一定程度上代表代码成分。从成分看来，实际上 VirtualDOM 的代码并不多，大部分代码都放在跟真实 DOM 相关的操作里面，尽管从认知上来看，这些操作并不需要写如此多的代码。我认为导致代码量大的原因大致有以下几点。

### 注重安全性

严格的 Virtual DOM 检查以及 ReactDOM 操作，包括 props 类型，经典 DOM 元素的标签名，属性名，CSS属性名，标签嵌套合法性，支持的原生 DOM 事件类型，等等都有明确的定义。React 毫不吝啬地在所有涉及安全问题的地方使用白名单来提高安全性，这也是代码量大的原因之一：

``` js
var HTMLDOMPropertyConfig = {
  isCustomAttribute: RegExp.prototype.test.bind(
    /^(data|aria)-[a-z_][a-z\d_.\-]*$/
  ),
  Properties: {
    /**
     * Standard Properties
     */
    accept: null,
    acceptCharset: null,
    accessKey: null,
    action: null,
    allowFullScreen: MUST_USE_ATTRIBUTE | HAS_BOOLEAN_VALUE,
    allowTransparency: MUST_USE_ATTRIBUTE,
    alt: null,
    async: HAS_BOOLEAN_VALUE,
    autoComplete: null,
    // autoFocus is polyfilled/normalized by AutoFocusUtils
    // autoFocus: HAS_BOOLEAN_VALUE,
    autoPlay: HAS_BOOLEAN_VALUE,
    capture: MUST_USE_ATTRIBUTE | HAS_BOOLEAN_VALUE,
    cellPadding: null,
    cellSpacing: null,
    charSet: MUST_USE_ATTRIBUTE,
    challenge: MUST_USE_ATTRIBUTE,
    checked: MUST_USE_PROPERTY | HAS_BOOLEAN_VALUE,
    classID: MUST_USE_ATTRIBUTE,
    // To set className on SVG elements, it's necessary to use .setAttribute;
    // this works on HTML elements too in all browsers except IE8. Conveniently,
    // IE8 doesn't support SVG and so we can simply use the attribute in
    // browsers that support SVG and the property in browsers that don't,
    // regardless of whether the element is HTML or SVG.
    className: hasSVG ? MUST_USE_ATTRIBUTE : MUST_USE_PROPERTY,
    cols: MUST_USE_ATTRIBUTE | HAS_POSITIVE_NUMERIC_VALUE,
    colSpan: null,
    content: null,
    contentEditable: null,
    contextMenu: MUST_USE_ATTRIBUTE,
    controls: MUST_USE_PROPERTY | HAS_BOOLEAN_VALUE,
    coords: null,
    crossOrigin: null,
    data: null, // For `<object />` acts as `src`.
    dateTime: MUST_USE_ATTRIBUTE,
    default: HAS_BOOLEAN_VALUE,
    defer: HAS_BOOLEAN_VALUE,
    //  ... 省略150行
}
```

### 实现同构

ReactDOM 在前端渲染需要做同构处理，即将服务端返回的第一遍渲染出来的 html DOM 绑定到前端 Virtual DOM 树，这增加了不少 ReactDOM 和 ReactDOMServer 的共用代码，因为必须兼顾浏览器端和服务端渲染的一致性。

除此之外，ReactDOMServer 即使在前端渲染不会用到，但是为了前后端使用同一份代码，所以依然打包进了 react.js，这也增加了一些体积。

### 开发者友好

大量的提示文本，针对每个模块方法的误用或被抛弃的方法，都有具体详细的提示，这些提示混淆压缩后长度不变：
```js
warning(
  owner._warnedAboutRefsInRender,
  '%s is accessing getDOMNode or findDOMNode inside its render(). ' +
  'render() should be a pure function of props and state. It should ' +
  'never access something that requires stale data from the previous ' +
  'render, such as refs. Move this logic to componentDidMount and ' +
  'componentDidUpdate instead.',
  owner.getName() || 'A component'
);
```

另外，喜欢使用完整的语句作为方法及属性名，如 `registrationNameDependencies`，也是导致容量大的原因，因为方法名和属性名即使经过压缩混淆，长度也不会改变。以下是压缩后的 React 入口模块代码（格式化）：

``` js
"use strict";
var r = e(35),
o = e(45),
a = e(61),
i = e(23),
u = e(104),
s = {};
i(s, a),
i(s, {
  findDOMNode : u("findDOMNode", "ReactDOM", "react-dom", r, r.findDOMNode),
  render : u("render", "ReactDOM", "react-dom", r, r.render),
  unmountComponentAtNode : u("unmountComponentAtNode", "ReactDOM", "react-dom", r, r.unmountComponentAtNode),
  renderToString : u("renderToString", "ReactDOMServer", "react-dom/server", o, o.renderToString),
  renderToStaticMarkup : u("renderToStaticMarkup", "ReactDOMServer", "react-dom/server", o, o.renderToStaticMarkup)
}),
s.__SECRET_DOM_DO_NOT_USE_OR_YOU_WILL_BE_FIRED = r,
s.__SECRET_DOM_SERVER_DO_NOT_USE_OR_YOU_WILL_BE_FIRED = o,
t.exports = s
```

### 过度模块化

react.js 里面包含了151个模块的定义，平均每个模块化增加的额外代码量：

```js
// 模块编译后生成代码
2 : [function (e, t, n) {
    // 原模块定义代码
  }, {
    106 : 106,
    136 : 136,
    63 : 63
  }
]
```

每个模块增加的代码约为45Byte，加上统一处理函数，共约为7KB的大小。虽然看起来不多，但实际上这占了压缩后的 react.min.js（133KB） 的 5% 的大小。至于真的需要写那么多个模块吗，我觉得是不用的，至少一个模块里面只有一个函数这种（如 onlyChild 模块）是可以集中写的。除此之外，虽然有如此庞大的模块集合，React 的模块间耦合还是很高，模块间相互调用十分繁多。

### 其他依赖

觉得 React 大的原因除了 react.js 本身的庞大外，还有需要和 React 搭配使用的库也有很多，包括 React 自己的 addons，封装好的 http 请求，Flux，Redux 等框架，处理复杂状态的 Immutable.js 等等等。另外如果要适配 IE8 还要引入一堆 polyfill 也增加了不少容量。

## 结论

React 确实很大，但是也并非大而无当，起码出发点是好的。但是有没有优化的空间呢，我认为是有的，起码如果区分开发版和发行版能有效的去掉对用户来说没有用的开发者提示。

## 题外话

我们知道我们在程序中需要使用 react-dom.js 来使用 ReactDOM，然而实际上这个文件并不真的包含 ReactDOM 的实现，按照上面的分析 ReactDOM 是 react.js 的一部分:

``` js
React.__SECRET_DOM_DO_NOT_USE_OR_YOU_WILL_BE_FIRED = ReactDOM;
React.__SECRET_DOM_SERVER_DO_NOT_USE_OR_YOU_WILL_BE_FIRED = ReactDOMServer;
```

那么 react-dom.js 到底干了啥，我们来看看：

``` js
function(React) {
  return React.__SECRET_DOM_DO_NOT_USE_OR_YOU_WILL_BE_FIRED;
}
```

你TM逗我。其实这也可以理解，React 内部模块的耦合决定了 ReactDOM 不可能单独抽取出来用，react-dom.js 这个模块文件只是给我们暴露一个入口。
