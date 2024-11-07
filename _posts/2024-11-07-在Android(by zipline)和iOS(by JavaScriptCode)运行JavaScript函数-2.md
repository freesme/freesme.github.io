---
layout: post
title: 在Android中和iOS中运行JavaScript函数-2
date: 2024-11-07 23:18 +0800
categories: [TIL, Demo]
---
## zipline-android 调用JavaScript库

接上文 [在Android中运行JavaScript函数-1](https://freesme.github.io/posts/%E5%9C%A8Android%E4%B8%AD%E8%BF%90%E8%A1%8CJavaScript%E5%87%BD%E6%95%B0-1/)，以下是一个**可用**的通过Zipline调用JavaScript 库的例子：

```groovy
// 引入依赖库
implementation("app.cash.zipline:zipline-android:1.17.0")
```

```kotlin
import android.content.Context
import androidx.test.core.app.ApplicationProvider
import app.cash.zipline.Zipline
import kotlinx.coroutines.*
import kotlinx.coroutines.test.StandardTestDispatcher
import org.junit.Assert.*
import org.junit.Test

class AstrolabeTest {

    @Test
    fun testAstrolabe() = runBlocking {
        val testDispatcher = StandardTestDispatcher()

        // 创建 Zipline 实例，传入 dispatcher 参数
        val zipline = Zipline.create(dispatcher = testDispatcher)

        // 获取应用程序上下文
        val context = ApplicationProvider.getApplicationContext<Context>()

        // 定义必要的全局变量
        val initScript = """
            var self = this;
            var navigator = { userAgent: 'zipline' };
            var console = {
                log: function(message) { },
                error: function(message) { },
                warn: function(message) { },
                info: function(message) { },
                debug: function(message) { }
            };
        """.trimIndent()

        zipline.quickJs.evaluate(initScript)

        // 加载要调用的 example.min.js 文件
        val jsSource = loadJsFromAssets(context, "example.min.js")

        // 捕获评估 example.min.js 时的异常
        try {
            zipline.quickJs.evaluate(jsSource)
        } catch (e: Exception) {
            println("Error evaluating example.min.js: ${e.message}")
            fail("Failed to evaluate example.min.js: ${e.message}")
            return@runBlocking
        }

        // 检查 'foo' 是否已定义
        val fooDefined = zipline.quickJs.evaluate("typeof foo !== 'undefined';") as Boolean
        if (!fooDefined) {
            println("'foo' is not defined after evaluating foo.min.js")
            fail("'foo' is not defined after evaluating foo.min.js")
            return@runBlocking
        }

        // 定义要执行的 JavaScript 脚本
        val script = """
        JSON.stringify(foo.bar('param1', 'parma2'));
        """.trimIndent()

        // 执行脚本并获取结果字符串
        val result: String
        try {
            result = zipline.quickJs.evaluate(script) as String
        } catch (e: Exception) {
            println("Error executing script: ${e.message}")
            fail("Failed to execute script: ${e.message}")
            return@runBlocking
        }

        // 打印结果字符串
        println("Foo#bar Result String: $result")
    }

    // 从 assets 中加载 JS 文件
    private fun loadJsFromAssets(context: Context, fileName: String): String {
        return context.assets.open(fileName).bufferedReader(Charsets.UTF_8).use { it.readText() }
    }
}

```

经过排查，`Zipline`不能成功调用js库的原因是原项目webpack打包时设置`libraryTarget: 'umd'将库打包成 **通用模块定义（Universal Module Definition，UMD）** 格式，将 UMD 格式的特点是兼容多种模块系统，包括 CommonJS、AMD 和作为全局变量的方式，虽然UMD兼容性广泛，但在没有模块系统的环境下，可能无法正确地将库暴露为全局变量。

``` javascript
module.exports = {
    mode: 'none',
    entry: './src/index.ts',
    output: {
      path: path.resolve(__dirname, 'dist'), // 指定打包文件的目录
      filename: `example.min.js`, // 打包后文件的名称
      library: 'foo', // 将打包后的代码作为一个全局变量可直接调用
      libraryTarget: 'var', // 此处原本为 umd
      umdNamedDefine: true, // 为UMD模块命名
    }...
```

在 **Zipline（QuickJS 引擎）** 的环境中：

- **不支持模块系统**：Zipline 不支持 CommonJS、AMD 或 ES Modules 等模块系统。
- **UMD 包裹可能无法正确执行**：由于环境中不存在 `define`、`module.exports` 等，UMD 包裹的代码可能无法正确地将库挂载到全局对象上。
- **全局对象识别问题**：UMD 包裹在尝试将库挂载到全局对象时，无法正确识别全局对象（如 `window`、`global`、`self` 等）。

因此，当使用 `'umd'` 作为 `libraryTarget` 时，Zipline 环境无法正确加载库，导致 `foo` 变量未被定义。

将 `libraryTarget` 修改为 `'var'` 后，Webpack 会以简单的方式将库导出为全局变量：

```javascript
var foo = (function() {
  // 库代码
})();
```

这样，`foo` 变量直接在全局作用域中被定义，且不依赖任何模块系统。

---

## iOS JavaScriptCore调用JavaScript库

```swift
import XCTest
import JavaScriptCore

class AstrolabeTests: XCTestCase {
    func testAstrolabe() {
        // 创建一个 JSContext
        let context = JSContext()
        
        // 处理异常
        context?.exceptionHandler = { context, exception in
            if let exception = exception {
                print("JavaScript Exception: \(exception)")
            }
        }
        
        // 设置全局对象
        if let globalObject = context?.globalObject {
            // 定义 self
            globalObject.setValue(globalObject, forProperty: "self")
        } else {
            XCTFail("无法获取 JSContext 的全局对象")
            return
        }
        
        // 加载 example.min.js 文件
        guard let jsSourcePath = Bundle(for: type(of: self)).path(forResource: "example.min", ofType: "js") else {
            XCTFail("无法找到 example.min.js 文件")
            return
        }
        
        do {
            // 使用指定的编码方式读取文件内容
            let jsSourceContents = try String(contentsOfFile: jsSourcePath, encoding: .utf8)
            context?.evaluateScript(jsSourceContents)
        } catch {
            XCTFail("加载 JavaScript 文件时出错: \(error)")
            return
        }
        
        // 检查 foo 是否已定义
        if context?.objectForKeyedSubscript("foo").isUndefined == true {
            XCTFail("foo 未定义")
            return
        }
        
        // 调用 foo.bar 函数并返回 JSON 字符串
        let script = """
        JSON.stringify(foo.bar('1','2'));
        """
        
        if let result = context?.evaluateScript(script) {
            // 获取结果字符串
            if let jsonString = result.toString() {
                print("Result JSON String: \(jsonString)")
                } else {
                    XCTFail("无法将结果转换为数据")
                }
            } else {
                XCTFail("无法获取结果字符串")
            }
    }
}

```

iOS的调用相对容易，并且支持UMD格式打包的js文件。



