#Vue.js the bug of UIwebview.MutationObserver not working exception in iOS 9.3.*

### Vue.js version
1.0.24

### Reproduction Link
https://github.com/miccycn/vue-bug-demo

### Steps to reproduce
IOS 9.3.x系统的以下浏览器中打开demo页面：

* QQ浏览器
* QQ Webview
* 支付宝webview
* 微博webview
* UC浏览器
* Opera：

执行以下动作：

* 1.在灰色区域内随意高频大幅度滑动（让滚动条有滚动）；
* 2.点击左上角的按钮


### What is Expected? What is actually happening?

点击按钮后，弹出“Right”的是Vue正常工作，弹出“Error”是Vue出现了异常。


我需在按钮上的数字要输出的是arr.length，我在这里用原生方法做了一个强制判断：

<code>this.arr.length == document.getElementById("button").textContent</code>

而在IOS 9.3.x的这些出现异常的浏览器中这时候arr.length的真实数值和按钮上显示的不一样，所以会弹“Error”。

目前已知在IOS 9.3.x以下浏览器中运行是正常的：

* 360浏览器
* firefox
* safari
* chrome
* 微信webview

在IOS9.3及以下的系统版本中所有浏览器都是正常的。


通过审查Vue的源码，发现问题出在util/env.js中全局MutationObserver的监听。

```javascript
  /* istanbul ignore if */
  if (typeof MutationObserver !== 'undefined' && !(isWechat && isIos)) {
    var counter = 1
    var observer = new MutationObserver(nextTickHandler)
    var textNode = document.createTextNode(counter)
    observer.observe(textNode, {
      characterData: true
    })
    timerFunc = function () {
      counter = (counter + 1) % 2
      textNode.data = counter
    }
  } else {
    // webpack attempts to inject a shim for setImmediate
    // if it is used as a global, so we have to work around that to
    // avoid bundling unnecessary code.
    const context = inBrowser
      ? window
      : typeof global !== 'undefined' ? global : {}
    timerFunc = context.setImmediate || setTimeout
  }
```

v1.0.24里这段不计入代码覆盖率测试的代码中，还特意对IOS的微信webview做了强制降级处理（函数的异步执行不使用MutationObserver而采用setTimeout(0)），所以在微信webview下没有这个异常。尝试取消对IOS的微信的特殊处理后，微信webview也出现了这个问题。


经过实验测试基本可以认为是浏览器底层对MutationObserver的异步执行出现了问题，在浏览器屏幕滚动或回弹的过程中如果同时让touch的一些默认事件触发，会产生对MutationObserver的阻塞，以至于MutationObserver的回调函数不去执行。造成后后面一系列的异常。


IOS 9.3对系统底层的touch、click等很多事件进行了重写，取消了300ms延迟可能会连带着在一些莫名其妙的地方带来阻塞，特别容易对一些涉及异步的原生方法产生影响。

有异常的国产浏览器的共同点是他们都是UIWebView，表现正常的浏览器（safari除外）均采用了WKWebView。

（补充：在IOS中，WKWebView是IOS8以后才有的，比UIWebView更新，性能也更好，现在还没推广开来，国内厂商大都还在用UIWebView。）


我最后提交了一个PR，改成对IOS 9.3的所有UIWebView在MutationObserver这个方法上做降级处理。（UIWebView不支持IndexedDB）

把源码中对ios微信的hack删掉了，毕竟实在太dirty……不过PR里面我对IOS9.3的判断这里我处理得也很dirty，这个与系统的UIwebview相关的bug估计以后不一定会修复，用系统版本来做判断毕竟不是长久之计。这个issue的异常就需要尤大大进一步考虑了。


如果不改Vue的源码，在这里对touch事件全部preventDefault掉，所有默认的触屏事件都用其他库来实现也是解决方法之一。到时我们的业务不允许我们preventDefault掉所有touchmove，所以只有在Vue源码中找原因了。
