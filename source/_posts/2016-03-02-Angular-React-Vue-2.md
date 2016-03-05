---
title: ARV 渲染实现比较之 React
date: 2016-03-02 19:06:55
comments: true
categories:
 - 技术
tags:
 - Javascript
 - Angular
 - React
 - Vue
---

{% post_link Angular-React-Vue-Rendering-1 上一篇 %} 我总结了 [Angular][1] 的渲染实现，今天我们来看看 [React][2]。

## React

React 的渲染套路简单来说就是虚拟DOM树（Virtual DOM）。React 的思想有点儿像 Web Component，但是他又是一个独立的实现，可以纯粹使用 js 逻辑实现了前端组件化。看似 HTML + JS 的新语言 JSX，实际上会编译成完全的 js 代码，所有的 HTML 都转换成了 JS 的 DOM 操作，只不过这里操作的 DOM 并不是真正的 DOM，而是 React 实现的一套映射到真正 DOM 的虚拟 DOM。并且 React 的重点在于 'React'，即视图响应数据变化。

<!--more-->

这里讨论的 React 目前是 0.14 版本的，鉴于对这个版本 0.12 版本做了很大的更改，所以并不确定更新到 1.0 后本文还是否适用，请注意。

### StartUp

React 的 jsx 启动代码大概如下：

``` js
ReactDOM.render(
  <h1>Hello, world!</h1>,
  document.getElementById('example')
);
```

### JSX

虽然使用 React 一般使用React专用的 JSX 语法，但是运行时实际上会编译为 JS 语法，比如 JSX 的：

``` js
var root = <ul className="my-list">
             <li>Text Content</li>
           </ul>;
```

会编译成类似以下的 JS：

``` js
var root = React.createElement('ul', { className: 'my-list' },
             React.createElement('li', null, 'Text Content')
           );
```

为了不那么绕，所以我接下来将直接采用 JS 语法来分析。

### ReactElement

ReactElement 是组成 React 虚拟 DOM 的基本元素，它对应于真实 DOM 中的 HTMLElement，比如 div，li，或我们自定义的 Component。一个 ReactElement 可以是以下类型：
 - ReactDOMElement（即HTML原生支持的元素）
 - ReactComponentElement（我们定义的 Component 元素）

创建 ReactElement 方法如下：

``` js
var element = React.createElement(type, props, children, ...);
```

其中第一个参数指定元素的标签名或 ReactClass，第二个参数可以指定元素的属性（props），比如 className 之类的，三个参数以及更多参数指定元素的内部元素，类型是 ReactNode。

### ReactNode

ReactNode 即是 React 虚拟 DOM 树的节点，它对应于真实 DOM 的节点。一个 ReactNode 可以是下面的类型：
 - ReactText（文字，数字）
 - ReactElement
 - ReactFragment（一个元素为ReactNode的数组）

利用上面这些结构类型，就可以创建一棵由 ReactElement 构成的树。

### ReactClass & ReactComponent

有了上面的普通 ReactElement 和 ReactNode 就已经可以创建一棵树了，通过创建一些工厂函数也能重用一些元素结构。但是到目前为止也只实现了 View 而缺少 ViewModel，没有真正做到组件化。

ReactClass 就是用来产生可重用的 ReactComponent 的类型，其实就是一个构造函数。下面是定义一个类名为 MyComponent 的组件类的方法：

``` js
var MyComponent = React.createClass({render: function() {...}});
```

其中的 render 属性是一个必须实现的属性，他必须返回一个 ReactElement，React 就是通过这个 render 方法来将一个 ReactComponent 渲染成 ReactElement 的。有了 ReactClass 我们就可以为产生的 ReactComponent 赋予状态属性，然后在 render 方法根据状态的不同，返回结构不同的 ReactElement。

注意以上定义好的 MyComponent 只是一个 ReactClass，要用它生产出 ReactComponent 元素，还需要像生成 ReactElement 那样去掉用 createElement 方法：

``` js
var element = React.createElement(MyComponent, props, child);
```

如果觉得 ReactClass 和 ReactComponent 理解起来还是比较抽象，那么可以将 ReactClass 想象成标签名，即 `div`,`video`，这种，但是他不是原生的标签，所以使用前需要先用 `createClass` 定义好。而 ReactComponent 就是跟 ReactElement 一个级别的东西。

### ReactDOM & 第一遍渲染

有了上面的 ReactNode，ReactElement，ReactComponent，我们就可以完全的定制一棵有状态控制的虚拟DOM树，之所以说是虚拟的，是因为它只是逻辑上的数据结构，跟我们肉眼看到的页面并没有直接的联系。

我们通过 ReactDOM 这个子库来做跟真实DOM相关的操作，比如将一棵虚拟 DOM 树渲染到一个 HTML 的 DOM 元素中：

``` js
var component = ReactDOM.render(element, container, callback);
```

其中第一参数为一棵虚拟DOM数的根节点元素，第二个参数为一个 HTML DOM 元素，返回结果是绑定成功后生成的component 实例的引用（ref）。这里又多了一个概念 ReactElement 的实例，据我们所知 ReactComponent 跟 ReactElement 是一个级别的东西，是虚拟DOM树的基本结构，为什么还有实例？简单来说，只有一个虚拟的 ReactElement 被绑定到一个真实 DOM 元素上，它内部才能正常运作，所以只有绑定后才能产生一个真正可以调用的实例。

