---
layout: post
title: "webpack打包文件分析"
date: 2018-8-22 12:17
tags: "webpack"
---

本文将从简单的打包文件开始，分析webpack在commonjs和es6标准情况下的处理以及tree shaking是怎么剔除多余代码的。

## commonjs规范打包文件分析
- 源文件
    ```js
    // index.js
    const util = require('./util');
    console.log(util.hello())
    // util.js
    function hello () {
      return 'hello'
    }
    function bye () {
      return 'bye'
    }
    module.exports = {
        hello,
        bye
    }
    ```
<!-- more -->

- 可以看到webpack打包生成是一个立即执行函数，modules参数是各个模块的代码， 其中`/* 1 */`对应的是index.js，`/* 2 */`对应的是util.js,`/* 0 */`是执行1的主模块文件，可以看到模块函数有三个参数`module`、`exports`、`__webpack_reuire__`，这些都是在立即执行函数内部传递的

    <p><img src='https://image-static.segmentfault.com/274/771/2747719228-5b7b6557c7d93_articlex' width="500"></p>

- 立即函数的内部主要定义了`__webpack_require__`函数及其属性，最后一句话`return __webpack_require__((__webpack_require__.s = 0));`表示执行moduleId为0的模块

    <p><img src='https://image-static.segmentfault.com/415/192/4151923559-5b7b6a8558693_articlex' width="500"></p>

- `__webpack_require__`函数中缓存已经调用的模块，初始化exports等参数并将这些参数传入module的执行中
    ```js
    var installedModules = {}; // 缓存已引入的模块
    function __webpack_require__(moduleId) {
        if (installedModules[moduleId]) { //直接返回引入模块
          return installedModules[moduleId].exports;
        }
        //初始化模块
        var module = (installedModules[moduleId] = {
          i: moduleId,
          l: false,
          exports: {}
        });
        // 执行对应的模块
        modules[moduleId].call(
          module.exports,
          module,
          module.exports,
          __webpack_require__
        );
        // 表示模块已经加载
        module.l = true;
        return module.exports;
    }
    ```
    ```js
    // 挂载模块
    __webpack_require__.m = modules;
    
    // 挂载缓存
    __webpack_require__.c = installedModules;
    
    // 实际是Object.prototype.hasOwnProperty
    __webpack_require__.o = function(object, property) {
        return Object.prototype.hasOwnProperty.call(object, property);
    };
    // 定义对象取值的函数
    __webpack_require__.d = function(exports, name, getter) {
        if (!__webpack_require__.o(exports, name)) {
          Object.defineProperty(exports, name, {
            configurable: false,
            enumerable: true,
            get: getter
          });
        }
      };
    // 判断是否为es模块，如果是则默认返回module['default'],否则返回module
    __webpack_require__.n = function(module) {
        var getter =
          module && module.__esModule
            ?
              function getDefault() {
                return module["default"];
              }
            :
              function getModuleExports() {
                return module;
              };
        __webpack_require__.d(getter, "a", getter);
        return getter;
     };
    ```
## 小结
    - webpack打包文件是一个立即执行函数IIFE
    - 模块文件在外包裹了一层函数，以数组的形式作为参数传入立即执行函数中
    - IIFE内部有缓存已执行函数的机制，第二次执行直接返回exports
    - IIFE 最后执行id为0的模块`__webpack_require__((__webpack_require__.s = 0));`将整个modules执行串联起来
