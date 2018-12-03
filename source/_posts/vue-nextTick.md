---
layout: post
title: "vue-nextTick"
date: 2018-08-31 9:22
tags: vue
---

## macrotack和microtask
- js每次事件循环只从macrotack中读取一个并任务执行，同一个事件循环会把microtask中的任务执行完毕并且先于macrotask

## 数据更新的处理函数
- 两个macrotask中可能穿插着ui重渲染，所以在microtask中在ui重渲染之前把所有的数据更新，一次渲染就能得到最新的DOM结构，减少开销;所以优先把更新数据的操作放在microtask队列中,批处理更新

## vue中的nextTick的实现
<!-- more -->
- vue中的nextTick根据浏览器的特性封装，优先级如下
   - vue@2.5+ Promise > setImmediate > MessageChannel > setTimeout。
   - vue@<2.5 Promise > MutationObserver > setTimeout

    - Promise
        - 以上函数只有Promise是将回调放进microTak队列中，所以是最优方案，只有在Promise不存在的情况下才会走其他方法。

    - 将setTimeout放到优先级的最后的原因：
       - setimeout中的回调并不是放在microtask而是macrotask中，对于重渲染是有开销的；
       - 在回调之前要不断做超时检查, 用setTimeout最快也是4ms发以后调用回调
       - 虽然setTimeout不是最优方案但是可作为兼容性最好的方案

    - MessageChannel 和 MutationObserver 
        - 由于兼容性问题，vue2.5开始抛弃了MutationObserver
        - MessageChannel拥有port1，port2两个属性，让其中一个port监听onMessage，另一个port发送消息即可，不需要像setTimeout做超时检测，它作为非ie浏览器兼容方案。
        - onMessage回调会被注册为macroTask。

    - setImmediate也是将回调函数放进marcoTak中，优点是不需要做超时检测，目前只有ie浏览器实现。 
