---
title: CentOS 7安装Kubernetes
date: 2018-01-18 13:35:23
tags: [Kubernetes, CentOS]
categories: Technology
---

网上搜索K8S的安装教程有很多，但是过程各不相同，只好找到官方说明，照着爬一遍坑试试。在此记下爬坑历程，为以后安装铺好路。

## 1. 安装和配置Docker-CE
参考[官方安装说明](https://docs.docker.com/engine/installation/linux/docker-ce/centos/)，一切都还顺利。

+ 如果以前安装过旧版Docker，似乎无法升级至新版Docker，需要卸载，我本地是新安装的系统，此步略过。
  ```
  sudo yum remove docker docker-common docker-selinux docker-engine
  ```

+ 添加Docker官方repository。
  ```
  sudo yum install -y yum-utils device-mapper-persistent-data lvm2
  sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
  ```

+ 安装Docker-CE。
  ```
  #sudo yum install docker-ce
  ```

  > 注意：由于后面安装的kubeadm版本是v1.9.1，官方声明兼容Docker的最高版本是17.03。如果担心兼容性可以安装指定版本的Docker。
  > 另外，Docker的17.03版本与17.12版本（默认安装）间替换了selinux，因此需要先卸载selinux再安装。安装完成后，需要重启CentOS系统[1]
  > ```
    sudo yum remove docker-ce container-selinux
    sudo rm -rf /var/lib/docker
    yum list docker-ce --showduplicates | sort -r | grep 17.03
    - docker-ce.x86_64            17.03.2.ce-1.el7.centos            docker-ce-stable 
    - docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable 
    - docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable

    sudo yum install --setopt=obsoletes=0 docker-ce-selinux-17.03.2.ce-1.el7.centos docker-ce-17.03.2.ce-1.el7.centos
    reboot now
    systemctl enable docker && systemctl start docker
    ```

+ 设置Docker的Cgroup Driver与K8S的一致。
  > Docker的Cgroup Driver可以使用`docker info`命令查看。
  > K8S的Cgroup Driver参考`/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`文件。
  ```
  cat >> /etc/docker/daemon.json <<EOF
  {
    "exec-opts": ["native.cgroupdriver=systemd"]
  }
  EOF

  systemctl restart docker
  ```

+ 安装完毕后，Docker默认不会运行，而且开机不会自动启动，需要设置systemd服务
  ```
  sudo systemctl enable docker
  sudo systemctl start docker
  ```

+ 使用下面命令查看docker服务状态
  ```
  sudo systemctl status docker
  ```

## 2. 系统设置
安装K8S前，需要对系统做如下设置：

+ 根据官方建议设置K8S的iptables配置文件。
  ```
  cat <<EOF >  /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  EOF

  sysctl --system
  ```

+ 关闭firewalld服务。其实，也可以根据`kubeadm init`的提示设置firewalld许可`6443`和`10250`端口
  ```
  systemctl stop firewalld && systemctl disable firewalld
  ```

+ 关闭swap[2]，然后删除`/etc/fstab`文件中swap自动挂载相关设置。可以使用`cat /proc/swaps`命令查看swap是否关闭。
  ```
  swapoff -a
  ```

## 3. 安装和配置Kubernetes
这里介绍的是使用kubeadm安装K8S（目前还处于beta阶段）。首先安装的是如下三个工具：

+ kubeadm：引导集群的工具。
+ kubelet：核心工作组件，运行在集群各主机上，负责如启动pods和容器等工作。
+ kubectl：操控集群的命令行工具。

由于以上工具相互独立，可以分别安装，因此官方建议安装时要保证版本统一，不同版本混用可能会出现问题。

+ 安装kubeadm，kubelet和kubectl的命令如下：
  ```
  cat <<EOF > /etc/yum.repos.d/kubernetes.repo
  [kubernetes]
  name=Kubernetes
  baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
  enabled=1
  gpgcheck=1
  repo_gpgcheck=1
  gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
  EOF

  setenforce 0
  yum install -y kubelet kubeadm kubectl
  systemctl enable kubelet && systemctl start kubelet
  ```

  > 注意：由于国内环境问题，可能无法访问<https://packages.cloud.google.com>，可考虑使用SSR+ProxyChains。

+ 使用`kubeadm init`命令初始化集群。其中，`--pod-network-cidr=10.244.0.0/16`参数是使用Flannel设置Pod Network；`--apiserver-advertise-address=<ip-master-address>`参数用于设置API Server绑定的网络接口，如果没有设置，将使用默认网关绑定的网络接口。
  ```
  kubeadm init --kubernetes-version=v1.9.1 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=10.1.0.69
  ```

  > 注意：使用`kubeadm init`命令时，会联网查询K8S的版本号，国内网络可能无法连接到google服务器，导致命令执行失败，错误信息如下：
  > ```
      kubeadm init
      unable to get URL "https://dl.k8s.io/release/stable-1.9.txt": Get https://storage.googleapis.com/kubernetes-release/release/stable-1.9.txt: net/http: TLS handshake timeout
    ```
  > 我们可以在命令中指定K8S的版本号，避免联网查询。
  > ```
    kubeadm init --kubernetes-version=v1.9.1
    ```

+ `kubeadm init`过程会连接[Google镜像服务器](https://cloud.google.com)下载很多image，国内网络又会无法连接。可以参考官方的离线解决方案[3]，使用Docker hub获取指定的镜像（参考[网上教程](http://blog.csdn.net/shida_csdn/article/details/78480241)）。

  而且，在Docker hub上可以搜索到很多其他人做好的镜像，版本匹配就可以直接pull。（如[这里](https://hub.docker.com/u/mirrorgooglecontainers/)）

  ```
  docker pull mirrorgooglecontainers/kube-apiserver-amd64:v1.9.1
  docker pull mirrorgooglecontainers/kube-controller-manager-amd64:v1.9.1
  docker pull mirrorgooglecontainers/kube-scheduler-amd64:v1.9.1
  docker pull mirrorgooglecontainers/kube-proxy-amd64:v1.9.1
  docker pull mirrorgooglecontainers/etcd-amd64:3.1.10
  docker pull mirrorgooglecontainers/pause-amd64:3.0
  docker pull mirrorgooglecontainers/k8s-dns-sidecar-amd64:1.14.7
  docker pull mirrorgooglecontainers/k8s-dns-kube-dns-amd64:1.14.7
  docker pull mirrorgooglecontainers/k8s-dns-dnsmasq-nanny-amd64:1.14.7

  docker tag mirrorgooglecontainers/kube-apiserver-amd64:v1.9.1 gcr.io/google_containers/kube-apiserver-amd64:v1.9.1
  docker tag mirrorgooglecontainers/kube-controller-manager-amd64:v1.9.1 gcr.io/google_containers/kube-controller-manager-amd64:v1.9.1
  docker tag mirrorgooglecontainers/kube-scheduler-amd64:v1.9.1 gcr.io/google_containers/kube-scheduler-amd64:v1.9.1
  docker tag mirrorgooglecontainers/kube-proxy-amd64:v1.9.1 gcr.io/google_containers/kube-proxy-amd64:v1.9.1
  docker tag mirrorgooglecontainers/etcd-amd64:3.1.10 gcr.io/google_containers/etcd-amd64:3.1.10
  docker tag mirrorgooglecontainers/pause-amd64:3.0 gcr.io/google_containers/pause-amd64:3.0
  docker tag mirrorgooglecontainers/k8s-dns-sidecar-amd64:1.14.7 gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.7
  docker tag mirrorgooglecontainers/k8s-dns-kube-dns-amd64:1.14.7 gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.7
  docker tag mirrorgooglecontainers/k8s-dns-dnsmasq-nanny-amd64:1.14.7 gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.7
  ```

  > 注意：添加tag时不要漏掉`google_containers`，完整的前缀是`gcr.io/google_containers/`

+ `kubeadm init`命令执行失败后，可能影响到下一次执行该命令，这时需要执行重置命令。

  ```
  kubeadm reset
  ```

+ 配置常规用户如何使用kubectl访问集群。如果不执行下面操作，将无法使用kubectl命令。

  ```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```

+ 安装Pod Network（使用Flannel）。

  ```
  kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
  ```


## 4. 重置集群

+ 重置集群需要先将各node节点从master中移除（master节点也需要执行该操作）。

  ```
  kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
  kubectl delete node <node name>
  ```

+ 移除成功后，在各节点上执行重置命令（含master节点）。

  ```
  kubeadm reset
  ```

  ​

### 参考链接

1. [Docker服务无法启动 - Error starting daemon: error initializing graphdriver: devmapper](http://blog.csdn.net/loophome/article/details/75299068)
2. [best way to disable swap in linux](https://serverfault.com/questions/684771/best-way-to-disable-swap-in-linux)
3. [running kubeadm without an internet connection](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#running-kubeadm-without-an-internet-connection)
4. [使用kubeadm安装Kubernetes 1.9](https://blog.frognew.com/2017/12/kubeadm-install-kubernetes-1.9.html)