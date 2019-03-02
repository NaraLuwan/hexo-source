---
title: spring实现定时任务
date: 2017-03-15 14:55:06
tags: Spring
categories: 
- Java
- Web
---
今天在项目里需要实现一个定时任务，每隔3个小时将过滤的广告通过邮件上报给运营一次。考虑了一下，从实现的技术上可以有三种做法：
- Java自带的java.util.Timer类，这个类允许调度一个java.util.TimerTask任务。使用这种方式可以让你的程序按照某一个频度执行，但不能在指定时间运行。
- 使用Quartz，这是一个功能比较强大的的调度器，可以让程序在指定时间执行，也可以按照某一个频度执行，不过配置起来稍显复杂。
- Spring3.0以后自带的task，其实就是一个轻量级的Quartz，而且使用起来比Quartz简单许多。支持注解和配置文件两种方式。

综合考虑，最后使用第三种方案，在这里记录一下：<!-- more -->
首先在spring的配置文件头中配置命名空间及描述
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:task="http://www.springframework.org/schema/task"
xsi:schemaLcation="http://www.springframework.org/schema/task http://www.springframework.org/schema/task/spring-task-3.0.xsd">
```
然后配置task任务扫描注解，可以理解为定时器开关
```
<task:annotation-driven/>
```
最后配置扫描位置：
```
<context:component-scan base-package="com.test"/>
```
java代码：
```java
package com.test;
public interface TestService {  
       public void test();  
}  
@Component  //import org.springframework.stereotype.Component;  
public class MyTestServiceImpl  implements TestService {  
      @Scheduled(cron="0 0/5 * * * ?")   //每5分钟执行一次  
      @Override  
      public void test(){  
            System.out.println("定时任务执行...");  
      }  
}
```
执行后控制台每5分钟就会打印出：定时任务执行...   了
这里有两个地方需要注意：
1.定时器方法所在的类上要配置注解@Component,而不是接口
2.除了上边用到的这种通过@Scheduled注解外还可以通过配置文件实现
```
<task:scheduled-tasks>   
        <task:scheduled ref="testTask" method="test" cron="0 0/5 * * * ?"/>   
</task:scheduled-tasks>
```
这里cron的值是通过一种cron表达式来配置的，Cron表达式是一个字符串，字符串以5或6个空格隔开，分为6或7个域，每一个域代表一个含义，Cron有如下两种语法格式： 
- Seconds Minutes Hours DayofMonth Month DayofWeek Year 
- Seconds Minutes Hours DayofMonth Month DayofWeek
每一个域可出现的字符如下：
- Seconds:可出现", - * /"四个字符，有效范围为0-59的整数 
- Minutes:可出现", - * /"四个字符，有效范围为0-59的整数 
- Hours:可出现", - * /"四个字符，有效范围为0-23的整数 
- DayofMonth:可出现", - * / ? L W C"八个字符，有效范围为0-31的整数 
- Month:可出现", - * /"四个字符，有效范围为1-12的整数或JAN-DEc 
- DayofWeek:可出现", - * / ? L C #"四个字符，有效范围为1-7的整数或SUN-SAT两个范围。1表示星期天，2表示星期一， 依次类推 
- Year:可出现", - * /"四个字符，有效范围为1970-2099年
每一个域都使用数字，但还可以出现如下特殊字符，它们的含义是： 
- \* : 表示匹配该域的任意值，假如在Minutes域使用*, 即表示每分钟都会触发事件
- ? : 只能用在DayofMonth和DayofWeek两个域。它也匹配域的任意值，但实际不会。因为DayofMonth和 DayofWeek会相互影响。例如想在每月的20日触发调度，不管20日到底是星期几，则只能使用如下写法： 
13 13 15 20 * ?, 其中最后一位只能用？，而不能使用*，如果使用*表示不管星期几都会触发，实际上并不是这样
- \- : 表示范围，例如在Minutes域使用5-20，表示从5分到20分钟每分钟触发一次
- / : 表示起始时间开始触发，然后每隔固定时间触发一次，例如在Minutes域使用5/20,则意味着5分钟触发一次，而25，45等分别触发一次
- , : 表示列出枚举值值。例如：在Minutes域使用5,20，则意味着在5和20分每分钟触发一次
- L : 表示最后，只能出现在DayofWeek和DayofMonth域，如果在DayofWeek域使用5L,意味着在最后的一个星期四触发
- W : 表示有效工作日(周一到周五),只能出现在DayofMonth域，系统将在离指定日期的最近的有效工作日触发事件。例如：在 
DayofMonth使用5W，如果5日是星期六，则将在最近的工作日：星期五，即4日触发。如果5日是星期天，则在6日(周一)触发；如果5日在星LW:这两个字符可以连用，表示在某个月最后一个工作日，即最后一个星期五
- \# : 用于确定每个月第几个星期几，只能出现在DayofMonth域。例如在4#2，表示某月的第二个星期三

<hr />