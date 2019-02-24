## 部署kubectl 命令行工具

- ### 准备二进制命令包

```
[root@master software]# wget https://dl.k8s.io/v1.13.3/kubernetes-client-linux-amd64.tar.gz
[root@master software]# tar zxf kubernetes-client-linux-amd64
[root@master ~]# cd kubernetes/client/bin
[root@master bin]# cp kubectl /opt/kubernetes/bin/
[root@master bin]# scp kubectl node01:/opt/kubernetes/bin/
[root@master bin]# scp kubectl node02:/opt/kubernetes/bin/
```

- ### 创建 admin 证书签名请求

```
[root@master ~]# cd /usr/local/src/ssl/
[root@master ssl]# vim admin-csr.json
{
  "CN": "admin",
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
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
```

- ### 生成 admin 证书和私钥：

```
[root@master ssl]# cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes admin-csr.json | cfssljson -bare admin
[root@master ssl]# ls -l admin*
-rw-r--r-- 1 root root 1009 Mar  5 12:29 admin.csr
-rw-r--r-- 1 root root  229 Mar  5 12:28 admin-csr.json
-rw------- 1 root root 1675 Mar  5 12:29 admin-key.pem
-rw-r--r-- 1 root root 1399 Mar  5 12:29 admin.pem

[root@master src]# mv admin*.pem /opt/kubernetes/ssl/
```

- ### 设置集群参数

```
[root@master src]# kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://192.168.175.3:6443
Cluster "kubernetes" set.
```

- ### 设置客户端认证参数

```
[root@master src]# kubectl config set-credentials admin \
   --client-certificate=/opt/kubernetes/ssl/admin.pem \
   --embed-certs=true \
   --client-key=/opt/kubernetes/ssl/admin-key.pem
User "admin" set.
```

- ### 设置上下文参数

```
[root@master src]# kubectl config set-context kubernetes \
   --cluster=kubernetes \
   --user=admin
Context "kubernetes" created.
```

- ### 设置默认上下文

```
[root@master src]# kubectl config use-context kubernetes
Switched to context "kubernetes".
```

- ### 使用kubectl工具

```
[root@master ~]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-1               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"} 
```

- ### node节点部署kubectl

  ```
  [root@master ~]# scp -r .kube/ node01:/root
  [root@master ~]# scp -r .kube/ node02:/root
  [root@node02 ~]# kubectl get cs
  NAME                 STATUS    MESSAGE              ERROR
  scheduler            Healthy   ok                   
  controller-manager   Healthy   ok                   
  etcd-0               Healthy   {"health": "true"}   
  etcd-2               Healthy   {"health": "true"}   
  etcd-1               Healthy   {"health": "true"}   
  [root@node01 ~]# kubectl get cs
  NAME                 STATUS    MESSAGE              ERROR
  scheduler            Healthy   ok                   
  controller-manager   Healthy   ok                   
  etcd-0               Healthy   {"health": "true"}   
  etcd-2               Healthy   {"health": "true"}   
  etcd-1               Healthy   {"health": "true"}   
  ```

  