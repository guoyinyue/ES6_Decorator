# hybrid 和  h5 如何通信

标签（空格分隔）： hybrid h5

---

<h2 style="color:#666666">hybrid h5与native交互开发实现原理</h2>
<h3 style="color:#6a6a6a;">关于混合开发</h3>
hybrid的发家史就不谈了，关于它的使用场景和使用方式以及hybrid的基础知识我看到过叶小钗的关于这方面的文章，很具体很详细，下面就hybrid实现部分做一部分记录和探讨。叶小钗hybrid博客链接如下：
[http://www.cnblogs.com/yexiaochai/p/4921635.html][1]


  两个文件
  **bridge.js 和 c.hybrid.shell.js**
  native和h5的之间的互调又称之为桥协议。比较常见的第三方库有webViewJavaScriptBridge。原理大致相同，都是通过靠native捕获请求来实现的（http，https，file之类的请求都能被native捕获），因此这里也是用发送请求然后被native捕获。但是有一点需要记住native可以调用js代码，但是js不能调用native方法。正因为js是不能直接调用native的method所以，需要借助

     - (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType

  每次在重新定向URL的时候，这个方法就会被触发。然后根据request url来做一些操作，由于不懂OC这门奇葩的语音所以就闭着眼睛不瞎说了，这里面估计就类似于我们的switch case之类的做一个路由分发的操作。
  先说说bridge.js 因为它比较简单。这个模块的作用就是封装了一系列的工具对象。它们在页面被加载的时候就被注入到window对象上去了。
  ![此处输入图片的描述][2]


可以看到window对象多了这么多属性。
举个栗子：形如下面的代码
```js
     /**
        * @description 设置状态栏样式
        * @brief 设置状态栏样式
        * @method app_set_status_bar_style
        * @param statusBarStyle {String} 状态栏样式：lightContent/darkContent。页面默认为 lightContent。lightContent 对应为白色文字，黑色半透明底色；darkContent 为黑色文本，白色半透明底色。系统限制，样式固定，不可自定义。
        * @callback set_status_bar_style
        * @since v6.15
        * @author wei_l
        * @example
          //调用  
        CtripUtil.app_set_status_bar_style("darkContent");
        //调用完成之后，返回的数据格式
        var json_obj =
        {
            tagname:"set_status_bar_style"
        }
        app.callback(json_obj);
    */
    app_set_status_bar_style:function(statusBarStyle) {
        if (!Internal.isSupportAPIWithVersion("6.15")) {
            return;
        }

        var params = {};
        params.statusBarStyle = statusBarStyle;
        Internal.callNative("Util", "setStatusBarStyle", params, "set_status_bar_style");
    }
```
h5开发码农直接`CtripUtil.app_set_status_bar_style("darkContent");`这样就可以调用native让状态栏的顶部变成黑色。这里注意一下有一个“set_status_bar_style” 参数，下面回调的时候会用得到，稍后说到。接着细致进入代码内 发现主要是调用了callNative方法。Internal对象是一个全局对象，封装一些公共的方法，比如版本判断大小，格式化字符串，内部组装url之类的。先看看callNative的内部实现：
```js
/**
     * @brief 内部调用native API
     * @description  内部使用，调用native API
     * @param {String} moduleName 需要处理业务逻辑的模块名，对应native的ClassName
     * @param {String} nativeApiName 需要处理业务逻辑的函数名，对应native的MethodName
     * @param {JSON} params 传递给函数的业务参数
     * @param {String} tagName 业务处理完成，返回callback的名字
     * @method callWin8App
     * @since v5.3
     */
    callNative:function(moduleName, nativeApiName, params, tagName) {
        if (!moduleName || !nativeApiName) {
            alert("param Error:["+moduleName+"],["+nativeApiName+"]")
            return;
        }
        var paramString = Internal.makeParamString(moduleName, nativeApiName, params, tagName)
        if (Internal.isIOS) {
            var url = Internal.makeURLWithParam(paramString);
            Internal.loadURL(url);
        }
        else if (Internal.isAndroid){
            nativeModule = window[moduleName+"_a"];
            if (nativeModule) {
                nativeFunction = nativeModule[nativeApiName];
                if (nativeFunction) {
                    nativeFunction.call(nativeModule, paramString);
                }
            }
        }
        else if (Internal.isWinOS) {
            Internal.callWin8App(paramString);
        }
   }
```
简单，一目了然，把目光聚焦在 ` Internal.loadURL(url); ` 上面，再次进入loadUrl：
```js

/**
     * @brief 内部URL跳转
     * @description 内部隐藏iframe，做URL跳转
     * @method loadURL
     * @param {String} url 需要跳转的链接
     * @since v5.2
     */
    loadURL:function(url) {
        if (url.length == 0) {
            return;
        }
        var iframe = document.createElement("iframe");
        var cont = document.body || document.documentElement;

        iframe.style.display = "none";
        iframe.setAttribute('src', url);
        cont.appendChild(iframe);

        setTimeout(function(){
            iframe.parentNode.removeChild(iframe);
            iframe = null;
        }, 200);
    }

```
现在一切就更明朗了，通过iframe发送了一个请求，200ms后移除了这个iframe。这里是不是可以用一个空img标签发送请求然后移除img我还没有试过。毕竟它的作用只是发送一个url请求而已。而这个url请求大多是形如：

> ctrip://wireless   也就是ctrip://协议
eg. ctrip://wireless/h5?page=index.html#foods 就是美食的一个hybrid地址。

bridge.js文件就是这么easy~
下面说说c.hybrid.shell.js。这个模块也比较容易理解，刚刚我们用   `CtripUtil.app_set_status_bar_style("darkContent");` 调用了native让它给我们设置一下状态栏的颜色，如果h5开发人员需要一个回调怎么办呢？
给`app_set_status_bar_style`设置一个回调，like  app_set_status_bar_style(parames,fn)？这种形式是不行的，因为我们没办法把函数塞过去，毕竟我们是通过url来传递沟通的。c.hybrid.shell.js可以来解决这个问题。
在说hybrid.shell之前，我们还要说一下 `__bridge_callback` 这个函数。 native在实现了h5的调用之后，要通知h5，native部分功能已经完成了，那么OC会调用js的代码，之前说过js是不能直接调用native的，但是native 是可以调用js的，因为native用于UIWebView 提供的 `stringByEvaluatingJavaScriptFromString` 方法，这个方法可以把js脚本当做字符串一样执行，有点类似于JavaScript的eval函数。
```js
// Swift
webview.stringByEvaluatingJavaScriptFromString("Math.random()")
// OC
[webView stringByEvaluatingJavaScriptFromString:@"Math.random();"];
```
请native同事调试后发现native最后调用了`__bridge_callback` 方法。回过头来看看`__bridge_callback` 方法：

```js
/**
 * @brief app回调bridge.js
 * @description 将native的callback数据转换给H5页面的app.callback(JSON)
 * @method __bridge_callback
 * @param {String} param native传给H5的字符串,该字符串在app组装的时候做过URLEncode
 * @since v5.2
 * @author jimzhao
 */
function __bridge_callback(param) {
    param = CtripTool.ctripParamDecode(param);
    var jsonObj = JSON.parse(param);
    if (jsonObj != null) {
        if (jsonObj.param  && jsonObj.tagname && jsonObj.tagname == "web_view_finished_load" && jsonObj.param.platform ) {
            Internal.isInApp = true;
            Internal.appVersion = jsonObj.param.version;
            Internal.osVersion = jsonObj.param.osVersion;
            if (Internal.isWinOS) {
                window.navigator.userAgent.winPhoneUserAgent = window.navigator.userAgent+"_CtripWireless_"+Internal.appVersion;
                console = CtripConsole;                
            }
            else if (Internal.isIOS) {
                console = CtripConsole;
            }
        }
        var val = null;
        try {
            val = window.app.callback(jsonObj);
        } catch (e) {
            console.log("callback execute error:"+e);
        }
        if (Internal.isWinOS) {
            if (val) {
                val = "true";
            } else {
                val = "false";
            }
        }
        return val;
    }
    return -1;
};

```
其他代码都可以忽略，最主要的一行`window.app.callback(jsonObj);` 它调用了window对象下面的一个属性的方法。这个方法就是在hybrid.shell中定义的。它的作用类似于回调函数的容器，通过传递过来的tagname来寻找对应的callback or callbacks 。来看看w.app.callback是如何实现的吧：
```js
         W.app.callback = function(options) {
                var params, err;
                // 容错处理：某些 Hybrid 方法在回调时未严格遵守约定。
                if (typeof options == 'string') {
                    try {
                        options = JSON.parse(decodeURIComponent(options));
                    } catch (ex) {
                        return; // 老版本中此处未终止函数执行。
                    }
                }
                var tagname = options.tagname;
                params = HYBRID.fninfo(tagname).paramsMixed ? options : options.param;

                // 容错处理
                if (typeof params == 'string') {
                    try {
                        params = JSON.parse(params);
                    } catch (ex) {}
                }
                if (options.error_code) {
                    /^(\((-?\d+)\))?(.+)$/.exec(options.error_code);
                    err = new Error();
                    err.number = parseInt(RegExp.$2, 10);
                    err.message = RegExp.$3;
                }
                var sequenceId = params ? params.sequenceId : undefined;
                var upons     = _ME.fn('find', tagname, _ME.SN.UPON),
                    posts     = _ME.fn('find', tagname, _ME.SN.POST),
                    callbacks = _ME.fn('find', tagname, sequenceId );
                var abort = false;
                // 继承一个古老的容错处理。
                // @TODO 自 Lizard 2.1 起不再支持该容错处理。
                var extFn;
                if (W.Lizard && W.Lizard.version == '2.0') {
                    extFn = W.Lizard.facadeMethods;
                    if (extFn) {
                      extFn = extFn[tagname];
                    }
                    if (typeof extFn == 'function') {
                      callbacks = callbacks.concat(extFn);
                    } else {
                      extFn = undefined;
                    }
                }
                if (upons.length + callbacks.length) {
                    _.find(posts, function(fn) {
                        try { params = fn(params, err); }
                        catch (ex) { return _ME.abort(ex); }            
                    });
                    var fnTry = function(sequenceId, callback) {
                        _ME.fn('try', tagname, sequenceId, callback);
                        abort = callback(params, err) === false;
                    };
                    if (!abort) {
                      _.find(upons, function(callback) { return fnTry(_ME.SN.UPON, callback); });
                    }
                    if (!abort) {
                      _.find(callbacks, function(callback) { return fnTry(sequenceId, callback); });
                    }
                    // 如果仅有 extFn 方法时，应返回 undefined，此时不会终止 Native 默认功能的执行。
                    return (extFn && upons.length + callbacks.length == 1) ? undefined : true;
                }
            };

```
简单说说这个函数，第十一行获取了传递给这个函数对象的tagname属性，也就是上文说到的“set_status_bar_style” 调用每个native方法，在bridge桥函数中最后一个参数都是固定写死的，这个需要native和h5协商好。因为需要靠这个参数来找寻注册的回调函数数组。第四十行 通过tagname和sequenceId 寻找callbacks sequenceId 稍后再说，先看看`_ME.fn('find', tagname, sequenceId );`这个函数。
```js
        fn: (function() {
            // 存储容器
            var STORAGE = {};
            return function(action, tagname, sequenceId, callback) {
                // 参数兼容
                if (!callback && (sequenceId instanceof Function)) {
                    callback = sequenceId;
                    // delete sequenceId; // Uncaught SyntaxError: Delete of an unqualified identifier in strict mode.
                    sequenceId = undefined;  // _.noop() returns undefined
                }
                if (!sequenceId) {
                  sequenceId = _ME.SN.DEFAULT;
                }
                var storage, index, times;
                // 初始化容器。
                storage = _ME.ifHasnot(STORAGE, tagname, {});
                storage = _ME.ifHasnot(storage, sequenceId, { fns: [], times: [] });

                // 某些情况下，回调函数可能已在队列中，我们需要取得其位置。
                index = _.indexOf(storage.fns, callback);
                // 如回调函数未在队列中，则认为其当前执行次数为零。
                times = index < 0 ? 0 : storage.times[index];   
                switch (action) {
                    // 注册回调函数（可无限次使用）。
                    case 'on':
                    // 注册回调函数（仅供使用一次）。
                    case 'once':
                        if (callback) {
                            // 调整执行次数
                            times = (action == 'once') ? 1 : W.Infinity;
                            // 将回调函数在队列中的位置设定为末尾
                            if (index < 0) {
                              index = storage.fns.length;
                            }
                            storage.times[index] = times;
                            storage.fns[index] = callback;
                        }
                        return;
                    // 返回所有已注册的回调函数。
                    case 'find':
                        return storage.fns;
                    // 削减已注册回调函数的可使用次数，自动删除即将失效的回调函数。
                    case 'try':
                        // 调整执行次数
                        storage.times[index] = --times;
                        if (times === 0) { // 执行次数已消耗完毕，则取消该回调函数的注册
                            arguments[0] = 'off';
                            _ME.fn.apply(_ME, arguments);
                        }
                        return;
                    case 'off':
                        if (callback) {
                            var reject = function(arr, index) {
                                return _.union(_.first(arr, index), _.rest(arr, index+1));
                            };
                            storage.fns = reject(storage.fns, index);
                            storage.times = reject(storage.times, index);
                        }
                        else {
                            storage.fns = [];
                            storage.times = [];
                        }
                        return;
                }
            };
        }
```
_ME是一个自执行的对象，里面有一个fn方法，fn返回了一个闭包，我们就是执行这个闭包函数，find 是方法操作名，通过switch语句可以发现它通过storage对象获取callbacks。既然有获取，必然有注册，因此这边fn有一个on操作方法名，用来注册tagname对应的callbak。举个栗子，如果我要注册set_status_bar_style这个tagname的回调方法，我需要

    cHybridShell.fn("on",'set_status_bar_style',sequenceId,function(){
        console.log(“这是我的回调函数”);
    });
那么on操作做了些什么事呢，看到16 17 行代码，它把回调函数塞进了storage这个变量里面了，所以最终的数据结构就是这样的：
```js
    STORAGE={
        set_status_bar_style ：{
                sequenceId1：{
                    fns：[]
                },
                sequenceId2：{
                    fns：[]
                }
        }
    }
```
这里用到了sequenceId，这个id的左右是什么呢？

    storage = _ME.ifHasnot(storage, sequenceId, { fns: [], times: [] });
    STORAGE这个自由变量是根据 tagname和sequenceId 来唯一确定：{fns：[]} 这个storage的。如果我同一个native时间注册几个不同的函数怎么区分相应的回调函数呢，就是通过这个sequenceid 或者如果即便回调函数式同一个函数，但是是并发调用hybridApi 这时候怎么区分呢，答案依然是 通过sequenceId。


> W.app.callback 最后调用_.find() 方法一个一个地执行了回调函数。至此，hyrbid与h5的交互结束。这是最简单最通用的一个方法，通过url实现。兼容ios所有版本。苹果 从 IOS7开始 加入了JavaScriptCore 这个库来实现交互。




----------
*last word*
目前前端 实现这类库文件采用的依然是AMd的requirejs，关于requirejs的实现方式链接如下：
[requirejs实现方式解读][3]


  [1]: http://www.cnblogs.com/yexiaochai/p/4921635.html
  [2]: http://oyzj2oggn.bkt.clouddn.com/QQ%E6%88%AA%E5%9B%BE20171107105622.png
  [3]: https://zybuluo.com/archer--lhb/note/941162
