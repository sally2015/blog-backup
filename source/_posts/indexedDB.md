---
layout: post
title: "indexedDB存储失败分析"
date: 2018-09-13 19:17
tags: "indexedDB"
---

### 问题描述
大数据查询项目有一个用户的浏览器报transaction aborted错误，无法正常使用ide。
这个项目因为是大数据结果集展示，重度依赖indexedDB数据，减少请求开销。
所以一旦db报错完全无法使用。

<!-- more -->

### 问题分析
chrome官方文档存储失败有两种可能，一个是浏览器兼容问题，
但这个用户浏览器是chrome60以上，可以排除是兼容性问题。
另一个可能是如果在用户的硬盘所剩无几的情况下也可能失败。


### indexedDB对于存储的限制

根据MDN里面描述indexedDB的存储限制还是比较大的。
firefox是无限量的，而chrome的要根据可用可用空间和shared pool来计算。
shared pool可以达到可用磁盘空间的1/3，每个应用最多占有shared pool的20%。
如果目前可用空间是60g，shared pool可以占用20g，这个应用程序占用shared pool的20%就是最多是4g。

这个用户在生产环境本地硬盘容量大小是100G，现在只剩1个多G。
如果按照上述所说计算理论上分配下来只有几十M供chrome可用。
在手动清除数据库后，只打开ide一个标签页的情况下还是报错。
另外因为是因为是生产环境在citrix只有chrome浏览器，没有办法测试同样情况下firefox是否可用。

### 错误提示
操作indexedDB是用了dexie的1.5.1版本，在2.0.0版本之后，
会使用QuotaExceededError 代替 AbortError: Transaction aborted模糊的错误。
后面版本先升级到2.0以上，捕获硬盘不足的错误，给用户提示。
后续要考虑在捕获到硬盘不足的情况下是不是做一个不使用indexedDB的方案。

### 小结
这个项目算是比较特殊，以往用indexedDB只是为了优化用户体验。即使报错也不会有影响。
最后indexedDB入坑需谨慎。


### 常见问题收集：
Unhandled rejection: NotFoundError: Failed to execute 'transaction' on 'IDBDatabase': One of the specified object stores was not found.
当更新了数据库的结构时要更新数据库的版本
