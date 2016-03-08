---
title: ARV 渲染实现比较之 Angular
date: 2016-03-02 11:09:55
comments: true
categories:
 - 技术
tags:
 - Javascript
 - Angular
 - React
 - Vue
---

重新开始写博客的第一篇，还是来点干货，在闭关一年多之间，我在项目中使用过 [Angular][1], [React][2], 和 [Vue][3]，今天我想说一下他们在渲染方面的一些实现的异同。

首先，要澄清一件事情，这几个东西虽然都可以用来渲染，但是却是不同级别的东西。Angular是一个功能十分强大的 MVVM（Model-View-ViewModel）框架，拥有包括模块化、服务在内的大而全的解决方案；而 React 只是一个组件化虚拟 DOM 库，也就是 View-ViewModel 部分；Vue 也是一个组件化的 View-ViewModel 库，但是他使用了类似 Angular 的模版语法。

Angular 虽然功能很全，但是这就意味着灵活性的降低。包括他自己定义的一套 html 扩展语法到js代码的映射关系，定义服务和过滤器的复杂写法，想用一个非 Angular 系的插件都要回想一下如何遵守他的规定来写扩展，大大降低了我们写代码的积极性。虽然 Angular2 号称解决了很多 Angular1 版本的缺点，但是既然他还在Beta阶段，我也不好去做什么评价。至少现在还没有多少人会去真正的使用它，所以今天这里的 Angular 仅代表他的第一代的产品。

<!--more-->

扯远了，忍不住吐槽了一下 Angular ，今天只讨论他们之间渲染的异同。至于 [Polymer][4] 这种真正基于 Web Component 的Polyfill 我只能说太超前了，一个组件至少一个请求我还是接受不能，而且虽然是一个 Polyfill，但是却不兼容目前大部分的运行环境，也并不知道未来是否真的能成为一套完善的标准，所以暂时不纳入讨论范围。有机会倒是可以研究一下其效率，这是后话了。

## Angular

这里假设读者对 Angular 的术语有一定的了解，比如 scope，filter，directive，controller。Angular 响应 ViewModel 变化的方法简单来说就是脏值检查（dirty-check），以下介绍这个脏值检查的基本概念。

### Start Up

整个 Angular 应用的启动代码类似下面这样：

```js
$compile($document)($rootScope);
$rootScope.$digest();
```

### Scope

Scope 相当于 MVVM 中的 ViewModel，所有跟渲染视图有关的变量，事件处理方法，都会绑定到 Scope 对象上，这样就可以在视图模版对这些值和方法进行调用。 Scope 可以有子 Scope，子 Scope 跟父 Scope 具有类似于 prototype 的继承关系。而整个系统的顶级 Scope 就是 $rootScope。

### Template Linking

`$compile` 是一个 Angular 模版编译器，他将在DOM树中的所有 directive 绑定到对应的DOM元素，并进行初始化。编译器执行后会返回一个绑定函数，这个绑定函数可以用来将编译后的模版绑定到一个 Scope 对象，这个过程称为模版关联（template linking）：

``` js
var linker = $compile(element);
linker($scope);
```

在这个过程中，一些 directive 会创建子 $scope，比如 `ng-controller`，`ng-repeat` 等等。这样 ViewModel 终于和 View 绑定起来了。

但是绑定后并不会马上渲染，绑定后执行 Scope 对象的 `$digest` 方法，这个模版才真正运作起来。

### Watch & 视图更新

在分析 `$digest` 之前，先了解一下 Scope 的变化是怎么映射到视图的。`$scope.$watch` 方法是用来监听 Scope 中的值变化的基本方法。他的语法是：

``` js
$scope.$watch(watchExpression, listener, [objectEquality]);
```

简单来说，绑定监听器后，如果 `watchExpression` 的运算结果变化了，`listener` 就会被调用。第三个参数 `objectEquality` 则代表是否用深度拷贝的方法来比较对象的变化，默认情况下，对于表达式运算结果为对象的监听器只采用引用对比。另外 `watchExpression` 必须是幂等的，即对于相同的输入，运算结果必须一致。

