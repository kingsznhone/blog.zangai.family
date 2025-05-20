---
title: ASP.NET CORE 数据模型的自定义base58校验
date: 2025-05-21 03:53:04
tags: 
    - C#
cover: /img/2025-05-21/cover.jpg
---

## Attribute 设计
最近在玩Solana，SOL链上的钱包地址和btc一样

使用Base58字符串进行可读性转换

所以在程序中经常需要校验输入的字符串是否为合法的Base58格式

如果能把Base58字符串也容纳到Data Annotation的校验体系里

自动校验从webapi接口传入的data class json里的地址字段，岂不美哉

就可以从接口内部实现中剔除校验Base58字符串相关的逻辑了

`System.ComponentModel.DataAnnotations`提供了一个基础类`DataTypeAttribute`

只需要继承这个类就可以自定义校验逻辑

引入SimpleBase包作为解码库，实现起来不困难

每一个base58字符串都是由`byte[]`编码得来

所以这个annotation应该有一个可变参数`expectedLength`，检查解码后的`byte[]`长度是否符合预期

``` cs
using System.ComponentModel.DataAnnotations;
using SimpleBase;
public class Base58Attribute : DataTypeAttribute
{
    private readonly int _expectedLength;

    public Base58Attribute(int expectedLength) : base(DataType.Custom)
    {
        _expectedLength = expectedLength;
        ErrorMessage = $"必须是有效的 Base58 字符串，且解码后长度必须为 {_expectedLength} 字节。";
    }

    public override bool IsValid(object value)
    {
        if (value is string base58String && !string.IsNullOrEmpty(base58String))
        {
            try
            {
                byte[] decoded = Base58.Bitcoin.Decode(base58String);
                return decoded.Length == _expectedLength;
            }
            catch
            {
                return false; // 解码失败，说明不是合法的 Base58 字符串
            }
        }
        return true; // 为空或者为Null视为有效
        
    }
}
```

## 使用

使用这个annotation只需要把它挂载到对应的属性上即可

如果输入字符串为`Null`或者为`""`，校验器返回通过

这样就可以直接用在可选的钱包字段上，64代表私钥，32代表公钥

```cs
class WalletInput
{
    [Base58(64)]
    public string? MainWallet { get; set; }
    [Base58(32)]
    public string? Program {get;set;}
}
```

如果是必须输入的钱包字段，则需要结合`[Required]`校验器一起

结合起来使用就可以避免`Null`字段进入API

`AllowEmptyStrings`指示是否允许空字符串通过校验

如果设置为false，则空字符串和null字符串都不能通过校验

```cs
class WalletInput
{
    [Base58(64)]
    [Required]
    public string MainWallet { get; set; }
    [Base58(32)]
    [Required(AllowEmptyStrings = false)]
    public string Program {get;set;}
}
```


