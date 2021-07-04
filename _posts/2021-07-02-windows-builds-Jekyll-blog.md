---
layout: post
title:  Windows系统搭建Jekyll主题博客
date:   2021-07-02 21:52:23 +0800
image:  https://image.wsdydeni.top/peter-thomas-ZgU60B3Wbzs-unsplash.jpg
tags:   博客
---


## 安装 Ruby

进入[https://rubyinstaller.org/downloads/](https://rubyinstaller.org/downloads/)

![](https://image.wsdydeni.top/Ruby%E4%B8%8B%E8%BD%BD%E7%95%8C%E9%9D%A2%E5%9B%BE.png)

点击 `Archives` 查看历史版本

选择 [](https://github.com/oneclick/rubyinstaller2/releases/download/RubyInstaller-2.6.5-1/rubyinstaller-devkit-2.6.5-1-x86.exe) 版本安装

![](https://image.wsdydeni.top/Ruby%E5%AE%89%E8%A3%85%E6%B5%81%E7%A8%8B1.png)

![](https://image.wsdydeni.top/Ruby%E5%AE%89%E8%A3%85%E6%B5%81%E7%A8%8B2.png)

勾选全部安装

![](https://image.wsdydeni.top/Ruby%E5%AE%89%E8%A3%85%E6%B5%81%E7%A8%8B3.png)

点击Finish安装MSYS2

![](https://image.wsdydeni.top/Ruby%E5%AE%89%E8%A3%85%E6%B5%81%E7%A8%8B4.png)

输入1按下Enter开始第一步的安装,2是可选的，1，3是必须安装的。

![](https://image.wsdydeni.top/Ruby%E5%AE%89%E8%A3%85%E6%B5%81%E7%A8%8B5.png)

第一步安装成功,此时输入2，3都是可以的。

![](https://image.wsdydeni.top/Ruby%E5%AE%89%E8%A3%85%E6%B5%81%E7%A8%8B6.png)

第二步失败也没事，反正是可选的，接下来输入3。

![](https://image.wsdydeni.top/Ruby%E5%AE%89%E8%A3%85%E6%B5%81%E7%A8%8B7.png)

第三步好像也失败了，不过没事。由于老版本和墙墙的原因，失败或许难以避免。

重新打开一个命令行窗口，输入 `ruby -v` 和 `gem -v` 来查看版本。

![](https://image.wsdydeni.top/Ruby%E5%AE%89%E8%A3%85%E6%B5%81%E7%A8%8B8.png)

好像都安装成功了，那就不管那么多了。

***

## 安装 Jekyll

gem 在我理解里就类似于 npm 吧，都是版本依赖管理。

{% highlight shell %}


gem install jekyll -v 3.8.5

{% endhighlight %}

输入以上命令安装，百度了很久，这个版本比较稳。

![](https://image.wsdydeni.top/Ruby%E5%AE%89%E8%A3%85%E6%B5%81%E7%A8%8B9.png)

貌似是安装成功了，完活。

***

## 搭建博客

这里我选择了 [zolan](https://github.com/artemsheludko/zolan) 主题 

这主题啥功能都没有，就图一个首页看着顺心，后面自己折腾吧。

![](https://image.wsdydeni.top/%E5%8D%9A%E5%AE%A2%E9%A2%84%E8%A7%88%E6%95%88%E6%9E%9C%E5%9B%BE.png)

最终的效果看起来还行，生命不息，折腾不止！