# 二进制部署flannel网络

- ### 为Flannel生成证书

```
[root@master ssl]# vim flanneld-csr.json
{
  "CN": "flanneld",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShenZhen",
      "L": "ShenZhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

- ### 生成证书

```
[root@master ssl]# cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld
```

- ### 分发证书

```
[root@master ssl]# cp flanneld*.pem /opt/kubernetes/ssl/
[root@master ssl]# scp flanneld*.pem 192.168.175.4:/opt/kubernetes/ssl/
[root@master ssl]# scp flanneld*.pem 192.168.175.5:/opt/kubernetes/ssl/
```

- ### 下载Flannel软件包

```
[root@master ~]# cd /opt/kubernetes/software
# wget
 https://github.com/coreos/flannel/releases/download/v0.10.0/flannel-v0.10.0-linux-amd64.tar.gz
[root@master software]# tar zxf flannel-v0.10.0-linux-amd64.tar.gz
[root@master software]# cp flanneld mk-docker-opts.sh /opt/kubernetes/bin/
复制到linux-node2节点
[root@master software]# scp flanneld mk-docker-opts.sh 192.168.175.4:/opt/kubernetes/bin/
[root@master software]# scp flanneld mk-docker-opts.sh 192.168.175.5:/opt/kubernetes/bin/
复制对应脚本到/opt/kubernetes/bin目录下。
[root@master ~]# cd /opt/software/kubernetes/cluster/centos/node/bin/
[root@master bin]# cp remove-docker0.sh /opt/kubernetes/bin/
[root@master bin]# scp remove-docker0.sh 192.168.175.4:/opt/kubernetes/bin/
[root@master bin]# scp remove-docker0.sh 192.168.175.5:/opt/kubernetes/bin/
```

- ### 配置Flannel

```
[root@master ~]# vim /opt/kubernetes/cfg/flannel
FLANNEL_ETCD="-etcd-endpoints=https://192.168.175.3:2379,https://192.168.175.4:2379,https://192.168.175.5:2379"
FLANNEL_ETCD_KEY="-etcd-prefix=/kubernetes/network"
FLANNEL_ETCD_CAFILE="--etcd-cafile=/opt/kubernetes/ssl/ca.pem"
FLANNEL_ETCD_CERTFILE="--etcd-certfile=/opt/kubernetes/ssl/flanneld.pem"
FLANNEL_ETCD_KEYFILE="--etcd-keyfile=/opt/kubernetes/ssl/flanneld-key.pem"
复制配置到其它节点上
[root@master ~]# scp /opt/kubernetes/cfg/flannel 192.168.175.4:/opt/kubernetes/cfg/
[root@master ~]# scp /opt/kubernetes/cfg/flannel 192.168.175.5:/opt/kubernetes/cfg/
```

- ### 设置Flannel系统服务

```
[root@master ~]# vim /usr/lib/systemd/system/flannel.service
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
Before=docker.service

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/flannel
ExecStartPre=/opt/kubernetes/bin/remove-docker0.sh
ExecStart=/opt/kubernetes/bin/flanneld ${FLANNEL_ETCD} ${FLANNEL_ETCD_KEY} ${FLANNEL_ETCD_CAFILE} ${FLANNEL_ETCD_CERTFILE} ${FLANNEL_ETCD_KEYFILE}
ExecStartPost=/opt/kubernetes/bin/mk-docker-opts.sh -d /run/flannel/docker

Type=notify

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
复制系统服务脚本到其它节点上
# scp /usr/lib/systemd/system/flannel.service 192.168.175.4:/usr/lib/systemd/system/
# scp /usr/lib/systemd/system/flannel.service 192.168.175.5:/usr/lib/systemd/system/
```

## Flannel CNI集成

- ### 下载CNI插件

```
https://github.com/containernetworking/plugins/releases
wget https://github.com/containernetworking/plugins/releases/download/v0.7.1/cni-plugins-amd64-v0.7.1.tgz
[root@master ~]# mkdir /opt/kubernetes/bin/cni
[root@master src]# tar zxf cni-plugins-amd64-v0.7.1.tgz -C /opt/kubernetes/bin/cni
[root@master ~]# scp -r /opt/kubernetes/bin/cni/* 192.168.175.4:/opt/kubernetes/bin/cni/
[root@master ~]# scp -r /opt/kubernetes/bin/cni/* 192.168.175.5:/opt/kubernetes/bin/cni/
```

- ### 创建Etcd的key(只需要在master上执行)

```
/opt/kubernetes/bin/etcdctl --ca-file /opt/kubernetes/ssl/ca.pem --cert-file /opt/kubernetes/ssl/flanneld.pem --key-file /opt/kubernetes/ssl/flanneld-key.pem \
      --no-sync -C https://192.168.175.3:2379,https://192.168.175.4:2379,https://192.168.175.5:2379 \
set /kubernetes/network/config '{ "Network": "10.2.0.0/16", "Backend": { "Type": "vxlan", "VNI": 1 }}' >/dev/null 2>&1
```

- ### 启动flannel

```
[root@master ~]# systemctl daemon-reload
[root@master ~]# systemctl enable flannel
[root@master ~]# chmod +x /opt/kubernetes/bin/*
[root@master ~]# systemctl start flannel
```

```
[root@node1 ~]# systemctl daemon-reload
[root@node1 ~]# systemctl enable flannel
[root@node1~]# chmod +x /opt/kubernetes/bin/*
[root@node1 ~]# systemctl start flannel
```

```
[root@node1 ~]# systemctl daemon-reload
[root@node1 ~]# systemctl enable flannel
[root@node1~]# chmod +x /opt/kubernetes/bin/*
[root@node1 ~]# systemctl start flannel
```

- ### 查看服务状态

```
[root@master ~]# systemctl status flannel
```

## 配置Docker使用Flannel

```
[root@master ~]# vim /usr/lib/systemd/system/docker.service
[Unit] #在Unit下面修改After和增加Requires
After=network-online.target firewalld.service flannel.service
Wants=network-online.target
Requires=flannel.service

[Service] #增加EnvironmentFile=-/run/flannel/docker
Type=notify
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd $DOCKER_OPTS
```

- ### 将配置复制到另外两个阶段

```
# scp /usr/lib/systemd/system/docker.service 192.168.175.4:/usr/lib/systemd/system/
# scp /usr/lib/systemd/system/docker.service 192.168.175.5:/usr/lib/systemd/system/
```

- ### 重启Docker

```
[root@master ~]# systemctl daemon-reload
[root@master ~]# systemctl restart docker
[root@node1 ~]# systemctl daemon-reload
[root@node1 ~]# systemctl restart docker
[root@node2 ~]# systemctl daemon-reload
[root@node2 ~]# systemctl restart docker
```

