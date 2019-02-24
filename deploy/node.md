## 部署kubelet

- ### 二进制包准备

```
[root@master ~]# cd /usr/local/src/kubernetes/server/bin/
[root@master bin]# cp kubelet kube-proxy /opt/kubernetes/bin/
[root@master bin]# scp kubelet kube-proxy node01:/opt/kubernetes/bin/
[root@master bin]# scp kubelet kube-proxy node02:/opt/kubernetes/bin/
```

- ### 创建角色绑定

```
[root@master ~]# kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
clusterrolebinding "kubelet-bootstrap" created
```

- ### 创建 kubelet bootstrapping kubeconfig 文件 设置集群参数

```
[root@master ~]# kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://192.168.175.3:6443 \
   --kubeconfig=bootstrap.kubeconfig
Cluster "kubernetes" set.
```

- ### 设置客户端认证参数

```
[root@master ~]# kubectl config set-credentials kubelet-bootstrap \
   --token=9fcb620d7053c386b945af9b670bcfaf \
   --kubeconfig=bootstrap.kubeconfig   
User "kubelet-bootstrap" set.
```

- ### 设置上下文参数

```
[root@master ~]# kubectl config set-context default \
   --cluster=kubernetes \
   --user=kubelet-bootstrap \
   --kubeconfig=bootstrap.kubeconfig
Context "default" created.
```

- ### 选择默认上下文

```
[root@master ~]# kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
Switched to context "default".
[root@master ~]# cp bootstrap.kubeconfig /opt/kubernetes/cfg
[root@master ~]# scp bootstrap.kubeconfig 192.168.175.4:/opt/kubernetes/cfg
[root@master ~]# scp bootstrap.kubeconfig 192.168.175.5:/opt/kubernetes/cfg
```

- ### 部署kubelet 设置CNI支持

```
[root@node01 ~]# mkdir -p /etc/cni/net.d
[root@node01 ~]# vim /etc/cni/net.d/10-default.conf
{
        "name": "flannel",
        "type": "flannel",
        "delegate": {
            "bridge": "docker0",
            "isDefaultGateway": true,
            "mtu": 1400
        }
}

[root@node02 ~]# mkdir -p /etc/cni/net.d
[root@node02 ~]# vim /etc/cni/net.d/10-default.conf
{
        "name": "flannel",
        "type": "flannel",
        "delegate": {
            "bridge": "docker0",
            "isDefaultGateway": true,
            "mtu": 1400
        }
}
```

- ### 创建kubelet目录

```
[root@node01 ~]# mkdir /var/lib/kubelet
[root@node02 ~]# mkdir /var/lib/kubelet
```

- ### 创建kubelet服务配置

```
[root@k8s-node1 ~]# vim /usr/lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/opt/kubernetes/bin/kubelet \
  --address=192.168.175.4 \
  --hostname-override=192.168.175.4 \
  --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.0 \
  --experimental-bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \
  --kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \
  --cert-dir=/opt/kubernetes/ssl \
  --network-plugin=cni \
  --cni-conf-dir=/etc/cni/net.d \
  --cni-bin-dir=/opt/kubernetes/bin/cni \
  --cluster-dns=10.1.0.2 \
  --cluster-domain=cluster.local. \
  --hairpin-mode hairpin-veth \
  --allow-privileged=true \
  --fail-swap-on=false \
  --logtostderr=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log
Restart=on-failure
RestartSec=5
```

```
[root@k8s-node1 ~]# scp /usr/lib/systemd/system/kubelet.service node2:/usr/lib/systemd/system/
注意：修改对service ip
```

- ### 启动Kubelet

```
[root@node01 ~]# systemctl daemon-reload
[root@node01 ~]# systemctl enable kubelet
[root@node01 ~]# systemctl start kubelet
```

```
[root@node02 ~]# systemctl daemon-reload
[root@node02 ~]# systemctl enable kubelet
[root@node02 ~]# systemctl start kubelet
```

- ### 查看服务状态

```
[root@node01 ~]# systemctl status kubelet
```

- ### 查看csr请求 注意是在master上执行。

```
[root@master ~]# kubectl get csr
```

批准kubelet 的 TLS 证书请求

```
[root@master ~]# kubectl get csr|grep 'Pending' | awk 'NR>0{print $1}'| xargs kubectl certificate approve
```

执行完毕后，查看节点状态已经是Ready的状态了

