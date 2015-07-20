---
layout: post
title: "关于在64位 Windows 7 中部署中文化的Octopress"
date: 2012-03-23 15:52
comments: true
tags:  
 - Octopress
 - 技术
---

<blockquote>
前言——可以在Linux环境下部署的话，还是尽可能在Linux下部署吧...
</blockquote>

真的不是开玩笑的，除非你像我一样喜欢折腾。即使没有Linux系统，能够运行虚拟机的话，装个虚拟的Linux系统也比直接在Windows中部署要简单。

一般的安装步骤，在<a href="http://octopress.org/docs/">Octopress的文檔</a>中就有详细的说明。而在Windows 7中部署，则可参考<a href="http://sinosmond.github.com/blog/2012/03/12/install-and-deploy-octopress-to-github-on-windows7-from-scratch/">Sinosmond的一篇文章</a>。
具体的部署过程，我就不再重复了，只是在部署过程中有几点是需要注意的。

<!--more-->

<h2>Ruby</h2>
Octopress要求的Ruby的版本是1.9.2，最好使用该版本，因为不同版本间的函数库有可能有出入，导致某些插件无法运行。这里经过我的折腾，发现最新版1.9.3也是支持的，目前使用起来没有什么问题，但是需要将octopress根目录下的“.rvmrc”文件中的一行改成
```
rvm use 1.9.3
```

<h2>分支</h2>
若要使用github的个人page，建立repo时设置的名字必须是<code>&lt;yourname&gt;.github.com</code>，这里<code>&lt;yourname&gt;</code>指的是你的github用户名。这样就可以让你的页面可以通过地址<code>http://&lt;yourname&gt;.github.com</code>，来访问，如果不是这样命名的话，你的github pages只能通过<code>github.com/&lt;reponame&gt;</code>访问。
使用<code>rake setup_github_pages</code>后，需要输入github pages的repo地址，格式是<code>git@github.com:&lt;yourname&gt;/&lt;yourname&gt;.github.com.git</code>。

使用<code>rake deploy</code>后，会将public活页夹下的所有文件拷贝到分支管理目录_deploy活页夹中，也即是<code>&lt;yourname&gt;.github.com</code>的master分支目录，然后上传到github。如需对源代码进行版本管理，需要另外建立source分支，并使用基本的git命令进行版本管理。

<h2>中文化</h2>
Octopress原本就是一个英文的框架，所以并没有考虑很多使用其他语言会导致的问题。在尝试中文化时，可能会遇到一些问题，还好这些问题都是能解决的。

<h4>rake generate失败</h4>
如果直接使用原框架书写中文博文，会在generate时失败，提示出现非法UTF-8字符。首先要确认是否已经按照<a href="http://sinosmond.github.com/blog/2012/03/12/install-and-deploy-octopress-to-github-on-windows7-from-scratch/">Sinosmond的文章</a>配置，并设置好<code>LANG</code>等环境变量。如果仍出现该提示是因为原本的文件和生成的文件都是ASCII编码的，如果直接输入中文当然不能被识别。正确的做法是，若要在文件中书写中文，首先将文件保存为<code>UTF-8 without BOM</code>编码格式，然后再进行书写。注意是<code>UTF-8 without BOM</code>而不是单纯的<code>UTF-8</code>编码，如果存成后者，在generate时不会出错，但是生成页面时会出现奇怪的现象。

<h4>中文化文章分类</h4>
如果直接使用中文的文章分类，在deploy后会发现，点击文章分类后出现404错误。这是因为在在generate时，<code>category_generator.rb</code>插件将根据分类名称生成分类页面活页夹，而生成的活页夹是中文的，这在URL中是不允许的，因此无法定位到该页面。这里有<a href="http://geron.heroku.com/blog/2012/octo-cate-cn-spo">Geron的一篇文章</a>，介绍如何为octopress提供中文分类支持。但是我使用该方法后，并没有成功应用。他提供的方法是直接将中文的文章分类转换为url中的编码(就是那种类似<code>%3d</code>这样代表文字的编码)。我使用后，确实令中文的活页夹变成了URL编码的活页夹，这样URL就跟目录相一致。我在本地也测试成功，但是上传github后依然出现404错误，并且考虑到这种方法会产生意义不明显的URL，所以只好采用别的方法。这里我想到了一个将分类名称跟索引分离的方法。即是在一个分类变量中，同时储存一个要显示的名称，还有一个要生成的路径名，这一串字符作为分类索引，并令显示与实现分离。

