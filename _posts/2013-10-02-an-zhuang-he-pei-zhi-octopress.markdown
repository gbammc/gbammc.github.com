---
layout: post
title: "安装和配置Octopress"
date: 2013-10-02 21:26
comments: true
categories: Octopress
keywords: ios,octopress,ruby,gem,github
description: 然而因为 Octopress 不更新了，现在已经转成 Jekyll。
---

在平时的学习中，深深受益于网络上很多乐意分享自己学习成果的大牛，因此也想找一个途径来记录并分享自己的学习情况，并希望藉此能促进自己的练级速度和深度，所以最终也决定以写blog的形式来做这件事。

最近程序猿界流行用Octopress + Github Pages的方式来搭建自己的个人blog，所以也就跟风学习一下。（主要是觉得Octopress是开源的，自由度很高，不像某度，某sdn等，会强插各种没用的广告或效果。__Less is more.__ 越纯粹，越好。）

---

## 安装：

不知道什么原因，跟着[Octopress](http://octopress.org/)官网的教程一步步做，在输入`rake deploy`后会出现以下问题：

``` bash
## Pushing generated _deploy website
To git@github.com:gbammc/gbammc.github.io.git
 ! [rejected]        master -> master (non-fast-forward)
error: failed to push some refs to 'git@github.com:gbammc/gbammc.github.io.git'
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart. Merge the remote changes (e.g. 'git pull')
hint: before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```

又经过多次搜索大牛的文章以后，终于在按照[这里](http://evsseny.appspot.com/2012/03/30/Octopress-blog-install.html)的步骤以后安装完成。为免日后找不到，就在这里再列一下吧。

### 安装rvm和ruby

首先安装rvm：

``` bash
bash -s stable < <(curl -s https://raw.github.com/wayneeseguin/rvm/master/binscripts/rvm-installer)
```	

然后设置classpath ( shell 下 )：

``` bash
	echo '[[ -s "$HOME/.rvm/scripts/rvm" ]] && . "$HOME/.rvm/scripts/rvm"
	source ~/.bash_profile
```

最后安装ruby：

``` bash	
rvm install 1.9.3 && rvm use 1.9.3
rvm rubygems latest
```

最新版的Octopress只支持ruby 1.9.3版本以上，所以已经安装旧版ruby的同学还需升级一下。
	
### 安装Octopress

先从github上将源码clone下来：

``` bash
git clone git://github.com/imathis/octopress.git octopress    # 从github clone octopress的源代码，后面的octopress是本地存放文件夹的名称，可以自定
cd octopress      # 进入所选的文件夹
ruby --version    # 应该显示Ruby 1.9.3
```

然后安装一些依赖：

``` bash
gem install bundler
bundle install 
rake install    # 安装主题
rake preview    # 本地预览 （http://localhost:4000/）
```

### 把blog部署到github：

创建一个名称为user_name.github.com的repo（必须使用username.github.com这种格式命名仓库，页面生成需要几分钟的时间才可以正常访问```http://user_name.github.com```或者```http://user_name.github.io```）

``` bash
rake setup_github_pages # 和github创建关联
git@github.com:your_username/your_username.github.com.git   # 按提示输入github URL
rake generate # 把你所有编辑的内容生成你的Blog静态页面
rake deploy   # <span class="goog_qs-tidbit goog_qs-tidbit-0">如果检查没有任何问题就可以 push 你的 blog 到 github master branch</span> 
＃ 状态检查
cd ~/octopress
git status   # 应该显示 On branch source
cd _deploy/  # 应该显示 On branch master
＃ 最后提交到source branch
git add .
git commit -m 'first commit'
git push origin source # 如果这一步出错，请再次检查仓库名称是否按要求命名，同时检查Admin面板下Default Branch是否为 master
```

但是到这里，当我去看Admin面板时，却只有source这个分支，而没有master分支，只有当我提交一篇文章后，master分支才出现。可能是因为Octopress的源文件都存放于source分支下，而deploy的都在master分支上。[这里](http://huanggang.me/archives/654)有相关解释

	还有很重要的一步是把你的修改(文本修改，不包含”_deploy”目录，”deploy”目录保存;
	”rake generate”生成的静态页面内容，会被”rake deploy”命令提交到”master branch”)
	放到你的github pages(“source” branch)上

### 更新Octopress

``` bash
git pull octopress master     # Get the latest Octopress
bundle install                # Keep gems updated
rake update_source            # update the template's source
rake update_style             # update the template's style
```

### 写博客方法

写博客主要是用以下几个命令，[这里](http://octopress.org/docs/blogging/)有详细介绍：

* rake new_post[‘article name’] 生成博文框架，然后修改在source/_posts/下生成的文件即可
* rake generate 生成静态文件
* rake watch 检测文件变化，实时生成新内容
* rake preview 在本机4000端口生成访问内容
* rake deploy 发布文件

博文是采用markdown语法，另外增加了一些扩充的插件，markdown的介绍文章网上可以搜到很多，比如[这个](http://daringfireball.net/projects/markdown/)。

---

## 我的配置

### 分享和评论

因为GFW的原因，我们需要先删除一些东西，不然会造成加载很慢的情况。

* /_config.yml，把里面的twitter相关信息全部删掉
* /source/_includes/custom/head.html 把google的自定义字体去掉，原因同上。
   
在/_config.yml 这个配置文件里的每一项都有相应的注释，可以改一些博客头，作者名之类的东西。

而配置评论和分享到微博功能。步骤如下：

* 在_config.yml中增加一项： weibo_share: true
* 修改 source/_includes/post/sharing.html ，增加：

```
// 下面的大括号是全角的，如果复制，请自行改成半角
｛% if site.weibo_share %｝
｛% include post/weibo.html %｝
｛% endif %｝
```

* 增加文件：source/_includes/post/weibo.html
* 访问 http://www.jiathis.com/ ，获取分享的代码
* 访问 http://uyan.cc/ ，获得评论的代码
* 将上面2步的代码都加入到weibo.html中即可
   
### 主题

如果需要使用像我这样的主题，那么请参考这个：[Introducing an Octopress Theme, Greyshade!](http://shashankmehta.in/archive/2012/greyshade.html)

还需要带微博或Dribble分享的话，请参考这个：[为 Octopress 的 Greyshade 主题增加新浪微博和 Dribbble 的支持](http://www.imallen.com/blog/2013/05/12/add-support-for-weibo-and-dribbble-to-greyshade.html)

### 更多参考博文(注：侧栏设置并不适用于Greyshade主题)：
[Octopress主题改造](http://shanewfx.github.io/blog/2012/08/13/improve-blog-theme/)

[我的Octopress配置](http://www.yanjiuyanjiu.com/blog/20130402/)
