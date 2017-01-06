---
layout: post
title: "Hybrid模式下支付宝钱包APP探秘"
date: 2014-11-25 13:16:12 +0800
comments: true
categories: hybrid
---

![image](/images/custom_images/zhifubaoqianbao.jpg)

<!-- more -->

探秘对象是iOS系统8.3.0版本支付宝钱包，拿到ipa文件之后，进行解压缩处理，从中我们可以看到这几类文件，png的图像、amr结尾的文件、plist文件、bundle文件等文件格式。

在其中有两个文件特别的眼熟，那就是`cordovaios.txt`和`WebViewJavascriptBridge.js.txt`文件，打开发现这两个文件没有经过改造，还是原来的配方。amr文件在[上一篇](http://codefunny.github.io/blog/2014/11/17/hybird-ios/)已经说明是可压缩文件，大家可以解压缩看看里面的内容，就发现实际上都是一堆js、html、css文件。我们后面再来探秘这一块。

现在我要去找一找传说的`AliPayBridge.js`在哪里呢，有一个`H5Service.bundle`引起了我的注意,打开显示里面的内容，在后面有几个js和plist文件，真相开始浮出水面了，打开h5_bridge.js文件，里面终于找到`window.AlipayJSBridge`的身影。

我的javascript功能还处于初级阶段，也只能大概的了解一下作用，JSAPI主要定义了几大方法，
> call、trigger、_invokeJS、_handleMessageFromObjC、_fetchQueue、loadPlugin

从名字就可以知道其用途，前面的好说，最后一个`loadPlugin`，用于载入一个或多个动态插件，采用js来实现动态加载，这个应该就是支付宝钱包最重要的一个方法了。

还有一个`h5_bridge_for_scanApp.js`，里面有一个scanAppCallFunMap的Dictionary变量，看里面定义了几个重要的方法调用。

前面提过`cordovaios.txt`，看来阿里是集众家之长，后面再去看一看了，现在我比较关心的是，如何集成这些独立的插件，还有就是如何有效的管理，实际上就是开发框架和安全问题了。

前面我也提到了阿里的arale前端框架，这个框架在11000002.amr里面看到过，在淘宝和天猫页面，我发现了他们有使用另外一个框架KISSY。每个amr里面都有一个`Manifest.xml`文件和`CERT`文件，前面应该是配置文件，后面应该是安全相关的证书文件。这一套有机体如何配合完成整个框架还不清楚，继续摸索吧。

这是zepto.js的一个小游戏，我替换了里面的data.js并修改了里面的一些参数，放在博客里面，发现前端的功能越来越好玩了，[点击去玩一玩吧](
http://codefunny.github.io/class/app/lucky.html)