<a href="https://github.com/denjones/denjones.github.com/commit/1d4f3b9433a4d77e31530c4d5f20611c9b9829e2#diff-1">这里是我对<code>category_generator.rb</code>的修改</a>。

修改后的分类格式变更为<code>&lt;分类显示名称&gt;{&lt;分类目录名称&gt;}</code>。比如说你想建立一个“随笔”分类，你想让分类页面保存在一个叫“essay”的目录中，你就要在文章markdown文件的头部加入这样的一行：
```
categories:  随笔{essay}
```
如果希望归类到多个分类，则需要这样写：
```
categories:
 - 随笔{essay}
 - Octopress{octopress}
```

除此之外，还要对一些用于显示分类名称的页面做一些修改。把其中的<code>category</code>修改为<code>category[/[ ^ { ]*/]</code>，因为这里的<code>category</code>已经变成了<code>&lt;分类显示名称&gt;{&lt;分类目录名称&gt;}</code>的格式，需要使用正则表达式取出<code>&lt;分类显示名称&gt;</code>这一部份用于显示。

我的octopress框架已经对相关部份做了处理，是一个比较完善的中文版本的Octopress，如果不喜欢折腾的可以直接在Github <a href="https://github.com/denjones/denjones.github.com/tree/source">clone本博客的框架的开源代码</a>，然后再把其中的<code>_post</code>等目录中的多余文件去掉，修改为自己的框架进行使用。

<h2>代码高亮</h2>
其实这个问题才是在64位Windows 7中部署Octopress会遇到的难题。Octopress已经自带了代码高亮(Highlighting)的相关插件，使用的是<a href="http://pygments.org/">pygments</a>这款插件。但是这款插件是用Python语言写的，所以在本地运行时，需要有安装Python环境。因此进入<a href="http://www.activestate.com/activepython/downloads">Python的主页</a>下载安装包进行安装。好了，既然是64位的Windows7系统，那么首选当然是ActivePython的64-bit版本。兴高采烈的下载安装后，发现问题来了，如果设置了代码高亮的<code>lang</code>属性，generate时会出现错误<code>Liquid error: Could not open library ‘.dll’: The specified module could not be found.</code>。查看错误消息发现在执行<code>rubypython.rb</code>中的函数时，产生了错误。Google后得知rubypython对Windows支持不好，因此需要手动修改其中的一些代码。

对ruby目录下的<code>lib\ruby\gems\ruby 1.9.x>\gems\rubypython-0.5.x\lib\rubypython\pythonexec.rb</code><a href="https://github.com/bendoerr/rubypython/commit/1349aea1c6faa459c4be8474e4a7e878f08459c2">作此修改</a>。

一般来说这样就可以解决问题，但是在这里这个错误<code>Liquid error: Could not open library ‘.dll’: The specified module could not be found.</code>依然出现。考虑是不是64位的问题，于是进入<code>C:/windows/sysWOW64</code>下，并没有发现Python的相关dll，于是到<code>C:/windows/system32</code>下，将<code>python27.dll pythoncom27.dll pywintypes27.dll</code>拷贝到<code>C:/windows/sysWOW64</code>下。generate发现错误变成<code>Liquid error: Could not open library ‘C:/windows/system32/python27.dll’: The specified module could not be found.</code>。然后我就开始百思不得其解。最后没有办法，尝试安装了ActivePython的32-bit版本，问题迎刃而解。估计是rubypython对64位的python环境支持不好，无法打开64位Python的dll。所以在选择ActivePython版本时，请使用32-bit版本。

<h2>结语</h2>
至此，中文版的Octopress在64位windows7中部署成功。折腾了那么久，总算有所回报。但是想到要在别的Windows机器上写博客，也要经过如此复杂的环境配置，我就觉得蛋疼。还好中文框架源码已经使用了版本管理，并不需要对框架进行重复的修改。总的来说在Linux下部署Octopress要比在Windows中简单得多，若经不起折腾，还是不由选用Windows + Octopress这种组合。但是既然你选用了Octopress，证明你的折腾能力还是有的，因为Octopress是一款面向Hacker的博客框架，就在使用Octopress的过程中，享受折腾带来的乐趣吧。