## ES6模块使用打包分析
- 源文件
```js
// // index.js
import addCounter from './util'

addCounter()
//util.js
let counter = 3;
export default function addCounter () {
  return counter++
}
```
```js 
// bundle.js
/* 1 */
/***/ (function(module, __webpack_exports__, __webpack_require__) {

"use strict";
Object.defineProperty(__webpack_exports__, "__esModule", { value: true });
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_0__util__ = __webpack_require__(2);
// // index.js
Object(__WEBPACK_IMPORTED_MODULE_0__util__["a" /* default */])()

/***/ }),
/* 2 */
/***/ (function(module, __webpack_exports__, __webpack_require__) {

"use strict";
/* harmony export (immutable) */ __webpack_exports__["a"] = addCounter;
//util.js
let counter = 3;
function addCounter () {
  return counter++
}
```
- 在ES6打包中，引入其他模块文件的index.js有这样一句代码`Object.defineProperty(__webpack_exports__, "__esModule", { value: true });`标记当前引入`import`文件为es6模块。
- 结合前面IIFE函数中的`__webpack_require__.n`当`__esModule`为true是，`__webpack_require__.d(getter, "a", getter);`获取a的是module.default的值，上面打包文件中`Object(__WEBPACK_IMPORTED_MODULE_0__util__["a" /* default */])()`体现了这一点，保证 counter 变量取的是值的引用。
### ES6和cjs值传递的区别
- CommonJS 输出是值的拷贝，ES6模块输出的是值的引用
- es6值引用源文件
    ```js
    // // index.js
    import { counter, addCounter} from './util'
    console.log(counter);
    addCounter()
    console.log(counter);
    
    //util.js
    export let counter = 3;
    export function addCounter () {
      return counter++
    }
    ```
    
    ```js
    // bundle.js
    /* 1 */
    /***/ (function(module, __webpack_exports__, __webpack_require__) {
    
    "use strict";
    Object.defineProperty(__webpack_exports__, "__esModule", { value: true });
    /* harmony import */ var __WEBPACK_IMPORTED_MODULE_0__util__ = __webpack_require__(2);
    // // index.js
    
    console.log(__WEBPACK_IMPORTED_MODULE_0__util__["b" /* counter */]); // 3
    Object(__WEBPACK_IMPORTED_MODULE_0__util__["a" /* addCounter */])() 
    console.log(__WEBPACK_IMPORTED_MODULE_0__util__["b" /* counter */]); // 4
    
    /***/ }),
    /* 2 */
    /***/ (function(module, __webpack_exports__, __webpack_require__) {
    
    "use strict";
    /* harmony export (binding) */ __webpack_require__.d(__webpack_exports__, "b", function() { return counter; });
    /* harmony export (immutable) */ __webpack_exports__["a"] = addCounter;
    //util.js
    let counter = 3;
    function addCounter () {
      return counter++
    }
    
    /***/ })
    ```
- 在`/* 2 */`中counter（“b”）的定义使用了__webpack_require__.d方法，也就是直接赋值在exports的属性上，import时获取的也就是引用值了

- cjs值引用源文件
    ```js
    // index.js
    const {counter, addCounter} = require('./util');
    console.log(counter);
    addCounter()
    console.log(counter);
    
    //util.js
    let counter = 3;
    function addCounter () {
        counter = counter+1
    }
    module.exports = {
        counter,
        addCounter
    }
    ```
    ```js
    // bundle.js
    /* 1 */
    /***/ (function(module, exports, __webpack_require__) {
    // index.js
    const {counter, addCounter} = __webpack_require__(2);
    console.log(counter); // 3
    addCounter()
    console.log(counter); //3
    
    
    /***/ }),
    /* 2 */
    /***/ (function(module, exports) {
    
    let counter = 3;
    function addCounter () {
        counter = counter+1
    }
    module.exports = {
        counter,
        addCounter
    }

    ```
