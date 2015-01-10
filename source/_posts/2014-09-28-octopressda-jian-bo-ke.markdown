---
layout: post
title: "octopress搭建博客"
date: 2014-09-28 20:11:47 +0800
comments: true
categories: other
---
像黑客一样写博客！

OK，终于有自己的博客站点了，这里记下怎么倒置的。   

octopress 官网：[http://octopress.org/](http://octopress.org/)   

首先各种配置ruby，git，github等就不赘述，百度或者官网都有具体步奏，照做即可。或者可以看下这个[视频](http://happycasts.net/episodes/36)，happypeter老师讲的很清楚，同时这个网站上有非常精彩的其他工具教程视频，很值得学习。   <!-- more -->
配置好后，就可以使用自己配置的github page地址来访问了。我基本是按照视频的方式来搭建以及管理的。如果喜欢我的博客也可直接clone，然后修改下就可以了，当然博客内容还是得删掉的。   

搭建好后就需要进行一些配置了，比如title,subtitle,author，这样以后你发表的博文才会以你的名义嘛。这些配置都在博客根目录_config.yml文件中，打开进行编辑，我用的是mac，所以这些操作都是很方便的，vim直接打开就可以了，window可以使用相应工具就行。这是一个配置文件，像刚才说到的博客名称，作者，以及第三方插件等都是在这个文件中配置的。这里先改下title，subtitle，author，保存退出然后rake preview， 打开浏览器，输入localhost:4000应该就能看到效果了。   

###文章编写
使用下边这条命令就可以生产一片新的博文，博文位于`source/_post`文件夹下
```
rake new_post["博客名称"] 
```
octopress使用Markdown进行编写，所以有一个合适的Markdown编辑器就很重要了，我在Mac下使用的是Mou编辑器，它可以是对编写文件进行对照，非常方便，如图所示，当然有其他的编辑器可选。
{% img  /images/1.png %} 
编辑完成后，我们就可以使用下边两个命令将其推送到github中。
```
$rake generate
$rake deploy
```

###换主题
默认主题其实也蛮好的，要是自己想弄的不一样一点的，那就自己动手，改一通代码，或者像我这样即懒又不会多少前端的，当然第三方主题最省事了。   
这里有收集的octopress第三方主题：[octopress theme](https://github.com/imathis/octopress/wiki/3rd-Party-Octopress-Themes)    
里边有很多漂亮的主题，看上自己喜欢的，clone搞定。每个主题github主页都有相应的安装步奏，照做即可。然后你就可以看到自己想要的主题了。比如foxslide   
``` 
$ cd blog
$ git submodule add https://github.com/sevenadrian/foxslide .themes/foxslide
$ git submodule update --init
$ rake install['foxslide']
$ rake generate
```   

###换域名
如果使用github page托管octopress，都可通过github page地址进行访问，比如我的ypengju.github.com，就可访问。要是想通过自己的域名访问，那只需简单的配置即可。

1. 在source目录下创建CNAME文件，在其中添加域名。比如我的CNAME文件中是ypengju.com
2. 添加域名解析，在你的域名服务商网站中，对你申请的域名添加解析，添加A解析，设值为207.97.227.245
3. 再添加CNAME解析，其值为你的github page地址，如我的ypengju.github.com    

之后将你的修改使用rake generate，rake deploy推送到github上，等上一段时间，就可通过绑定的域名进行访问了。
###加评论
octopress默认是支持disqus评论工具的。只需在_config.yml文件中，在disqus_short_name：后添加你的disqus账号名就可以在每篇博文后添加评论功能。但是dispus在国内并不好用，所以我采用的是友言评论，与其相同的还有多说等。

1. 注册友言账号，注册地址：[友言](http://www.uyan.cc/)   
2. 获取代码，将其添加到`source/_includes/post/share_comment.html`文件中。默认没有此html文件，创建一个即可。    
```
<!-- UY BEGIN -->
<div id="uyan_frame"></div>
<script type="text/javascript" src="http://v2.uyan.cc/code/uyan.js?uid=xxxxxxx"></script>
<!-- UY END -->
```  
3. 在`source/_includes/post/share.html`使用`include`将share_comment.html文件包含进来，在引入时先判断有没有定义comment_share变量  
{% raw %}
```
{% if site.comment_share %}
  {% include post/share_comment.html %}
{% endif %}
```
{% endraw %}
4. 在`_config.ymb`中添加comment_share变量 设置为true。如`comment_share: true`这样就可以在配置文件中控制是否需要评论系统。   
###加分类
当文章多了，有一个分类对于查阅就非常方便了，我是使用了github上的一个插件，只需简单配置。插件地址[octopress-category-list](https://github.com/ctdk/octopress-category-list)   
根据README提示，将文件拷贝到相应目录。然后在`_config.yml`进行配置。这个插件提供了三种category类型，根据自己喜好，将相应插件路径配置在`default_asides`这一项中即可。

octopress还支持其他很多插件，比如分类，tag，微博分享等，可根据自己需要，对其进行配置。这里是github上有人整理的octopress插件。   
octopress 插件：[octopress plugin](https://github.com/imathis/octopress/wiki/3rd-party-plugins)    
