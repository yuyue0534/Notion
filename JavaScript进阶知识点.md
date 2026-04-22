## JavaScript进阶知识点全览

一、作用域、闭包、与执行上下文
- 执行上下文与调用栈：每次函数调用都会创建新的执行上下文，包含 variableEnvironment、lexicalEnvironment 和 this 绑定，执行上下文压入调用栈，返回后弹出。
- 词法作用域：js采用词法作用域，函数的作用域在调用的时候确定，而非调用时，属性查找沿着[[Environment]]逐层向上。
- 闭包：函数 + 其外部词法环境的引用，核心用途：
   - 封装私有变量
   - 柯里化、偏应用函数
   - 记忆化
   - 循环中用 let 或 IIFE 保存正确的状态
 
- 变量提升
   - var 声明提升，值为 undefined
   - function 整体提升
   - let、const，存在暂时性死区，访问会抛错 ReferenceError
 
二、异步编程与事件循环
- 事件循环（event loop）：执行顺序： 同步代码 -> 微任务队列（promise.then, queueMicroTask, MutationObserver） -> 宏任务队列（setTimeout, setInterval, I/O）；每一轮宏任务执行前、先清空全部微任务。
- promise深入：
   - 三种状态：pending -> fulfilled/ reject（不可逆）；
   - promise.all（全部成功才成功）、allSettled（全部落定）、race（最快的）、any（最快成功的）。
   - promise构造函数的执行器是同步执行的。
 
- async / await：async函数始终返回promise； await暂停当前微任务； 关键性能点：循环中的await是串行的，需要并行时 用promise.all。
- generator 与迭代器协议：`function*` 返回迭代器，`yield` 暂停执行；实现 `[Symbol.iterator]`使得自定义的对象支持`for...of`，`yield*` 可委托给另一个可迭代对象。
- 任务调度API：
   - requestAnimationFrame：每一帧渲染前执行，适合动画；
   - requestIdleCallback：浏览器空闲时执行，适合低优先级任务；
   - MessageChannel：创建低优先级宏任务。
 

三、原型、继承、与 类：
- 原型链：每个对象有`Prototype`，属性查找沿着链往上，终点为 object.prototype (其 Prototype 为 null);
- new操作符原理（四步）：
   - 创建空对象；
   - 设置 Prototype 为 Fn.prototype；
   - 以新对象为 this执行构造函数；
   - 若构造函数返回对象 则用它，否则返回新对象。

**寄生组合式继承**
```
function Child(...arg){
   Parent.call(this, ...arg)      //借用构造函数
}
Child.prototype =Object.create(Parent.prototype)
Child.prototype.contructor = Child
```

**ES6 class语法糖**
- 本质仍然基于原型，extends自动处理原型链；
- 子类构造函数必须调用super()；
- static方法属于类本身， 不在实例上；
- 私有字段 field （ES2022）真正的私有；

**Mixin模式**： JS不支持多继承，用Object.assign(Target.prototype, MixinA, MixinB)或高阶函数组合行为；

四、this绑定与函数机制
- this的四种绑定原则（优先级从低到高）:

| 规则 | 示例 | this指向 |
| --- | --- | --- |
| 默认绑定 | fn() | 全局对象/undefined(严格模式) |
| 隐式绑定 | obj.fn() | obj |
| 显式绑定 | fn.call(ctx) | ctx |

- 箭头函数无自身this，捕获外层词法this，不可被`call/ bind/ apply`改变;






