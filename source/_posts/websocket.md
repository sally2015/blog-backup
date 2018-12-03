---
layout: post
title: "webscoket使用总结"
date: 2018-11-9 09:07
tags: "websocket"
---

### 项目需求
DataWorkClouds是一个类似于阿里云DataWorks的大数据处理研发平台，支持数据集成，在线代码编辑、查询云上数据等功能的在线编辑器。这次改造主要是将以前的在脚本查询数据执行后的日志、历史、状态等需要实时反馈给用户的接口从http轮询改为websocket即时通信和推送消息。

<!-- more -->

### websocket基础知识
- 为了实现websocket通信，客户端首先发起了一次http get请求，请求报文如下。其中connection:upgrade和upgrade:websocket是告诉服务器通信协议发生改变，改变为websoket协议。Sec-WebSocket-Key是一个base encode的值，由客户端随机生成，用于验证服务器。
```
    Connection: Upgrade
    Host: xxx.xxx.xxx
    Origin: http://localhost:5000
    Sec-WebSocket-Key: ELlxT2NAsGHc9/F980hVzw==
    Sec-WebSocket-Version: 13
    Upgrade: websocket
```
- 支持websocket的服务器在确认以上请求后，返回状态码为101 Switching protocals的响应。
- 其中字段Sec-WebSocket-Accept是由服务器对客户端发送的Sec-WebSocket-Key进行认证和加密后的结果，以帮助客户端确认是真实可靠的服务器。
```
General:
    Request URL: ws://10.255.0.134:11005/ws/api/entrance/execute
    Request Method: GET
    Status Code: 101 Switching Protocols
Response Headers:
    Connection: Upgrade
    Date: Fri, 09 Nov 2018 00:41:27 GMT
    Sec-WebSocket-Accept: qcC7cougCawcQ/D3qYCIHQb/6I0=
    Sec-WebSocket-Extensions: permessage-deflate
    Upgrade: WebSocket
```
- socket.io是一个封装很好的库，但是这次并没有用到任何库；socket.io在配套适用一系列后端相应库的时候处理事件非常方便，但这次项目后端使用的库前端并不能控制，这时候用库反而有些多余了。实际上使用websocket只需要几行代码。
- 创建socket实例和最简单的用法
```js
var ws = new WebSocket("wss://echo.websocket.org");

ws.onmessage = function(evt) {
  console.log( "Received Message: " + evt.data);
  ws.close();
};

ws.onclose = function(evt) {
  console.log("Connection closed.");
};  
```
- 调用构造函数的时候new WebSocket(url)过程：
  - url解析器解析url地址并返回urlRecord
  - 如果解析urlRecord失败，抛出"SyntaxError" DOMException.（使用try catch，并发出downgrade事件，改成http请求）
  - 建立连接，返回新的websocket对象。如果建立连接失败，会触发websocket连接算法失败，该算法确定websocket连接关闭，从而引发close事件。

- socket建立后的readystate状态
 - 0：尚未连接(当发送数据触发重连时，会立即重新建立连接，此时readystate可能为0，则需要做延迟发送的操作，否则会抛出InvalidStateError"DOMException)
 - 1:连接成功
 - 2：触发了close事件，正在关闭socket
 - 3：socket已经关闭

### 项目中的注意点
- socket init之后一个页面连接且仅连接一个websocket，因为频繁接收websocket推送消息，所以除非长时间没有数据交互服务器主动断连，否则前端不会主动close。 

- 监听close事件和状态码的处理
 - 1001：1001是空闲状态处理（Idle timeout），是由于长时间没有任何数据交互，超过了服务器设置的时间而返回的状态码。考虑到此时确实已经长时间没有数据交互，可能是用户开了tab闲置，为了减轻服务器压力，不会立即重连，而是下次前端发送消息（比如再次执行脚本sendData）才重新连接，而其他错误码close立即重连。
 - 1006错误：一般是服务器挂掉或者断网的情况。后台一台节点服务器挂掉的情况，由于后端服务是服务器集群，前端自动重连可以连接到下一个节点的服务器。

- error事件的处理
 - 一开始总是纠结既想着去处理onclose也想着onerror的时候怎么处理，什么时候应该降级使用http请求。实际上应该让它避免发生error事件，如果是无法避免的触发的错误最后都是导致close发生，最后只需要处理close事件即可。最常见会发生error的场景是，socket的readystate为0时就去调用send方法，判断当前状态延迟发送能够直接避免这个问题。

- http降级处理
 - 网络原因断连的情况并不需要降级为http请求，只需要socket重连即可，socket在成功连接前是先发送get http请求的（101 Switching Protocols），而后才upgrade为Websocket协议。这一步实际上跟http请求轮询的方案是类似的。
 - 项目中只有在new Websocket时由于解析url失败的情况才会选择降级处理，因为此时是实例化socket本身出现了问题。
