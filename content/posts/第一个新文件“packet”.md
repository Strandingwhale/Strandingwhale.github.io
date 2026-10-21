---
title: "第一个新文件“packet”"
date: 2026-10-22
draft: false
tags: ["go", "游戏开发", "游戏项目"]
categories: ["游戏开发"]
description: "第一个文件"
---

# 前言

    我开始建造我的游戏，暂且叫他无底深城吧。首先是构建后端，我打算用go语言来做。我之前学过一点，但不多，也算从头再来。

# 第一个文件

    “packet.go”是将消息 ID + protobuf 二进制 → 组装成 `[4字节长度][2字节msgId][payload]` 的包（大端）并从字节缓冲区里不断读，切出完整包，处理半包、粘包的功能文件

![Encode是用于编码的函数](/images/packet1.png)

![Decode是用于解码的函数](/images/packet2.png)

这些花了我几个小时， 总之收获还蛮多的。而且这和网上的课程相去甚远，像C++里的STL，别人不教，就算你精通了基础，也还是不一样。好了先到这，我睡了。