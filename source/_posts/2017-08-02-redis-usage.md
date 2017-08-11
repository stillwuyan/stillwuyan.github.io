---
title: Redis使用汇总
date: 2017-08-02 19:11:08
tags: Redis
categories: Technology
---
基于网络资料整理的Redis使用说明，备忘参考。

## 基本操作
+ 启动Redis服务命令`redis-server /etc/redis.conf`（不指定`redis.conf`文件也可以启动服务，但是不建议这样做）。
+ Redis服务启动后可以使用`redis-cli`命令进入命令行交互模式。
  ```
  $ redis-cli
  redis 127.0.0.1:6379> ping
  PONG
  redis 127.0.0.1:6379> set mykey somevalue
  OK
  redis 127.0.0.1:6379> get mykey
  "somevalue"
  ```
+ Redis服务运行过程中，会将数据自动的保存在本地，可以在`redis-cli`中使用`SAVE`命令执行本地保存。关闭Redis服务时使用`SHUTDOWN`命令也会执行本地保存。
  `$ redis-cli shutdown`

## 数据操作
+ 可以在[try.redis.io](https://try.redis.io/)网站上学习数据操作的基本命令。

## 参考链接
1. [Redis quick start](https://redis.io/topics/quickstart)