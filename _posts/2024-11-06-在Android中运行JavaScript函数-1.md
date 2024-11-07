---
layout: post
title: 在Android中运行JavaScript函数-1
date: 2024-11-06 23:54 +0800
categories: [TIL, Demo]
---

## 起因

在开发任务中，为了快速实现某个功能使用到了一个开源的JavaScript库实现一些相对复杂的数据逻辑计算，Android和iOS客户端为数据的最终使用方。由于这个功能的计算逻辑受业务限制是固定的，基本确认不会有改动，在服务器开发一个接口没有太大的必要性，所以将这个库的调用的放在客户端本地实现。

## 实现

目前使用在 WebView 中加载本地 HTML 页面，并在页面加载完成后调用一个 JavaScript 函数，然后将函数的返回结果传回 Android 并进行处理，已经可以满足功能上使用这个JavaScript库：

```kotlin
package com.example.javascripttest

import android.content.Context
import android.util.Log
import android.webkit.JavascriptInterface
import android.webkit.WebChromeClient
import android.webkit.WebView
import android.webkit.WebViewClient
import androidx.test.core.app.ApplicationProvider
import androidx.test.platform.app.InstrumentationRegistry
import androidx.test.ext.junit.runners.AndroidJUnit4
import org.json.JSONObject

import org.junit.Test
import org.junit.runner.RunWith

import org.junit.Before
import java.util.concurrent.CountDownLatch
import java.util.concurrent.TimeUnit

@RunWith(AndroidJUnit4::class)
class JavaScriptRunTest {
    private lateinit var webView: WebView
    private var somethingData: String? = null
    private val latch = CountDownLatch(1)

    @Before
    fun setUp() {
        // 获取应用上下文
        val context = ApplicationProvider.getApplicationContext<Context>()
        // 初始化 WebView
        InstrumentationRegistry.getInstrumentation().runOnMainSync {
            webView = WebView(context)
            webView.settings.javaScriptEnabled = true
            webView.webChromeClient = object : WebChromeClient() {
                override fun onConsoleMessage(
                    message: String?,
                    lineNumber: Int,
                    sourceID: String?
                ) {
                    Log.d("WebView", "Console: $message")
                }
            }
            webView.webViewClient = WebViewClient()
            webView.addJavascriptInterface(JavaScriptInterface(), "Android")
        }
    }

    @Test
    fun testSomethingData() {
        InstrumentationRegistry.getInstrumentation().runOnMainSync {
            webView.loadDataWithBaseURL(
                "file:///android_asset/",
                """
                <html>
                    <head>
                        <script src="file:///android_asset/somelib.min.js"></script>
                    </head>
                    <body>
                        <script>
                            document.addEventListener('DOMContentLoaded', function() {
                                if (typeof somelib === 'undefined') {
                                    Android.sendSomethingData("Error: somelib is undefined");
                                    return;
                                }
                                try {
                                    var result = somelib.dofunc('param1', 'param2');
                                    Android.sendSomethingData(JSON.stringify(result));
                                } catch (error) {
                                    Android.sendSomethingData("Error: " + error.message);
                                }
                            });
                        </script>
                    </body>
                </html>
                """.trimIndent(),
                "text/html", "UTF-8", null
            )
        }

        // 等待 JavaScript 完成执行
        try {
            val success = latch.await(15, TimeUnit.SECONDS)
//            assertTrue(success, "Timed out waiting for JavaScript execution.")
//            assertNotNull(somethingData, "Something data should not be null")
//            assertTrue(!somethingData!!.startsWith("Error"), "JavaScript execution failed: $somethingData")
            val jsonObject = somethingData?.let { JSONObject(it) }

            println("Something JSONObject: $jsonObject")
        } catch (e: Exception) {
            e.printStackTrace()
        }
    }

    private inner class JavaScriptInterface {
        @JavascriptInterface
        fun sendSomethingData(data: String) {
            somethingData = data
            latch.countDown() // 解除等待
        }
    }
}

```

## 拓展 Zipline

在探索过程中尝试了 [cashapp/Zipline](https://github.com/cashapp/zipline) (前身 [Duktape-Android](https://github.com/cashapp/zipline#duktape-android) )，Zipline通过在Kotlin/JVM或Kotlin/Native程序中嵌入QuickJS JavaScript引擎（这是一个小而快速的JavaScript引擎）实现执行JavaScript代码，Zipline主要为高效执行嵌入式 JavaScript 而设计，特别是为了在移动设备上良好运行。它专注于在移动平台上的执行性能，并且对内存和启动时间进行了优化。适用于移动端逻辑动态更新、较高性能需求。

由于Zipline 不需要加载浏览器引擎的其他组件（HTML 解析、CSS 处理、页面渲染等），它的 JavaScript 执行速度往往比 WebView 更快，尤其是在只涉及 JavaScript 逻辑执行而不需要 UI 渲染的场景下（符合我当前的需求）。

### 简单调用示例

1.导入依赖

```groovy
implementation("app.cash.zipline:zipline:1.17.0")
```

2.在 `assets` 中创建一个JavaScript 文件形`example.js` 定义一个函数：

```javascript
function greet(name) {
    return "Hello, " + name + "!";
}
```

3.在Kotlin中调用

```kotlin
import app.cash.zipline.Zipline
import android.content.Context
import android.util.Log
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch

fun executeJavaScriptWithZipline(context: Context) {
    CoroutineScope(Dispatchers.IO).launch {
        try {
            // Initialize Zipline
            val zipline = Zipline.create()

            // Load JavaScript code from assets
            val jsCode = context.assets.open("example.js").bufferedReader().use { it.readText() }

            // Load and execute JavaScript code
            zipline.loadJsModule("example", jsCode)

            // Call the JavaScript function defined in example.js
            val result = zipline.quickJs.evaluate("greet('Zipline')")
            Log.d("ZiplineTest", "Result of greet: $result")

        } catch (e: Exception) {
            Log.e("ZiplineTest", "Error executing JavaScript", e)
        }
    }
}

```

~~在简单的JavaScript函数调用中Zipline执行并没有什么问题，当我测试业务需要使用的JS库时，可能是由于该库依赖了一些浏览器环境的一些对象，出现了一些错误，我尝试通过在 Kotlin 中注入相关定义: `val defineSelfScript = "var self = this;"` 由于时间限制的原因未能解决，最后放弃了这个方案，使用了最初的WebView方案。~~

[2024-11-07-在Android(by zipline)和iOS(by JavaScriptCode)运行JavaScript函数-2.md](https://freesme.github.io/posts/%E5%9C%A8Android(by%20zipline)%E5%92%8CiOS(by%20JavaScriptCode)%E8%BF%90%E8%A1%8CJavaScript%E5%87%BD%E6%95%B0-2/)