## 混合cjs和es6打包分析
### import引入cjs模块
```js
//index.js
import { bye } from './bye'
import hello from './hello'

bye();
hello();

//hello
function hello() {  
    console.log('hello')
}
module.exports = hello;

//bye.js
function bye() {  
    console.log('bye')
}
module.exports = {
    bye
};
```
```js
// bundle.js
/* 1 */
/***/ (function(module, __webpack_exports__, __webpack_require__) {

"use strict";
Object.defineProperty(__webpack_exports__, "__esModule", { value: true });
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_0__bye__ = __webpack_require__(2);
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_0__bye___default = __webpack_require__.n(__WEBPACK_IMPORTED_MODULE_0__bye__);
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_1__hello__ = __webpack_require__(3);
/* harmony import */ var __WEBPACK_IMPORTED_MODULE_1__hello___default = __webpack_require__.n(__WEBPACK_IMPORTED_MODULE_1__hello__);



Object(__WEBPACK_IMPORTED_MODULE_0__bye__["bye"])();
__WEBPACK_IMPORTED_MODULE_1__hello___default()();

/***/ }),
/* 2 */
/***/ (function(module, exports) {

function bye() {  
    console.log('bye')
}
module.exports = {
    bye
};

/***/ }),
/* 3 */
/***/ (function(module, exports) {

function hello() {  
    console.log('hello')
}
module.exports = hello;

/***/ })
/******/ ]);
```
- 除了在index.js在es模块设置`__esModule`标记，注意到调用时候的不用
- `Object(__WEBPACK_IMPORTED_MODULE_0__bye__["bye"])()`引入bye是以一个对象的形式引入的。
- `__WEBPACK_IMPORTED_MODULE_1__hello___default()();`在处理cjs时每个模块有一个defalut值，如果是cjs的情况会返回module的getter，而源文件引入Hello是default值，直接执行返回的getter

