# Android 混合编程：WebView 实践

**关于作者**

> 郭孝星，程序员，吉他手，主要从事 Android 平台基础架构方面的工作，欢迎交流技术方面的问题，可以去我的[Github](https://github.com/guoxiaoxing)提 issue 或者发邮件至guoxiaoxingse@163.com与我交流。

**文章目录**

- 一 基本用法
- 二 代码交互
- 三 性能优化

第一次阅览本系列文章，请参见[导读](./doc/导读.md)，更多文章请参见[文章目录](./README.md)。

## 一 基本用法

WebView 也是 Android View 的一种, 我们通常用它来在应用内部展示网页, 和以往一样, 我们先来简单看一下它的基本用法。

添加网络权限

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

在布局中添加 WebView

```xml
<?xml version="1.0" encoding="utf-8"?>
<WebView  xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/webview"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent"
/>
```

使用 WebView 加载网页

```
WebView myWebView = (WebView) findViewById(R.id.webview);
myWebView.loadUrl("http://www.example.com");
```

以上就是 WebView 的简单用法, 相比大家已经十分熟悉, 下面我们就来逐一看看 WebView 的其他特性。

### WebView 基本组件

了解了基本用法, 我们对 WebView 就有了大致的印象, 下面我们来看看构建 Web 应用的三个重要组件。

#### WebSettings

WebSettings 用来对 WebView 做各种设置, 你可以这样获取 WebSettings:

```java
WebSettings webSettings = mWebView .getSettings();
```

WebSettings 的常见设置如下所示:

JS 处理

- setJavaScriptEnabled(true); //支持 js
- setPluginsEnabled(true); //支持插件
- setJavaScriptCanOpenWindowsAutomatically(true); //支持通过 JS 打开新窗口

缩放处理

- setUseWideViewPort(true); //将图片调整到适合 webview 的大小
- setLoadWithOverviewMode(true); // 缩放至屏幕的大小
- setSupportZoom(true); //支持缩放，默认为 true。是下面那个的前提。
- setBuiltInZoomControls(true); //设置内置的缩放控件。 这个取决于 setSupportZoom(), 若 setSupportZoom(false)，则该 WebView 不可缩放，这个不管设置什么都不能缩放。
- setDisplayZoomControls(false); //隐藏原生的缩放控件

内容布局

- setLayoutAlgorithm(LayoutAlgorithm.SINGLE_COLUMN); //支持内容重新布局
- supportMultipleWindows(); //多窗口

文件缓存

- setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK); //关闭 webview 中缓存
- setAllowFileAccess(true); //设置可以访问文件

其他设置

- setNeedInitialFocus(true); //当 webview 调用 requestFocus 时为 webview 设置节点
- setLoadsImagesAutomatically(true); //支持自动加载图片
- setDefaultTextEncodingName("utf-8"); //设置编码格式
- setPluginState(PluginState.OFF); //设置是否支持 flash 插件
- setDefaultFontSize(20); //设置默认字体大小

#### WebViewClient

WebViewClient 用来帮助 WebView 处理各种通知, 请求事件。我们通过继承 WebViewClient 并重载它的方法可以实现不同功能的定制。具体如下所示:

- shouldOverrideUrlLoading(WebView view, String url) //在网页上的所有加载都经过这个方法,这个函数我们可以做很多操作。比如获取 url，查看 url.contains(“add”)，进行添加操作

- shouldOverrideKeyEvent(WebView view, KeyEvent event) //处理在浏览器中的按键事件。

- onPageStarted(WebView view, String url, Bitmap favicon) //开始载入页面时调用的，我们可以设定一个 loading 的页面，告诉用户程序在等待网络响应。

- onPageFinished(WebView view, String url) //在页面加载结束时调用, 我们可以关闭 loading 条，切换程序动作。

- onLoadResource(WebView view, String url) //在加载页面资源时会调用，每一个资源（比如图片）的加载都会调用一次。

- onReceivedError(WebView view, int errorCode, String description, String failingUrl) //报告错误信息

- doUpdateVisitedHistory(WebView view, String url, boolean isReload) //更新历史记录

