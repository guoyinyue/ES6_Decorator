



####_javascript_ require.js 源码分析

---------------------------------------
**javascript 模块化**

      js在语法上并没有支持模块化，知道ES2015 才添加Module机制。到此之前
      浏览器依然不支持Module，Babel转码的也是使用的commonJs规范。
      现在的模块实现方式:
  1)<span style="color:#6a6a6a;padding-left:15px;font-size:18px;">通过一个对象来实现模块</span>
```js
var moduleA = {
    privateVar:'private var',
    privateMethod:function(){},
    publicMehod:function(){}
 }
```
<span style="color:#606060;padding-left:20px;font-size:16px;">这种方式的弊端很明显，污染了全局变量。如果一不小心修改了moduleA的值，那后果就严重了。其次只要能获取moduleA的引用，那么久能获取模块内部所以的变量和函数，即便是私有的.</span>

2)<span style="color:#6a6a6a;padding-left:15px;font-size:18px;">通过IIFE来实现模块</span>
```js
var moduleB = (function($){
    var privateBVar = '3';
    var privateBFn = fucntion(){$('body').css({...})};
    var publicBVar = 4;
    var publicBFn = function(){};
    return {
        publicBFn:publicBFn,
        publicBVar:publicBVar
    }
    })(jquery)

```
<span style="color:#606060;padding-left:20px;font-size:16px;">这种方式可以解决模块内部私有变量被修改和引用的问题。这种方式依然存在一个污染全局变量的问题，jQuery的定义方式和上面类似，同样是给window对象注入了一个jQuery属性，然后在这样一个函数作用域里面实现自己的方法。设想当我们使用Backbone的时候，由于Backbone强依赖于underscore，jQuery或者zepto，那么在引入Backbone.js之前必须先引入jQuery。如果依赖变多，假设jQuery又依赖别的模块，管理这些依赖关系其实也是非常头疼的问题。尤其是在浏览器端，资源的下载速度也可能影响代码的执行顺序（h5添加defer async）</span>
<h3 style="font-size:20px;padding-left:5px;">AMD</h3>
<span style="color:#606060;padding-left:0px;font-size:16px;">RequireJS是一款遵循AMD规范协议的JavaScript模拟加载器.</span>
特点：异步加载、模块化、依赖管理.</br>
requirejs 源码2150行，去掉注释大概1200多行。但是这写代码运行起来的时候会随着依赖关系的复杂程度远远超过1200行。原因是因为它会存在大量检查依赖看似递归调用的函数执行。
我从自己写的一个简单requirejs调用来窥探内部代码的调用流程。</br>
```js
<script src="js/require.js" data-main="main"></script>
```
    这是requirejs的入口文件，加载完requirejs后执行代码 一共两处调用，下面是第一次初始化调用：
