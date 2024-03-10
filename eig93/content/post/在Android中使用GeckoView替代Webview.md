+++
author = "EIG93"
title = "在Android中使用GeckoView替代Webview"
date = "2024-03-10"
description = "Geckoview在Android中的简单使用"
tags = [
    "markdown",
    "css",
    "html",
]
categories = [
    "Android",
    "hybrid",
]
series = ["android-guide"]
aliases = ["android-guide"]
+++

这篇文章主要介绍了使用GeckoView时需要注意到的一些内容
<!--more-->

# 在 Android 中使用 GeckoView 替代 Webview

之前开发一个 Android 项目项目，是混合开发，因为国内默认的手机浏览器大部分都经过魔改，以至于有些功能在系统默认的 Webview 中的表现不一致。因为项目中使用到了 WebRTC 来实现语音的功能，但是在一部分厂家的手机上出现了没有声音的现象，判断是部分的系统默认的 Webview 不支持 h264 编码，为了在各个手机上有一致的体验，选择寻找 Webview 的替代，最终选择了 Geckoview(开源，Mozilla 团队开发维护)，官方的教程[教程](https://firefox-source-docs.mozilla.org/mobile/android/geckoview/consumer/geckoview-quick-start.html).

GeckoView 发布了三个版本:Stable, Beta 和 Nightly，稳定性递减。这骗文章中的例子使用的依赖版本(Stable)：

```gradle
implementation 'org.mozilla.geckoview:geckoview:115.0.20230726201356'
```

使用的版本号可以从[Maven 仓库](https://mvnrepository.com/search?q=GeckoView)中搜索,也可以直接去官方给的[地址](https://maven.mozilla.org/?prefix=maven2/org/mozilla/geckoview/)中查找自己想要的版本。
基本的使用，照着官网就可以直接打开网页了。不过 Geckoview 和我们平时使用的 Webview 还是有很大的不同。在 Webview 中很多属性我们只需要在 WebSettings 设置就可以了。而一些页面交互的处理，使用自定义的 WebChromeClient，包括全屏，权限的授权处理和文件的选择等都可以通过 WebChromeClient 提供的回调方法去解决，这块网上的资料很丰富。但是在 GeckoView 中是没有 WebChromeClient 的，而是在 GeckoSession 类中提供了很多委托接口让开发者自己去实现一些回调方法做处理。
比如为了在网页中使用麦克风我们需要添加权限，GeckoSession 提供了一个接口 PermissionDelegate，里面有各样的回调方法，其中我们需要在方法 onMediaPermissionRequest()中添加我们的权限。

```kotlin
 val permissionDelegate = object : GeckoSession.PermissionDelegate {
        override fun onAndroidPermissionsRequest(
            session: GeckoSession,
            permissions: Array<out String>?,
            callback: GeckoSession.PermissionDelegate.Callback
        ) {
            mCallback = callback
            requestAndroidPermissions(permissions)
        }

        override fun onContentPermissionRequest(
            p0: GeckoSession,
            p1: GeckoSession.PermissionDelegate.ContentPermission
        ): GeckoResult<Int>? {

            return super.onContentPermissionRequest(p0, p1)
        }

        override fun onMediaPermissionRequest(
            session: GeckoSession,
            uri: String,
            video: Array<out GeckoSession.PermissionDelegate.MediaSource>?,
            audio: Array<out GeckoSession.PermissionDelegate.MediaSource>?,
            callback: GeckoSession.PermissionDelegate.MediaCallback
        ) {

            //音频，麦克风相关权限
            var videoPermission: GeckoSession.PermissionDelegate.MediaSource? = null
            var audioPermission: GeckoSession.PermissionDelegate.MediaSource? = null

            video?.let {
                if (it.isNotEmpty()) {
                    videoPermission = it[0]
                }
            }

            audio?.let {
                if (it.isNotEmpty()) {
                    audioPermission = it[0]
                }
            }

            //注意，我们这里添加了全局的mediaCallback，为了在原生代码获取到权限之后，给Web端的权限请求授权.
            mediaCallback = callback
            requestForMediaPermissions(videoPermission, audioPermission)
        }
    }

```

实现了这样的一个接口之后，我们需要设置给 GeckoSession。

```kotlin
  mSession?.permissionDelegate = permissionDelegate
```

这样，当Web端请求打开麦克风的时候我们会在回调方法 onMediaPermissionRequest 中获取到权限参数，并通过原生代码去请求权限，等用户同意了权限之后，我们再通过 GeckoSession.PermissionDelegate.MediaCallback 去授权Web端请求的权限。

```kotlin
//这个方法在后面的代码中被调用
 fun grantMediaPermissions(videoPermission : GeckoSession.PermissionDelegate.MediaSource?,
                          audioPermission: GeckoSession.PermissionDelegate.MediaSource?) {
        mediaCallback?.grant(videoPermission, audioPermission)
 }



fun requestForMediaPermissions(videoPermission : GeckoSession.PermissionDelegate.MediaSource?,
                                    audioPermission: GeckoSession.PermissionDelegate.MediaSource?) {
    XXPermissions.with(this).permission(Permission.RECORD_AUDIO).permission(Permission.CAMERA).request(object : OnPermissionCallback {
        override fun onGranted(permissions: MutableList<String>, allGranted: Boolean) {
            //给Web端的请求授权
            grantMediaPermissions(videoPermission, audioPermission)
        }

        override fun onDenied(permissions: MutableList<String>, doNotAskAgain: Boolean) {
            if (permissions.size == 2) {
                    ToastUtil.showToast("授权失败，请先到设置页面授权本地录音和相机权限")
            }

            if (permissions.size == 1) {
                if (permissions[0] == Permission.RECORD_AUDIO) {
                        ToastUtil.showToast("请先到设置页面授权本地录音权限")
                }else if (permissions[0] == Permission.CAMERA){
                        ToastUtil.showToast("请先到设置页面授权相机权限")
                }
            }

            XXPermissions.startPermissionActivity(this@GeckoActivity, permissions)
            }

        })
    }
```

这样我们就完成了一个麦克风权限的请求并且授权的代码。比较重要的是理解 GeckoView 在一些操作上很多时候都是需要设置委托类的。再举个例子:
获取Web端页面的生命周期，在 Webview 中我们都是通过 WebViewClient 的 onPageStarted 和 onPageFinished 来判断的，但是在 GeckoView 中也是通过一个委托的实现来判断：

```kotlin
    val progressDelegate = object : ProgressDelegate {
        override fun onPageStart(session: GeckoSession, url: String) {
            super.onPageStart(session, url)
        }

        override fun onPageStop(session: GeckoSession, success: Boolean) {
            super.onPageStop(session, success)
            mHandler?.postDelayed({ hideLoading() }, 1000)

        }
    }

    //使用的时候，设置给GeckoSession
    mSession?.progressDelegate = progressDelegate
```

以上是一些设置方面的内容，而使用 Webview 我们通常都会有和原生端交互的需求。在 Webview 中还是比较简单的，直接使用注解“@JavascriptInterface”来操作，而在 GeckoView 中则有很大的不同，GeckoView 使用插件的方式。

```kotlin
       //Web端交互插件相关代码.这里是在本地做测试
        mRuntime?.webExtensionController
            ?.ensureBuiltIn("resource://android/assets/sample-ext/", "sample@mock.com")?.accept {

                var extension = it

                runOnUiThread {
                    if (extension != null) {
                        Log.e("extension", "插件安装成功")
                        //这里插件安装成功之后骂我们还是通过一个委托实现对象messageDelegate，来作为消息交互的回调,并指定一个名称"browser"，随意只要可以和Javascript中的名称对应上，Javascript代码将在文章下方展示
                        mSession?.webExtensionController?.setMessageDelegate(extension, messageDelegate, "browser")
                }
             }

        }


        //sample-page是放到本地的一个包含测试页面的文件夹，可以改成远程网页,这里需要先将项目assets文件夹下面的sample-page复制到app的私有目录,注意这个只是为了测试，真的项目网页是通过请求获取到的，可以忽略这一步

        val samplePageFolder = File(filesDir, "sample-page")
        if (!samplePageFolder.exists()) {

            // Copy sample page to app's private storage
            // This is required because there is no way to match web extension's content_scripts
            // to resource://android/assets/... and jar:file://... URLs which are used for
            // communication between page and native app (by web extension in-between).
            samplePageFolder.mkdirs()
            for (file in assets.list("sample-page")!!) {
                assets.open("sample-page/$file").copyTo(File(samplePageFolder, file).outputStream())
            }
        }

```

messageDelegate相关代码:

```kotlin
  private var mPort: WebExtension.Port? = null
   var messageDelegate = object: WebExtension.MessageDelegate {
        override fun onMessage(
            nativeApp: String,
            message: Any,
            sender: WebExtension.MessageSender
        ): GeckoResult<Any>? {

            Log.e("收到来自Web端的消息==", message.toString())
            if (message is JsonObject) {
                //判断消息格式并做消息处理
            }
            return null
        }

        override fun onConnect(port: WebExtension.Port) {
//            super.onConnect(port)
            Log.e("MessageDelegate", "onConnect");

            mPort = port;
            mPort?.setDelegate(mPortDelegate);

            var obj = JSONObject()
            obj.put("msg", "hello world")
            mPort?.postMessage(obj)

        }
    }
```
项目中插件存放的目录:

![插件目录](../../public/jscode.png "插件目录")

manifest.json文件内容(重要，插件配置相关):
```json
{
  "manifest_version": 2,
  "name": "sample-extension",
  "version": "1.0",
  "description": "Sample extension",
  "browser_specific_settings": {
    "gecko": {
      "id": "sample@mock.com"//配置一个自定义的id，kotlin代码中使用.
    }
  },
  "content_scripts": [//配置和Web端交互的js文件
    {
      "matches": [
      "<all_urls>"//所有的url都通过communicate.js交互
      ],
      "js": [
        "communicate.js"
      ],
      "run_at": "document_start"
    }
  ],
  "background": {
    "scripts": [
      "background.js"
    ]
  },
  "permissions": [//权限相关
    "nativeMessaging",
    "nativeMessagingFromContent",
    "geckoViewAddons",
    "webRequest",
    "webRequestBlocking",
    "tabs",
    "webNavigation",
    "<all_urls>"
  ]
}
```

communicate.js文件内容：

```javascript

console.log('Content:start!');

//https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/runtime/connectNative

//kotlin使用到browser
const port = browser.runtime.connectNative("browser");

//https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Sharing_objects_with_page_scripts
let JSBridge = {

    //这个方法提供给Web端发送内容到原生,Web端的Javascript代码需要去调用
    postMessage: function (message) {
        port.postMessage(message);
    }
}

window.wrappedJSObject.JSBridge = cloneInto(
    JSBridge,
    window,
    { cloneFunctions: true });

port.onMessage.addListener(message => {
    console.log("Received message from native app: " + JSON.stringify(message));

//这里是从原生代码中接收到信息并返回给Web端页面
//    port.postMessage(`Received: ${JSON.stringify(message)}`);
    if (window.wrappedJSObject && window.wrappedJSObject.webPageCallback) {
        //webPageCallback 这个方法需要在Web端页面中定义，文章往下有代码
        window.wrappedJSObject.webPageCallback( JSON.stringify(message));
    }
});
```
background.js文件我们暂时没有使用到，这一块属于插件内容，具体可以参考浏览器的插件开发.
接下来是本地测试的web页面，放在sample-page目录里面，主要是code.js
```Javascript
console.log("Hello from the sample code!");

function sendMessageToNative(message) {

    var jsonObj = {"type": message};

    jsonObj.type = message
    console.log("发送给原生端信息" + JSON.stringify(jsonObj));
    
    //这个方法在communicate.js中定义了
    window.JSBridge.postMessage(message);
}

//这个方法在communicate.js中被调用转发给原生代码
function webPageCallback(message) {
    console.log("收到原生端的信息" + message);
    document.getElementById("message").innerHTML = message;
}

```
接着是page.html：

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link rel="manifest" href="path/to/manifest.json">
    <title>Sample Extension</title>
    <script src="code.js">
    </script>
</head>
<body>
<p>Test page 测试测试</p>
<p id="message">hhh</p>

//点击按钮发送消息
<button onclick="sendMessageToNative('hello world')">SEND MESSAGE</button>
</body>
</html>
```

最后我们可以回去看一下messageDelegate的代码，结合messageDelegate,整个Javascript和原生的交互就完成了，和Webview还是有很大的不同的。

Webview中大部分原先WebViewClient和WebChromeClient的工作，都交给了GeckoSession中各种委托接口的实现，在Javascript和原生的交互中，Webview使用"@JavascriptInterface"注解来实现比较简洁，而Geckoview则使用了插件的形式，会复杂一些，另外，Geckoview最终会被打包到apk中，会增加包体积，好处是可以解决各个厂家魔改系统浏览器内核导致体验不一致带来的各种适配问题。

最后，代码中的一些内容参考了开源项目:(https://github.com/mozilla/geckoview)和(https://github.com/truefedex/GeckoViewNativeWebApp)









