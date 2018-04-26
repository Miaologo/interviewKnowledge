webview内容自适应高度

# WKWebView

> webViewDidFinishLoad代理方法被调用时，页面并不一定完全展现完成，可能有图片还未加载出来，导致此时获取的高度是并不是最终高度，过会儿图片加载出来后，浏览器会重新排版，而我们在这之前给了一个错误高度，导致显示异常

如何能在webViewDidFinishLoad之后获取到网页内容高度的变化？

答案：监听！

以下是OLEContainerScrollView的使用方式

```
[webview.scrollView addObserver:self forKeyPath:NSStringFromSelector(@selector(contentSize)) options:NSKeyValueObservingOptionOld context:KVOContext];

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    if (context == KVOContext) {
        // Initiate a layout recalculation only when a subviewʼs frame or contentSize has changed
        if ([keyPath isEqualToString:NSStringFromSelector(@selector(contentSize))]) {
            UIScrollView *scrollView = object;
            CGSize oldContentSize = [change[NSKeyValueChangeOldKey] CGSizeValue];
            CGSize newContentSize = scrollView.contentSize;
            if (!CGSizeEqualToSize(newContentSize, oldContentSize)) {
                //[self setNeedsLayout];
                //[self layoutIfNeeded];
                CGSize fittingSize = [self.webView sizeThatFits:CGSizeZero];
                self.webView.frame = CGRectMake(0, 0, fittingSize.width, fittingSize.height);
            }
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(frame))] ||
                   [keyPath isEqualToString:NSStringFromSelector(@selector(bounds))]) {
            UIView *subview = object;
            CGRect oldFrame = [change[NSKeyValueChangeOldKey] CGRectValue];
            CGRect newFrame = subview.frame;
            if (!CGRectEqualToRect(newFrame, oldFrame)) {
                [self setNeedsLayout];
                [self layoutIfNeeded];
            }
        }
    } else {
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
    }
}
```


```
- (void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation {
    CGFloat height = 0.0;
    [webView sizeToFit];
    height = webView.scrollView.ContentSize.height;
    CGRect webFrame = webView.frame;
    webFrame.size.height = height;
    webView.frame = webFrame;

}

``` 

```
- (void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation {
 [webView evaluateJavaScript:@"document.body.offsetHeight" completionHandler:^(id data, NSError * _Nullable error) {
  CGFloat height = [data floatValue];
    //ps:js可以是上面所写，也可以是document.body.scrollHeight;在WKWebView中前者offsetHeight获取自己加载的html片段，高度获取是相对准确的，但是若是加载的是原网站内容，用这个获取，会不准确，改用后者之后就可以正常显示，这个情况是我尝试了很多次方法才正常显示的
    CGRect webFrame = webView.frame;
    webFrame.size.height = height;
    webView.frame = webFrame;
}];
  
}
```

```
遍历WKWebView的所有子视图，找到中间的WKContentView,获取到它的frame设定给webView
- (void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation{
     if(webView.subViews.count > 0) {
     for(UIView *view in webView.subViews) {
     
          if([view iskindOfClass:NSClassFromString:@"WKContentView"]){
          webView.frame = view.frame;
          }
   }
   
```
>需要自己打印出来webView.subViews去观察一下，有很大不同，前两种方法是比较常用的，一般情况下能满足要求，特别一点是：我在做项目的时候发现，有时候第一次获取高度是获取不到的，这个时候我就强制在第一次获取之后隔1s再去获取一次，刷新webView，这个时候就能获取到了

# UIWebView

```
根据webview内嵌的scrollView的contentSize.height去计算高度：
-(void)webViewDidFinishLoad:(UIWebView *)webView {

    CGFloat height = 0.0;
    [webView sizeToFit];
    height = webView.scrollView.ContentSize.height;
    CGRect webFrame = webView.frame;
    webFrame.size.height = height;
    webView.frame = webFrame;
}

```

```
-(void)webViewDidFinishLoad:(UIWebView *)webView {

    CGFloat height = [[webView stringByEvaluatingJavascriptFromString:@"document.body.offsetHeight"] floatValue];
    //ps:js可以是上面所写，也可以是document.body.scrollHeight;个人觉得两者在UIWebView中都可以，但是在WKWebView中就不同了，后面会有介绍
    CGRect webFrame = webView.frame;
    webFrame.size.height = height;
    webView.frame = webFrame;
}
```

```
遍历UIWebView的所有子视图，找到中间的UIWebViewScrollView或者UIWebBrowserVeiw,获取到它的frame设定给webView
-(void)webViewDidFinishLoad:(UIWebView *)webView {
     if(webView.subViews.count > 0) {
     for(UIView *view in webView.subViews) {
    // if([view iskindOfClass:NSClassFromString:@"UIWebViewScrollView"]){
  //   webView.frame = view.frame;
   //  }
           UIScrollView *scrollView = webView.subViews[0];
          for (UIView *view in scrollView.subViews) {
          if([view iskindOfClass:NSClassFromString:@"UIWebBrowserVeiw"]){
           webView.frame = view.frame;
         }
      }
   }
 }
```


# 参考文档

https://www.jianshu.com/p/dd022c868802

