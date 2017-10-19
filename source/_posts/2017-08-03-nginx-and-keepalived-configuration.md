---
title: Nginx+Keepalived实现高可用负载均衡服务
date: 2017-08-03 09:49:20
tags: [Nginx, Keepalived]
categories: Technology
---
```
docker run -it --privileged=true --name lb debian bash
apt-get update
apt-get install keepalived nginx

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass MrUse
    }
    virtual_ipaddress {
        172.17.0.99
    }
}

# cat /etc/keepalived/keepalived.conf 
global_defs {
   notification_email {
   #  acassen@firewall.loc   # 指定keepalived在发生切换时需要发送email到的对象，一行一个
   #  sysadmin@firewall.loc
   }
   #notification_email_from Alexandre.Cassen@firewall.loc  # 指定发件人
   #smtp_server 192.168.200.1     # smtp 服务器地址 
   #smtp_connect_timeout 30       # smtp 服务器连接超时时间
   router_id LVS_DEVEL            # 标识本节点的字符串,通常为hostname,但不一定非得是hostname,故障发生时,邮件通知会用到
}
###　新增　###
vrrp_script chk_httpd {
    script "/etc/keepalived/check_and_start_httpd.sh"   # apache httpd 服务检测并试图重启
    interval 2                    # 每2s检查一次
    weight -5                     # 检测失败（脚本返回非0）则优先级减少5个值
    fall 3                        # 如果连续失败次数达到此值，则认为服务器已down
    rise 2                        # 如果连续成功次数达到此值，则认为服务器已up，但不修改优先级
}

vrrp_instance VI_1 {              # 实例名称
    state MASTER                  # 可以是MASTER或BACKUP，不过当其他节点keepalived启动时会自动将priority比较大的节点选举为MASTER
    interface eth0                # 节点固有IP（非VIP）的网卡，用来发VRRP包做心跳检测
    virtual_router_id 51          # 虚拟路由ID,取值在0-255之间,用来区分多个instance的VRRP组播,同一网段内ID不能重复;主备必须为一样
    priority 100                  # 用来选举master的,要成为master那么这个选项的值最好高于其他机器50个点,该项取值范围是1-255(在此范围之外会被识别成默认值100)
    advert_int 1                  # 检查间隔默认为1秒,即1秒进行一次master选举(可以认为是健康查检时间间隔)
    authentication {              # 认证区域,认证类型有PASS和HA（IPSEC）,推荐使用PASS(密码只识别前8位)
        auth_type PASS            # 默认是PASS认证
        auth_pass 1111            # PASS认证密码
    }
    virtual_ipaddress {
        192.168.14.166            # 虚拟VIP地址,允许多个,一行一个
    #    192.168.200.17
    }
    ###　新增　###
    track_script {                # 引用VRRP脚本，即在 vrrp_script 部分指定的名字。定期运行它们来改变优先级，并最终引发主备切换。
        chk_httpd          
    }                
}

state BACKUP     # 此值可设置或不设置，只要保证下面的priority不一致即可
interface eth0   # 根据实际情况选择网卡
priority 40      # 此值要一定小于Master机器上的值，最好相差不少于50

# cat /etc/keepalived/check_and_start_httpd.sh 
#!/bin/bash
counter=$(ps -C httpd --no-heading|wc -l)
if [ "${counter}" = "0" ]; then
    service httpd start
    sleep 2
    counter=$(ps -C httpd --no-heading|wc -l)
    if [ "${counter}" = "0" ]; then
        /etc/rc.d/init.d/keepalived stop
    fi
fi 该脚本的目的是用来检测httpd服务是否存在，如果不存在就重启，重启失败就关闭本机keepalived以便VIP切换到Backup机器上。该脚本由keepalived进行调用。
```