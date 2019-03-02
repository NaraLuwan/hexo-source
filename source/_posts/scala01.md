---
title: scala 学习笔记
tags: [scala]
categories: scala
date: 2019-01-12 16:53:21
description: Recording the process of learning scala
---
<p class="description"></p>

下午抽时间学习了下 scala 的基本语法，有 java 基础看起来很快，这里主要记录一下与 java 不同的地方。

## scala 简介

Scala 是 Scalable Language 的简写，是一门多范式的编程语言。

## scala 特性

- 面向对象：每个值都是对象。对象的数据类型以及行为由类和特征描述。
- 函数式编程：函数也能当成值（对象）来使用。
- 静态类型：通过编译时检查，保证代码的安全性和一致性。
- 扩展性：可以以库的形式无缝添加新的语言结构，自动闭包等。
- 并发性：使用Akka作为默认 Actor 实现，可以复用线程。

<!-- more -->

## 数据类型

与 Java 有着相同的数据类型：

> Byte,Char,Short,Int,Long,Float,Double,Boolean

不同的是这些数据类型在 scala 中都是类，也就是说scala没有java中的原生类型。在scala可以对数字等基础类型调用方法。

## 变量

使用关键词 var 声明变量，使用关键词 val 声明常量。声明变量的时候可以不指定类型，编译器会根据初始值自动推断。

## 访问修饰符

与 Java 有着相同的访问修饰符：

>private，protected，public。

不同的是 Scala 中的 private 限定符比 Java 更严格，在嵌套类情况下，外部类不能访问被嵌套类的私有成员。

## 运算符

和 Java 不同的是，Scala 中算术运算符没有 ++/-- 操作符，要使用 +=/-=。

## 条件/循环

if else 使用和 java 一样。循环语句不支持 break 或 continue 语句，需要依赖 scala.util.control._ 包中的 Breaks 对象。

## 方法/函数

方法是类的一部分，而函数是一个对象可以赋值给一个变量。换句话来说在类中定义的函数即是方法。使用 val 语句可以定义函数，def 语句定义方法。

## 闭包

闭包是一个函数，返回值依赖于声明在函数外部的一个或多个变量，可以简单的认为是可以访问一个函数里面局部变量的另外一个函数。类似于 JavaScript 中的闭包。

## 数组/集合

主要是定义数组的语法和 java 不同，以及一些专有的 api 如：concat

## 类/接口

使用关键词 object 声明类，使用关键词 trait 声明接口（特征）。与 java 中 interface 不同的是，trait 可以定义属性和方法的实现。

在 scala 中同样是单继承多实现。

## 模式匹配

对应 Java 里的 switch，但是写在选择器表达式之后。即： 选择器 match {备选项}。

## 异常处理

同 java 一样，catch 子句是按次序捕捉的，越具体的异常越要靠前，如果抛出的异常不在catch字句中，该异常则无法处理，会被向上层抛出到调用者处。

不同的是 catch 子句的语法借用了模式匹配的思想来做异常的匹配，是一系列case字句。

## 文件I/O

Scala 进行文件写操作，直接用的都是 java中 java.io.File 包下的 I/O 类。进行文件读操作需要依赖 scala.io.Source 包下类及伴生对象。

## 练习代码

Github: [AwesomeScala](https://github.com/NaraLuwan/AwesomeScala)

<hr />