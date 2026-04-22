## JavaScript进阶知识点全览

一、作用域、闭包、与执行上下文
- 执行上下文与调用栈：每次函数调用都会创建新的执行上下文，包含 variableEnvironment、lexicalEnvironment 和 this 绑定，执行上下文压入调用栈，返回后弹出。
- 词法作用域：js采用词法作用域，函数的作用域在调用的时候确定，而非调用时，属性查找沿着[[Environment]]逐层向上。
