---
title: Netty实现心跳处理
date: 2021-03-06 19:54:04
tags: Netty
categories: Java
updated:
keywords:
description:
top_img:
comments:
cover:
toc:
toc_number:
copyright:
copyright_author:
copyright_author_href:
copyright_url:
copyright_info:
mathjax:
katex:
aplayer:
highlight_shrink:
aside:
---


在Socket通信中为了保证Server和Client连接的有效,一般会使用心跳来检测Server和Client通信是否畅通.
## Server 
1. 心跳处理handler
```java
package com.github.whymesay.toy.monarch.transport.common;

import com.github.whymesay.toy.monarch.common.domain.Message;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.timeout.IdleStateEvent;
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.atomic.AtomicInteger;

/**
 * 心跳处理 每个客户端连接的channel 都有一个handler实例
 *
 * @author whymesay cyhyx521@gmail.com
 * @date 2020/10/3 17:50
 */
@Slf4j
public class HeartbeatServerHandler extends SimpleChannelInboundHandler<Message> {

    /**
     * timeout count
     */
    private AtomicInteger timeoutCount = new AtomicInteger(0);

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, Message msg) throws Exception {
        timeoutCount.set(0);
        if (msg.getType() == Message.PING) {
            log.info("receive client heartbeat");
            ctx.writeAndFlush(Message.PONG_MSG);
        }
    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            // 如果是心跳消息 超过时间客户端没有连接 下线
            handlerHeartbeatTimeout(ctx);
        } else {
            super.userEventTriggered(ctx, evt);
        }
    }

    /**
     * handler heartbeat timeout
     *
     * @param ctx ctx
     */
    private void handlerHeartbeatTimeout(ChannelHandlerContext ctx) {
        // todo 处理
        if (timeoutCount.getAndIncrement() >= 5) {
            ctx.close();
            log.warn("monarch timeout count more than 5,close!");
        } else {
            log.warn("monarch timeout, count: {}", timeoutCount.get());
        }
    }
}

```
2. 注册Server Handler,配置IdleStateHandler

服务端需要添加IdleStateHandler,并且配置读取超时时间,超过时间没有获取到客户端的消息,触发event
```java
	//多余的代码省略,具体代码可以查看底部的github地址
        socketChannel.pipeline()
                // when idle , send  heartbeat
                .addLast("IdleStateHandler", new IdleStateHandler(MonarchConstant.HEART_BEAT_TIME_OUT * MonarchConstant.HEART_BEAT_TIME_OUT_MAX_TIME, 0, 0, TimeUnit.SECONDS))
                // byte to message
                .addLast("MonarchDecoder", new MonarchDecoder(globalConfig.getSerializeConfig()))
                // message to byte
                .addLast("MonarchEncoder", new MonarchEncoder(globalConfig.getSerializeConfig()))
                // heartbeat
                .addLast("HeartbeatServerHandler", new HeartbeatServerHandler())
                .addLast("MonarchServerHandler", new MonarchServerHandler(this));

```

## client

1. Client Handler

```java
package com.github.whymesay.toy.monarch.transport.common;

import com.github.whymesay.toy.monarch.common.domain.Message;
import com.github.whymesay.toy.monarch.transport.monarch.constant.MonarchConstant;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.timeout.IdleStateEvent;
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.atomic.AtomicInteger;

/**
 * 心跳处理 每个客户端连接的channel 都有一个handler实例
 *
 * @author whymesay cyhyx521@gmail.com
 * @date 2020/10/3 17:50
 */
@Slf4j
public class HeartbeatClientHandler extends SimpleChannelInboundHandler<Message> {

    /**
     * timeout count
     */
    private AtomicInteger timeoutCount = new AtomicInteger(0);

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, Message msg) throws Exception {
        timeoutCount.set(0);
        if (msg.getType() == Message.PONG) {
            log.info("receive server heartbeat");
        }
    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            // 如果是心跳消息 超过时间客户端没有请求服务端
            handlerHeartbeatTimeout(ctx);
        } else {
            super.userEventTriggered(ctx, evt);
        }
    }

    /**
     * handler heartbeat timeout
     *
     * @param ctx ctx
     */
    private void handlerHeartbeatTimeout(ChannelHandlerContext ctx) {
        if (timeoutCount.getAndIncrement() >= MonarchConstant.HEART_BEAT_TIME_OUT_MAX_TIME) {
            //todo 当前连接以及断开 尝试重连
            ctx.close();
            log.warn("monarch timeout count more than {},close!", MonarchConstant.HEART_BEAT_TIME_OUT_MAX_TIME);
        } else {
            // 超时未发送数据 客户端主动发送消息到server
            ctx.writeAndFlush(Message.PING_MSG);
            log.warn("monarch timeout, count: {}", timeoutCount.get());
        }
    }
}

```


2. 注册Client Handler,配置IdleStateHandler


客户端和服务端基本相同,不同是我们需要配置IdleStateHandler的写超时,一旦没有向服务端发送消息,那么主动向服务端发起请求,说明自己还活着.

```java
        socketChannel.pipeline()
                .addLast("IdleStateHandler", new IdleStateHandler(0, MonarchConstant.HEART_BEAT_TIME_OUT, 0))
                // byte to message
                .addLast("MonarchDecoder", new MonarchDecoder(globalConfig.getSerializeConfig()))
                // message to byte
                .addLast("MonarchEncoder", new MonarchEncoder(globalConfig.getSerializeConfig()))
                .addLast("HeartbeatClientHandler", new HeartbeatClientHandler())
                .addLast("MonarchClientHandler", new MonarchClientHandler(this));
```

## 代码地址
>[https://github.com/whymesay/toy-monarch/tree/master/src/main/java/com/github/whymesay/toy/monarch/transport/monarch](https://github.com/whymesay/toy-monarch/tree/master/src/main/java/com/github/whymesay/toy/monarch/transport/monarch)