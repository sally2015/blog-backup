---
layout: post
title: "调用堆栈与作用域闭包"
date: 2018-11-28 19:53
tags: js
---

### 导读
- 什么是执行上下文？执行栈又是什么？
- js的基本类型和引用类型是如何存放在内存空间的？
- 闭包的定义，为什么要用闭包，闭包的使用场景
- 闭包为什么没清除，为什么闭包有可能导致内存泄漏？

<!-- more -->

### 执行上下文
- 类型：全局执行上下文（window）、函数执行上下文（函数被调用时创建新的函数执行上下文）
- 创建过程：
    1.进行执行上下文：给变量对象添加形参、函数声明、变量声明等初始的属性值
    2.代码执行：修改变量对象的属性值
- 进入执行上下文时变量对象
    - 1.函数的所有形参：没有实参，属性值为undefined
    - 2.变量声明：如果变量名称跟已经声明的形参或者函数相同，则变量声明不会干扰已经存在的这类属性
    - 3.函数声明：如果变量对象存在相同名称的属性则替换它，也就是函数变量声明层级更高

```js
//  变量声明不干扰形参
function g (a) {
    console.log(a)
    var a;
    console.log(a)
    a = 1
    console.log(a)
}
g(4); // 4,4,1
```

```js
// 函数声明优先级更高
function g (a) {
    console.log(a)
    var a = 1;
    function a() {}
    console.log(1)
}
g(4); // f (a) {}, 1
```

### 执行栈（调用栈）
- 用于管理多个被创建的函数执行上下文
- 具有栈的特点：后进先出
- 首次运行js代码会创建一个全局执行上下文，push到调用栈中
- 函数调用时，创建新的函数执行上下文也被push进当前执行栈。

#### 分析结果一样的两个片段调用栈
```js
//代码片段一
var scope = "global scope";
function checkscope(){
    var scope = "local scope";
    function f(){
        return scope;
    }
    return f();
}
checkscope();
```
```js
//片段一调用栈伪代码
ECStack.push(<checkscope> functionContext);
ECStack.push(<f> functionContext);
ECStack.pop(<f> functionContext);
ECStack.pop(<checkscope> functionContext);
```
<p><img src="/assets/context/context-1.png"></p>

```js
// 代码片段二
var scope = "global scope";
function checkscope(){
    var scope = "local scope";
    function f(){
        return scope;
    }
    return f;
}
checkscope()();
```


```js
//片段二调用栈伪代码
ECStack.push(<checkscope> functionContext);
ECStack.pop(<checkscope> functionContext);
ECStack.push(<f> functionContext);
ECStack.pop(<f> functionContext);
```
<p><img src="/assets/context/context-2.png"></p>





https://github.com/mqyqingfeng/Blog/issues/4