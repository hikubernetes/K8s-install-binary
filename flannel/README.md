# YAML部署flannel网络

- 下载flannel

  ```
  [root@master ~]# wget https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml
  ```

  - 修改配置

    ```
    net-conf.json: |
        {
          "Network": "10.1.0.0/16",
          "Backend": {
            "Type": "vxlan"
          }
        }
        
    注：此处的配置要会写入etcd集群
    ```

  - 修改镜像

    ```
    image: registry.cn-shanghai.aliyuncs.com/gcr-k8s/flannel:v0.10.0-amd64
    ```

  - 多网卡

    ```
    如果Node有多个网卡的话，参考flannel issues 39701，
    https://github.com/kubernetes/kubernetes/issues/39701
    目前需要在kube-flannel.yml中使用--iface参数指定集群主机内网网卡的名称，
    否则可能会出现dns无法解析。容器无法通信的情况，需要将kube-flannel.yml下载到本地，
    flanneld启动参数加上--iface=<iface-name>
        containers:
          - name: kube-flannel
            image: registry.cn-shanghai.aliyuncs.com/gcr-k8s/flannel:v0.10.0-amd64
            command:
            - /opt/bin/flanneld
            args:
            - --ip-masq
            - --kube-subnet-mgr
            - --iface=eth1
    ```

- 启动

  ```
  kubectl apply -f kube-flannel.yml
  ```

- 查看

  ```
  kubectl get pods -n kube-system
  kubectl get svc -n kube-system
  ```

  