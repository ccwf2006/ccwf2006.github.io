---
layout: post
title: 使用 CGPDFDocument、CATiledLayer 和 UIPageViewController 做简单的 PDF 阅读器（一）
date: 2017-02-15
categories: iOS
---
# 0
最近项目中需要给客户看一些比较私密的文件，为了防止盗取和篡改，需要用 PDF 格式，为 app 添加了一个简单的 PDF 阅读模块。
# 打开 PDF
iOS 打开 PDF 是非常方便的事情，系统原生支持。

```
- (BOOL)openPDFFile:(NSString*)filePath{
    NSURL* pdfURL = [NSURL fileURLWithPath:filePath];
    CFURLRef cfpdfURL = CFURLCreateWithFileSystemPath(CFAllocatorGetDefault(), CFStringCreateWithCString(CFAllocatorGetDefault(), pdfURL.path.UTF8String, kCFStringEncodingUTF8), kCFURLPOSIXPathStyle, NO);
    _pdfDocument = CGPDFDocumentCreateWithURL(cfpdfURL);
    CFRelease(cfpdfURL);
    if (_pdfDocument == nil) {
        return NO;
    }
    _totalPageNum = CGPDFDocumentGetNumberOfPages(_pdfDocument);
    return YES;
}
```


```
CFStringCreateWithCString(CFAllocatorGetDefault(), pdfURL.path.UTF8String, kCFStringEncodingUTF8)
```

CGPDFDocumentCreateWithURL这里的 path，可以使用 NSURL 的fileURLWithPath:来获取，并用CFStringCreateWithCString转换。

_pdfDocument 是 CGPDFDocumentRef 类型的，在打开 PDF 文件以后，一直持有 PDF 文件的引用。

```
_totalPageNum = CGPDFDocumentGetNumberOfPages(_pdfDocument);
```

使用这个方法获取 PDF 总页数，供翻页的时候使用。

```
- (PDFContentViewController* _Nullable)showPage:(size_t)pageNum{
    if (_totalPageNum > 0 && pageNum <= _totalPageNum && pageNum > 0) {
        if (_currentPage != nil) {
            _currentPage = nil;
        }
        _currentPage = CGPDFDocumentGetPage(_pdfDocument, pageNum);
        PDFContentViewController* pdfContentViewController = [[PDFContentViewController alloc]init];
        [pdfContentViewController showPdfPage:_currentPage];
        pdfContentViewController.pageNumber = pageNum;
        return pdfContentViewController;
    }else{
        return nil;
    }
}
```

这里使用 CGPDFDocumentGetPage 来获取 PDF 文件中某一页的引用。
第一个参数是对 pdf 文件的引用，第二个参数可以直接取到某一页。这里很奇快的是，page 的下标是从1开始的，向CGPDFDocumentGetPage传入0会直接报错。

PDFContentViewController是一个显示 PDF 页面的类，每个 PDF 页面创建一个类。刚好给 UIPageViewController 做翻页效果使用。UIPageViewController 用起来很坑，这个以后再说。

# PDF文件显示
## 绘制 pdf 文件
绘制 pdf 文件跟平时绘制图片差不多

```
CGSize pdfPageOriginalPageSize = CGPDFPageGetBoxRect(_pdfPage, PDFContentDefaultPDFBox).size;
    CGSize imageSize = CGSizeMake(self.pdfContentView.bounds.size.width, self.pdfContentView.bounds.size.height);
    UIGraphicsBeginImageContext(imageSize);
    CGContextRef context = UIGraphicsGetCurrentContext();
    CGContextSetFillColorWithColor(context, self.contentBackground.CGColor);
    CGContextSetStrokeColorWithColor(context, self.contentBackground.CGColor);
    CGContextFillRect(context,CGRectMake(0, 0, imageSize.width, imageSize.height));
    CGContextSaveGState(context);
    CGContextTranslateCTM(context, 0, imageSize.height);
    CGContextScaleCTM(context, 1, -1);
    CGContextScaleCTM(context, imageSize.width/(pdfPageOriginalPageSize.width > 0?pdfPageOriginalPageSize.width:imageSize.width), imageSize.height/(pdfPageOriginalPageSize.height > 0?pdfPageOriginalPageSize.height:imageSize.height));
    CGContextDrawPDFPage(context, _pdfPage);
    CGContextRestoreGState(context);
    UIImage* thumbImg = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    self.pdfContentView.layer.contents = (__bridge id _Nullable)(thumbImg.CGImage);
```

关键是这里
CGContextDrawPDFPage(context, _pdfPage);
这里绘制的时候，会context 的 CTM 是相对于 pdf 文件来变换的。
如果不变换 CTM 就会绘制一个倒立的图像。并且，上面的代码中，pdf 页面的大小远远大于 UIGraphicsBeginImageContext(imageSize) 生成的 Image，于是就看到一个倒立的，只绘制了一部分的 pdf 文件。
这段代码是绘制了一个 Thumbnail Image，所以就没有绘制高清的图片，如果需要绘制高清的图片，例如在 7s plus 上显示的图片，至少要绘制一个 3x 于屏幕分辨率的图片，缩小后显示出来。

```
CGContextScaleCTM(context, imageSize.width/(pdfPageOriginalPageSize.width > 0?pdfPageOriginalPageSize.width:imageSize.width), imageSize.height/(pdfPageOriginalPageSize.height > 0?pdfPageOriginalPageSize.height:imageSize.height));
```

绘制的关键在这里，将整个 pdf 页面，缩放到与即将生成的 image 同样大小，就可以把整个页面都绘制到 image 上面了。

在计算 scale 的时候，也可以用

```
CGPDFPageGetDrawingTransform(_PDFPageRef, kCGPDFCropBox, self.bounds, 0, true)
```

