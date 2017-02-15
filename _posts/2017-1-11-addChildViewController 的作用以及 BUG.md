---
layout: post
title: addChildViewController 的作用以及 BUG
date: 2017-1-11
categories: iOS
---
苹果的文档是这样说的:

**This method creates a parent-child relationship between the current view controller and the object in thechildController parameter. This relationship is necessary when embedding the child view controller’s view into the current view controller’s content. If the new child view controller is already the child of a container view controller, it is removed from that container before being added.**
**This method is only intended to be called by an implementation of a custom container view controller. If you override this method, you must callsuper in your implementation.**

大意就是要求,如果 B VC 的 view 作为 A  VC 的 view 的 subView.那么也应该调用 addChildViewController 把B VC 设置为 A VC 的 childViewController.

这样做有几个好处:
1. 在 A VC 参与 ViewController 跳转的时候,可以把生命周期回调(viewDidLoad viewWillAppear 等)传给 B VC.
2. B VC 可以拿到 A VC 的 navigationItem 等对象,方便控制.
3. B VC 的 view 的 frame 受到 A VC 的控制.

这样一看好处不少,但是有个坑.
平时Status Bar 的高度是20px, 但是打电话或者录音的时候再进入 app,Status Bar 高度会变成40px. 这个时候,系统会自动把 UIWindow 的 rootViewController.view 下移20px.
但是不知道为什么,调用addChildViewController以后,B VC 的view 也会下移20px, 这样以来, StatusBar 高度增加了20px, 但是 B VC 累计向下移动了40px就导致 B VC 与 StatusBar 之间有一个20px 的空隙,并且通话结束后不会恢复.

所以,addChildViewController 以后,还是要手动设置一下 B VC 的 view 的 frame 来避免这个 BUG.

<http://stackoverflow.com/questions/17192005/what-does-addchildviewcontroller-actually-do>