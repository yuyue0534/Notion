## JavaScript进阶知识点全览

**一、作用域、闭包、与执行上下文**
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
 
**二、异步编程与事件循环**
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
 

**三、原型、继承、类**
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

**四、this绑定与函数机制**
- this的四种绑定原则（优先级从低到高）:

| 规则 | 示例 | this指向 |
| --- | --- | --- |
| 默认绑定 | fn() | 全局对象/undefined(严格模式) |
| 隐式绑定 | obj.fn() | obj |
| 显式绑定 | fn.call(ctx) | ctx |
| new绑定 | new fn() | 新创建的实例 |

- 箭头函数无自身this，捕获外层词法this，不可被`call/ bind/ apply`改变;
- 柯里化（currying）:简单来说，这个函数的作用是：把一个接收多个参数的函数，转换成一个可以“分步”接收参数的函数。

```
const curry = fn => {
   const arity = fn.length;
   return function curried(...args) {
      return args.length >= arity ? fn(...args) : (...more)=>curried(...args, ...more)
   }
}
```
- 函数组合（compose、pipe）
```
const compose = (...fns) => x =>fns.reduceRight( (v, f) =>f(v), x );
const pipe = (...fns) => x=> fns.reduce( (v,f) => f(v),x )
```

**五、内存管理与垃圾回收**
- V8的堆内存结构：
   - 新生代：存活时间短的对象，Scavenge（复制）算法，速度快；
   - 老生代：存活时间长的对象，标记清除 + 标记整理。
 
- 内存泄漏常见场景：
   - 意外全局变量：（忘写 var/let/const）；
   - 被遗忘的定时器或事件监听器；
   - 闭包意外持有大对象引用；
   - 变量持有已经从文档移除的dom节点（游离节点）；

**六、dom操作与事件系统**
- 事件流三阶段：捕获（自顶向下） -> 目标 -> 冒泡（自底向上）；addEventListener 在捕获阶段触发；
- 事件委托：将监听器挂在父节点，通过e.target判断来源，大幅减少监听器数量，也支持动态添加的子元素；
- 重排（Reflow）vs 重绘（Repaint）：
   - 改变几何属性（宽高位置）-> 触发重排（代价最高）；
   - 只改变颜色 -> 只会触发重绘；
   - 优化：使用documentFragment 批量操作，读写分离，transform开启GPU合成层；
 
- 现代观察者API：
   - Mutationobserver：异步监听dom变更（微任务回调）；
   - IntersectionObserver：监听元素与视口交叉，用于懒加载；
   - ResizeObserver：监听元素尺寸变化；

**ES6+ 核心新特性**
- Proxy与Reflect：
```
const handler = {
   get(target, key, receiver) {
      console.log(`读取 ${key}`)
      return Reflect.get(target, key, receiver)
   }
};
const proxy = new Proxy(obj, handler);
Proxy拦截13 种操作，是vue3响应式原理的核心，Reflect提供与陷阱对应的默认行为。
```
- Map、Set、WeakMap、WeakSet；
   - Map：任意类型的键，保持插入顺序，`.size`属性；
   - Set:唯一值的集合，可用于数组去重；
   - WeakMap/WeakSet：键为弱引用，不可枚举，适用于存储私有数据或做缓存；
 
- ES2020-2024精选：
```
//空值合并 + 可选链
const name = user?.profile?.name ?? 'defaultName';

//promise.any （任一成功）
const fastest = await Promise.any([p1, p2, p3])

//深克隆
const clone = structuredClone(obj);

//负索引
const last = arr.at(-1);

// Object.hasOwn（替代 hasOwnProperty）
Object.hasOwn(obj, 'key')

//Top Level await (ESM中)
const data = await fetch('./api').then(r => r.json() )

//数组分组
const grouped = arr.group(item => item.type)
```

**八、模块化与工程化**

ESM vs CJS 核心区别
| 特性 | ESM | CJS |
| --- | --- | --- |
| 导入时机 | 静态（编译期间） | 动态（运行时） |
| 绑定类型 | Live binding（实时） | 值拷贝 |
| Tree Shaking | 支持 | 支持 |
| 顶层this | undefined | module.exports |

Tree Shaking： 基于ESM静态分析，打包器标记未使用的导出，压缩时删除；在package.json中设置 “sideEffects: false” 告知打包器这个包 无副作用；

动态导入：
```
//路由懒加载示例：
const route = {
   component: () => import('./views/Home.vue')
}
```

常用设计模式：
- 观察者模式：subject直接通知observer（如 EventEmitter）；
- 发布订阅模式：通过事件总线中介解耦（如$emit/ $on）；
- 单例模式：ES Module的顶层export 天然时单例；
- 装饰器模式（AOP）：不修改原函数、包裹添加前置/后置逻辑。

**九、性能优化**

 防抖与节流
```
//防抖：最后一次触发, N ms后执行
function debounce(fn, delay) {
   let timer;
   return function(...args) {
      clearTimeout(timer);
      timer = setTimeout(() => fn.apply(this, args), delay);
   }
}
//节流，每次间隔M ms 最多一次
function throttle(fn, interval) {
   let last = 0;
   return function(...args) {
      const now = Date.now();
      if(now - last >= interval) {
         last = now;
         fn.apply(this, args)
      }
   }
}
```

V8隐藏类与内联缓存优化
- 隐藏类：对象类型属性相同时共享隐藏类，访问更快；
- 内联缓存（IC）：单态（monomorphic）> 多态（polymorphic）> 超态（megamorphic）
- 避免去优化：不要动态改变对象结构，避免混合类型数组；

Web Worker
```
//主线程
const worker = new Worker('./heavy.js');
worker.postMessage({data: largeArray});
worker.onmessage = function(e){
   console.log(e.data);
}
//heavy.js
self.onmessage = function(e) {
   const result = heavyCompute(e.data);
   self.postMessage(result);
}
```

