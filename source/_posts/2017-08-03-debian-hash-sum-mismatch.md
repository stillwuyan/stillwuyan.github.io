---
title: Ubuntu/Debian使用代理时apt遇到Hash sum mismatch解决方法
date: 2017-08-03 16:58:28
tags: [Debian, Ubuntu, apt-get]
categories: Technology
---
在Docker中使用Ubuntu 16.04或者Debian 9.x镜像时，`apt install`遇到下面问题：
```
Get:3 http://cdn-fastly.deb.debian.org/debian stretch/main amd64 libgdbm3 amd64 1.8.3-14 [30.0 kB]
Err:3 http://cdn-fastly.deb.debian.org/debian stretch/main amd64 libgdbm3 amd64 1.8.3-14
  Hash Sum mismatch
  Hashes of expected file:
   - SHA256:fbce0e2500aa970ed03665d15822265ff8d31c81927b987ae34e206b9b5ab0b6
   - MD5Sum:4bd924fc8be5471a12d1e0204c74d6c3 [weak]
   - Filesize:30042 [weak]
  Hashes of received file:
   - SHA256:7ef8809e6d91161dce0a4fecb3e34f132dd510446d22edb64c9581d3af5afe0a
   - MD5Sum:4109d27679f5cc5cd63f54719d648519 [weak]
   - Filesize:30042 [weak]
  Last modification reported: Tue, 07 Jun 2016 10:14:24 +0000
```
## 解决方法
在网上搜索`apt hash sum mismatch`，主要有以下几种解决方案：
+ 清除cache缓存并删除下载文件，该方法应该是解决本地缓存内容与repo上不一致导致的问题。但我尝试该方法后问题依然再现。
```
$ sudo apt-get clean
$ sudo rm -rf /var/lib/apt/lists/*
$ sudo apt-get update --fix-missing 
```
+ 更换apt更新源，一般推荐使用163或者aliyun。感觉提速的意义更大，尝试各种源依然无果。
```
http://mirrors.163.com/
http://mirrors.aliyun.com/
```
+ 使用代理上网，由于Proxy的设置问题导致mismatch，需要设置`/etc/apt/apt.conf.d/fixbadproxy`。与我的情况相符，按照下面操作解决问题！
```
$ sudo cat >/etc/apt/apt.conf.d/99fixbadproxy <<EOF
> Acquire::http::Pipeline-Depth "0";
> Acquire::http::No-Cache=True;
> Acquire::BrokenProxy=true;
> EOF
sudo rm -rf /var/lib/apt/lists/*
sudo apt-get update
```

## 参考链接
[stack overflow相关问答](https://stackoverflow.com/a/34272296)
[github上对proxy问题的解决方法](https://gist.github.com/trastle/5722089)
[APT Hash sum mismatch错误的常见解决方法总结](http://blog.csdn.net/chenming_hnu/article/details/54572708)
（总结的很全面）