- onFormResubmission(WebView view, Message dontResend, Message resend) //应用程序重新请求网页数据

- onReceivedHttpAuthRequest(WebView view, HttpAuthHandler handler, String host,String realm) //获取返回信息授权请求

- onReceivedSslError(WebView view, SslErrorHandler handler, SslError error) //让 webview 处理 https 请求。

- onScaleChanged(WebView view, float oldScale, float newScale) //WebView 发生改变时调用

- onUnhandledKeyEvent(WebView view, KeyEvent event) //Key 事件未被加载时调用

#### WebChromeClient

WebChromeClient 用来帮助 WebView 处理 JS 的对话框、网址图标、网址标题和加载进度等。同样地, 通过继承 WebChromeClient 并重载它的方法也可以实现不同功能的定制, 如下所示:

- public void onProgressChanged(WebView view, int newProgress); //获得网页的加载进度，显示在右上角的 TextView 控件中

- public void onReceivedTitle(WebView view, String title); //获取 Web 页中的 title 用来设置自己界面中的 title, 当加载出错的时候，比如无网络，这时 onReceiveTitle 中获取的标题为"找不到该网页",

- public void onReceivedIcon(WebView view, Bitmap icon); //获取 Web 页中的 icon

- public boolean onCreateWindow(WebView view, boolean isDialog, boolean isUserGesture, Message resultMsg);

- public void onCloseWindow(WebView window);

- public boolean onJsAlert(WebView view, String url, String message, JsResult result); //处理 alert 弹出框，html 弹框的一种方式

- public boolean onJsPrompt(WebView view, String url, String message, String defaultValue, JsPromptResult result) //处理 confirm 弹出框

- public boolean onJsConfirm(WebView view, String url, String message, JsResult result); //处理 prompt 弹出框

### WebView 生命周期

#### onResume()

WebView 为活跃状态时回调，可以正常执行网页的响应。

#### onPause()

WebView 被切换到后台时回调, 页面被失去焦点, 变成不可见状态，onPause 动作通知内核暂停所有的动作，比如 DOM 的解析、plugin 的执行、JavaScript 执行。

#### pauseTimers()

当应用程序被切换到后台时回调，该方法针对全应用程序的 WebView，它会暂停所有 webview 的 layout，parsing，javascripttimer。降低 CPU 功耗。

#### resumeTimers()

恢复 pauseTimers 时的动作。

#### destroy()

关闭了 Activity 时回调, WebView 调用 destory 时, WebView 仍绑定在 Activity 上.这是由于自定义 WebView 构建时传入了该 Activity 的 context 对象, 因此需要先从父
容器中移除 WebView, 然后再销毁 webview。

```java
mRootLayout.removeView(webView);
mWebView.destroy();
```

### WebView 页面导航

#### 页面跳转

当我们在 WebView 点击链接时, 默认的 WebView 会直接跳转到别的浏览器中, 如果想要实现在 WebView 内跳转就需要设置 WebViewClient, 下面我们先来
说说 WebView、WebViewClient、WebChromeClient 三者的区别。

- WebView: 主要负责解析和渲染网页
- WebViewClient: 辅助 WebView 处理各种通知和请求事件
- WebChromeClient: 辅助 WebView 处理 JavaScript 中的对话框, 网址图标和标题等

如果我们想控制不同链接的跳转方式, 我们需要继承 WebViewClient 重写 shouldOverrideUrlLoading()方法

```java
    static class CustomWebViewClient extends WebViewClient {

        private Context mContext;

        public CustomWebViewClient(Context context) {
            mContext = context;
        }

        @Override
        public boolean shouldOverrideUrlLoading(WebView view, String url) {
            if (Uri.parse(url).getHost().equals("github.com/guoxiaoxing")) {
                //如果是自己站点的链接, 则用本地WebView跳转
                return false;
            }
            //如果不是自己的站点则launch别的Activity来处理
            Intent intent = new Intent(Intent.ACTION_VIEW, Uri.parse(url));
            mContext.startActivity(intent);
            return true;
        }
    }
```

关于 shouldOverrideUrlLoading()方法的两点说明:

1 方法返回值

