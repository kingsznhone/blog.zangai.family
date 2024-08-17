---
title: 网络编程慎用localhost
date: 2024-08-17 22:40:56
tags: 
  - .NET
cover: ./img/localhost-meaning.png
---

前几天写web app的时候，在本机开了一个dummy host，然后在.NET项目中使用`HttpClient`访问http://localhost:7410/start

然而发现接口调用的延迟大概是两三秒，在dummy服务器上的接口测速从接到请求到处理完成返回200OK只用了2毫秒

延迟出在了.NET程序里，这么反常的延迟，肯定不是代码性能方面的问题

最后这个问题硬控我俩小时，在无数chrome标签页的助攻下终于明白了原因

问题的具体讨论在这个链接

https://github.com/dotnet/runtime/issues/23581#issuecomment-354391321

提炼一下就是runtime在碰到localhost域名的时候会调用windows的系统接口

.NET开发组为了保持runtime对域名解析流程简单可靠，不考虑给localhost加上短路

localhost在ipv4和ipv6上有两个目标地址，127.0.0.1或者::1

单纯短路的话就没法使用::1上的服务器了

dummy server监听的是127.0.0.1，我猜延迟就是在dotnet基础库等待::1超时，才切换到127.0.0.1

现在就是不清楚别的语言库会不会有同样的问题，这次是真的糊我一脸大的

为了安全起见，规范编程从我做起，以后避免使用localhost

