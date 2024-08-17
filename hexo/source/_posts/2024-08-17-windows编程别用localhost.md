---
title: windows编程慎用localhost
date: 2024-08-17 22:40:56
tags: 
  - .NET
cover:
---


前几天写web app的时候，在本机开了一个dummy host，然后在.NET项目中使用`HttpClient`访问http://localhost:7410/start

然而发现接口调用的延迟大概是两三秒，在dummy服务器上的接口从接到请求到处理完成返回200OK只用了几毫秒

那延迟就出在了.NET程序里，这么反常的延迟，肯定不是性能问题

最后这个问题硬控我俩小时，在无数chrome标签页的助攻下终于明白了原因

问题的具体讨论在这个链接

https://github.com/dotnet/runtime/issues/23581#issuecomment-354391321

提炼一下就是runtime在碰到localhost域名的时候会调用windows的系统接口，把localhost转换成127.0.0.1。

巨大延迟就出在这一来一回上，windows系统的DNS解析localhost的用时非常长，不知道是不是和我挂了梯子有关系。

.NET开发组为了保持runtime对域名解析流程简单可靠，不考虑给localhost加上短路

现在就是不清楚别的编程语言runtime会不会有同样的问题，这次是真的糊我一脸

我在写这篇博客的时候，也注意到node.js的本地测试启动的也是localhost:4000，每次打开网站的时候都会卡死几秒才开始加载

之前还以为是node.js有点慢， 现在感觉应该也是同样的windows DNS问题

为了安全起见，规范编程从我做起，以后避免使用localhost

