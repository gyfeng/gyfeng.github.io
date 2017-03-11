---
layout:     post
title:      "你好啊，代码狗！"
subtitle:   "懂得分享，才会成长"
date:       2017-02-25 12:00:00
author:     "CodeDoge"
header-img: "img/"
header-mask: 0.3
catalog:    true
tags:
    - 生活
---
> 博客 - 程序员分享与表达自己的方式。

## 一个假的程序猿
&#8195;&#8195;工作以来，一直都没有好好的静下心来好好记录工作、学习或生活的状态与感悟。看着周围的同事、朋友，都有自己的专属博客，将自己的学习与感悟与大家分享，一直觉得自己就是一个假的程序猿。  
&#8195;&#8195;作为一个程序猿，连Blog都没有，一定是假的，哈哈。其实写Blog的心一直都有，只是，作为一个"看外表"的人，对大众化的Blog平台都提不起兴趣，至于自己搭建Blog吧，对于一个前端白痴来说，做出来的东西太丑，丑得连自己都看不下去，还是继续做一个假的程序猿好了。

## 有一个真程序猿的梦想
&#8195;&#8195;前几天在阿里云上花了几十大洋买了个域名(_这里没有广告，没有广告，没有，没. . ._)，后来想想还是挺冲动的，但是仔细的想想也不能浪费啊，也没啥放的，就用来做博客吧。由于有之前的教训，不打算从头做起，准备找一个博客平台，然后将自己的域名解析到上面，简单方便，又不失比格。
## 开干吧，阿猿
&#8195;&#8195;程序猿的风格就是，说干就干。经多方面考察和研究(_这也行？_)，最终，选定了[GitHub Pages](https://pages.github.com/)作为页面服务器，将自己的域名解析到GitHub的方式来搭建自己的专属Blog。  
#### 选用GitHub Pages的几个理由:
- 不用自己管理服务器，省心省事
- 强大的托管平台，不怕文章丢失
- 有强大的版本管理功能，方便进行版本管理
- 重要的一点，免费
- 支持域名解析到其Pages页面
- 使用Markdown语法，简单、简洁
- 对Jekyll的定制非常容易，灵活
- . . .&nbsp;&nbsp;&nbsp;. . .&nbsp;&nbsp;&nbsp;还有很多很多

&#8195;&#8195;当然，因为服务器在国外，访问速度上肯定是个缺点，但因Blog更多属于静态内容，变化很小，有可进行大量缓存，除第一次访问有点慢之后，以后都还能接受。

## 创建GitHub仓库
&#8195;&#8195;[GitHub Pages](https://pages.github.com/) 有两种，一种是针对用户，一种是针对单个项目的。我们选择前者。首先，我们在GitHub上创建一个命名为username.github.io的项目，当然，这里的username是指自己的用户名了。这里我自己的就是gyfeng.github.io了。  
&#8195;&#8195;这样，就可以访问`https://username.github.io/`看看效果了，这里的`username.github.io`是GitHub免费提供为你的域名，有没有很嗨啊？GitHub提供了多种默认模板可供选择。设置方法：  
1.&nbsp;&nbsp;进入项目设置页面  
![GitHub设置](/img/in-post/hello-codedoge/github-project-setting.png)
2.&nbsp;&nbsp;找到GitHub Pages设置，使劲戳"Choose a theme"按钮
![主题选择](/img/in-post/hello-codedoge/github-launch-theme-chooser.png)
3.&nbsp;&nbsp;选择喜欢的主题吧
![主题选择](/img/in-post/hello-codedoge/github-theme-choose.png)
## 从Hello World开始
&#8195;&#8195;不如俗套一把，就从Hello World开始吧。
1. 首先把username.github.io仓库clone到本地：
```
git clone https://github.com/username/username.github.io.git
```
2. 进入工作目录：
```
cd username.github.io
```
3. 修改首页文件：
```
echo "Hello World" > index.html
```
4. 然后上传至GitHub。  
&#8195;&#8195;剩下的就是打开你的Blog了。记得，多按几下Ctrl+F5(_都是猿，别问为什么_)，就可以看到熟悉的Hello World了。  
&#8195;&#8195;什么？需要编写Html？还能不能好好的写博客了？ 别急，慢慢来嘛。

## Jekyll现身
&#8195;&#8195;刚对有简单提及到`Jekyll`，没错，GitHub Pages 就是使用 [Jekyll](http://jekyllcn.com/) 将我们的各种将纯文本转换为静态页面。
### 安装Jekyll
因为Jekyll是基于ruby的，所以需要先安装ruby
```
$ sudo apt-get install ruby-full
``` 
安装gem
```
$ sudo gem install rubygems-update
```
安装jekyll  
```
$ sudo gem install jekyll bundler
```  
验证是否安装成功
```
$ jekyll -v
jekyll 3.4.0
```
显示版本号即表示安装成功　　
**_注意_：**可能有些地方网络不好，安装不成功，需要设置ruby源：
```
$ gem sources --remove https://rubygems.org/
$ gem sources --add http://gems.ruby-china.org/
$ gem sources -l
http://gems.ruby-china.org/
# 确保只有 gems.ruby-china.org
```