返回 true: Android 系统会处理 URL, 一般是唤起系统浏览器。
返回 false: 当前 WebView 处理 URL。

由于默认放回 false, 如果我们只想在 WebView 内处理链接跳转只需要设置 mWebView.setWebViewClient(new WebViewClient())即可

```java
/**
     * Give the host application a chance to take over the control when a new
     * url is about to be loaded in the current WebView. If WebViewClient is not
     * provided, by default WebView will ask Activity Manager to choose the
     * proper handler for the url. If WebViewClient is provided, return true
     * means the host application handles the url, while return false means the
     * current WebView handles the url.
     * This method is not called for requests using the POST "method".
     *
     * @param view The WebView that is initiating the callback.
     * @param url The url to be loaded.
     * @return True if the host application wants to leave the current WebView
     *         and handle the url itself, otherwise return false.
     */
    public boolean shouldOverrideUrlLoading(WebView view, String url) {
        return false;
    }
```

2 方法 deprecated 问题

shouldOverrideUrlLoading()方法在 API >= 24 时被标记 deprecated, 它的替代方法是

```
        @Override
        public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request) {
            view.loadUrl(request.toString());
            return true;
        }
```

但是 public boolean shouldOverrideUrlLoading(WebView view, String url)支持更广泛的 API 我们在使用的时候还是它,
关于这两个方法的讨论可以参见:

http://stackoverflow.com/questions/36484074/is-shouldoverrideurlloading-really-deprecated-what-can-i-use-instead  
http://stackoverflow.com/questions/26651586/difference-between-shouldoverrideurlloading-and-shouldinterceptrequest

#### 页面回退

Android 的返回键, 如果想要实现 WebView 内网页的回退, 可以重写 onKeyEvent()方法。

```java
@Override
public boolean onKeyDown(int keyCode, KeyEvent event) {
    // Check if the key event was the Back button and if there's history
    if ((keyCode == KeyEvent.KEYCODE_BACK) && myWebView.canGoBack()) {
        myWebView.goBack();
        return true;
    }
    // If it wasn't the Back key or there's no web page history, bubble up to the default
    // system behavior (probably exit the activity)
    return super.onKeyDown(keyCode, event);
}
```

#### 页面滑动

关于页面滑动, 我们在做下拉刷新等功能时, 经常会去判断 WebView 是否滚动到顶部或者滚动到底部。

我们先来看一看三个判断高度的方法

```java
getScrollY();
```

该方法返回的是当前可见区域的顶端距整个页面顶端的距离,也就是当前内容滚动的距离.

```java
getHeight();
getBottom();
```

该方法都返回当前 WebView 这个容器的高度

```
getContentHeight();
```

返回的是整个 html 的高度, 但并不等同于当前整个页面的高度, 因为 WebView 有缩放功能, 所以当前整个页面的高度实际上应该是原始 html 的高度
再乘上缩放比例. 因此, 判断方法是:

```java
if (webView.getContentHeight() * webView.getScale() == (webView.getHeight() + webView.getScrollY())) {
    //已经处于底端
}

if(webView.getScrollY() == 0){
    //处于顶端
}
```

以上这个方法也是我们常用的方法, 不过从 API 17 开始, mWebView.getScale()被标记为 deprecated

