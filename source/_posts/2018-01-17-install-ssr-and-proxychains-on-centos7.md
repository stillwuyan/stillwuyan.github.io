---
title: CentOS 7安装SSR Client和ProxyChains
date: 2018-01-17 18:07:33
tags: [SSR, Proxychains, CentOS]
categories: Technology
---

SSR是个神器，但是它默认提供的是socks5代理，而Linux系统下更多是使用http/https代理，通常设置一下环境变量就可以了。（当然如果应用默认不使用该变量，还需要在应用内进行设置）

```
export http_proxy=http://ip:port
export https_proxy=http://ip:port
```

直到我发现另一个神器：ProxyChains，它支持http，socks5等常用代理方式，可以构建多级代理，而且它不影响全局环境（不需要设置环境变量），只需要在执行的应用命令前加上`proxychains`即可。

下面介绍在CentOS 7中安装和使用SSR与ProxyChains Client的步骤：

## 1. 安装SSR Client 

SSR相对于SS增强了通信成功性，推荐使用。

+ 下载SSR源码

  ```
  cd ~/Downloads
  git clone -b manyuser https://github.com/shadowsocksrr/shadowsocksr.git
  ```

+ 新建配置文件，添加server连接相关信息。

  ```
  cat > /etc/shadowsocks.json <<EOF
  {
  "server":"12.34.56.78",
  "server_ipv6":"::",
  "server_port":8388,
  "local_address":"127.0.0.1",
  "local_port":1080,
  "password":"happy2017",
  "timeout":300,
  "udp_timeout":60,
  "method":"aes-128-ctr",
  "protocol":"auth_aes128_md5",
  "protocol_param":"",
  "obfs":"tls1.2_ticket_auth",
  "obfs_param":"",
  "fast_open":false,
  "workers":1
  }
  EOF

  ```

+ 启动daemon形式启动SSR Client，可以在`/var/log/shadowsocksr.log`文件中查看日志信息。

  ```
  cd shadowsocksr/shadowsocks
  python local.py -c /etc/shadowsocks.json -d start
  tail -f /var/log/shadowsocksr.log
  ```

+ 停止SSR Client。

  ```
  python local.py -c /etc/shadowsocks.json -d stop
  ```

+ 修改配置文件后，需要重启SSR Client新配置才能生效。

## 2. 安装ProxyChains

proxychains-ng是proxychains的加强版，主要有以下功能和不足：

+ 支持http/https/socks4/socks5
+ 支持认证
+ 远端dns查询
+ 多级代理模式
+ 不支持udp/ICMP转发
+ 少部分程序和在后台运行的可能无法代理

安装和使用步骤如下：

+ 下载proxychains-ng的源码。

  ```
  cd ~/Downloads
  git clone https://github.com/rofl0r/proxychains-ng
  ```

+ 编译安装。

  ```
  cd proxychains-ng
  ./configure --prefix=/usr --sysconfdir=/etc
  make && make install
  ```

+ 安装proxychains.conf配置文件，配置文件默认安装在`/etc/proxychains.conf`

  ```
  make install-config
  ```

+ 在配置文件中添加代理服务器信息。

  ```
  [ProxyList]
  socks5  127.0.0.1 1086
  ```

+ 使用`proxychains4`命令上网

  ```
  proxychains4 curl www.baidu.com
  ```

+ 也可以使用`proxychains4`打开shell，然后在shell中执行的程序默认都会使用proxychains的代理。

  ```
  proxychains4 -q /bin/bash
  ```

### 参考链接

+ [ShadowsocksR Clients and Server](https://dcamero.azurewebsites.net/shadowsocksr.html)
+ [通过ProxyChains-NG实现终端下任意应用代理](https://www.hi-linux.com/posts/48321.html)