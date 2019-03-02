---
title: AES + RSA + Hash 实现 C-S 安全交互 
tags: [加解密]
categories:
  - 加解密
date: 2019-01-31 10:30:53
---
## 概述

AES 由于其执行速度快，易于硬件实现，破解难度大等优势，被广泛用于数据的加密。

既然是对称加密，那如何保证秘钥的安全传输？很容易想到用 RSA 加密秘钥。由于只能用私钥解密，而私钥不需要交互双方都知道也就不用通过网络传输，只要私钥不泄露信息就是安全的。

但如果别人截取到请求后伪造数据也用 RSA 公钥加密这种情况呢？也就是如何保证数据的准确性？这个时候就需要签名校验。

本文基于 AES + RSA + Hash 实现一套完整的足够安全的加解密算法。

<!-- more -->

## 流程图

![交互流程图](/uploads/encrypt/AES_RSA_Hash.png)

## 实现

### Client 端

1. 生成 AES 密钥。
2. 使用生成的 AES 密钥对请求的明文数据进行加密，得到 EncryptData。
3. 使用 Server 端提供的接口获取RSA公钥。
4. 使用获取到的 RSA 公钥对 AES 密钥进行加密，得到 EncryptAesKey。
5. 生成签名（CRC 或 Hash 都可以，简单点可以只对 AES 秘钥按一定的规则转换后 Hash）。
6. 将 EncryptAesKey EncryptData 和 Hash 一起发送给 Server 端。

### Server 端

1. 生成 RSA 密钥对,并提供接口给 client 获取 RSA 公钥（或者直接私下明文约定好）。
2. 响应 Client 的 Http 请求，获取到 EncryptAesKey EncryptData 和 Hash。
3. 使用 RSA 私钥 EncryptAesKey 进行 RSA 解密，得到 AES 密钥 AesKey。
4. 按照约定的规则对 AesKey 进行转换后再生成签名，校验获取到的 Hash 字段，如不通过就不用继续后边的处理了。
5. 使用最终的 AesKey 对 EncryptData 进行 AES 解密，得到明文数据。
6. 做响应的处理，返回结果。

** 注：返回结果的加解密逆推回去即可。**

## 一些扩展

1. 签名前最好对参与签名的字段先 Base64 编码一下，避免一些特殊字符导致签名校验不通过，返回结果最好也编码下。
2. 请求和返回数据最好增加时间戳或 UUID 字段，这样生成的签名基本不会出现重复的情况，而且每次都有变动。

<hr />