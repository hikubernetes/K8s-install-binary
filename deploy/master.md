# 部署k8s集群master节点

- ## 软件包下载

  ```
  [root@master software]# wget https://dl.k8s.io/v1.13.3/kubernetes-server-linux-amd64.tar.gz
  [root@master software]# tar zxf kubernetes-server-linux-amd64.tar.gz
  [root@master software]# cd kubernetes
  [root@master kubernetes]# cp server/bin/kube-apiserver /opt/kubernetes/bin/
  [root@master kubernetes]# cp server/bin/kube-controller-manager /opt/kubernetes/bin/
  [root@master kubernetes]# cp server/bin/kube-scheduler /opt/kubernetes/bin/
  ```

- ## 创建生成CSR的json配置文件

  ```
  [root@master ssl]# cd /usr/local/src/ssl
  [root@master ssl]# vim kubernetes-csr.json
  {
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "192.168.175.3",
      "10.1.0.1",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
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

- ## 生成kubernetes证书和私钥

  ```
  [root@master ssl]# cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
     -ca-key=/opt/kubernetes/ssl/ca-key.pem \
     -config=/opt/kubernetes/ssl/ca-config.json \
     -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
  [root@master ssl]# cp kubernetes*.pem /opt/kubernetes/ssl/
  [root@master ssl]# scp kubernetes*.pem 192.168.175.4:/opt/kubernetes/ssl/
  [root@master ssl]# scp kubernetes*.pem 192.168.175.5:/opt/kubernetes/ssl/
  ```

- ## 创建 kube-apiserver 使用的客户端 token 文件

```
[root@master ssl]#  head -c 16 /dev/urandom | od -An -t x | tr -d ' '
9fcb620d7053c386b945af9b670bcfaf 
[root@master ssl]# vim /opt/kubernetes/ssl/ bootstrap-token.csv
9fcb620d7053c386b945af9b670bcfaf,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
```

- ## 创建基础用户名/密码认证配置

```
[root@master ssl]# vim /opt/kubernetes/ssl/basic-auth.csv
admin,admin,1
readonly,readonly,2
```

- ## 部署Kubernetes API Server

```
[root@master ~]# vim /usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
ExecStart=/opt/kubernetes/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \
  --bind-address=192.168.175.3 \
  --insecure-bind-address=127.0.0.1 \
  --authorization-mode=Node,RBAC \
  --runtime-config=rbac.authorization.k8s.io/v1 \
  --kubelet-https=true \
  --anonymous-auth=false \
  --basic-auth-file=/opt/kubernetes/ssl/basic-auth.csv \
  --enable-bootstrap-token-auth \
  --token-auth-file=/opt/kubernetes/ssl/bootstrap-token.csv \
  --service-cluster-ip-range=10.1.0.0/16 \
  --service-node-port-range=20000-40000 \
  --tls-cert-file=/opt/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/opt/kubernetes/ssl/kubernetes-key.pem \
  --client-ca-file=/opt/kubernetes/ssl/ca.pem \
  --service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
  --etcd-cafile=/opt/kubernetes/ssl/ca.pem \
  --etcd-certfile=/opt/kubernetes/ssl/kubernetes.pem \
  --etcd-keyfile=/opt/kubernetes/ssl/kubernetes-key.pem \
  --etcd-servers=https://192.168.175.3:2379,https://192.168.175.4:2379,https://192.168.175.5:2379 \
  --enable-swagger-ui=true \
  --allow-privileged=true \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/opt/kubernetes/log/api-audit.log \
  --event-ttl=1h \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

- ###  启动API Server服务

```
[root@master ~]# systemctl daemon-reload
[root@master ~]# systemctl enable kube-apiserver
[root@master ~]# systemctl start kube-apiserver
```

- ### 查看API Server服务状态

```
[root@master ~]# systemctl status kube-apiserver
```

- ## 部署Controller Manager服务

```
[root@master ~]# vim /usr/lib/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/kubernetes/bin/kube-controller-manager \
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --allocate-node-cidrs=true \
  --service-cluster-ip-range=10.1.0.0/16 \
  --cluster-cidr=10.2.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem \
  --service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \
  --root-ca-file=/opt/kubernetes/ssl/ca.pem \
  --leader-elect=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

- ### 启动Controller Manager

```
[root@master ~]# systemctl daemon-reload
[root@master scripts]# systemctl enable kube-controller-manager
[root@master scripts]# systemctl start kube-controller-manager
```

- ### 查看服务状态

```
[root@master scripts]# systemctl status kube-controller-manager
```

## 部署Kubernetes Scheduler

```
[root@master ~]# vim /usr/lib/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/kubernetes/bin/kube-scheduler \
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --leader-elect=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

- ### 部署服务

```
[root@master ~]# systemctl daemon-reload
[root@master scripts]# systemctl enable kube-scheduler
[root@master scripts]# systemctl start kube-scheduler
[root@master scripts]# systemctl status kube-scheduler
```