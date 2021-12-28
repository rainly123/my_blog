---
title: "golang避坑之gorm默认0值"
date: 2021-09-16T15:39:00+08:00
description: PHP，golang工程师，项目管理，软件架构
toc: true
draft: true
---

## 问题
gorm是我们在golang开发中操作数据库的常用框架，但是这个框架有一个十分常见的坑，当我们在做updates操作的时候，会将不需要修改的字段置为0值或空值，原因是传给updates函数的参数是一个struct类型，而updates函数内部会利用反射将struct类型的字段解析成mysql对应的字段，很显然，如果你没有对某些字段赋值，那么updates内部就会将这些字段视做0值或控制，最后在执行update语句的时候就修改了我们不想修改的字段。

## 解决方案
- 直接传map类型
这样的话，updates内部通过反射得到的字段就是我们想要的字段了
- 标签json中添加omitempty字段，如：json:"cn_name,omitempty"
这种情况适用于字段很多的情况下的struct，手动转map的工作量比较大，这时候我们可以先转成json，通过添加omitempty字段，为0和空值的字段就不会出现在json数据内，接下来json转map就可以得到我们想要的字段了。