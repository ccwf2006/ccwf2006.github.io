---
layout: post
title: WKWebView 同步 Cookie
date: 2017-1-24
categories: iOS
---
iOS8 以后推出了 WKWebview，显示更快效率更高.

但是 WKWebview 有几个很坑的问题:

1. Cache 与系统分开，并且在 iOS8上面无法清除(iOS9增加了相关方法).
2. Cookie 与系统分开，web 的 cookie 与native 的 cookie 是分开的.
3. 不走 NSURLProtocol，无法自定义网络请求.

So， 需要 native 与 web 统一 cookie 就无从谈起了，甚至 webview 于 webview 之间的 cookie 同步也有问题.

# 先来解决 webview 与 webview 之间的同步问题.
个 webview 页面里登录之后,另一个 webview 依旧是未登录的状态.

这个比较容易处理,让两个 webview 使用同一个 WKProcesspool 就可以了.

```Objective-C
self.sharedProcessPool = [[WKProcessPool alloc]init];
webViewConfiguration = [[WKWebViewConfiguration alloc] init];
WKUserContentController *contentController = [[WKUserContentController alloc] init];
webViewConfiguration.userContentController = contentController;
webViewConfiguration.processPool = self.sharedProcessPool;
```

上面代码里,WKProcesspool 简单写了一下.实际使用的时候,需要把 sharedProcessPool 做成单例模式,每一个需要同步的 WKWebview 都设置这个 processPool.

但是,注意官方文档有这样的描述:
**The process pool associated with a web view is specified by its web view configuration. Each web view is given its own Web Content process until an implementation-defined process limit is reached; after that, web views with the same process pool end up sharing Web Content processes.**

implementation-defined process limit is reached 这个条件达到的时候,会发生什么?我不确定.

# 再来解决自定义 Cookie 的问题

这个服务器一般有这样的需求,在 App 中 webview 发起请求,带一个特殊的 Cookie 字段来表示来源.

这个可以在合适的时候修改 NSRequest 来解决.

cookie 这个参数,是需要设置的 Cookie 字符串,形式是类似:key1=val1;key2=val2这样子的.每一个 key val 对,对应着一个NSHttpCookie 对象.
```Objective-C
- (void) webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler{
    
    
if ([navigationAction.request allHTTPHeaderFields][@"Cookie"] && [[navigationAction.request allHTTPHeaderFields][@"Cookie"] rangeOfString:cookie].length > 0) {
        decisionHandler(WKNavigationActionPolicyAllow);
    }else{
        NSMutableURLRequest *request= [NSMutableURLRequest requestWithURL:url];
        [request setValue:cookie forHTTPHeaderField:@"Cookie"];
        [webView loadRequest:request];
         decisionHandler(WKNavigationActionPolicyCancel);
    }

}
```

可以在这里把 NSCookieStorage 里的 native 存储的 cookie 传进去.

**这种方式实质是在发送给服务器的请求中伪造了 Cookie, 服务器在某些情况下回认的,但是,  web 前端那边是肯定拿不到的这些 Cookie,当 web 前端发 ajax 请求的时候,更是不会带上这些自定义的 Cookie.**

# 打通 web 前端与 native 之间的 cookie 存储.

似乎 WKProcesspool 创建的时候,会从 NSCookieStorage 里面读取 Cookie 之前看过一段 FireFox 的源码,他们是通过替换 WKProcesspool 来做到隐私浏览器的.

但是.....为什么叫 pool 呢?里面肯定有复用和缓存.

对一个已经存在的 WebView 替换 WKProcesspool 是不会立即生效的,即使替换了,也要过一段时间,等 WebView 持有的 process 失效,再次从 WKProcesspool 取 process 的时候才生效.然而,黄花菜已经凉了...

还有一种曲线救国的方法,就是用 JS 来做 cookie 的读取与存储.

```Objective-C
KUserContentController* userContentController = WKUserContentController.new;  
WKUserScript * cookieScript = [[WKUserScript alloc]   
    initWithSource: @"document.cookie = 'TeskCookieKey1=TeskCookieValue1';document.cookie = 'TeskCookieKey2=TeskCookieValue2';"  
    injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:NO];  
// again, use stringWithFormat: in the above line to inject your values programmatically  
[userContentController addUserScript:cookieScript];  
WKWebViewConfiguration* webViewConfig = WKWebViewConfiguration.new;  
webViewConfig.userContentController = userContentController;  
WKWebView * webView = [[WKWebView alloc] initWithFrame:CGRectMake(/*set your values*/) 
```
只有部分无关痛痒的 cookie 能存取.用处并不大.

至关重要的 sessionid 这个 cookie, 早已被服务器设置了 HttpOnly 属性!然后被 WebView 保护起来,不允许通过前端的方式修改.

想要修改sessionid, 只能从 WebView 提供的接口入手,但是, WKwebview 没有这样的接口.

所以, native 与 web 的 sessionid 依然是不同步的,服务器的登录状态无法保持.

网上所谓的 UIWebView  WKWebView 和 Native 之间同步 Cookie 的方法,只要看到里面有 Javascript 代码的,都存在这个问题,根本原因就在于HTML 页面是不能通过 Javascript 代码来修改 HttpOnly 属性是 true 的 Cookie .

相信没有哪个后台会冒着 sessionid 被冒用的风险,把 sessionid 属性的 HttpOnly 设置成 false.

最后,还是换回 UIWebView 吧!

<http://stackoverflow.com/questions/25797972/cookie-sharing-between-multiple-wkwebviews>
<http://stackoverflow.com/questions/27043103/losing-cookies-in-wkwebview>