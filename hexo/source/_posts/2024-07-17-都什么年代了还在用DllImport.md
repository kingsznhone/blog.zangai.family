---
title: 都什么年代了还在用[DllImport]?
date: 2024-07-17 00:58:14
tags:
  - .NET
cover: /img/2024-07-17/cover.jpg
---

# 概述

C#的项目调用本机代码，比如C++写的DLL，是windows开发经常遇到的事情

`[DllImport]`作为二十多年来广为流传的代码蜈蚣，已经承担了太多

作为C#早期就出现的核心功能，`[DllImport]`在运行时生成IL stub会导致额外的性能开销

在framework时代，.NET存在的意义，是作为一个依附于windows的便捷开发平台，性能上的考虑不多。

随着时代发展，.NET如果再固步自封，一定会因为性能问题，被淘汰在历史的长河中。

随着Ryujit逐渐替换老旧的IL编译器，.NET的性能也越来越强。

.NET 6是一次浴火重生，性能大幅度增强，标准库和工具链里也塞了很多新玩具，但还有一个问题有待解决，就是与本机代码的交互

其实现在离解决interop性能问题仅有一步之遥，在.NET 5中加入的源生成器就是关键

在.NET 7中通过源生成器，在编译时生成IL stub，就可以更高效地调用本机代码，基本消除了调用开销

在一些计算密集型的热点，比如三角函数，指对数函数的运算，就可以把热点抽出来用C++重写，然后编译成DLL导出给C#程序使用。

把多线程和异步IO这种复杂的任务管理交给C#，再把计算热点交给Native Code

一方面享受着C#的开发便利和托管安全，又得到了native code的狂暴性能，这就相当于给C#开了个外挂的氮气加速。

# [DllImport]

考虑这样一个计算热点

``` C
#include "math.h"
struct Term
{
    double ss;
    double cc;
    double aa;
    double bb;
};

double Substitution(struct Term* terms, int length, double tj, double tit) {
    double result = 0;
    for (int n = 0; n < length; n++) {
        double u = terms[n].aa + terms[n].bb * tj;
        double su = sin(u);
        double cu = cos(u);
        result += tit * (terms[n].ss * su + terms[n].cc * cu);
    }
    return result;
}
```
使用如下方式导出为NativeAccelerator.dll
``` C
__declspec(dllexport) double Substitution(struct Term* terms,int length, double tj, double tit);
```

在C#代码中只需要引入这个DLL即可使用Native函数

``` C# 
public class Calculator
{
    [DllImport("NativeAccelerator.dll",CharSet =CharSet.Unicode)]
    private static extern double Substitution(Term[] terms, int length, double tj, double tit);
}
```

在.NET 6 环境下使用`[DllImport]`调用Native Code，benchmark出来的运行时间是纯C#实现的70%左右。

在计算密集型任务，将热点抽出来进行Native加速是很有必要的。

# [LibraryImport]

换用`[LibraryImport]`，需要注意的一点是源生成器会在运行时额外生成一部分代码，所以需要把所在的Class修饰符加上partial，对应的函数声明也得有partial。

``` C#
public partial class Calculator
{
    [LibraryImport("NativeAccelerator.dll", EntryPoint = "Substitution", StringMarshalling = StringMarshalling.Utf16)]
    private static partial double Substitution(Term[] terms, int length, double tj, double tit);
}
```
在win_x64平台下，不需要再考虑Calling Convention的问题，Cpp代码和C#代码都默认遵守Microsoft x64 ABI.

都2024年了，应该也没必要再研究x86的那些屎山了。

本机代码调用换成`[LibraryImport]`以后，benchmark出来的计算速度是纯C#实现的30%。相比`[DllImport]`，节省了40%的动态IL stub生成的开销

![Benchmark](https://raw.githubusercontent.com/kingsznhone/VSOP2013.NET/main/README/NativeAccelerate.png)


整体结果可以说是非常令人满意了。就算使用纯Cpp实现算法，也不一定会比这个混合计算快。

计算热点函数占用程序90%以上的CPU时间，反而有可能因为cpp水平太菜写出奇怪的代码，导致速度比用纯C#实现还慢。

# 多说一点

现在的.NET 8 的性能在.NET 7 的基础上更进一步。

即将于年底发布的.NET 9 还有性能上的大幅度增强，JIT语言性能差已经是20年前的陈旧思维了

微软前几个月发布的Garnet，作为redis的平替，使用纯C#实现，跑分比原版redis还要高。

在如今的.NET上，计算密集型程序写不出native级别的性能，就是菜

~~菜，就多练，写不出来就别写。以前是以前，现在是现在~~

.NET 的高性能离不开神一样的底层转译和基础库，向默默无闻的coreclr和ryujit开发者致敬。

![Alt Text](img/2024-07-17/1.gif)
