---
title: WebSocket之maxMessageSize
date: 2017-03-31 19:34:18
tags: 
- WebSocket
categories: 
- JavaScript应用
---

>踩过一次的坑不要踩第二次

用websocket发送图片，各种中断，纠结好几天才解决，这个坑踩的真是心累。

```
@OnMessage(maxMessageSize=1048576)
public void onMessage(String message, Session session){
    System.out.println("Fuck you!");
}
```