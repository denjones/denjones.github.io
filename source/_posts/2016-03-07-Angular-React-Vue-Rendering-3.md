---
title: ARV 渲染实现比较之 Vue
date: 2016-03-07 10:47:32
comments: true
categories:
 - 技术
tags:
 - Javascript
 - Angular
 - React
 - Vue
---



之前已经分析了 [Angular][1] 和 [React][2] 的渲染：
 - {% post_link Angular-React-Vue-Rendering-1 %}
 - {% post_link Angular-React-Vue-Rendering-2 %}

今天来分析 [Vue][3]。

## Vue

Vue 是一个新兴的前端视图框架，他做的事情跟 React 很类似，实现了前端组件化以及 View-ViewModel 绑定，但是同时又支持跟 Angular 很类似的模版语法，以及比 Angular 更方便的 filter 和 directive 定义方法，大大提高了开发效率。React 和 Angular 都熟悉的开发者转到 Vue 使用起来十分方便得心应手。

如果只是 React 跟 Angular 两者优点的结合，那 Vue 也没那么出彩，然而他的渲染效率甚至是 React 的 10 倍之多，这也是很多开发者转向 Vue 的理由。

<!--more-->

### Start Up

Vue 的启动代码大致如下：

``` js
var exampleVM = new Vue({
  el: '#example-1',
  data: exampleData
})
```

而定义一个 VueComponent 的代码如下：

``` js
// 定义
var MyComponent = Vue.extend({
  template: '<div>A custom component!</div>',
  data: {}
})

// 注册
Vue.component('my-component', MyComponent)
```

注册后就可以在 HTML 或其他模版中使用这个 VueComponent：

``` html
  <my-component></my-component>
```

### Vue 实例

无论直接使用 `new Vue()` 还是 `new MyComponent()` 还是在模版编译过程中使用 VueComponent  `<my-component></my-component>` 都会产生一个对应到 DOM 元素的 Vue 实例，这跟 React 的 ReactComponent 实例十分类似。

每个 Vue 实例都有一个 `data` 属性，它相当于 Vue 的 ViewModel 部分，只有赋予到 `data` 的数据，才能在视图模版中被调用，这一点跟 React 的 state 十分相似。

### 模版编译

在 Vue 实例化的过程中，会对模版（DOM 元素或者字符串模版，字符串模版会先转换成 DOM 元素）进行编译。这个过程跟 Angular 的 $compile 过程类似，他会递归地对所有普通元素的 directive 进行初始化，然后返回一个绑定函数，用于绑定 Vue 实例与初始化好的 DOM 元素。通过这个函数绑定 DOM 和 Vue 实例，就可以建立起 ViewModel 到 View 的关联。如果模版中使用了 VueComponent 子元素，那么这个 VueComponent 将会被实例化。

### Watcher & 视图更新

在 Vue 实例化的过程中会对 Vue 实例的 `data` 属性赋初值（data的值应该是一个对象），Vue 会遍历 `data` 对象的每一个属性，并将其转换成ES5 getter/setter。这是 Vue 能监听到 `data` 变化的前提。

然后与 Angular 的 `$watch` 类似，Vue directive 初始化过程也需要创建对应的 Watcher，不过这些 Watcher 不需要用户手动去定义，Vue 会自动根据 params 和 directive 表达式来创建。如果这些监听的值发生变化，Watcher 就会调用 directive 的 `update` 方法，从而更新视图。所以基本上所有的视图更新操作都是在 `update` 方法中进行，一个 directive 的定义方式如下，相比 Angular 要简单许多：

``` js
Vue.directive('my-directive', {
  bind: function () {
    // do preparation work
  },
  update: function (newValue, oldValue) {
    // do something based on the updated value
  },
  unbind: function () {
    // do clean up work
  }
})
```

到目前为止 Watcher 的行为都跟 Angular 类似，但是 Watcher 怎么监听到表达式的变化呢，难道像 Angular 一样进行 dirty-check？上面提到 Vue 实例化时会为 `data` 的属性创建 getter/setter，这是利用了 ES5 的 defineProperty 特性。当然这些 getter/setter 并不是简单的读写值，当 Watcher 进行表达式的第一次求值运算时，每接触一个 getter/setter，这个 getter/setter，就会将这个属性记录到 Watcher 的依赖列表。之后若这个属性的值被改变，那就意味着这个属性的 setter 将被调用。setter 设值后将调用依赖到这个属性的 Watcher 重新求值，这就可以监听到表达式的变化并触发视图更新。

这就是 Vue 对实例化后再添加的 `data` 属性无法响应的原因，虽然可以通过 Vue 实例的 `$set` 方法添加新的属性，但是这将导致所有跟这个 `data` 有依赖关系的 Watcher 重新求值，所以还是预先定义好 `data` 的属性比较靠谱。

### 异步更新

实际上 `update` 方法的调用是异步的，因为 Watcher 可能依赖到多个数据更新，如果每个数据更新都调用 `update` 方法会导致同一个 `update` 方法调用多次，而只有最后一次的 `update` 的结果才是需要的。 为了解决这个问题 Vue 会维护一个异步更新队列，并进行去重后再进行更新。因此对 `data` 并不会马上更新到视图，而只会在下一个心跳（setTimeout）更新，这跟 Angular 中对 `$scope` 修改又是类似的。

### 性能

Vue 的性能可以说是无可挑剔的，利用 ES5 的 defineProperty 特性，准确而直接地响应了数据的变更，当且仅当必须更新视图时，才会去更新渲染，直截了当。一般情况下无需担心 Vue 的性能问题，只要渲染集合数据时，如果对整个集合引用进行替换，他跟 Angular 会有同样的问题，无法得知变动前后集合元素的对应关系，但这同样的可以通过添加 `track-by` 指定一个元素的稳定 id 来解决。而且 Vue 压缩后的大小是 Angular，React 之中最小的，同样提高了加载速度。但是跟 React 差不多，他只是一个 View-ViewModel 库，如果需要加入 Model 部分，可能还要引入 Redux，Vuex 之类的库。当然，使用 plain object 也是可以的，我在一个项目中为了追求简便就是这么干的。

然而使用 ES5 的特性提高了性能也成为了 Vue 的短板，他无法支持 IE8 及不支持 ES5 defineProperty 特性的浏览器，这个特性也没法通过 polyfill 来补救，而兼容 IE8 恰好是 Angular 跟 React 都做到的。由于比较新 Vue 的插件也是比较少的，但是简单的语法让我们重新造轮子也非常得心应手，相比之下如果你让我在 Angular 下面造轮子，我还得去重新理解一遍他的概念，认真比对文档，生怕语法有什么弄错了。

### 小结

相比于 Angular 和 React，我这篇写的实在太短了，因为他的概念简单而熟悉，性能也没有多大问题，如果不需要支持 IE8 以下，无疑是个好的选择。如果需要开发手机端 WEB 应用应该可以毫无顾虑地用 Vue 了。

至此 ARV 渲染实现比较即将完结了，总结留到下一期。

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
