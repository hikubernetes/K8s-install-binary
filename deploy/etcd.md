# ETCD集群高可用部署

- ## 下载etcd软件包

  ```
  [root@master software] 
  wget https://github.com/coreos/etcd/releases/download/v3.2.18/etcd-v3.2.18-linux-amd64.tar.gz
  [root@master software]# tar zxf etcd-v3.2.18-linux-amd64.tar.gz
  [root@master software]# cd etcd-v3.2.18-linux-amd64
  [root@master etcd-v3.2.18-linux-amd64]# cp etcd etcdctl /opt/kubernetes/bin/ 
  [root@master etcd-v3.2.18-linux-amd64]# scp etcd etcdctl node01:/opt/kubernetes/bin/
  [root@master etcd-v3.2.18-linux-amd64]# scp etcd etcdctl node02:/opt/kubernetes/bin/
  ```

- ## 创建etcd证书签名请求

  ```
  [root@master ssl]# cd /usr/local/src/ssl
  [root@master ssl]# vim etcd-csr.json
  {
    "CN": "etcd",
    "hosts": [
      "127.0.0.1",
  "192.168.175.3",
  "192.168.175.4",
  "192.168.175.5"
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

- ## 生成etcd证书和私钥

  ```
  [root@master src]# cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
    -ca-key=/opt/kubernetes/ssl/ca-key.pem \
    -config=/opt/kubernetes/ssl/ca-config.json \
    -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
  会生成以下证书文件
  [root@master src]# ls -l etcd*
  -rw-r--r-- 1 root root 1045 Mar  5 11:27 etcd.csr
  -rw-r--r-- 1 root root  257 Mar  5 11:25 etcd-csr.json
  -rw------- 1 root root 1679 Mar  5 11:27 etcd-key.pem
  -rw-r--r-- 1 root root 1419 Mar  5 11:27 etcd.pem
  [root@master ~]# cp etcd*.pem /opt/kubernetes/ssl
  [root@master ~]# scp etcd*.pem node01:/opt/kubernetes/ssl
  [root@master ~]# scp etcd*.pem node02:/opt/kubernetes/ssl
  ```

- ## 设置etcd配置文件

  ```
  [root@master ~]# vim /opt/kubernetes/cfg/etcd.conf
  #[member]
  ETCD_NAME="etcd-node1"
  ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
  #ETCD_SNAPSHOT_COUNTER="10000"
  #ETCD_HEARTBEAT_INTERVAL="100"
  #ETCD_ELECTION_TIMEOUT="1000"
  ETCD_LISTEN_PEER_URLS="https://192.168.175.3:2380"
  ETCD_LISTEN_CLIENT_URLS="https://192.168.175.3:2379,https://127.0.0.1:2379"
  #ETCD_MAX_SNAPSHOTS="5"
  #ETCD_MAX_WALS="5"
  #ETCD_CORS=""
  #[cluster]
  ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.175.3:2380"
  # if you use different ETCD_NAME (e.g. test),
  # set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
  ETCD_INITIAL_CLUSTER="etcd-node1=https://192.168.175.3:2380,etcd-node2=https://192.168.175.4:2380,etcd-node3=https://192.168.175.5:2380"
  ETCD_INITIAL_CLUSTER_STATE="new"
  ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"
  ETCD_ADVERTISE_CLIENT_URLS="https://192.168.175.3:2379"
  #[security]
  CLIENT_CERT_AUTH="true"
  ETCD_CA_FILE="/opt/kubernetes/ssl/ca.pem"
  ETCD_CERT_FILE="/opt/kubernetes/ssl/etcd.pem"
  ETCD_KEY_FILE="/opt/kubernetes/ssl/etcd-key.pem"
  PEER_CLIENT_CERT_AUTH="true"
  ETCD_PEER_CA_FILE="/opt/kubernetes/ssl/ca.pem"
  ETCD_PEER_CERT_FILE="/opt/kubernetes/ssl/etcd.pem"
  ETCD_PEER_KEY_FILE="/opt/kubernetes/ssl/etcd-key.pem"
  ```

- ## 创建ETCD SERVICE

  ```
  [root@master ~]# vim /etc/systemd/system/etcd.service
  [Unit]
  Description=Etcd Server
  After=network.target
  
  [Service]
  Type=simple
  WorkingDirectory=/var/lib/etcd
  EnvironmentFile=-/opt/kubernetes/cfg/etcd.conf
  # set GOMAXPROCS to number of processors
  ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /opt/kubernetes/bin/etcd"
  Type=notify
  
  [Install]
  WantedBy=multi-user.target
  ```



- ## 重新加载系统服务

  ```
  [root@master ~]# systemctl daemon-reload && mkdir /var/lib/etcd/
  [root@master ~]# systemctl enable etcd && systemctl enable etcd
  ```

- ## 分发配置文件和证书到node01和node02

  ```
  [root@master ~]# scp /opt/kubernetes/cfg/etcd.conf node01:/opt/kubernetes/cfg/
  [root@master ~]# scp /etc/systemd/system/etcd.service node01:/etc/systemd/system/
  [root@master ~]# scp /opt/kubernetes/cfg/etcd.conf node02:/opt/kubernetes/cfg/
  [root@master ~]# scp /etc/systemd/system/etcd.service node02:/etc/systemd/system/
  ```

- ## node节点修改配置文件

  注意：node节点不用重新生成证书，直接修改service配置文件中相对应节点角色ip和节点名称，然后启动即可。三个节点要同时启动，只启动一个节点，会失败。

- ## 验证etcd集群是否健康

  ```
  [root@master ~]# etcdctl --endpoints=https://192.168.175.3:2379 \
    --ca-file=/opt/kubernetes/ssl/ca.pem \
    --cert-file=/opt/kubernetes/ssl/etcd.pem \
    --key-file=/opt/kubernetes/ssl/etcd-key.pem cluster-health
  member 435fb0a8da627a4c is healthy: got healthy result from https://192.168.175.3:2379
  member 6566e06d7343e1bb is healthy: got healthy result from https://192.168.175.4:2379
  member ce7b884e428b6c8c is healthy: got healthy result from https://192.168.175.5:2379
  cluster is healthy
  ```