调用了 `ReactDOM.render` 之后，这个虚拟 DOM 树的结构将按初始状态原封不动的渲染成真实 DOM 结构，并完成内部 ReactElement 到 DOMElement 的绑定。

### 数据更新

React 的 ViewModel 实际上就是 ReactComponent 的 state 和 props，View 将根据这些状态和属性来改变。但是 React 不会去监听这些状态和属性的变化，每次改变 state 和 props 都需要手动调用 ReactComponent 的
`setState` 和 `setProps` 方法，这些方法会通知 Component 更新渲染，也可以调用 `forceUpdate` 强制更新。更新渲染实际上是通过 render 方法实现的，因此 render 方法内部一般不可以再修改状态，否则有可能导致渲染死循环。

### Reconciliation

如果每次改动都需要重新渲染，那 React 造这些轮子也没啥价值了。接下来的 Reconciliation 才是 React 渲染实现的重点。

Reconciliation 这个词怎么翻译，我也不确定，估计可以称之为“调解”，调解因从一棵虚拟 DOM 树（局部）变成另一棵 DOM 树引起的“冲突”。

如果要准确比较两棵虚拟 DOM 树的差异并分解成最少的节点操作，可能需要 O(n3) 的复杂度，这样最节省操作真实 DOM 的成本，但是显然这相对于直接重新渲染而言是更耗费性能的。React 使用了一种较为粗暴的 O(n) 方法来找到一个较优解，在 DOM 操作和对比差异之前取得了良好的平衡。

元素级别的对比：
 - 如果对比某个节点的类型发生变化，则重新渲染该节点
 - 类型相同的一般DOM元素（非 ReactComponent），可以在 O(n) 时间内找到其属性的变化并重现
 - 类型相同的 ReactComponent 不会被替换，将使用新组件的 props 去调用旧组件的 `component[Will/Did]ReceiveProps()` 方法并重新调用他的 render 方法

列表级别的对比：
 - 按照顺序对两个列表的每一个列表项进行元素级别的对比操作，若最后多了就删掉，少了就插入新的元素
 - 如果一个列表项元素设置了 key 属性，则会对 key 值相同的项进行对比，并重现对应的调整位置，删除，插入操作

利用以上规则来对比虚拟 DOM 树的变化并将操作重现到真实 DOM 上，节省了大部分重新创建删除 DOM 元素的成本，也让对比操作的复杂度保持在一个可控范围内。

### 性能

由 Reconciliation 的过程可以看出，提高性能的方法有几种：
 - 如果两个 ReactComponent 结构相近，并且会相互切换，尝试合并成一个
 - 所有列表项都加上 key 属性，并且令每个 key 在列表内稳定且唯一
 - 实现 ReactComponent 的 shouldComponentUpdate 方法，避免无谓的重新渲染

其中如果你的 ReactComponent 的渲染结果是幂等的（即针对相同的 props 和 state 其渲染结果也一致），那么可以直接使用 React 提供的 PureRenderMixin，她相当于帮你实现了 shouldComponentUpdate 方法，如果 props 和 state 没有变化（对象级别为引用对比），将直接返回 false 防止重复渲染。

但是如果你的 prop 或 state 属性是对象，而你必须对比对象内部属性的变化，可以使用 Facebook 的另外一个库 [Immutable][5]，他可以用来创建不可变的对象，即要改变对象的属性必须拷贝并创建一个新的对象，这就令通过对比引用来对比对象属性变化成为可能。

但是我个人觉得这样每修改一次就创建新的对象也是一笔开销，同时还要使用 Immutable 定义的属性方法去修改对象，如果可以自己实现 shouldComponentUpdate 去做必要的比较还是可行的。不过如果真的想用不可变数据的话建议看看 [森(Mori)](http://swannodette.github.io/mori/)，比 Immutable 要快一点。

### 小结

React 渲染速度很快，这要得益于他将所有模版操作都预先编译成了虚拟 DOM 创建操作，这节省了运行时编译的开销；得益于“调和”算法，虚拟 DOM 结构改变映射到真实 DOM 也节省了大量不必要的操作。

然而 React 的优势又不止性能，与 ES6 高度兼容，搭配 Webpack 轻易实现模块化，前后端可以实现同构（isomorphic）等等都是 React 的加分点。React 可以说是做 View-ViewModel 这一块只做一件事，并且做到了极致。但是这也意味着只使用 React 可能是不够的，还需要用到 Flux，Redux 之类的数据模型和事件处理框架。然而单单 React 一个库压缩后就达到了 200k 左右的大小，加上其他诸如 React-Router，jQuery，Immutable 等等大小就更加令人瞠目结舌，虽然今天网速已经很快，但是第一次加载仍然会让人觉得等待很漫长，毕竟需要先加载完 React 才能进行渲染，除非通过同构服务端渲染来进行第一遍渲染。

Vue 下期再写了。

[1]: https://angular.io
[2]: https://facebook.github.io/react
[3]: http://vuejs.org/
[4]: https://www.polymer-project.org/1.0/
[5]: https://facebook.github.io/immutable-js/
