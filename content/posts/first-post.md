+++
date = '2026-05-21T17:48:43+08:00'
draft = false
title = 'First Post'
tags: ["标签1", "标签2"]
categories: ["分类1"]
+++

```java
public static void sendMsg(DataPacket prototype) {
        for (Channel channel : clients) {
            if (channel.isActive()) {
                ByteBuf byteBuf = prototype.toByteBufMsg();
                prototype.setPayload(byteBuf);
                log.info("体信息:{}", CodecUtils.byteBufToHex(byteBuf));
                log.info("原始包中的buf: {}", CodecUtils.byteBufToHex(prototype.getPayload()));
                log.info("发送消息给客户端: {}", channel.remoteAddress());
                channel.writeAndFlush(prototype);
            }
        }
    }

```

