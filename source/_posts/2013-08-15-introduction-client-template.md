---
layout: post
title: "前端模板的选择"
date: 2013-08-15 20:01
comments: true
tags:
 - 随笔
 - 技术
 - Javascript
---

最近接到一个任务，让我选一个前端模板。但是我其实连js模版实战经验都没有，为什么要让我选呢。不过接到任务就得照办了。首先得先知道什么是前端模板呢，稍微查了些百科，简单来说就是利用内嵌于HTML中的一些标记，来直观迅速生成HTML字符串的工具了，类似于后台用PHP标记<?=xxx>来生成HTML这样，而不需要繁琐的字符串拼接操作。需要在前端用javascript来使用，所以叫做前端模板。那么类似后台有多种语言，前端模板也有多种多样，以至于有人在github上做了一个应用方便大家选择前端模板：[TemplateChooser](http://garann.github.io/template-chooser/)

<!--more-->

虽然其中没有包括所有的模板，但是人们最常用的都在里面了，点开左边的标签设置选项，右面剩下的模板名就是符合要求的了。

第一项 Is this for use on the client or the server，就是模板需要运行在前端还是后台，对于前端模板当然选前台，但是不少模板是可以运行在前后端的。

第二项 How much logic should it have，表明你需要模板中可以包含多少逻辑处理。其中 the entirety of JS 选项代表模板中需要可以运行JS代码；just the basics 则代表模板中仅包含基本的逻辑，比如判断这类的；none at all 则表示模板中不包含任何逻辑，换而言之几乎只能做变量替换的操作。

第三项 Does it need to be one of the very fastest，表示你是否对速度有特别严苛的要求，yes or no。选完yes后会发现只剩三种doT.js、Microtemplating、Underscore templates，这是由一个应用来判定的：[jsPerf](http://jsperf.com/javascript-templating-shootoff-extended)。这三种在测试中是排行前三的。但是我们发现里面评测的模板并不完全，所以其实这项并没有太大的参考价值。

第四项 Do you need to pre-compile templates，表示你是否需要可以预编译的模板。首先要解释一下编译的意思，对于一般编程语言，编译就是让代码转换并优化成机器可以识别的机器码，从而机器可以直接运行。这里的编译意思类似，一个模板只是一个直观的数据字符串，我们需要让他变得可执行，那么我们就要让他转变成一个可以执行的函数对象，虽然这个对象一般也是以字符串的形式存在的。是否可以预编译的差别在于，如果你的模板是固定不变的，那么如果预先编译好并将编译结果保存起来，要使用的时候就不需要再动态编译，可以减少你分析模板的时间。

第五项 Do you need compile-time partials，表示你是否需要在编译时的模板拆分。这里的拆分其实并不是准确的表达，但是我找不到一个更好的词来形容了。实际上这个概念很简单，就是模块化的思想，可以将多个模板中重复使用的部分拆分出来单独存成一个模板文件，而让其他需要使用这个部分的模板导入这个模板文件。这种操作一般只在服务端模板中进行，因为在前端难以动态读取后台的分布式拆分文件。但是对于可以预编译的模板，则可以预先将分布式的文件编译好，然后在前台一一导入后使用。

第六项 Do you want a DOM structure, or just a string，那是问你的模板是一个字符串还是一个DOM结构。关于这点我也不太清楚，因为在我的印象中模板就是一个字符串，而里面包含了所渲染的HTML字符串的直观表示。选了DOM之后，同样只剩下三种模板dom.js、pure.js、Transparency。我简单地查看了一下pure.js模板（他同时也是一个不包含逻辑的模板），发现他并不是将一个模板保存为字符串，而是直接在HTML文件中为标签添加一些特殊的类，然后渲染的时候再用选择器出这个DOM对象进行渲染，换而言之他的模板是直接内嵌在HTML文件中的！那么确实如果希望应用逻辑是不大可能的，因为必须保证模板符合XHTML规范。

第七项 Aside from template tags, should it be the same language before and after rendering，这项比较好理解，除了模板标签之外，模板的语法要和模板渲染后的语法一样，那么在这里我们要渲染出HTML语言，结果是显然易见的，就是模板是一种在HTML语言中内嵌标签的语言。或许你会觉得很奇怪，难道还有什么除了插入标签外的神奇的语法吗？选No之后又再一次出现了三种模板dom.js、Jade templates、Microtemplating。我再一次简单的看了下dom.js，因为他同时也是一个使用DOM结构的模板。结果发现DOM是一种完全使用Javascript的模板！他的语法还是非常直观的：

``` javascript
header(
    h1('Heading'),
    h2('Subheading'));

nav(
    ul({ 'class': 'breadcrumbs' },
      li(a({ href: '/' }, 'Home')),
      li(a({ href: '/section/'}, 'Section')),
      li(a('Subject'))));

article(
    p('Lorem ipsum...'));

footer('Footer stuff');
```

明白了上面的含义的话，其实还是很好选择的。关于选项中的第二项可能有点争议，到底模板需要多少逻辑呢。如果从模板的出发点来看的话，其实答案是显然易见的。有些人可能觉得功能越强大的模板越好，但是事实恰好相反，人们发明了模板就是为了直观地设计输出数据的表现形式，让显示与逻辑数据分离。此时若为了让模板支持更强大的功能而硬是往里面添加复杂的逻辑，甚至让其支持javascript，是本末倒置的，这些明明可以直接在javascript中操作的东西又何必硬塞到模板里面让其可读性降低呢。然而不包含任何逻辑毕竟不现实，因此大多数模板都提供了最基本的逻辑能力，比如遇到空对象的特殊处理方法。所以这里一般选择just the basics。

最后剩下来的模板也只剩6个，但是到底要如何取舍，还是不能简单的决定。经过一些对比评测[1]，大家普遍认为以下三种是综合评分最高的：Mustache.js、Handlebars.js、dust.js。

[Mustache](https://github.com/janl/mustache.js/) 应该是最流行的模版了，显示与逻辑分离，模板语法清晰，但是逻辑功能太过缺乏，代码庞大而且效率比较低，已经停止维护超过两年。他的语法如下：
``` html
{ {#stooges} }
<b>{ {name} }</b>
{ {/stooges} }
```

[Handlebars](https://github.com/wycats/handlebars.js/)
是Mustache的改进，显示与逻辑分离，语法兼容Mustache，可以编译成代码，改进Mustache对路径的支持，但是若需要在服务端运行需要使用服务端Javascript引擎如Node.js。
``` html
{ {#each comments} }
<div class="body">{ {body} }</div>
{ {/each} }
```

[Dust](https://github.com/linkedin/dustjs)
也是显示与逻辑分离，语法比Mustache更清晰简便（少写一对花括号{}!），可以编译成代码，比Mustache提供更好的路径支持，支持更多的扩展功能，异步渲染，执行效率更高。但是若需要在服务端运行需要使用服务端Javascript引擎如Node.js。
``` html
{#names}
<li>{name}</li>{~n}
{/names}
```

以上三种都是最优秀的模板了，语法可以说是比较类似，但是无论从结果还是过程看Dust都有着无比强大的优势。目前Dust.js是由LinkedIn团队维护的，最近已更新到2.0版本，相比以前有更大的优化，所以我觉得选用dust.js作为模版是比较合适的。只可惜Dust.js毕竟没有胡子Mustache家族势力广泛，尤其在我大天朝，少有见到关于dust的技术文章。下次我打算尽我微薄之力，为大家介绍一下Dust的使用方法，普及一下dust这个LinkedIn推荐的模板。

参考：

[1] [The client-side templating throwdown: mustache, handlebars, dust.js, and more](http://engineering.linkedin.com/frontend/client-side-templating-throwdown-mustache-handlebars-dustjs-and-more)