### cjs引入es模块
- 源文件
```js
//index.js
const { bye } = require('./bye')
const hello = require('./hello')
bye();
hello.default();

//hello.js
function hello() {  
    console.log('hello')
}
export default hello;

//bye.js
export function bye() {  
    console.log('bye')
}
```
```js
// bundle.js
/* 1 */
/***/ (function(module, exports, __webpack_require__) {

const { bye } = __webpack_require__(2)
const hello = __webpack_require__(3)
bye();
hello.default();

/***/ }),
/* 2 */
/***/ (function(module, __webpack_exports__, __webpack_require__) {

"use strict";
Object.defineProperty(__webpack_exports__, "__esModule", { value: true });
/* harmony export (immutable) */ __webpack_exports__["bye"] = bye;
function bye() {  
    console.log('bye')
}

/***/ }),
/* 3 */
/***/ (function(module, __webpack_exports__, __webpack_require__) {

"use strict";
Object.defineProperty(__webpack_exports__, "__esModule", { value: true });
function hello() {  
    console.log('hello')
}
/* harmony default export */ __webpack_exports__["default"] = (hello);

/***/ })
/******/ ]);
```
- 源文件中的`export function bye编译成`__webpack_exports__["bye"] = bye;`export defalut编译成`__webpack_exports__["default"] = (hello)`；所以在默认引入hello调用时要手动调用default方法

# code spliting代码分离
## entry points入口分离
- 最简单最直接的分离代码方式，但是会有重复的问题,比如说引入lodash
- 顺便说一下lodash引入的是一整个对象,如果你是用的是`import {each} from 'lodash'`,还是会引入整个lodash库,目前最好的解决办法是按fp引入比如:`import each from 'lodash/fp/each';`
- 分别在js/index，js/bye中引入lodash，像下面的配置就会存在重复的each方法，如果不是按fp的方式引入lodash，则相当于引入了两个lodash大小的文件是对资源的浪费。
    ```js
    module.exports = {
    entry: {
        index: './src/index.js',
        bye: './src/bye.js'
    },
    output: {
        filename: 'js/[name].js'
    },
    ```
- 所以我们在webpack4之前使用CommonsChunkPlugin来抽取公共组件，webpack4开始使用`optimization`选项来做简化配置
    ```js
    //webapck4之前
    new webpack.optimize.CommonsChunkPlugin({
        name: 'commons',
        // (公共 chunk(commnon chunk) 的名称)
      
        filename: 'commons.js',
        // (公共chunk 的文件名)
      
        minChunks: 2,
        // (模块必须被2个 入口chunk 共享)
    })
    ```
    
## 动态导入
- webpack 提供了两种方式进行动态导入，符合 ECMAScript 提案的`import()`和webpack特定的方法`require.ensure()`
    > import() 调用会在内部用到 promises。如果在旧有版本浏览器中使用 import()，记得使用 一个 polyfill 库（例如 es6-promise 或 promise-polyfill），来 shim Promise。
- 简单的例子：在index.js中动态引入hello.js，打包后会生成一个0.js和index.js
    ```js
    //index,js
    import('./hello').then(hello => {
        hello.default();
    })
    ```
- 先来看index.js中[modules]引入的模块，可以发现引入的动态引入的hello.js文件并不存在index.js，实际是在新生成的`0.js`文件中，这里先按下不提，`/* 1 */`中最重要的两个函数`__webpack_require__.e`和`__webpack_require__.bind`成为动态import的关键
    ```js
    [
    /* 0 */,
    /* 1 */
    /***/ (function (module, exports, __webpack_require__) {
    
                __webpack_require__.e/* import() */(0/* duplicate */).then(__webpack_require__.bind(null, 0))
                .then(hello => {
                    hello.default();
                })
    
                /***/
            })
    /******/]
    ```
- 分析`__webpack_require__.e(0)`
    ```js
    /******/ 	__webpack_require__.e = function requireEnsure(chunkId) {
    /******/ 		var installedChunkData = installedChunks[chunkId];
    /******/ 		if (installedChunkData === 0) {
    /******/ 			return new Promise(function (resolve) { resolve(); });
                /******/
            }
    /******/
    /******/ 		// a Promise means "currently loading".
    /******/ 		if (installedChunkData) {
    /******/ 			return installedChunkData[2];
                /******/
            }
    /******/
    /******/ 		var promise = new Promise(function (resolve, reject) {
    /******/ 			installedChunkData = installedChunks[chunkId] = [resolve, reject];
                /******/
            });
    /******/ 		installedChunkData[2] = promise;
    /******/
    /******/ 		// start chunk loading
    /******/ 		var head = document.getElementsByTagName('head')[0];
    /******/ 		var script = document.createElement('script');
    /******/ 		script.type = "text/javascript";
    /******/ 		script.charset = 'utf-8';
    /******/ 		script.async = true;
    /******/ 		script.timeout = 120000;
    /******/
    /******/ 		if (__webpack_require__.nc) {
    /******/ 			script.setAttribute("nonce", __webpack_require__.nc);
                /******/
            }
    /******/ 		script.src = __webpack_require__.p + "js/" + chunkId + ".js";
    /******/ 		var timeout = setTimeout(onScriptComplete, 120000);
    /******/ 		script.onerror = script.onload = onScriptComplete;
    /******/ 		function onScriptComplete() {
    /******/ 			// avoid mem leaks in IE.
    /******/ 			script.onerror = script.onload = null;
    /******/ 			clearTimeout(timeout);
    /******/ 			var chunk = installedChunks[chunkId];
    /******/ 			if (chunk !== 0) {
    /******/ 				if (chunk) {
    /******/ 					chunk[1](new Error('Loading chunk ' + chunkId + ' failed.'));
                        /******/
                    }
    /******/ 				installedChunks[chunkId] = undefined;
                    /******/
                }
                /******/
            };
    /******/ 		head.appendChild(script);
    /******/
    /******/ 		return promise;
            /******/
        };
    ```
- `installedChunks`对象用来标记模块是否加载过，当installedChunks[chunkId]为0时表示已经加载成功
- `installedChunkData`保存新建的promise，用于判断模块是否处于加载中状态
- 通过创建script标签插入head的方式来加载模块
- 函数最后返回promise
- 经过script标签的方法，代码会引入`0.js`
    ```js
    webpackJsonp([0],[
        /* 0 */
        /***/ (function(module, __webpack_exports__, __webpack_require__) {
        
        "use strict";
        Object.defineProperty(__webpack_exports__, "__esModule", { value: true });
        function hello() {  
            console.log('hello')
        }
        /* harmony default export */ __webpack_exports__["default"] = (hello);
        
        /***/ })
    ]);
    ```
    - 可以看到是执行了一个`webpackJsonp`方法，分析下这个方法
    ```js
        var parentJsonpFunction = window["webpackJsonp"];
    /******/ 	window["webpackJsonp"] = function webpackJsonpCallback(chunkIds, moreModules, executeModules) {
    /******/ 		// add "moreModules" to the modules object,
    /******/ 		// then flag all "chunkIds" as loaded and fire callback
    /******/ 		var moduleId, chunkId, i = 0, resolves = [], result;
    /******/ 		for (; i < chunkIds.length; i++) {
    /******/ 			chunkId = chunkIds[i];
    /******/ 			if (installedChunks[chunkId]) {
    /******/ 				resolves.push(installedChunks[chunkId][0]);
                    /******/
                }
    /******/ 			installedChunks[chunkId] = 0;
                /******/
            }
    /******/ 		for (moduleId in moreModules) {
    /******/ 			if (Object.prototype.hasOwnProperty.call(moreModules, moduleId)) {
    /******/ 				modules[moduleId] = moreModules[moduleId];
                    /******/
                }
                /******/
            }
    /******/ 		if (parentJsonpFunction) parentJsonpFunction(chunkIds, moreModules, executeModules);
    /******/ 		while (resolves.length) {
    /******/ 			resolves.shift()();
                /******/
            }
            /******/
            /******/
        };
    ```
- 将webpackJsonp赋值前，先把之前的webpackJson存储为parentJsonpFunction，当多个入口文件的时候会出现这种情况
- 判断是否已经加载过有缓存，如果没有加载过则将resolve方法push进一个数组中存储
- 将modules对应的id值赋值为引入进来的模块
- 执行resolves队列
- __webpack_require__.e(0)返回的promise后执行`__webpack_require__.bind(null, 0)`，前面我们分析过`__webpack_require__`方法，主要作用是缓存已经调用的模块，初始化exports等参数并将这些参数传入module的执行中。后一个promise链then方法真正执行hello方法。

# Tree shaking
## 一些常用概念
* AST （abstract Syntax Tree）把js的每一个语句都转化为树中的一个节点
* DCE(dead code elimination)保证代码结果不变的前提下，去除无用代码
    ```js 
    DCE{
    	优点：减少程序体积、执行时间
         Dead Code{
        	程序中没有执行的代码（不可能进入的分支、return之后的语句）
            导致dead variable的代码（写入变量后不再读取的代码）
        }
    }
    ```

* tree shaking是DCE的一种方式，在打包时忽略没有用到的代码
* tree shaking通过扫描所有ES6的export。找出所有import标记为有使用和无使用，在后续压缩时进行剔除。因此必须遵循es6的模块规范（import&export）

* webpack2支持tree-shaking需要在babel处理中不能将es6转成commonjs，设置babel-preset-es2015的modules为false，表示不对es6模块进行处理。webpack3/webpack4可以不加这个配置。

<p><img src='http://i2.bvimg.com/661700/ebb0f57413af01cb.png' width="500"></p>


## Tree shaking步骤

- 所有import标记为`harmony import`
- 被使用过的export标记为`harmony export [type]`
- webpack4为了加快开发状态下的编译速度，`mode=development`时都认为是被使用的，不会出现`unused harmony export`[详情][4]
- 没被使用过的export标记为`harmony export [FuncName]`,其中[FuncName]为export的方法名称
- 之后在Uglifyjs（或者其他类似工具进行代码精简），就可以把unused部分去掉

## 方法的处理
    ```js
    //math.js
    export function square(x) {
        return x * x;
    }
    
    export function cube(x) {
        return x * x * x;
    }
    
    //index.js
    import { cube } from './math.js';

    function component() {
        var element = document.createElement('pre');
    
        element.innerHTML = [
            'Hello webpack!',
            '5 cubed is equal to ' + cube(5)
        ].join('\n\n');
    
        return element;
    }
    
    document.body.appendChild(component());
    ```
    
   ```js
    //bundle.js
  (function(module, __webpack_exports__, __webpack_require__) {
    
    "use strict";
    /* unused harmony export square */
    /* harmony export (binding) */ __webpack_require__.d(__webpack_exports__, "a", function() { return cube; });
    function square(x) {
        return x * x;
    }
    
    function cube(x) {
        return x * x * x;
    }
    
    /***/ }),
    /* 1 */
    /***/ (function(module, __webpack_exports__, __webpack_require__) {
    
    "use strict";
    __webpack_require__.r(__webpack_exports__);
    /* harmony import */ var _math_js__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(0);
    
    
    function component() {
        var element = document.createElement('pre');
    
        element.innerHTML = [
            'Hello webpack!',
            '5 cubed is equal to ' + Object(_math_js__WEBPACK_IMPORTED_MODULE_0__[/* cube */ "a"])(5)
        ].join('\n\n');
    
        return element;
    }
    
    document.body.appendChild(component());
    
    /***/ })
    /******/ ]);
    ```
## class的处理
- tree shaking对于类是整体标记为导出的，表明webpack tree shaking只处理顶层内容，类和对象内部都不会再被分别处理。因为类调用属性方法有很多情况，比如util[Math.random() > 0.5 ? 'hello' : 'bye']这种判断型的字符串方式调用

 ```js
    // index.js
    import Util from './util'
    
    let util = new Util()
    let result1 = util.hello()
    
    console.log(result1)
    
    // util.js
    export default class Util {
        hello() {
            return 'hello'
        }
    
        bye() {
            return 'bye'
        }
    }
    // util.js
    export default class Util {
        hello() {
            return 'hello'
        }
    
        bye() {
            return 'bye'
        }
    }
    ```
     ```js
    //bundle.js
    (function(module, __webpack_exports__, __webpack_require__) {
    
    "use strict";
    /* harmony export (binding) */ __webpack_require__.d(__webpack_exports__, "a", function() { return Util; });
    // util.js
    class Util {
        hello() {
            return 'hello'
        }
    
        bye() {
            return 'bye'
        }
    }
    
    /***/ }),
    /* 1 */
    /***/ (function(module, __webpack_exports__, __webpack_require__) {
    
    "use strict";
    __webpack_require__.r(__webpack_exports__);
    /* harmony import */ var _util__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(0);
    // index.js
    
    let util = new _util__WEBPACK_IMPORTED_MODULE_0__[/* default */ "a"]()
    let result1 = util.hello()
    console.log(result1)
    
    /***/ })
    /******/ ]);
    ```


# 副作用（side effects）
在执行某个方法或者文件时会对全局其他内容产生影响的代码。在导入时会执行特殊行为的代码，而不是仅仅暴露一个 export 或多个 export。举例说明，例如 polyfill，在各类prototype假如方法，它影响全局作用域，并且通常不提供 export。

## 模块引入带来的副作用

```js
//index.js
import Util from './util'
console.log('Util unused')