系统会根据第二个参数传入的值，将 pdf 裁剪之后放入第三个参数传入的 Rect 里面。

```
CGSize pdfPageOriginalPageSize = CGPDFPageGetBoxRect(_pdfPage, PDFContentDefaultPDFBox)
```

这个需要说明一下，每个 pdf 页面都有一个原始大小。虽然 PDF 格式是矢量图，但是，其中的图片等内容不是矢量的，这个原始大小也就比较容易理解了。第二个参数 CGPDFBox。系统会根据传入的值来返回一个裁剪后的 pdf 页面大小

```
typedef CF_ENUM (int32_t, CGPDFBox) {
  kCGPDFMediaBox = 0,
  kCGPDFCropBox = 1,
  kCGPDFBleedBox = 2,
  kCGPDFTrimBox = 3,
  kCGPDFArtBox = 4
};
```

具体的裁剪规则，还没有研究清楚。
此外，需要知道的是，PDF 里面的每个元素都是可以读取到的，iOS 也给我们提供了相应的方法。
如果有打开 pdf 中的 url 之类的需求，可以研究一下这方面的内容。



在回到显示部分，实际浏览时，如果存在放大缩小滑动等动作，就必须重新绘图。用 UIScrollview 的 contentOffset 来计算当前正在显示的区域，并绘制到屏幕上。

但是会产生几个问题：1.如果绘制与 UIScrollView 的 ContentSize 同样大小的图片，那么，当 Scale 非常大的时候，绘制的图片也非常大，最终会内存占用过高而崩溃。
2.手动计算正在显示的区域比较困难。

好在 CATiledLayer可以帮我们解决这个难题，但是 CATiledLayer 也有一些坑。
## 使用 CATiledLayer 显示 PDF
上面的绘图代码是显示缩略图的绘制代码，由于CATiledLayer 绘制图片是异步的，在绘制过程中，会存在一块一块闪烁的情况，在 CATiledLayer 下面放一个缩略图，效果比较好。
CATiledLayer是 iOS 提供的一个专门显示大图片的类，它可以分块加载，异步加载。并且有自己的内存管理机制，可以把屏幕外的部分及时移除。
在 UIScrollview 上面使用 CATiledLayer 时需要设置
levelsOfDetail 和 levelsOfDetailBias。
levelsOfDetail = 3指的是，UIScrollview 缩放倍数（scale）在1/2 1/4 1/8时使用同样的 scale 重绘屏幕。缩小时节省内存。
levelsOfDetailBias = 3 指的是UIScrollview 缩放倍数（scale）在2 4 8时使用同样的 scale 重绘屏幕。保持清晰度。
简直就是为展示 PDF 文件生的。
只要设置CATiledLayer的 delegate，并且在

```
- (void)drawLayer:(CALayer *)layer inContext:(CGContextRef)context
```

里面绘制就可以了，但是不能手动设置CATiledLayer的 content。
这里就是 delegate 的绘制方法

```
- (void)drawLayer:(CALayer *)layer inContext:(CGContextRef)context{
        PDFContentViewController* strongSelf = self;
        double screenScale = [UIScreen mainScreen].scale;
        CGSize pdfPageOriginalPageSize = CGPDFPageGetBoxRect(strongSelf->_pdfPage, PDFContentDefaultPDFBox).size;
        CGSize imageSize = CGSizeMake(layer.bounds.size.width, layer.bounds.size.height);
        CGContextSetFillColorWithColor(context, [UIColor whiteColor].CGColor);
        CGContextSetStrokeColorWithColor(context, [UIColor whiteColor].CGColor);
        CGContextFillRect(context,CGContextGetClipBoundingBox(context));
        CGContextSaveGState(context);
        CGContextTranslateCTM(context, 0, layer.bounds.size.height);
        CGContextScaleCTM(context, 1, -1);
        
        CGContextScaleCTM(context, imageSize.width/(pdfPageOriginalPageSize.width > 0?pdfPageOriginalPageSize.width:imageSize.width), imageSize.height/(pdfPageOriginalPageSize.height > 0?pdfPageOriginalPageSize.height:imageSize.height));
        CGContextDrawPDFPage(context, strongSelf->_pdfPage);
        CGContextRestoreGState(context);
}
```

需要注意的是：context在传入这个方法时，已经做过 CTM 变换了，它的 Clip 和 scale 都可以发生了改变。
获取 scale 使用

```
CGContextGetCTM(context).a
```

获取 clip 使用

```
CGContextGetClipBoundingBox(context)
```

通常情况下，这个 context 描绘的就是当前需要绘制的块，不要改变 scale 和 clip。可以忽略这个变换，直接在这个基础上继续变换 context 即可，就好像在一个普通的 layer 绘制全屏的图像一样。

但是这个地方是多线程的，如果处理不好，可能会引用的已经释放的 self 变量，导致崩溃。



![20161122211801277-2017215](http://ol02bld35.bkt.clouddn.com/20161122211801277-2017215.png)

这本书非常好，看过 PDF 之后已入正收藏。



![20161122211953103-2017215](http://ol02bld35.bkt.clouddn.com/20161122211953103-2017215.png)

内存占用也比较小。

现在已经完成了 PDF 文件的打开和绘制一个页面，下一步就要进行缩放和翻页等操作了


[http://www.cocoachina.com/bbs/read.php?tid-48894-keyword-CATiledLayer.html](http://www.cocoachina.com/bbs/read.php?tid-48894-keyword-CATiledLayer.html)
[http://stackoverflow.com/questions/2295151/catiledlayer-drawing-crash](http://stackoverflow.com/questions/2295151/catiledlayer-drawing-crash)
[https://developer.apple.com/library/content//qa/qa1637/_index.html](https://developer.apple.com/library/content//qa/qa1637/_index.html)