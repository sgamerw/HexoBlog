---
title: WebSocket替换tcp连接
layout: post
toc: false
comments: true
date: 2017-02-21 17:09:20
tags:
---

公司项目之前一直用的 TCP 来连接网络，现在需要用 WebSocket 来代替。公司项目开发环境是 cocos2d-x + protobuf。

一直对网络这部分不太了解，在网上查了下[资料](https://github.com/chukong/quick-cocos2d-x/blob/master/samples/websockets/scripts/WebSockets.lua)，了解了使用样例。

cocos2d-x 引擎会向脚本抛出4个消息。
* `kWebSocketScriptHandlerOpen` -- `连接建立`
* `kWebSocketScriptHandlerMessage` -- `接收消息`
* `kWebSocketScriptHandlerClose` -- `连接关闭`
* `kWebSocketScriptHandlerError` -- `连接错误`

首先把几个消息对应的回调函数与项目中原来 TCP 的接口对上，尝试发送消息，失败，服务端解析不出来消息格式，显示消息被分为了好几个包。和原来 TCP 发送消息对比，原来在 c++ 层的 tcp socket 类中还有对数据的处理，在数据前面加上了 16 byte 的头数据，包括 protobuf 加密数据长度、 md5 等信息。

这里有2点思考：
1. 是否需要在 protobuf 加密后的字节流前面加上这 16 byte 的数据。
2. 即时需要加上这些信息，这些逻辑处理应该独立出来，不应该和 socket 耦合，socket 应该职责单一，只负责发送数据，不负责处理数据逻辑。

再次发送，服务端成功解析数据。接下来在收到数据的时候也要把 16 byte 的头数据解析拿到 protobuf 加密数据的长度。参照原来 TCP 类的处理，只需拿到数据长度来解析后面真实的 protobuf 加密部分的数据。

后续测试中，排行榜数据的接收直接导致 APP 崩溃，经过排查，发现因为这条消息的数据量过大，引擎向客户端抛出了3次`接收消息`。第一次解析出来数据长度 10kb ，但是这个包只有 4kb 的数据，接下来又收到 4kb 的数据，第三个包 2kb 数据，才算把数据完整接收。参看了原来 TCP 的处理方案：

* 以消息头里解析出来的 protobuf 加密数据长度为标准，如果接收到的数据不足，则把当前包缓存起来，不向脚本抛出消息。后面来的包继续缓存数据，直至数据达到第一个包消息头解析出来的数据长度。此时消息接收完整，向脚本抛出`接收消息`。

我在查看了引擎在 c++ 封装的 WebSocket 类之后，看到 libwebsockets 库提供了 lws_protocols 这个结构体来配置接收包的相关配置。lws_protocols 中的 rx_buffer_size 字段即是接收包的缓存大小，如果数据量超过了缓存大小，就分多次接收了。所以另一个解决方案是：

* 修改 lws_protocols 类型的数据的 rx_buffer_size 字段，设置得大一些。

这个方案听起来并不怎么靠谱，因为数据超过了设置得字段还是会报错。更好的方案似乎还是原来 TCP 的处理方案。

这里有2点思考：
1. 每一个包接收到的数据都有一个 len 变量标示大小，如果 rx_buffer_size 足够大，每一条消息都不用分包来接收，其实这个 len 变量就和 数据头 16 byte 里解析出来的数据长度一致，这里是否需要添加 16 byte 的头信息是不是有冲突？数据头的 md5 数据并没有用上，也许更严谨的做法是还要依据这个 md5 值来验证 protobuf 加密数据的完整性。
2. 如果严谨的方法还是自己在消息头中传递消息体的大小，缓存数据直至完整接收，那么自己不传数据包大小的做法就很坑了，数据量超过默认的 rx_buffer_size 大小分包接收了，而又给脚本抛消息了就会报错。
