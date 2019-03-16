---
layout: post
author: czb1n
title:  "iOS加载本地HTML"
date:   2017-04-17 15:43:05
categories: [iOS]
tags: [iOS, WebView]
---

简介：读取本地的HTML文件来展示H5页面。

 - HTML文件会需要根据URL中不同的`Hash Tag`来显示不同的页面。例如`#!/register`显示注册页，`#!/login`显示登录页等等。
 - HTML文件还需要根据URL传入的参数请求数据。

iOS8以后，苹果推出了新框架`WebKit`。所以分别用`UIWebView`和`WKWebView`来实现看看。
以下仅当HTML文件的文件名为`index.html`。

WebView调试方法是在模拟器显示WebView之后，打开Safari的`"开发"`Tab的Simulator。

# UIWebView

``` Swift
NSString *filePath = [[NSBundle mainBundle] pathForResource:@"index" ofType:@"html"];
NSString *htmlString = [NSString stringWithContentsOfFile:filePath encoding:NSUTF8StringEncoding error:nil];
NSURL *url = [[NSURL alloc] initWithString:filePath];
[self.webView loadHTMLString:htmlString baseURL:url];
```

以上是把HTML文件读取成字符串再加载到WebView中，这时传递的URL还没有带上Hash Tag和参数。

除了加载HTML字符串以外还可以直接加载文件路径的请求：

``` Swift
NSString *filePath = [[NSBundle mainBundle] pathForResource:@"index" ofType:@"html"];
NSURL *url = [[NSURL alloc] initWithString:filePath];
NSURLRequest *request = [NSURLRequest requestWithURL:url];
[self.webView loadRequest:request];
```

那么要在URL上加上Hash Tag和传入参数很简单，只需要在URL上拼接上去就可以了。

``` Swift
NSString *component = @"#!/login?id=1&user=admin";
NSString *urlString = [filePath stringByAppendingString:component];
NSURL *url = [[NSURL alloc] initWithString:urlString];
```

但是这样有一个问题就是URL传入的时候一些特殊符号会有编码问题。

``` Swift
NSString *encodedComponent = (NSString *)CFBridgingRelease(
			CFURLCreateStringByAddingPercentEscapes(
			kCFAllocatorDefault,
			(CFStringRef)component,
			(CFStringRef)@"!$&'()*+,-./:;=?@_~%#",
			NULL,
			kCFStringEncodingUTF8
			)
			);
NSString *urlString = [filePath stringByAppendingString:component];
NSURL *url = [[NSURL alloc] initWithString:urlString];
```

到此就可以满足上述的目标了。
不过这里有一个其他问题，`UIWebView`会有内存占用非常高而且不释放导致溢出的问题，这里也顺便记录一下普遍的解决方法。

当收到系统的内存告警时，清除所有缓存的Response。

``` Swift
- (void)didReceiveMemoryWarning
{
    [super didReceiveMemoryWarning];
    
    [[NSURLCache sharedURLCache] removeAllCachedResponses];
}
```

当加载完一个链接之后，系统会把`WebKitCacheModelPreferenceKey`这个属性设为`1`，导致内存不会释放。所以实现`UIWebViewDelegate`中的`- (void)webViewDidFinishLoad:(UIWebView *)webView;`方法。在加载完一个链接之后把`WebKitCacheModelPreferenceKey`属性设为`0`，并禁用`WebKitDiskImageCacheEnabled`，`WebKitOfflineWebApplicationCacheEnabled`这两个缓存。

``` Swift
- (void)webViewDidFinishLoad:(UIWebView *)webView
{
    [[NSUserDefaults standardUserDefaults] setInteger:0 forKey:@"WebKitCacheModelPreferenceKey"];
    [[NSUserDefaults standardUserDefaults] setBool:NO forKey:@"WebKitDiskImageCacheEnabled"];
    [[NSUserDefaults standardUserDefaults] setBool:NO forKey:@"WebKitOfflineWebApplicationCacheEnabled"];
    [[NSUserDefaults standardUserDefaults] synchronize];
}
```

``` Swift
- (void)dealloc
{
    [self.webView stopLoading];
    [self.webView removeFromSuperview];
    self.webView = nil;
}
```

# WKWebView

`WKWebView`速度更快，内存占用少。使用的方法和`UIWebView`的方法差不多。

列几个关于`WKWebView`的问题：

 - 使用`WKWebView`的时候不能用上面的那种方式去拼接URL。
需要用：  
`- (nullable instancetype)initWithString:(NSString *)URLString relativeToURL:(nullable NSURL *)baseURL NS_DESIGNATED_INITIALIZER;`  
`+ (nullable instancetype)URLWithString:(NSString *)URLString relativeToURL:(nullable NSURL *)baseURL;`

``` Swift
NSURL *fileUrl = [[NSURL alloc] initWithString:filePath];
NSURL *url = [NSURL URLWithString:encodedComponent relativeToURL:fileUrl];
```

- 还有WebKit框架对跨域进行了安全性检查限制，不允许跨域。所以同样的方法可能在`UIWebView`能获取到数据，在`WKWebView`却获取不到数据。

**这个问题我暂时未想到较好的解决方法，欢迎指教。**或许可以把请求部分提到App中，得到数据之后再通过JavaScript传入HTML中。

- 看到其他资料说`WKWebView`使用`- (nullable WKNavigation *)loadRequest:(NSURLRequest *)request;`来读取本地的HTML无法读取成功，后台会出现如下的提示：Could not create a sandbox extension for /

**这个我在用模拟器测试的过程中并没有出现相应的错误。**
