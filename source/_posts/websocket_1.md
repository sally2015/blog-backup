---
layout: post
title: "webscoket异常关闭处理"
date: 2018-12-3 17:09
tags: "websocket"
---

### 问题描述

- 在测试环境中，切换了提供websocket连接服务器的ip和端口，相当于此时连接的服务器无法提供websocket的服务；
- 与之前提到的不同new webSocket(url)无法解析则会直接抛出一个抛出"SyntaxError" DOMException，用try catch捕获错误抛出downgrade事件，改成http请求即可
- 但这时new webSocket(url)中解析url是正常的，catch并不能捕获到异常,而是通过error事件1006状态码告知，奇怪的是，error和close事件并没有接受到任何原因，却直接在console面板中直接输出错误。
- 由于没有细分异常报错，前端一直在重连。

<!-- more -->
<p><img src='/assets/websocket/error.png' width="500"></p>

### 问题分析
- websocket连接过程中发生错误是会触发错误事件而不是直接throw error，这是因为`throw`操作必须是同步的。
- 如果同步抛出所有连接时的错误，脚本执行和ui渲染会被阻塞知道完全正规websocket握手。
- 由于异步连接，websocket就会通过error事件报错，因此try catch无法捕获到错误。
- ERR_CONNECTION_RESET这些错误因为安全问题，js无法访问到具体的信息，只会在console中告知用户。以下情况如果允许js访问将探测探测用户的本地网络，因此浏览器故意省略WebSocket API的WHATWG规范要求的任何有用信息。
    - A server whose host name could not be resolved.
    - A server to which packets could not successfully be routed.
    - A server that refused the connection on the specified port.
    - A server that failed to correctly perform a TLS handshake (e.g., the server certificate can't be verified).
    - A server that did not complete the opening handshake (e.g. because it was not a WebSocket server).
    - A WebSocket server that sent a correct opening handshake, but that specified options that caused the client to drop the connection (e.g. the server specified a subprotocol that the client did not offer).
    - A WebSocket server that abruptly closed the connection after successfully completing the opening handshake.

### 重连机制如何维护
- websocket的状态码分为保留和可供框架和应用细分两种，不能笼统使用1006一个状态码，需要后台配合约定不同状态码的处理。
- 之前为了不丢失消息，除了空闲状态的关闭，断连后都会立即尝试重连。
    1. 异常关闭可能由任何原因引起，在这种情况下如果每个客户端遇到异常关闭并立即且持续地的尝试重新连接，服务端可能会因为大量的客户端尝试重新连接遇到的拒绝服务攻击，最终结果可能是服务不能及时的恢复或恢复是更加困难。
    2. 为了解决1的问题，立即重连换成延时重连，如设置一个递增的时长（如[1,2,3,5,10,20]秒）来尝试恢复，这里可以使用rxjs比较方便。但是关键状态消息丢失的问题极有可能发生了。
    3. 要求严格的im通信需要ack确认接收，或者序列标记来解决2的问题。客户端与服务器进行消息接收确认，当客户端向服务器发送消息确认接收的数据包时，服务器才将持久存储中的消息删除，或修改为已接收的状态。当客户端发生断线时，消息依然在服务器端的存储中。客户端重新上线后，服务器端将未发送成功的消息重新进行发送。每个消息都应当有编号，客户端需要保存一份最近的消息ID，防止服务器发送重复的消息。
    4. 但是这个项目如果断连后前端还没有连接上则会丢包不存储，所以前端要在消息监听时做一些定时检测，超过时长给用户提示，重新执行。