//util.js
console.log('This is Util class')
export default class Util {
  hello () {
    return 'hello'
  }

  bye () {
    return 'bye'
  }
}
Array.prototype.hello = () => 'hello'
```
```js
//bundle.js
function(e, t, o) {
    e.exports = o(1);
  },
  function(e, t, o) {
    "use strict";
    Object.defineProperty(t, "__esModule", { value: !0 });
    o(2);
    console.log("Util unused");
  },
  function(e, t, o) {
    "use strict";
    console.log("This is Util class");
    Array.prototype.hello = () => "hello";
  }
```

- webpack依然会把未使用标记Util为`unused harmony export default`,在压缩时去处class Util的引入，但是前后的两句代码依旧保留。因为编译器不知道这两句代码时什么作用，为了保证代码执行效果不变，默认指定所有代码都有副作用。

## 方法带来的副作用

```js
//index.js
import {hello, bye} from './util'
let result1 = hello()
let result2 = bye()
console.log(result1)

// util.js
export function hello () {
  return 'hello'
}
export function bye () {
  return 'bye'
}
```
```js
//bundle.js
function(e, t, n) {
    e.exports = n(1);
  },
  function(e, t, n) {
    "use strict";
    Object.defineProperty(t, "__esModule", { value: !0 });
    var r = n(2);
    let o = Object(r.b)();
    Object(r.a)();
    console.log(o);
  },
  function(e, t, n) {
    "use strict";
    (t.b = function() {
      return "hello";
    }),
      (t.a = function() {
        return "bye";
      });
  }
