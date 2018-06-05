---
title: 使用CentOS 7搭建网关服务器
date: 2018-06-05 10:49:43
tags: [iptabels, CentOS]
categories: Technology
---

内网环境通常显示只有一台机器可以访问外网，其他机器如果想访问外网就需要经由该机器。虽然搭建代理服务器的方法更简单，但是要求其他机器显示设置代理信息，如果再使用到其他代理服务器，配置更加复杂。相对而言使用网关服务器，其他机器访问外网要更透明一些。

#### 1. 前期准备

+ 由于Centos7使用firewall作为防火墙，这里需要改为iptables

    ```
    systemctl stop firewalld
    systemctl disable firewalld
    yum install iptables-services
    systemctl start iptables
    systemctl enable iptables
    ```

+ 通常还需要关闭SELINUX

    ```
    vim /etc/selinux/config
    #SELINUX=enforcing
    SELINUX=disabled
    :wq
    setenforce 0 #使配置立即生效
    ```

#### 2. 开启转发支持

```
vim /etc/sysctl.d/99-sysctl.conf
net.ipv4.ip_forward = 1
:wq
```

#### 3. 使用iptables配置转发

+ 建立nat伪装。仅针对eno2网卡收到的192.168.1.0~192.168.1.255网段传过来的数据包。如果将192.168.1.0/24替换为192.168.1.23，则表示只针对IP地址为192.168.1.23传过来的数据包。

    ```
    iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eno2 -j MASQUERADE
    ```

+ 建立转发。所有eno1网卡收到的数据包都可以转发

    ```
    iptables -A FORWARD -i eno1 -j ACCEPT
    ```

+ 建立转发。仅针对192.168.1.0~192.168.1.255网段转发

    ```
    iptables -A FORWARD -s 192.168.1.0/24 -m state --state ESTABLISHED,RELATED -j ACCEPT
    ```

#### 4. 使用iptables配置访问规则

+ 设置默认不转发任何数据包

  ```
  iptables -A FORWARD -j DROP
  ```

+ 允许指定的IP的数据包被转发

  > iptables规则是从上至下匹配规则，因此添加例外时需要使用insert方式插入规则

  ```
  iptables -I FORWARD -s 192.168.1.23 -j ACCEPT
  ```

#### 5. 保存iptables规则

```
service iptables save
```

#### 6. 如果配置iptables后不起作用，可以重启iptables服务试试

````
service iptables restart
````