如果使用 `ng-repeat` 这种基于集合数据的监听，就会使用 `$watchCollection` 方法监听集合操作：

``` js
$scope.$watchCollection(obj, listener);
```

`$watchCollection` 实际上会穷举集合中的每一个 key 的值，只要有增删改的操作，`listener` 就会被调用。

一般的 Angular 用户可能接触这些方法不是很多，但是如果写过插件，写过 directive 的用户就可能接触的比较多，他们就会知道，实际上所有基于 Scope 中变量改变而引发的行为，都是通过 `$watch` 绑定的，包括使用最多的 directive `ng-bind`，只是这些 directive 都已经封装好可以直接用了。

所以，所有的视图更新，都是在 `listener` 中实现的，比如 `ng-bind`，在检测到表达式结果改变时，直接调用所在 element 的 html 方法将表达式的结果写到 element 的 innerHtml 里面去：

```js
scope.$watch(ngBindHtmlWatch, function ngBindHtmlWatchAction() {
  element.html($sce.getTrustedHtml(ngBindHtmlGetter(scope)) || '');
});
```

至于 Watch 是怎么监听到 `watchExpression` 的变化的，请继续往下看。

### Digest & 脏值检查

`$scope.$digest` 方法是用来检查 Scope 数据变化的方法，调用一次 $digest 会执行一次上面提到的脏值检查，这将执行所有在这个 Scope 及其子 Scope 用 `$watch` 绑定的 `watchExpression`，并将运算结果与上一次检查时的结果进行对比，若发生变化则调用对应的 `listener` 处理函数，并将本次 `$digest` 标记为 `dirty`。如果一次 `$digest` 执行完毕后结果为 `dirty`，那么将马上重新执行另外一次 `$digest`, 直到不会再产生 `dirty`。这就被称为 Digest 循环。

之所以在一次 `dirty` 的 `$digest` 之后重新执行一次 `$digest`，是因为 `listener` 中的逻辑可能涉及到修改 Scope，以至于之前检查过的 `watchExpression` 产生不同的计算结果。

至于 Digest 执行的时机，一般而言并不需要用户去手动执行，如果你的系统完全运行在 Angular 体系下， Angular 会在适当的时候去执行，比如使用 `$timeout`，`$http`，`ng-controller`，`ng-click`，`ng-model` 等，这也是 Angular 要重新造轮子的原因。

但是如果你要自己写 directive 或者想用 Angular 体系外的东西时，就得手动触发 Digest 循环：`$scope.$apply()`。

### 性能

总的来说，Angular 渲染的逻辑就是，在任何可能引发 Scope (ViewModel) 改变的地方，调用 Digest 循环进行脏值检查并在发现脏值后修改 DOM 树 (View) 进行渲染，直到 Scope (ViewModel) 稳定下来。

可以看出来，一次可能的变动，将遍历整个应用中绑定的监听器，效率十分低。不过优化的方法也很明显，就是减少绑定监听器的数量，比如在一些绑定后不会变化的地方在一次脏值出现后马上注销监听器（bind-once)，或者在可能发生脏值之前再注册监听器，之后马上注销监听器等等。另外使用 `ng-repeat` 时最好使用稳定的属性（比如id）来作为 `track by` 的跟踪属性，防止不必要的模版关联。

### 小结

Angular 虽然有点慢，但是无可否认，开发者会为此埋单，毕竟良好的兼容性，众多的插件，方便的双向绑定都得到了人们的中意。

由于文章有点长，所以 React 和 Vue {% post_link Angular-React-Vue-Rendering-2 下期再写 %}。

文章链接：
 - {% post_link Angular-React-Vue-Rendering-1 %}
 - {% post_link Angular-React-Vue-Rendering-2 %}
 - {% post_link Angular-React-Vue-Rendering-3 %}
 - {% post_link Angular-React-Vue-Rendering-4 %}

[1]: https://angular.io
[2]: https://facebook.github.io/react
[3]: http://vuejs.org/
[4]: https://www.polymer-project.org/1.0/