```
- bye方法调用后的返回值result2没有被使用，webpack删除了result2的这个变量，bye方法因为webpack调用了会影响什么，内部执行的代码有可能影响全局，所以即使执行后没有用到返回值也不能删除。

## 如何解决副作用
1、 pure_funcs配置，在webpack.config.js中增加没有副作用的函数名
```js
// index.js
import {hello, bye} from './util'
let result1 = hello()
let a = 1
let b = 2
let result2 = Math.floor(a / b)
console.log(result1)
```
```js
 plugins: [
    new UglifyJSPlugin({
      uglifyOptions: {
        compress: {
          pure_funcs: ['Math.floor']
        }
      }
    })
  ],
```
```js
//bundle.js 未使用pure_funcs
function(e, t, n) {
    "use strict";
    Object.defineProperty(t, "__esModule", { value: !0 });
    var r = n(2);
    let o = Object(r.a)();
    Math.floor(0.5);
    console.log(o);
  },
//bundle.js pure_funcs
function(e, t, n) {
    "use strict";
    Object.defineProperty(t, "__esModule", { value: !0 });
    var r = n(2);
    let o = Object(r.a)();
    console.log(o);
  },
```
- 这里之所有要换indexjs的内容正是因为pure_function的局限性，在uglify后函数大多变了名字，只有Math.floor这种全局方法不会被重命名。才会生效。

2、 webpack4 package.json里面配置sideEffects
```json
{
  "name": "your-project",
  "sideEffects": false
}
{
  "name": "your-project",
  "sideEffects": [
    "./src/some-side-effectful-file.js"
  ]
}
```
- false表示代码全部没有副作用，或者通过一个数组表示没有副作用的文件
- concatenateModule作用域提升；在webpack3可以加入webpack.optimize.ModuleConcatenateModulePlugin()对bundle文件进行优化串联起各个模块到一个闭包中，在webpack4中这种配置已经在mode=production中默认配置了
- 开启concatenateModule后bye的调用和本体都被消除了，这个功能的原理是将所有模块输出到一个方法内部，这样像bye这种没有副作用的方法便可以被轻易识别出来
- `` 过去 webpack 打包时的一个取舍是将 bundle 中各个模块单独打包成闭包。这些打包函数使你的 JavaScript 在浏览器中处理的更慢。相比之下，一些工具像 Closure Compiler 和 RollupJS 可以提升(hoist)或者预编译所有模块到一个闭包中，提升你的代码在浏览器中的执行速度。``

# tree shaking小结
- 使用es6模块
- 工具函数尽量使用单独导出不要使用一个类或者对象
- 声明sideEffects

## 参考链接：
- [webpack打包分析][1];
- [Webpack之treeShaking][2];
- [tree shaking][3]


  [1]: https://zacharykwan.com/2018/02/20/webpack%E5%8E%9F%E7%90%86/webpack%E6%A8%A1%E5%9D%97%E5%8C%96%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90%EF%BC%88%E4%B8%80%EF%BC%89%E2%80%94%E2%80%94%20webpack%20%E6%89%93%E5%8C%85%E6%96%87%E4%BB%B6%E5%88%86%E6%9E%90/
  [2]: https://mp.weixin.qq.com/s/Ue0kNOMQS7mH-2-9BhYk8Q
  [3]: https://webpack.docschina.org/guides/tree-shaking/
  [4]: https://github.com/easonyq/easonyq.github.io/issues/1#event-1791361391