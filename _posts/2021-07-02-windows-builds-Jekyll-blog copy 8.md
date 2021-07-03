---
layout: post
title:  Windows系统搭建Jekyll主题博客
date:   2021-07-02 21:52:23 +0800
image:  orange.jpg
tags:   其他4
music-id: 28563314
# description: 教你如何搭建博客
excerpt: This is a description to make up the numbers.This is a description to make up the numbers.This is a description to make up the numbers.This is a description to make up the numbers.This is a description to make up the numbers.This is a description to make up the numbers.This is a description to make up the numbers.This is a description to make up the numbers.This is a description to make up the numbers.
---


## 安装 Ruby

进入https://rubyinstaller.org/downloads/

![]({{ site.baseurl }}/images/image-20210702211805686.png)

点击 ==Archives== 查看历史版本

选择 [Ruby+Devkit 2.6.5-1 (x86)](https://github.com/oneclick/rubyinstaller2/releases/download/RubyInstaller-2.6.5-1/rubyinstaller-devkit-2.6.5-1-x86.exe) 版本安装

![]({{ site.baseurl }}/images/image-20210702212307934.png)

![]({{ site.baseurl }}/images/image-20210702212348201.png)

勾选全部安装

![]({{ site.baseurl }}/images/image-20210702212656459.png)

点击Finish安装MSYS2

![]({{ site.baseurl }}/images/image-20210702212800356.png)

输入1按下Enter开始第一步的安装,2是可选的，1，3是必须安装的。

![]({{ site.baseurl }}/images/image-20210702212954661.png)

第一步安装成功,此时输入2，3都是可以的。

![]({{ site.baseurl }}/images/image-20210702213215717.png)

第二步失败也没事，反正是可选的，接下来输入3。

![]({{ site.baseurl }}/images/image-20210702213433209.png)

第三步好像也失败了，不过没事。由于老版本和墙墙的原因，失败或许难以避免。

重新打开一个命令行窗口，输入 ==ruby -v== 和 ==gem -v== 来查看版本。

![]({{ site.baseurl }}/images/image-20210702213702159.png)

好像都安装成功了，那就不管那么多了。

***

## 安装 Jekyll

gem 在我理解里就类似于 npm 吧，都是版本依赖管理。

{% highlight shell %}


gem install jekyll -v 3.8.5

{% endhighlight %}

输入以上命令安装，百度了很久，这个版本比较稳。

![]({{ site.baseurl }}/images/image-20210702214133522.png)

貌似是安装成功了，完活。

***

## 搭建博客

这里我选择了 ==zolan== 主题 https://github.com/artemsheludko/zolan

这主题啥功能都没有，就图一个首页看着顺心，后面自己折腾吧。

![]({{ site.baseurl }}/images/image-20210702220255271.png)

最终的效果看起来还行，生命不息，折腾不止！