# k8s-install-binary

- 本文主要介绍如何使用官方提供的二进制包部署K8S（V1.13.3）集群。同时具有以下几大特性

  - TLS双向认证
  - RBAC授权
  - Flannel网络
  - ETCD高可用集群
  - Kube-proxy 启用ipvs

- k8s架构图

  ![](https://www.google.com/url?sa=i&source=images&cd=&cad=rja&uact=8&ved=2ahUKEwjnsZ-jj9LgAhVL7GEKHeReCH0QjRx6BAgBEAU&url=https%3A%2F%2Fdaihainidewo.github.io%2Fblog%2Fk8s-%25E6%259E%25B6%25E6%259E%2584%2F&psig=AOvVaw363YeraP4hj__THBqKJ7wg&ust=1551019085126296)



* 集群分布架构图

  | Role   | Hostname | IP            |
  | ------ | -------- | ------------- |
  | master | master   | 192.168.175.3 |
  | worker | node01   | 192.168.175.4 |
  | worker | node02   | 192.168.175.5 |

* 