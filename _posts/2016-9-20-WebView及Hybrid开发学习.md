---
layout: post
title: "WebView及Hybrid开发学习"
date: 2016-09-20
categories:
---

>过去心不可得，现在心不可得，未来心不可得。。。

最近接触了几个开发项目，需要接入活动页，介绍页等东西，甚至些商城之类的。想想现在就算不是Hybrid的开发模式，一个应用接入这么一两个H5页面还是常有的事。所以，把自己用到的、学到的写下来，做个总结。

### 简单使用WebView

使用WebView加载网页真是超级简单，

```
	private void setWebView() {
		//记住设置WebViewClient，不然应用不认，会使用外部浏览器来打开
        webView.setWebViewClient(new WebViewClient());
        webView.loadUrl("http://www.baidu.com");
    }
```

最后记得加上权限，

```
	<uses-permission android:name="android.permission.INTERNET" />
```

就是这样，打开百度还能搜两下，只是会发现百度是精简版，有些网页还显示不大对。这是因为没有为WebView设置JavaScript的支持。

```
	WebSettings settings = webView.getSettings();
    settings.setJavaScriptEnabled(true);//设置JavaScript可用
```

现在体验算蛮好了。

某些情况下，一些静态的不常更换的页面不需要频繁的网络请求加载，可以用 **setCacheMode** 来控制，有以下几个值：

```
	LOAD_DEFAULT，默认，可以由cache-control等缓存策略控制
	LOAD_NORMAL，同上，废弃
	LOAD_CACHE_ELSE_NETWORK，如果本地有缓存就加载缓存，否则，从网络获取
	LOAD_NO_CACHE，不缓存
	LOAD_CACHE_ONLY，只使用缓存
```

单页面的话，这个样子就可以了。要是涉及到页面内的加载可能需要返回的效果：

```
	@Override
    public void onBackPressed() {
        if (webView.canGoBack()) {
            webView.goBack();
        } else {
            super.onBackPressed();
        }
    }
```

我是这样做的。。。这里说一下WebView的两个重要的Client，**WebViewClient** 和 **WebChromeClient** ：

> WebViewClient 相当于浏览器内核，处理渲染，请求等。

> WebChromeClient 处理对话框，网站图标，网站title，加载进度等，可以理解为UI控制

最后，现在明显的加载网页的页面都会写一个顶部的ProgressBar，网上主要有两种写法，一个是对WebView进行封装，一个是直接布局:

```
	//看，WebChromeClient，进度
	webView.setWebChromeClient(new WebChromeClient() {
            @Override
            public void onProgressChanged(WebView view, int newProgress) {
                if (newProgress == 100) {
                    progressBar.setVisibility(View.GONE);
                } else {
                    if (progressBar.getVisibility() == View.GONE) {
                        progressBar.setVisibility(View.VISIBLE);
                    } else {
                        progressBar.setProgress(newProgress);
                    }
                }
            }
        });
```

我这里直接布局实现的，比较简单就不说了。要说的是ProgressBar直接用不知为何不理想，我将系统ProgressBar的Drawable文件copy了一份重新设置才使高度合适，不至于两边有padding似的。

### Hybrid开发学习

具体的就参考下面第一篇参考文章，写的很详细了。平时用的少，不做什么封装，不考虑耦合，从H5调用下Native的代码还是可以的。如果要在多个地方使用，形成一套规范就得多花时间了。

```
	webView.setWebViewClient(new WebViewClient() {
         @Override
         public boolean shouldOverrideUrlLoading(WebView view, String url) {
            //解析url，如果是指定的就拦截掉
            return super.shouldOverrideUrlLoading(view, url);
         }
    });
```

嗯，简单的就直接拦截url就好啦。

### 参考

[好好和h5沟通！几种常见的hybrid通信方式](http://zjutkz.net/2016/04/17/%E5%A5%BD%E5%A5%BD%E5%92%8Ch5%E6%B2%9F%E9%80%9A%EF%BC%81%E5%87%A0%E7%A7%8D%E5%B8%B8%E8%A7%81%E7%9A%84hybrid%E9%80%9A%E4%BF%A1%E6%96%B9%E5%BC%8F/)

[浅谈Hybrid技术的设计与实现第二弹](http://www.cnblogs.com/yexiaochai/p/5524783.html#autoid-8-1-0)