```
[root@master kubernetes-server-linux-amd64]# kubectl get node
NAME            STATUS   ROLES    AGE    VERSION
192.168.175.4   Ready    <none>   5d1h   v1.13.3
192.168.175.5   Ready    <none>   5d1h   v1.13.3
```



## 部署Kubernetes Proxy

- ### 配置kube-proxy使用LVS

```
[root@node01 ~]# yum install -y ipvsadm ipset conntrack
[root@node02 ~]# yum install -y ipvsadm ipset conntrack
```

- ### 创建 kube-proxy 证书请求

```
[root@master ssl]# cd /usr/local/src/ssl/
[root@master ssl]# vim kube-proxy-csr.json
{
  "CN": "system:kube-proxy",
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
   -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

- ### 分发证书到所有Node节点

```
[root@master ssl]# cp kube-proxy*.pem /opt/kubernetes/ssl/
[root@master ssl]# scp kube-proxy*.pem 192.168.175.4:/opt/kubernetes/ssl/
[root@master ssl]# scp kube-proxy*.pem 192.168.175.5:/opt/kubernetes/ssl/
```

- ### 创建kube-proxy配置文件

```
[root@master ssl]# kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://192.168.175.3:6443 \
   --kubeconfig=kube-proxy.kubeconfig
Cluster "kubernetes" set.

[root@master ssl]# kubectl config set-credentials kube-proxy \
   --client-certificate=/opt/kubernetes/ssl/kube-proxy.pem \
   --client-key=/opt/kubernetes/ssl/kube-proxy-key.pem \
   --embed-certs=true \
   --kubeconfig=kube-proxy.kubeconfig
User "kube-proxy" set.

[root@master ssl]# kubectl config set-context default \
   --cluster=kubernetes \
   --user=kube-proxy \
   --kubeconfig=kube-proxy.kubeconfig
Context "default" created.

[root@master ssl]# kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
Switched to context "default".
```

- ### 分发kubeconfig配置文件

```
[root@master ssl]# cp kube-proxy.kubeconfig /opt/kubernetes/cfg/
[root@master ssl]# scp kube-proxy.kubeconfig 192.168.175.4:/opt/kubernetes/cfg/
[root@master ssl# scp kube-proxy.kubeconfig 192.168.175.5:/opt/kubernetes/cfg/
```

- ### 创建kube-proxy服务配置

```
[root@node01 bin]# mkdir /var/lib/kube-proxy

[root@node01 ~]# vim /usr/lib/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/opt/kubernetes/bin/kube-proxy \
  --bind-address=192.168.175.4 \
  --hostname-override=192.168.175.4 \
  --kubeconfig=/opt/kubernetes/cfg/kube-proxy.kubeconfig \
--masquerade-all \
  --feature-gates=SupportIPVSProxyMode=true \
  --proxy-mode=ipvs \
  --ipvs-min-sync-period=5s \
  --ipvs-sync-period=5s \
  --ipvs-scheduler=rr \
  --logtostderr=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log

Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

 启动Kubernetes Proxy
[root@node01 ~]# systemctl daemon-reload
[root@node01 ~]# systemctl enable kube-proxy
[root@node01 ~]# systemctl start kube-proxy
```

```
[root@node02 bin]# mkdir /var/lib/kube-proxy

[root@node02 ~]# vim /usr/lib/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/opt/kubernetes/bin/kube-proxy \
  --bind-address=192.168.175.5 \
  --hostname-override=192.168.175.5 \
  --kubeconfig=/opt/kubernetes/cfg/kube-proxy.kubeconfig \
--masquerade-all \
  --feature-gates=SupportIPVSProxyMode=true \
  --proxy-mode=ipvs \
  --ipvs-min-sync-period=5s \
  --ipvs-sync-period=5s \
  --ipvs-scheduler=rr \
  --logtostderr=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log

Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

 启动Kubernetes Proxy
[root@node02 ~]# systemctl daemon-reload
[root@node02 ~]# systemctl enable kube-proxy
[root@node02 ~]# systemctl start kube-proxy
```

- ### 查看服务状态 查看kube-proxy服务状态

```
[root@node01 scripts]# systemctl status kube-proxy

检查LVS状态
[root@node01 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.1.0.1:443 rr persistent 10800
  -> 192.168.175.3:6443           Masq    1      0          0         
```

使用下面的命令可以检查状态：

```
[root@master ssl]#  kubectl get node
NAME            STATUS    ROLES     AGE       VERSION
192.168.175.4   Ready     <none>    22m       v1.10.1
192.168.175.5   Ready     <none>    3m        v1.10.1
```