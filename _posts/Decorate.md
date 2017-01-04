
### ES6_Decorator 学习
***
*好多知识不用就容易忘，写博客也是只为了帮助自己以后遗忘之时回过头来能快速想起.*
>修饰器（Decorator）是一个函数，用来修改类的行为。一言以蔽之，Decorator说到底是个函数。

值得注意的是，*Decorator修饰器* 改变类行为是**发生在代码编译时期而不是代码执行时期**，因此修饰器是不能直接作用于函数的，这是因为js存在函数提升，这个我们过会儿再说。

*Decorator修饰器* 是ES7的草案，早之前Babel已经支持这一特性，那这东西到底有啥用？

##### 修饰器最常用的几个场景大概是：
1. 绑定this指针@autobind
2. 日志功能
3. 函数节流@throttle
4. 函数去抖@debounce
5. 函数重载@override
6. 属性只读@readonly
还有混入（mixin）属性是否可枚举等等这里就不多说了，总之修饰器的用途还是比较广泛的。下面这个第三方的库很好的实现了以上功能。
传送门：[core-decorators.js](https://github.com/jayphelps/core-decorators.js)
