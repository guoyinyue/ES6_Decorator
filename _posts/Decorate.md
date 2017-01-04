
### ES6_Decorator 学习
***
*好多知识不用就容易忘，写博客也是只为了帮助自己以后遗忘之时回过头来能快速想起.*
>修饰器（Decorator）是一个函数，用来修改类的行为。一言以蔽之，Decorator说到底是个函数。

值得注意的是，*Decorator修饰器* 改变类行为是**发生在代码编译时期而不是代码执行时期**，因此修饰器是不能直接作用于函数的，这是因为js存在函数提升，而用在类上面就不会出现这种情况。

*Decorator修饰器* 是ES7的草案，早之前Babel已经支持这一特性，那这东西到底有啥用？

##### 修饰器最常用的几个场景大概是
1. 绑定this指针@autobind
2. 日志功能
3. 函数节流@throttle
4. 函数去抖@debounce
5. 函数重载@override
6. 属性只读@readonly
还有混入（mixin）属性是否可枚举等等这里就不多说了，总之修饰器的用途还是比较广泛的。下面这个第三方的库很好的实现了以上功能。
传送门：[core-decorators.js](https://github.com/jayphelps/core-decorators.js)

##### Decorator修饰器用法
<h7 style="color:#6a6a6a;">下面我们来看一段代码:</h7>
```js
  function decoratorable(target){
    target.addNewAttribute = true;
  }
  @decoratorable
  class MyTestableClass {}
  console.log(MyTestableClass.addNewAttribute) // true
```
这段代码很简单，*decoratorable修饰器* 给MyTestableClass类添加了一个静态属性，修饰器的第一个参数target是它修饰的类的类名。如果需要给修饰器添加额外的参数怎么办呢，只要在这个基础上再包装一层就好了。
```js
function decoratorable(isTestable) {
  return function(target) {
    target.addNewAttribute = isTestable;
  }
}
  @decoratorable(false)
  class MyTestableClass {}
  console.log(MyTestableClass.addNewAttribute) // false

```
*decoratorable修饰器* 也可以直接添加在类的某一个方法或者属性上面，我们接着看
```js
  function log(target, name, descriptor) {
    var oldValue = descriptor.value;

    descriptor.value = function() {
      console.log(`Calling "${name}" with`, arguments);
      return oldValue.apply(null, arguments);
    };

    return descriptor;
  }
  class Math {
    @log
    add(a, b) {
      return a + b;
    }
  }
  const math = new Math();
  math.add(3,5);
```
执行完add（）方法，我们会在控制台收到一段日志，这也是上面说的常用的场景之一日志功能，这有点类似于中间件的感觉，在调用一些函数方法之前做一些预备工作，deprecate 函数的功能也是类似的。
还有一个值得一提的是，修饰器可以实现自动发布事件，这是个比较有意思的实现:
```js
import postal from "postal/lib/postal.lodash";
export default function publish(topic, channel) {
  //包装一层函数用于传递额外的参数
  return function(target, name, descriptor) {
    const fn = descriptor.value;
    //获得原来的函数，并保存在fn变量里面，以便在新的函数里面通过该变量调用原来的函数，其实就是包装一下原来的函数
    descriptor.value = function() {
      let value = fn.apply(this, arguments);
      //函数执行的时候发布一个主题，那么订阅这个主题的所有订阅者都会得到相应调用自身的回调函数
      postal.channel(channel || target.channel || "/").publish(topic, value);
    };
  };
}
import publish from "path/to/decorators/publish";

class FooComponent {
  @publish("foo.some.message", "component")
  someMethod() {
    return {
      my: "data"
    };
  }
  @publish("foo.some.other")
  anotherMethod() {
    // ...
  }
}
let foo = new FooComponent();

foo.someMethod() // 在"component"频道发布"foo.some.message"事件，附带的数据是{ my: "data" }
foo.anotherMethod() // 在"/"频道发布"foo.some.other"事件，不附带数据

```
大体上使用这个层面就是这样的，下面谈谈实现原理。
其实 *Decorator修饰器* 的实现是利用了ES5的 **Object.defineProperty(target, name, descriptor);** 方法。首先我们来考虑一个普通的ES6类：

```js
  class Person {
    name() { return `${this.first} ${this.last}` }
  }
```
执行这一段class，给Person.prototype注册一个name属性，粗略的和如下代码相似：
```js
  Object.defineProperty(Person.prototype, 'name', {
    value: specifiedFunction,
    enumerable: false,
    configurable: true,
    writable: true
  });
```
javascript 在注册一个类属性的时候会给这个属性添加一个属性描述符，这个属性描述符包含了这个属性的可读可写，标志了这个属性是否可以被for in for of 这种遍历数据迭代接口的函数遍历出来。那么装饰器在给方法添加修饰器的时候，会在用Object.defineProperty为Person.prototype注册name属性之前，执行这段装潢器：
```js
  let descriptor = {
    value: specifiedFunction,
    enumerable: false,
    configurable: true,enumerable、
    writable: true
  };

  descriptor = readonly(Person.prototype, 'name', descriptor) || descriptor;
  Object.defineProperty(Person.prototype, 'name', descriptor);
```
我们能看出，装潢器只是在Object.defineProperty为Person.prototype注册属性之前，执行一个装饰函数，其属于一类对Object.defineProperty的拦截。

Decorator修饰器就先写到这里，以后想到更多再来补充。
