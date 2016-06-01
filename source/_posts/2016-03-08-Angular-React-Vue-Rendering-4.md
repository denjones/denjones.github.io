---
title: ARV 渲染实现比较总结
date: 2016-03-08 11:42:32
comments: true
categories:
 - 技术
tags:
 - Javascript
 - Angular
 - React
 - Vue
---

前面我已经分析了 [Angular][1]，[React][2]，[Vue][3] 的渲染实现及其性能：
 - {% post_link Angular-React-Vue-Rendering-1 %}
 - {% post_link Angular-React-Vue-Rendering-2 %}
 - {% post_link Angular-React-Vue-Rendering-3 %}

直观一点地的看他们之间的效率对比可以看一下这一个 [性能评估测试](http://jsperf.com/angular-vs-knockout-vs-ember/842)。

<!--more-->

这个测试的原理是针对每一个前端框架，创建一个数组模型到视图的绑定，并重复以下操作：清空列表数据，然后循环对数组插入100个元素。重复操作一段时间，最后将比较他们之间每秒可以达到的操作次数。他比较了包括 Angular，React，Vue 在内的数个流行框架，其中 [Mithril][7] 的数据十分高，是 Vue 的2到3倍，而且体积十分小压缩后只有十几k，但是很遗憾他并没有列入我们的讨论范围，因为他还在开发阶段，使用人数也并不多。虽然这个测试可能比较片面，现实开发中用到的数据并不会那么简单粗暴，但是他也一定程度上能代表这些框架的性能了。

下面这幅图是这些框架的渲染速度横向对比：

{% asset_img benchmark.png %}

在这幅图中，你几乎看不到 Angular 的数据，因为他每秒只能进行 100 次左右的操作。React 的操作数大概在 1000 - 2000 之间，而 Vue 则在 10000 次以上。这也印证了前面几篇文章的分析。

## Angular

Angular 很慢，因为他用了脏值检查这种被动的遍历方法穷举所有监听的表达式。在这个例子中使用了 `track by $index` 命令，因为数组项是纯字符串。每对数组做一次修改，所有数组项都被重新评估。

做 100 次操作，实际上要比较 100 x 100 = 10000 个数据，这些比较都是逻辑比较，速度应该还行，主要是这些比较导致了 10000 个模版关联，更新 DOM 树的操作，也就这种级别的运算速度了。但是他的慢到不能用吗，实际上也并不，因为正常需求来说不需要这么高频率的更新。

## React

React 每秒能进行 1000 次操作，大约是 Angular 的 10 倍，这得益于 React 的虚拟 DOM 树。这个例子中，使用了列表的序号作为 key。

做 100 次操作，需要做 100 * 100 = 10000 个虚拟 DOM 节点的差异对比，但是因为数组项对应的 DOM 的 key 没有变化，因此并不会对 DOM 树结构进行操作，而仅仅会更新这个元素内部的文字，这个操作的复杂度是线性的。但是这里的虚拟 DOM 节点的差异对比并不是简单的值的对比，耗时比 10000 次比较数据要多，性能的提高点在于省掉大量修改真实 DOM 树结构的时间

## Vue

Vue 每秒能进行 10000 次操作，又大约是 React 的 10 倍。每一次对数组的赋值，都直接触发了数据项的对比然后直接更新绑定 DOM 元素的内部文本。没有模版关联，也不用对比虚拟 DOM 节点差异。

做 100 次操作，只需执行 100 x 100 = 10000 求值并对比，然后在线性时间内更新文本，没有其他额外的耗时操作。

## 总结

漫长的分析后可以自信地说出这个结果了，渲染性能上 Vue > React > Angular。

## 题外话

虽然本文仅从渲染性能上对比 Angular，React 和 Vue，但是用于不用这些框架绝对不能仅看性能的。Angular 社区成熟，轮子丰富，开发快捷，适合开发面向客户的产品的需要追求沉稳需求；React 组件化思想浓厚，高度可重用，响应数据变化快捷，结合 Redux 等框架后开发具有复杂交互操作的产品十分便利，但是体积庞大，代码冗长书写速度较慢，适合开发对内的运营项目；Vue 轻便简洁，渲染速度极高，只做一件事并做到极致，但是缺少轮子，而且不支持 IE8，适合开发 Html5 项目，尤其是手机端 SPA。当然，最终的选择，还是看团队需求。

文章链接：
 - {% post_link Angular-React-Vue-Rendering-1 %}
 - {% post_link Angular-React-Vue-Rendering-2 %}
 - {% post_link Angular-React-Vue-Rendering-3 %}
 - {% post_link Angular-React-Vue-Rendering-4 %}
 - {% post_link Mithril-Rendering %}

[1]: https://angular.io
[2]: https://facebook.github.io/react
[3]: http://vuejs.org/
[4]: https://www.polymer-project.org/1.0/
[5]: https://facebook.github.io/immutable-js/
[6]: https://github.com/magnumjs/mag.js/
[7]: http://mithril.js.org/
