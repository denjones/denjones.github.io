---
title: ARV 渲染实现番外篇之 Mithril
date: 2016-03-14 17:21:32
comments: true
categories:
 - 技术
tags:
 - Javascript
 - Mithril
 - Angular
 - React
 - Vue
---

这个系列本来没有打算去研究 [Angular][1]，[React][2]，[Vue][3] 之外的框架，但是 {% post_link Angular-React-Vue-Rendering-4 上次的性能测试 %} 让 [Mithril][7] 这个框架格外引人注目，毕竟他的成绩超过了其他所有框架。因为我没有在项目中用过 Mithril，所以只能从其文档及代码中寻找他惊人能力背后的实现。因为这个是番外篇了，所以可能不会太严谨，科普性质吧。

## Mithril

Mithril 翻译过来就是秘银的意思，这是魔幻小说或游戏里面才有的虚构金属物质，基本上是最好的金属，可以打造出最上等的武器装备。可能作者希望这个库就是那么厉害的东西，而且用 Mithril 做出来的产品也是上等的吧。他的第一个版本于 2014 年 3 月发布，相比 React （2013 年 5 月），Vue （2013 年 12 月），都要晚一点，而现在也只发布到 0.2.3 版本。

<!--more-->

### Start Up

Mithril 的启动代码大致如下：

``` js
var app = {
  controller: function() {},
  view: function(ctrl) {
    return (
      m("body", [
        m("input"),
        m("button", "Add")
      ])
    );
  };
};
m.mount(document.getElementById("example"), app);
```

思维敏锐的你一定看出来了，这结构跟 ReactComponent 不是差不多。实际上，连 Mithril 的书写格式都可以使用类似 JSX 这种预编译模版来写，他们称之为 [MSX][8] 是利用 React 的 JSX 改写而来的：

``` js
var app = {
  controller: function() {},
  view: function(ctrl) {
    return (
      <body>
        <input>
        <button>Add</button>
      </body>
    );
  };
};
m.mount(document.getElementById("example"), app);
```

### Virtual DOM

怎么会出现 Virtual DOM，没错，Mithril 确实是用 Virtual DOM 实现的，你完全可以按照 React 的 Virtual DOM 来理解 Mithril 的 Virtual DOM。对 Virtural 建议先去看我 {% post_link Angular-React-Vue-Rendering-2 关于 React 渲染的分析 %}。但是他们确实在细节上又有些不同。

### Cell

cell 即是虚拟 DOM 元素，有一个 children 属性指定内部子元素，对应于 React 的 ReactElement。可以使用 `m` 命令创建：

``` js
m(tag, attrs, children, ...);
```

Cell 可以是一个普通 DOM 元素或者一个 Component 实例。

### Tag

代表一个 Cell 的类型，可以是一个查询字符串，类似 `div.classname#id[param=one][param2=two]`，或者一个 ComponentClass 对象。

### ComponentClass

ComponentClass 是一个自定义组件类对象，用于构造 Component 实例。包含 `view` 和 `controller` 属性，`view` 返回一个 Cell，而 `controller` 就像是一个初始化函数，将在第一次渲染前执行 ，在里面需要初始化 ViewModel：

``` js
var myComponent = {
  controller: function() {
    this.data = m.prop()
  },
  view: function() {
    return m("body")
  }
}
```

Mithril 的 ComponentClass 对应于 React 的 ReactClass，`view` 对应于 `render`， `controller` 类似 `componentWillMount` 方法。而 controller 需要做的事情包括初始化 ViewModel，这又跟 ReactClass 的 `getInitialState` 类似。

### CellCache

是指一个根 Cell 的快照，他包含了其孩子 Cell 在内的一棵完整的虚拟 DOM 树，每一个根 Cell 在第一遍渲染的时候都会获得一个全局唯一的 CellCacheKey，并在每次 Redraw 的时候根据这个 Key 取出对应的虚拟 DOM 树与当前的虚拟 DOM 树进行对比，完成对比后再将当前虚拟 DOM 树创建快照并替换。CellCache 跟原本的 Cell 对象是不同的引用，CellCache 会比原始 Cell 对象多出来一个 `nodes` 属性，包含跟这个 Cell 关联的所有真实 DOM 元素。

### Redraw

Redraw 指的是 Mithril 的一遍渲染过程，他会执行一次对虚拟 DOM 数的自顶向下差异比较（diff），然后进行仅必要的 DOM 操作以重绘视图。

Mithril 使用的 diff 算法跟 React 还是有一点区别的：

元素级别的对比：
 - 若一个 Cell 没有被缓存（Cached），则创建元素并保存 CellCache
 - 若一个 Cell 有对应的 CellCache，并满足以下至少一项，则创建新 DOM 元素，否则在当前元素上做修改：
   - Tag 不一致
   - 属性名（attrs）有增减或改变
   - id/key属性的值有改变
 - 若一个 Cell 的父节点发生变化，则将自己绑定到新的父元素上

