---
layout: post
title: 吐槽 UIWebView 和 WKWebview
date: 2017-03-15
categories: iOS
---
**又是一年3.15，苹果爸爸前几天禁用了 JSPatch，就先不吐槽了，这里吐槽一下 iOS 系统上的两个 WebView--UIWebView 和 WKWebview**

# UIWebView 先天缺陷

据说想当年，UIWebView 会造成内存泄漏。这里就不吐槽了，反正我入行的时候都出 iOS7 了，什么内存泄漏之类的也没见过。

这里想说说的就是，在 Hybrid 大行其道的情况下，Native 开发中经常需要与 Web 打交道。

## UIWebView 没有 Javascript 调用 Native 的接口！

伟大的人民发明了三种方式，iFrame URL 拦截以及 iOS7 以后的 JavascriptCore。
1. iFrame 方式没怎么用过，不过在 UIWebView 里面加载 iFrame 本身就是个坑。因为，如果网页中存在 iFrame，UIWebView 会多次调用

```
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType

- (void)webViewDidStartLoad:(UIWebView *)webView

- (void)webViewDidFinishLoad:(UIWebView *)webView
```

等生命周期回调，我们滴 APP 可是需要在每次 load 新 URL 的时候清空一些 Web 设置到 Native 中的特性的，于是，区分 mainFrame 和 iFrame 就非常重要了。还好 iOS 中有这种判断方式:

```
[request.mainDocumentURL.absoluteString isEqualToString:request.URL.absoluteString]
```

判断处理一下就好了。
扯的有点远了，先不论坑不坑，在 UIWebView 里面添加 iFrame 可是会启动一个新的 Context 的，效率如何？没测试过，估计不怎么样吧。

2. 使用 URL 拦截的方式
我们目前的项目中，使用的就是 URL 拦截的方式。需要注意的就是，通过 URL 传参，通常需要把参数进行 URLEncode 和 URLDecode，否则……经常掉坑里。

```
res = [self.webViewAdapter navigationRequestFilter:request.URL withNativeMethodExecComplete:^(GZJSBridgeCallBackType callbacktype, NSString *callbackStr) {
    if (callbacktype == GZJSBridgeNoCallBack) {
                
     }
    if (callbacktype == GZJSBridgeCallBackFunc) {
        [self.webView stringByEvaluatingJavaScriptFromString:callbackStr];
    }
    if (callbacktype == GZJSBridgeCallBackUrl) {
        [self.webView loadRequest:[NSURLRequest requestWithURL:[NSURL URLWithString:callbackStr]]];
    }
    if (callbacktype == GZJSBridgeCallBackReloadPage) {
        [self.webView reload];
    }
}];
```

**这是我们的项目中在 shouldStartLoadWithRequest 对 URL 进行拦截的代码，res 是是否需要拦截 URL，callbackblock 的作用是执行完 URL 所对应的原生方法后，回调 Web 页面的方式**

3. 使用 JavascriptCore 的方式
自从 iOS7 有了 JavascriptCore Framework，这种方式就成了效率最高的方式。但是，这种方式也是有缺陷的。
首先，在 UIWebView 中获取 JavascriptCore 的实例需要用这种方法：
```
[webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"]
```
调用私有变量哎，被苹果爸爸发现就不好了。（手动捂脸）
其次，UIWebView 中每一次加载 URL 都会生成新的 JavascriptContext 实例，需要重新获取，这是很坑的，具体下面讲。

## UIWebView 难以在 Html 执行 Javascript 之前嵌入一段 Native 需要加载的 Javascript。
在项目中，Html 与 Native 交互中调用的方法是由 Native 端指定的，想要在 Html 端调用，必须把这些方法挂到浏览器 window 对象的下面。老大直接指定，Native 方法都添加到 window.nativeCommon 对象下面。
在 Android 中，可以直接指定一个 Javascript 的 Handler 供 Javascript 调用 Native 方法。实现这样的功能是非常简单的。
但是在 iOS 的 UIWebView 中，必须在 Native 端执行 stringByEvaluatingJavaScriptFromString 把方法添加到 window.nativeCommon 对象下面才行。
也就是必须这样写:

```
- (void)webViewDidFinishLoad:(UIWebView *)webView{
        [self registerJSMethod];//向 window 对象添加一些 Javascript 方法
}
```

然而，这里存在的问题是 webViewDidStartLoad 和 webViewDidFinishLoad 是根据 Html 渲染的状态来回调的，与 Html 中 Javascript 执行的状态无关。
也就是说，在 Html 开始渲染的时候，会回调 webViewDidStartLoad 这个方法，此时，window 对象不一定是当前 Html 页面的 window 对象（因为 UIWebView 初始化 window 对象可能没有完成）。
当 Html 框架渲染完成之后，会回调 webViewDidFinishLoad 这个方法，并且，在 Web 端会执行 window.onload 这个方法，开始执行 Javascript 代码。
Web 端的同事通常会在 window.onload 的时候做一些 native 调用（例如显示分享按钮），这个时候如果在 Safari 控制台调试，回报找不到 window.nativeCommon 对象，毕竟在执行 window.onload 方法的时候，Native 这边还没有把 window.nativeCommon 对象生成呢！
解决这个问题，一种是像微信那样，生成 window.nativeCommon 之后，调用 Web 页面的一个特定方法通知。
另一种是直接把 window.nativeCommon 以及这个对象中包含的一系列方法打包成一个 js 文件让 Html 页面医用一下。最后一种也是最烂的方法就是让 Html 那边写一个 setTimeout 延时调用。
因为一开始不知道这样坑，目前我们在使用最烂的方法解决问题。

如果不使用 JS 方法注入，使用 JavascriptCore 来解决 Html 调用 Native 也存在这个问题。
webViewDidStartLoad 中获取 JavaScriptContext，可能会获取到前一个页面的 JavaScriptContext，对这个强引用，还可能导致延时释放的问题。
webViewDidFinishLoad 中获取 JavaScriptContext，同样存在时机太晚的问题。
所以 UIWebView 就是这样坑。

#WKWebview 的先天缺陷
WKWebview 的问题主要在于，在 app 中使用 WKWebview，WebView 完全是在另外一个进程中运行的。注意！是另外一个进程，如果用 Xcode 的 Instruments 进行调试就可以看到，每次打开一个 WKWebview，都会启动一个 WebCore 的进程。
WKWebview 与 app 之间的通信应该是通过 IPC 通信的方式来进行的。
这样就导致了 Native 与 WKWebview 之间存在着 Cookie 同步等等一系列问题。

这种问题只能有限解决，具体可以参考这个链接<http://blog.csdn.net/tencent_bugly/article/details/54668721>

但是，我们的项目有这样的需求：打开一个 Web 页面，如果没有登录--->跳转到 Native 的登录页面---->登录完成--->回到 Web 页面。
这时候，由于 WKWebview 与 app 之间通信的问题，Cookie 是不同步的，登录状态无法传递到 Web 页面。
所以，到目前为止，项目中也没有使用 WKWebview。