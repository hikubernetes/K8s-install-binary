#### 系统初始化	

* 关闭selinux

  ```
  [root@master ~]# setenforce 0
  [root@master ~]# vim /etc/sysconfig/selinux 
  SELINUX=disabled
  ```

* 关闭firewalld

  ```
  [root@master ~]# systemctl stop firewalld && systemctl disable firewalld
  ```

* 关闭swap

  ```
  [root@master ~]# swapoff -a
  [root@master ~]# vim /etc/fstab 
  #/dev/mapper/centos-swap swap                    swap    defaults        0 0
  ```

* ntp校对时间

  ```
  [root@master ~]# yum install ntp -y
  [root@master ~]# systemctl start ntpd && systemctl enable ntpd
  ```



* 配置内核参数以及加载ipvs

  ```
  [root@master ~]vim /etc/sysctl.conf
  net.ipv6.conf.all.disable_ipv6 = 1
  net.ipv6.conf.default.disable_ipv6 = 1
  net.ipv6.conf.lo.disable_ipv6 = 1
  
  vm.swappiness = 0
  net.ipv4.neigh.default.gc_stale_time=120
  net.ipv4.ip_forward = 1
  
  # see details in https://help.aliyun.com/knowledge_detail/39428.html
  net.ipv4.conf.all.rp_filter=0
  net.ipv4.conf.default.rp_filter=0
  net.ipv4.conf.default.arp_announce = 2
  net.ipv4.conf.lo.arp_announce=2
  net.ipv4.conf.all.arp_announce=2
  
  
  # see details in https://help.aliyun.com/knowledge_detail/41334.html
  net.ipv4.tcp_max_tw_buckets = 5000
  net.ipv4.tcp_syncookies = 1
  net.ipv4.tcp_max_syn_backlog = 1024
  net.ipv4.tcp_synack_retries = 2
  kernel.sysrq = 1
  
  # iptables透明网桥的实现
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  net.bridge.bridge-nf-call-arptables = 1
  ```

  ```
  [root@master ~]cat >/etc/modules-load.d/k8s-ipvs.conf<<EOF
  ip_vs
  ip_vs_rr
  ip_vs_wrr
  ip_vs_sh
  nf_conntrack_ipv4
  EOF
  ```

* 安装docker

  ```
  [root@master ~]# wget http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
  [root@master ~]# yum list docker-ce --showduplicates | sort -r
  [root@master ~]yum install -y docker-ce
  [root@master ~]systemctl start docker && systemctl enable docker
  ```

- 创建部署目录

  ```
  [root@master ~]mkdir -p /opt/kubernetes/{cfg,bin,ssl,log}
  ```

- 下载软件包
  - [官网下载](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.13.md#downloads-for-v1133)，需要能科学上网
  - [我的文件服务器](https://sw.hiecho.cn/k8s/k8s-install-v1.13.3.tar.gz)

### 注意：以上操作所有节点都必须执行，执行完成，重启各个节点。

