---
title: 拍拍贷微服务rpc框架--raptor
date: 2018-07-18 19:16:13
tags: [微服务,rpc,契约驱动]
categories: [微服务]
---

# ![](raptor-rpc/logo.png)说明

这是我在拍拍贷负责开发的微服务RPC框架，已经拍拍贷全面推广，生产稳定运行。

该RPC框架参考、借鉴了大量已有rpc框架的设计，主要特性如下：

1. 契约驱动(Contract-First)开发模式，采用protobuf契约，自动生成服务器端接口和客户端代码
2. 基于HTTP协议，一套组件同时覆盖内部服务开发和对外开放场景
3. RPC/REST混合模式，既可以使用客户端以RPC/HTTP/JSON方式调用，也可以通过浏览器以REST/HTTP/JSON方式调用
4. 支持多种强类型客户端自动生成，Java/C#/Python/iOS/Android...
5. 设计实现简单轻量，依赖少，可以和Spring(Boot)无缝集成

该框架已经开源，源码地址: [https://github.com/ppdai-incubator/raptor](https://github.com/ppdai-incubator/raptor)

详细参考文档请参考: [https://github.com/ppdai-incubator/raptor/wiki](https://github.com/ppdai-incubator/raptor/wiki)