> This method was deprecated in API level 17. This method is prone to inaccuracy due to race conditions
> between the web rendering and UI threads; prefer onScaleChanged(WebView,

因为 scale 的获取可以用一下方式:

```java
public class CustomWebView extends WebView {

public CustomWebView(Context context) {
    super(context);
    setWebViewClient(new WebViewClient() {
        @Override
        public void onScaleChanged(WebView view, float oldScale, float newScale) {
            super.onScaleChanged(view, oldScale, newScale);
            mCurrentScale = newScale
        }
    });
}
```

关于 mWebView.getScale()的讨论可以参见:

https://developer.android.com/reference/android/webkit/WebView.html

http://stackoverflow.com/questions/16079863/how-get-webview-scale-in-android-4

### WebView 缓存实现

在项目中如果使用到 WebView 控件, 当加载 html 页面时, 会在/data/data/包名目录下生成 database 与 cache 两个文件夹。
请求的 url 记录是保存在 WebViewCache.db, 而 url 的内容是保存在 WebViewCache 文件夹下。

控制缓存行为

```java
WebSettings webSettings = mWebView.getSettings();
//优先使用缓存
webSettings.setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK);
//只在缓存中读取
webSettings.setCacheMode(WebSettings.LOAD_CACHE_ONLY);
/不使用缓存
WwebSettings.setCacheMode(WebSettings.LOAD_NO_CACHE);
```

清除缓存

```java
clearCache(true); //清除网页访问留下的缓存，由于内核缓存是全局的因此这个方法不仅仅针对webview而是针对整个应用程序.
clearHistory (); //清除当前webview访问的历史记录，只会webview访问历史记录里的所有记录除了当前访问记录.
clearFormData () //这个api仅仅清除自动完成填充的表单数据，并不会清除WebView存储到本地的数据。
```

### WebView Cookies

添加 Cookies

```java
public void synCookies() {
    if (!CacheUtils.isLogin(this)) return;
    CookieSyncManager.createInstance(this);
    CookieManager cookieManager = CookieManager.getInstance();
    cookieManager.setAcceptCookie(true);
    cookieManager.removeSessionCookie();//移除
    String cookies = PreferenceHelper.readString(this, AppConfig.COOKIE_KEY, AppConfig.COOKIE_KEY);
    KJLoger.debug(cookies);
    cookieManager.setCookie(url, cookies);
    CookieSyncManager.getInstance().sync();
}
```

清除 Cookies

```java
CookieManager.getInstance().removeSessionCookie();
```

### WebView 本地资源访问

当我们在 WebView 中加载出从 web 服务器上拿取的内容时，是无法访问本地资源的，如 assets 目录下的图片资源，因为这样的行为属于跨域行为（Cross-Domain），而 WebView 是禁止
的。解决这个问题的方案是把 html 内容先下载到本地，然后使用 loadDataWithBaseURL 加载 html。这样就可以在 html 中使用 file:///android_asset/xxx.png 的链接来引用包里
面 assets 下的资源了。

```java
private void loadWithAccessLocal(final String htmlUrl) {
    new Thread(new Runnable() {
        public void run() {
            try {
                final String htmlStr = NetService.fetchHtml(htmlUrl);
                if (htmlStr != null) {
                    TaskExecutor.runTaskOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            loadDataWithBaseURL(htmlUrl, htmlStr, "text/html", "UTF-8", "");
                        }
                    });
                    return;
                }
            } catch (Exception e) {
                Log.e("Exception:" + e.getMessage());
            }

            TaskExecutor.runTaskOnUiThread(new Runnable() {
                @Override
                public void run() {
                    onPageLoadedError(-1, "fetch html failed");
                }
            });
        }
    }).start();
}
```

**注意**

- 从网络上下载 html 的过程应放在工作线程中
- html 下载成功后渲染出 html 的步骤应放在 UI 主线程，不然 WebView 会报错
- html 下载失败则可以使用我们前面讲述的方法来显示自定义错误界面

## 二 代码交互

### Android 原生方案

关于 WebView 中 Java 代码和 JS 代码的交互实现, Android 给了一套原生的方案, 我们先来看看原生的用法。后面我们还会讲到其他的开源方法。

JavaScript 代码和 Android 代码是通过 addJavascriptInterface()来建立连接的, 我们来看下具体的用法。

1 设置 WebView 支持 JavaScript

```java
webView.getSettings().setJavaScriptEnabled(true);
```

2 在 Android 工程里定义一个接口

```java
public class WebAppInterface {
    Context mContext;

    /** Instantiate the interface and set the context */
    WebAppInterface(Context c) {
        mContext = c;
    }

    /** Show a toast from the web page */
    @JavascriptInterface
    public void showToast(String toast) {
        Toast.makeText(mContext, toast, Toast.LENGTH_SHORT).show();
    }
}
```

**注意**: API >= 17 时, 必须在被 JavaScript 调用的 Android 方法前添加@JavascriptInterface 注解, 否则将无法识别。

3 在 Android 代码中将该接口添加到 WebView

```java
WebView webView = (WebView) findViewById(R.id.webview);
webView.addJavascriptInterface(new WebAppInterface(this), "Android");
```

这个"Android"就是我们为这个接口取的别名, 在 JavaScript 就可以通过 Android.showToast(toast)这种方式来调用此方法。

4 在 JavaScript 中调用 Android 方法

```js
<input type="button" value="Say hello" onClick="showAndroidToast('Hello Android!')" />

<script type="text/javascript">
    function showAndroidToast(toast) {
        Android.showToast(toast);
    }
</script>
```

在 JavaScript 中我们不用再去实例化 WebAppInterface 接口, WebView 会自动帮我们完成这一工作, 使它能够为 WebPage 所用。

**注意**:

由于 addJavascriptInterface()给予了 JS 代码控制应用的能力, 这是一项非常有用的特性, 但同时也带来了安全上的隐患,

> Using addJavascriptInterface() allows JavaScript to control your Android application. This can be a very useful feature or a dangerous
> security issue. When the HTML in the WebView is untrustworthy (for example, part or all of the HTML is provided by an unknown person or
> process), then an attacker can include HTML that executes your client-side code and possibly any code of the attacker's choosing. As such,
> you should not use addJavascriptInterface() unless you wrote all of the HTML and JavaScript that appears in your WebView. You should also
> not allow the user to navigate to other web pages that are not your own, within your WebView (instead, allow the user's default browser
> application to open foreign links—by default, the user's web browser opens all URL links, so be careful only if you handle page navigation
> as described in the following section).

下面正式引入我们在项目中常用的两套开源的替代方案

### jockeyjs 开源方案

[jockeyjs](https://github.com/tcoulter/jockeyjs)是一套 IOS/Android 双平台的 Native 和 JS 交互方法, 比较适合用在项目中。

> Library to facilitate communication between iOS apps and JS apps running inside a UIWebView

jockeyjs 对 Native 和 JS 的交互做了优美的封装, 事件的发送与接收都可以通过 send()和 on()来完成。我们先简单的看一下 Event 的发送与接收。

Sending events from app to JavaScript

```java
// Send an event to JavaScript, passing a payload
jockey.send("event-name", webView, payload);

//With a callback to execute after all listeners have finished
jockey.send("event-name", webView, payload, new JockeyCallback() {
    @Override
    public void call() {
        //Your execution code
    }
});
```

Receiving events from app in JavaScript

```java
// Listen for an event from iOS, but don't notify iOS we've completed processing
// until an asynchronous function has finished (in this case a timeout).
Jockey.on("event-name", function(payload, complete) {
  // Example of event'ed handler.
  setTimeout(function() {
    alert("Timeout over!");
    complete();
  }, 1000);
});
```

Sending events from JavaScript to app

```java
// Send an event to iOS.
Jockey.send("event-name");

// Send an event to iOS, passing an optional payload.
Jockey.send("event-name", {
  key: "value"
});

// Send an event to iOS, pass an optional payload, and catch the callback when all the
// iOS listeners have finished processing.
Jockey.send("event-name", {
  key: "value"
}, function() {
  alert("iOS has finished processing!");
});
```

Receiving events from JavaScript in app

```java
//Listen for an event from JavaScript and log a message when we have receied it.
jockey.on("event-name", new JockeyHandler() {
    @Override
    protected void doPerform(Map<Object, Object> payload) {
        Log.d("jockey", "Things are happening");
    }
});

//Listen for an event from JavaScript, but don't notify the JavaScript that the listener has completed
//until an asynchronous function has finished
//Note: Because this method is executed in the background, if you want the method to interact with the UI thread
//it will need to use something like a android.os.Handler to post to the UI thread.
jockey.on("event-name", new JockeyAsyncHandler() {
    @Override
    protected void doPerform(Map<Object, Object> payload) {
        //Do something asynchronously
        //No need to called completed(), Jockey will take care of that for you!
    }
});


//We can even chain together several handlers so that they get processed in sequence.
//Here we also see an example of the NativeOS interface which allows us to chain some common
//system handlers to simulate native UI interactions.
jockey.on("event-name", nativeOS(this)
            .toast("Event occurred!")
            .vibrate(100), //Don't forget to grant permission
            new JockeyHandler() {
                @Override
                protected void doPerform(Map<Object, Object> payload) {
                }
            }
);

//...More Handlers


//If you would like to stop listening for a specific event
jockey.off("event-name");

//If you would like to stop listening to ALL events
jockey.clear();
```

通过上面的代码, 我们对 jockeyjs 的使用有了大致的理解, 下面我们具体来看一下在项目中的使用。

1 依赖配置

下载代码: https://github.com/tcoulter/jockeyjs, 将 JockeyJS.Android 导入到工程中。

2 jockeyjs 配置

jockeyjs 有两种使用方式

方式一:

只在一个 Activity 中使用 jockey 或者多 Activity 共享一个 jockey 实例

```java
//Declare an instance of Jockey
Jockey jockey;

//The WebView that we will be using, assumed to be instantiated either through findViewById or some method of injection.
WebView webView;

WebViewClient myWebViewClient;

@Override
protected void onStart() {
    super.onStart();

    //Get the default JockeyImpl
    jockey = JockeyImpl.getDefault();

    //Configure your webView to be used with Jockey
    jockey.configure(webView);

    //Pass Jockey your custom WebViewClient
    //Notice we can do this even after our webView has been configured.
    jockey.setWebViewClient(myWebViewClient)

    //Set some event handlers
    setJockeyEvents();

    //Load your webPage
    webView.loadUrl("file:///your.url.com");
}
```

方式二:

另一种就是把 jockey 当成一种全局的 Service 来用, 这种方式下我们可以在多个 Activity 之间甚至整个应用内共享 handler. 当然我们同样需要
把 jockey 的生命周期和应用的生命周期绑定在一起。

```java
//First we declare the members involved in using Jockey

//A WebView to interact with
private WebView webView;

//Our instance of the Jockey interface
private Jockey jockey;

//A helper for binding services
private boolean _bound;

//A service connection for making use of the JockeyService
private ServiceConnection _connection = new ServiceConnection() {
    @Override
    public void onServiceDisconnected(ComponentName name) {
        _bound = false;
    }

    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
        JockeyBinder binder = (JockeyBinder) service;

        //Retrieves the instance of the JockeyService from the binder
        jockey = binder.getService();

        //This will setup the WebView to enable JavaScript execution and provide a custom JockeyWebViewClient
        jockey.configure(webView);

        //Make Jockey start listening for events
        setJockeyEvents();

        _bound = true;

        //Redirect the WebView to your webpage.
        webView.loadUrl("file:///android_assets/index.html");
    }

}

///....Other member variables....////


//Then we bind the JockeyService to our activity through a helper function in our onStart method
@Override
protected void onStart() {
    super.onStart();
    JockeyService.bind(this, _connection);
}

//In order to bind this with the Android lifecycle we need to make sure that the service also shuts down at the appropriate time.
@Override
protected void onStop() {
    super.onStop();
    if (_bound) {
        JockeyService.unbind(this, _connection);
    }
}
```

以上便是 jockeyjs 的大致用法.

## 三 性能优化

### 优化网页加载速度

默认情况 html 代码下载到 WebView 后，webkit 开始解析网页各个节点，发现有外部样式文件或者外部脚本文件时，会异步发起网络请求下载文件，但如果
在这之前也有解析到 image 节点，那势必也会发起网络请求下载相应的图片。在网络情况较差的情况下，过多的网络请求就会造成带宽紧张，影响到 css 或
js 文件加载完成的时间，造成页面空白 loading 过久。解决的方法就是告诉 WebView 先不要自动加载图片，等页面 finish 后再发起图片加载。

设置 WebView, 先禁止加载图片

```java
WebSettings webSettings = mWebView.getSettings();

//图片加载
if(Build.VERSION.SDK_INT >= 19){
    webSettings.setLoadsImagesAutomatically(true);
}else {
    webSettings.setLoadsImagesAutomatically(false);
}
```

覆写 WebViewClient 的 onPageFinished()方法, 页面加载结束后再加载图片

```java
@Override
public void onPageFinished(WebView view, String url) {
    super.onPageFinished(view, url);
    if (!view.getSettings().getLoadsImagesAutomatically()) {
        view.getSettings().setLoadsImagesAutomatically(true);
    }
}
```

**注意**: 4.4 以上系统在 onPageFinished 时再恢复图片加载时,如果存在多张图片引用的是相同的 src 时，会只有一个 image 标签得到加载，因而对于这样的系统我们就先直接加载。

### 硬件加速页面闪烁问题

4.0 以上的系统我们开启硬件加速后，WebView 渲染页面更加快速，拖动也更加顺滑。但有个副作用就是，当 WebView 视图被整体遮住一块，然后突然恢复时（比如使用 SlideMenu 将 WebView 从侧边
滑出来时），这个过渡期会出现白块同时界面闪烁。解决这个问题的方法是在过渡期前将 WebView 的硬件加速临时关闭，过渡期后再开启，如下所示:

过度前关闭硬件加速

```java
if(Build.VERSION.SDK_INT > Build.VERSION_CODES.HONEYCOMB){
    mWebView.setLayerType(View.LAYER_TYPE_SOFTWARE, null);
}
```

过度前开启硬件加速

```java
if(Build.VERSION.SDK_INT > Build.VERSION_CODES.HONEYCOMB){
    mWebView.setLayerType(View.LAYER_TYPE_HARDWARE, null);
}
```

以上就是本篇文章的全部内容, 大致就说这么多, 在实际的项目中我们通常会自己去封装一个 H5Activity 用来统一显示 H5 页面, 下面就提供了完整的 H5Activity, 封装了 WebView 各种特性与 jockeyjs 代码交互。

该 H5Activity 提供 WebView 常用设置、H5 页面解析、标题解析、进度条显示、错误页面展示、重新加载等功能。可以拿去稍作改造, 用于自己的项目中。

```java
package com.guoxiaoxing.webview;

import android.content.Context;
import android.graphics.Bitmap;
import android.net.ConnectivityManager;
import android.net.NetworkInfo;
import android.os.Build;
import android.os.Bundle;
import android.support.v7.app.AppCompatActivity;
import android.support.v7.widget.Toolbar;
import android.text.TextUtils;
import android.util.Log;
import android.view.KeyEvent;
import android.view.View;
import android.view.Window;
import android.webkit.JsResult;
import android.webkit.WebChromeClient;
import android.webkit.WebResourceError;
import android.webkit.WebResourceRequest;
import android.webkit.WebSettings;
import android.webkit.WebView;
import android.webkit.WebViewClient;
import android.widget.ProgressBar;

import com.jockeyjs.Jockey;
import com.jockeyjs.JockeyImpl;

public class H5Activity extends AppCompatActivity {

    public static final String H5_URL = "H5_URL";
    private static final String JOCKEY_EVENT_NAME = "JOCKEY_EVENT_NAME";
    private static final String TAG = H5Activity.class.getSimpleName();

    private Toolbar mToolbar;
    private ProgressBar mProgressBar;

    private Jockey mJockey;
    private WebView mWebView;
    private WebViewClient mWebViewClient;
    private WebChromeClient mWebChromeClient;

    private String mUrl;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        supportRequestWindowFeature(Window.FEATURE_NO_TITLE);
        setContentView(R.layout.activity_h5);
        setupView();
        setupSettings();
    }

    @Override
    protected void onStart() {
        super.onStart();
        setupJockey();
        setupData();
    }

    private void setupView() {
        mToolbar = (Toolbar) findViewById(R.id.h5_toolbar);
        mProgressBar = (ProgressBar) findViewById(R.id.h5_progressbar);
        mWebView = (WebView) findViewById(R.id.h5_webview);
    }

    private void setupSettings() {

        mWebView.setScrollBarStyle(WebView.SCROLLBARS_INSIDE_OVERLAY);
        mWebView.setHorizontalScrollBarEnabled(false);
        mWebView.setOverScrollMode(WebView.OVER_SCROLL_NEVER);

        WebSettings mWebSettings = mWebView.getSettings();
        mWebSettings.setSupportZoom(true);
        mWebSettings.setLoadWithOverviewMode(true);
        mWebSettings.setUseWideViewPort(true);
        mWebSettings.setDefaultTextEncodingName("utf-8");
        mWebSettings.setLoadsImagesAutomatically(true);

        //JS
        mWebSettings.setJavaScriptEnabled(true);
        mWebSettings.setJavaScriptCanOpenWindowsAutomatically(true);

        mWebSettings.setAllowFileAccess(true);
        mWebSettings.setUseWideViewPort(true);
        mWebSettings.setDatabaseEnabled(true);
        mWebSettings.setLoadWithOverviewMode(true);
        mWebSettings.setDomStorageEnabled(true);


        //缓存
        ConnectivityManager connectivityManager = (ConnectivityManager) this.getSystemService(Context.CONNECTIVITY_SERVICE);
        NetworkInfo info = connectivityManager.getActiveNetworkInfo();
        if (info != null && info.isConnected()) {
            String wvcc = info.getTypeName();
            Log.d(TAG, "current network: " + wvcc);
            mWebSettings.setCacheMode(WebSettings.LOAD_DEFAULT);
        } else {
            Log.d(TAG, "No network is connected, use cache");
            mWebSettings.setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK);
        }

        if (Build.VERSION.SDK_INT >= 16) {
            mWebSettings.setAllowFileAccessFromFileURLs(true);
            mWebSettings.setAllowUniversalAccessFromFileURLs(true);
        }

        if (Build.VERSION.SDK_INT >= 12) {
            mWebSettings.setAllowContentAccess(true);
        }

        setupWebViewClient();
        setupWebChromeClient();
    }

    private void setupJockey() {
        mJockey = JockeyImpl.getDefault();
        mJockey.configure(mWebView);
        mJockey.setWebViewClient(mWebViewClient);
        mJockey.setOnValidateListener(new Jockey.OnValidateListener() {
            @Override
            public boolean validate(String host) {
                return "yourdomain.com".equals(host);
            }
        });

        //TODO set your event handler
        mJockey.on(JOCKEY_EVENT_NAME, new EventHandler());
    }

    private void setupData() {
        mUrl = getIntent().getStringExtra(H5_URL);
        if (TextUtils.isEmpty(mUrl)) {
            //TODO show error page
        } else {
            mWebView.loadUrl(mUrl);
        }
    }

    private void setupWebViewClient() {
        mWebViewClient = new WebViewClient() {
            @Override
            public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request) {
                //TODO 处理URL, 例如对指定的URL做不同的处理等
                return false;
            }

            @Override
            public void onPageFinished(WebView view, String url) {
                super.onPageFinished(view, url);
            }

            @Override
            public void onPageStarted(WebView view, String url, Bitmap favicon) {
                super.onPageStarted(view, url, favicon);
            }

            @Override
            public void onReceivedError(WebView view, WebResourceRequest request, WebResourceError error) {
                super.onReceivedError(view, request, error);
            }
        };
        mWebView.setWebViewClient(mWebViewClient);
    }

    private void setupWebChromeClient() {
        mWebChromeClient = new WebChromeClient() {
            @Override
            public void onReceivedTitle(WebView view, String title) {
                super.onReceivedTitle(view, title);
                mToolbar.setTitle(title);

            }

            @Override
            public void onProgressChanged(WebView view, int newProgress) {
                super.onProgressChanged(view, newProgress);
                mProgressBar.setProgress(newProgress);
                if (newProgress == 100) {
                    mProgressBar.setVisibility(View.GONE);
                } else {
                    mProgressBar.setVisibility(View.VISIBLE);
                }
            }

            @Override
            public boolean onJsAlert(WebView view, String url, String message, JsResult result) {
                return super.onJsAlert(view, url, message, result);
            }
        };
        mWebView.setWebChromeClient(mWebChromeClient);
    }

    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        if ((keyCode == KeyEvent.KEYCODE_BACK) && mWebView.canGoBack()) {
            mWebView.goBack();
            return true;
        }
        return super.onKeyDown(keyCode, event);
    }
}
```
