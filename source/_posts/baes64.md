---
title: Base64编码原理
tags: [cs]
categories:
  - 编码
date: 2019-01-30 19:07:37
---

## 概述
> Base64是一种基于64个可打印字符来表示二进制数据的表示方法。由于每6个比特为一个单元，对应某个可打印字符。3个字节有24个比特，对应于4个Base64单元，即3个字节可由4
个可打印字符来表示。它可用来作为电子邮件的传输编码。在Base64中的可打印字符包括字母A-Z、a-z、数字0-9，这样共有62个字符，此外两个可打印符号在不同的系统中而不同。
  
> Base64常用于在通常处理文本数据的场合，表示、传输、存储一些二进制数据，包括MIME的电子邮件及XML的一些复杂数据。
  
概况来说，Base64 就是选出 64 个字符（小写字母a-z、大写字母A-Z、数字0-9、符号"+"、"/"），作为一个基本字符集，将其他所有符号都转换成这个字符集中的字符。

缺点：因为 Base64 将三个字节转化成四个字节，因此 Base64 编码后的文本，会比原文本大出三分之一左右。

优点：有些网关或系统只能使用ASCII字符。Base64就是用来将非ASCII字符的数据转换成ASCII字符的一种方法，而且base64特别适合在http，mime协议下快速传输数据。

<!-- more -->

## 步骤

1. 将每三个字节作为一组，一共是24个二进制位。

2. 将这24个二进制位分为四组，每个组有6个二进制位。

3. 在每组前面加两个00，扩展成32个二进制位，即四个字节。

4. 根据基本字符表，得到扩展后的每个字节的对应符号，这就是Base64的编码值。

## 实例

### 正常情况
![正常情况](/uploads/base64/base64_01.png)

### 需要补位的情况
![需要补位](/uploads/base64/base64_02.png)

## JDK API

JDK 1.8 版本以后 util 包里增加了 Base64 API。

```java
static class TestBase64 {
        public static void main(String[] args) {
            // Basic 编码：标准的BASE64编码，用于处理常规的需求。支持 byte[] -> byte[]  byte[] <-> String
            String encodeHelloStr = Base64.getEncoder().encodeToString("hello".getBytes());

            byte[] decodeHelloByte = Base64.getDecoder().decode(encodeHelloStr);
            System.out.println("hello".equals(new String(decodeHelloByte)));  // true

            // URL 编码：使用下划线替换URL里面的反斜线 /
            String urlEncoded = Base64.getUrlEncoder().encodeToString("index?a=xx".getBytes());
            System.out.println(urlEncoded);

            // MIME 编码：使用基本的字母数字产生BASE64输出，而且对MIME格式友好：每一行输出不超过76个字符，而且每行以“\r\n”符结束
            StringBuilder sb = new StringBuilder();
            for (int t = 0; t < 10; ++t) {
                sb.append(UUID.randomUUID().toString());
            }
            byte[] toEncode = sb.toString().getBytes();
            String mimeEncoded = Base64.getMimeEncoder().encodeToString(toEncode);
            System.out.println(mimeEncoded);
        }
    }
```

## 参考

[Base64笔记](http://www.ruanyifeng.com/blog/2008/06/base64.html)

[Java 8新特性探究（十一）Base64详解](http://www.importnew.com/14961.html)
<hr />