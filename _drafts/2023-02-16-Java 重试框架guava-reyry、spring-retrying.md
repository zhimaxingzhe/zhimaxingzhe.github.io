---
layout: post
title:  "如何接手别人的系统-遗留系统重建的道法术器势志（万字长文）"
categories: 方法论
tags:  方法论
author: zhimaxingzhe
link: https://zhimaxingzhe.github.io/
---

* content
{:toc}



引入重试依赖组件辅助开发重试逻辑，经过调研比对spring-retry、guava-retrying，发现前者仅可对异常进行重试，后者的重试策略更为灵活，支持RetryException、RetryListener、WaitStrategy、StopStrategy、BlockStrategy、AttemptTimeLimiter、Attempt。
采用guava-retrying，业务侵入小，简单开发即可使用，接口内配置RetryerBuilder，将函数交给RetryerBuilder执行即可。