![1](http://owznda5oi.bkt.clouddn.com/markdown-img-paste-20171101145442850.png)
```js
var requirejs, require, define;
```
    这是requirejs定义的三个全局变量，然后定义一个立即执行函数对这三个变量进行赋值。
```js
(function (global, setTimeout) {})(this, (typeof setTimeout === 'undefined' ? undefined : setTimeout)))
```
    浏览器环境里面this自然指向了window对象。
req({});这是require的入口函数，在此之前所有的代码几乎都是函数以及变量的定义。

```js
 Determine if have config object in the call.
      if (!isArray(deps) && typeof deps !== 'string') {
          // deps is a config object
          config = deps;
          if (isArray(callback)) {
              // Adjust args if there are dependencies
              deps = callback;
              callback = errback;
              errback = optional;
          } else {
              deps = [];
          }
      }
```
这里判断deps是否为数组或者字符串，require({})一个空对象，所以它会进入这个判断里面调整参数，requre([dep1,dep2],fn)这个一般在入口回调。require('dep1')参数为字符串的情况一般出现在定义一个模块里面，兼容CMD的写法我猜，比如
```js
var dep = require('./dep.js');
```

![1](http://owznda5oi.bkt.clouddn.com/markdown-img-paste-2017110116485865.png)
初始化context参数，调用newContext函数，整个代码去掉一些工具函数也没几个函数，newContext函数独独占据了1500多行。初始化后得到context的值:

![1](http://owznda5oi.bkt.clouddn.com/markdown-img-paste-20171101175805606.png)

这边的数据结构中:
1. Module是模块构造对象，defQueue只是起到一个中转globalDefQueue的中转站。
2. defined对象是所有模块定义完成后倒入exports结果的容器，defined[mod.map.id] = mod.exports，这是一种key-vaule的形式。
3. registry对象存放的是所有Module对象，当Module对象被enable之后，会加入enalbedRegistry中。如果checkloaded执行成功，并且该Module进入到defined状态，该Module会将exports结果放到defined对象中，同时从registry、enalbedRegistry移除。
4. enabledRegistry = {}：当一个Module被enable后，会加入enabledRegistry中。
context初始完结束后紧接着执行了

```javascript
dataMain = script.getAttribute('data-main');
```

它获取了data-main的值 以及路径。
再一次执行到了 req(cfg);函数，这个时候就不一样了
![1](http://owznda5oi.bkt.clouddn.com/markdown-img-paste-20171101182659543.png)
这个时候的依赖就变成了main.js （函数的入口依赖），紧接着 执行了
```js
context.configure(config);
```
进入configure函数后调用了
```js
context.require(cfg.deps || [], cfg.callback);
```
需要去require main模块
每个localRequire()函数最后调用context.nextTick函数 异步执行的原因是什么？应该是异步去加载依赖资源。
同步代码结束，开始执行队列里面的代码，也就是context.nextTick里面的回调，回调用于异步获取模块资源。
```js
requireMod.init(deps, callback, errback, {enabled: true});
```
<span style="color:#6a6a6a;">
这时候的deps为main.js，然后调用Module类的init初始化这个模块，init方法用于设置Module实例的属性，比如设置依赖回调等等，最主要的作用在于去enable自己和它的依赖。enable函数会把自身放置到enabledRegistry对象中（换句话说当一个Module被enable后，会加入enabledRegistry中。），当模块执行完毕以后调用cleanRegistry方法清除掉数组里面对应的模块。checkloaded函数会不断的检查enabledRegistry队列，init = undefined的Module均是正在加载中的。如果发生超时之后，init还未被设置为true（在completeLoad()中设置）, 则就可以throw err了。接着会循坏该模块的依赖项，依次对依赖进行ModuleMap，然后更新module.depMaps属性，依赖deps属性由原先的字符串变成了moduleMap对象，并为该模块的depCount属性+1 ，表示它依赖的模块的数目，同时为该依赖模块注册一个defined事件和callback并绑定this为该模块。当依赖enable它自己的依赖并触发自己的factory，也就是
```js
define(modulename,deps,factory)
```
后，触发defined事件这个回调函数会调用defineDep方法，defineDep里面有
```js
this.depCount -= 1;this.depExports[i] = depExports;
```
首先让**模块本身的依赖项数目减1**，然后将依赖的输出当做参数传递给depExports，作为模块factory的参数。然后check自身，**当depCount值等于0**的时候说明模块的依赖以及全部加载执行完毕了，接下来就执行自己的factory函数。
循环的最后调用
```js
context.enable(depMap, this);
```
来enable依赖的依赖。对main.js入口依赖enable的过程和以上类似，只不过这个时候main.js的依赖是个[]对象，因此一路向下直接走到了this.check();函数，这个函数检查模块是否准备好定义自身了。check函数里面判断模块是否inited过（模块具备了什么样的条件以后才会调用init方法），如果没有，那么去加载这个模块</span>
```js
module.fetch()->module.load()->context.load()->req.load()
```
req.load()函数就是去：
1）node = req.createNode(config, moduleName, url);创建script节点。
2）为添加的节点注册load事件，node.addEventListener('load', context.onScriptLoad, false);
当main.js加载下来后开始执行其中的代码

![1](http://owznda5oi.bkt.clouddn.com/markdown-img-paste-20171102200214313.png)
**require.config({})**一个对象设置基本的内部变量后后终于迎来了第一个require(['modulec','moduleb'],function(a,b){})类型的require调用。也就是之前说的第一个参数为数组。require()函数调用最终会调用context.require(deps, callback, errback);也就是localRequire()函数调用。执行完后出发main.js的loaded事件。这边之后从chrome调试工具可以看到。基本没做什么实事，都是在触发对一些匿名模块的defined事件回调，可以不管了。然后浏览器从context.nextTick队列里对require(['modulec','moduleb'])进行了初始化。通过
```js
requireMod = getModule(makeModuleMap(null, relMap));
```
创建一个匿名模块，moduleName类似于_@r6之类的，然后对_@r6进行init，这些上面的流程其实是一样的。接着对_@r6进行enable,将_@r6加入enabledRegistry，对_@r6的依赖进行defined事件注册，对_@r6的depCount值+1，同事enable这个dep，由于这个dep也就是modulec此时还不存在依赖项，因此会一路走到check函数，check最终如上面一般
```js
module.fetch()->module.load()->context.load()->req.load()，
```
让浏览器创建一个script节点去加载，然后结束。对于另一个依赖moduleb的处理也是一样的，让浏览器创建节点去加载moduleb，结束。然后回到_@r6，继续它代码的执行，走到check函数，由于此时它的两个依赖项都还没有触发defined事件，因此_@r6的depCount是2，下面的逻辑都不会走，直接结束。等到modulec下载完毕开始执行modulec里面的代码。（下载完后执行，触发loaded事件后就调用callGetModule然后（在解析completeLoad函数的时候通过defQueue.shift();取出这个模块的依赖进行init）init这个模块,即
```js
onScriptLoad()--->completeLoad()--->callGetModule()--->init()
```
）init之后的逻辑同上，先是调用this.enable()方法，激活这个模块，然后迭代每一个依赖项，为每一个依赖项注册defined事件，并且激活依赖模块，即调用enable方法去检查依赖的依赖，因为此时依赖项都是通过Module类初始化出来的，因此deps属性均为空数组，因而直接去fetch这个依赖项 而不会深层次再去enable这个依赖的依赖。等所有的模块都enable结束在去check自身，根据一些标志flag来执行check里面的逻辑是否执行本模块的factory。
over。
嵌套很深，不易描述，下面是main.js的代码：
```javascript

require.config({
  paths:{
    "modulea":'js/a',
    "moduleb":'js/b',
    "modulec":'js/c'
  }
});

require(['modulec','moduleb'],function(a,b){
	console.log(a);
	console.log(b);
	console.log(12345);
});

```
以及moduleb的代码：
```js
define(['modulec'],function(){
    console.log("定义模块b执行了");
});
```

以及modulec的代码
```js
define(function(){
	var arg = arguments.length;
	console.log(arguments.length);
	window.a =5;
	console.log("我是模块C，我执行了");
})
```
