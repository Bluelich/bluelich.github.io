---
title: 在macOS上实现多进程任务处理
date: 2017-07-05 21:55:09
tags: [iOS,macOS,Xcode,多进程]
---
## 前言
>前段时间,由于公司业务关系,开始了macOS的开发.总体来说macOS的开发是不太顺利的,主要是因为macOS的UI显示和交互处理和iOS完全不同,非常多的控件需要自定义,默认没有 back layer 等.
后来,因为业务处理需要用到多进程开发就简单的调研了下,多进程的开发开始比较容易的.

## 简介
>对于多进程,主要的实现方式就是xpc,xpc就是系统用底层的 `mach port` 和 `dispatch`等搞的一套高级API.通过xpc,依赖于主进程起一个新的进程,来处理任务,通过xpc的信道来传输消息.
使用xpc有很多优势,连接、休眠以及内存等都是系统自动处理的,我们只需要连接上xpc服务,然后按照自定义协议通信就可以了. <!--more-->

## Go
依赖于主项目创建一个target,选择`XPC Service`,然后设置好`bundle id`,点击`Finish`. Done!
现在你就已经可以和xpc进行通信了.
通信需要通过NSXPCConnection获取到对应进程的remote Object,按照协议,向这个Object发消息,接收回调就可以了.
需要注意的是,进程间通信传递的参数对象需要满足NSSecureCoding,大部分Foundation实现的一些常用的数据类型都满足这个协议,但是满足了NSSecureCoding的不一定都能传递,比如NSError就不行.而且不能传递返回值给对方进程,会导致崩溃,这个不太清楚是什么原因,有想了解的可以深挖.
其他情况下还想传递的对象参数可以在消息传递前把传递的参数类型放到一个集合里,指定传递的参数类型,这样也是可以的.

## 场景
使用xpc通常是多进程服务,给主app提供功能扩展,很多macOS系统App都使用了xpc来实现功能分离和数据隔离,可以通过`find /System/Library/Frameworks -name \*.xpc`来看下macOS系统中有哪些app使用xpc服务,比如说系统的iWork套件和Xcode就用了xpc.
我们可以把项目中的功能模块进行切分,把部分功能放到xpc服务中去,比如说图片处理,文字渲染预处理等.

## Refer
1 : https://objccn.io/issue-14-4/