列表级别的对比：
 - 为新列表中没有 Key 属性的元素创建全局唯一 Key
 - 若新列表中有，旧列表中没有的 Key，则标记为创建
 - 若新列表中有，旧列表中也有的 Key，则标记为移动
 - 若新列表中有，旧列表中没有的 Key，则标记为删除
 - 执行标记的 DOM 操作

比较算法自顶向下递归地遍历对比每一个 Cell 与 CellCache 及其 children，并作出对应的 DOM 修改操作。

### Redraw 时机

重绘会在组件 controller 执行后或事件触发后进行，也就是说自动的视图更新需要在 Mithril 的体系下进行，比如 `m.request`，`controller`，`onClick` 等，这与 Angular 类似。如果在体系外，比如使用 `setTimeout`，则需要在改变 ViewModel 的代码前后加上 `m.startComputation` 和 `m.endComputation` 这对计算标记：

``` js
setTimeout(function(){
  m.startComputation()
  vm.update()
  m.endComputation()
})
```

而实际上所有可以响应的 ViewModel 变动都是在 `m.startComputation` 和 `m.endComputation` 之间进行的， Mithril 体系内也是如此处理。一对计算标记内可以嵌套另外的计算标记对，重绘只会在嵌套最外层的 `m.endComputation` 调用后执行，这是通过计数器实现的，在调用 `m.startComputation` 时计数加一，调用 `m.endComputation` 时减一，当减为 0 时则会调用更低级的 `m.redraw` 方法。所以可以通过使用计算标记，可以保证将一系列连续的操作执行完毕后再进行重绘，避免在修改过程中因为异步操作引发的不必要的重绘操作。

然而，调用 `m.redraw` 也并不一定真正进行重绘，Mithril 在重绘上采取了跟（游戏）渲染引擎类似的锁帧操作，即在一个预设好的时间段内的数据改变，延迟到时间段结束的时候再进行重绘。对于游戏一般需要 30-60fps 以上的刷新率才能算是连贯，而 Mithril 则采用了 60fps，即一个重绘周期为 1/60s，约为 16ms。

### 性能

如果 Mithril 只有 diff，那么他跟 React 的性能基本上是差不多的。Mithril 在性能上致胜的关键在于重绘时机的选取：
 - `m.startComputation` 和 `m.endComputation` 保证了在一次连续（异步）操作内不会促发重绘
 - 锁帧操作保证了在肉眼可感知到的变化周期内（16ms）不会重复渲染

事实上我做了另外一个 [性能对比测试](http://jsperf.com/angular-vs-knockout-vs-ember/843)，与[上一次测试不同](http://jsperf.com/angular-vs-knockout-vs-ember/842)，我在 Mithril 的测试样本进行一遍变更后调用 `m.redraw(true)` 直接强制一遍重绘来取消锁帧的影响（这很公平，因为其他框架也可以应用这样的小技巧）。结果跟我想象中差不多：

{% asset_img benchmark.png %}

Mithril 的性能下降到 React 的一半左右。

### 小结

上面的测试结果可以看出来，单从 Virtual DOM 及 diff 算法上来说，Mithril 的效率并没有 React 高，但是 Mithril 仅仅使用了 2000 多行代码就实现了 Virtual DOM，甚至还包括 Router 在内的模块，致力于实现一个平台级的全能 MVC 框架。压缩后 20k 的大小相比 200k 以及需要额外库支持的 React，确实能省掉不少首次加载页面的时间。

但是在我看来他也仅仅实现了 View-ViewModel 部分以及封装了网络请求，并没有真正实现 Model 部分。他的创新之处也就剩下基于帧的重绘时机控制，然而，这个办法或多或少都属于小手段，并不是在逻辑上真正提高渲染效率，只是利用了人类的极限视觉来节省运算。而且这一招并不是没人想到过，只是很多框架都不需要如此极致的效率提高，因为渲染速度一般并不会影响到体验（除非要显示上万条列表数据）。只有在性能出现瓶颈的时候，才需要去做这样那样的小手段，而很多其他框架实际上都是可以另外实现锁帧操作的。

并没有很多人真的在用 Mithril 因为他跟 React 是如此类似（我甚至怀疑作者是不是看完 React 然后自己另外实现了一个），真的要在其中选大多数人也会选 React，毕竟社区健全完善，解决方案众多。但是 Mithril 给了我们很多灵感，比如 React 的代码量是不是可以更加精简，是不是可以通过锁帧进一步提高 React 的效率。

[1]: https://angular.io
[2]: https://facebook.github.io/react
[3]: http://vuejs.org/
[4]: https://www.polymer-project.org/1.0/
[5]: https://facebook.github.io/immutable-js/
[6]: https://github.com/magnumjs/mag.js/
[7]: http://mithril.js.org/
[8]: https://github.com/insin/msx
