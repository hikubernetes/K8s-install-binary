# k8s-install-binary

- ##### 本文主要介绍如何使用官方提供的二进制包部署K8S（V1.13.3）集群。同时具有以下几大特性

  - TLS双向认证
  - RBAC授权
  - Flannel网络
  - ETCD高可用集群
  - Kube-proxy 启用ipvs


* ##### 集群分布架构图

  | Role   | Hostname | IP            |
  | ------ | -------- | ------------- |
  | master | master   | 192.168.175.3 |
  | worker | node01   | 192.168.175.4 |
  | worker | node02   | 192.168.175.5 |

* ##### Realease Version

  * Os：centos7.5
  * Etcd : v3.3.1
  * Kubernetes: v1.13.3
  * Flannel : v0.10.0
  * Docker: 18.09.2-ce

* ##### 架构介绍

  * Etcd 三节点高可用部署
  * Apiserver，Controller-manager，Scheduler，部署在192.168.175.3
  * Kubelet，Kube-proxy部署在192.168.175.4,192.168.175.5
  * 同时Kube-proxy启用ipvs

* ##### k8s架构图

  ![](https://github.com/hikubernetes/k8s-install-binary/blob/master/images/k8s%E6%9E%B6%E6%9E%84.png)

  

#  k8s部署链接

- [集群初始化](https://github.com/hikubernetes/K8s-install-binary/blob/master/deploy/init.md)
- [CA证书制作](https://github.com/hikubernetes/K8s-install-binary/blob/master/deploy/ca-make.md)
- [ETCD集群部署](https://github.com/hikubernetes/K8s-install-binary/blob/master/deploy/etcd.md)
- [Master节点部署](https://github.com/hikubernetes/K8s-install-binary/blob/master/deploy/master.md)
- [Node节点部署](https://github.com/hikubernetes/K8s-install-binary/blob/master/deploy/node.md)
- [Flannel网络二进制部署](https://github.com/hikubernetes/K8s-install-binary/blob/master/deploy/flannel.md)
- [验证集群](https://github.com/hikubernetes/K8s-install-binary/blob/master/deploy/check.md)

# plugin 部署

- [Coredns部署](https://github.com/hikubernetes/K8s-install-binary/blob/master/coredns/README.md)

- [Flannel部署](https://github.com/hikubernetes/K8s-install-binary/blob/master/flannel/README.md)
- [ingress-nginx部署](https://github.com/hikubernetes/K8s-install-binary/blob/master/ingress-nginx/README.md)

