# CA证书制作

- ### 安装cfssl

```
[root@master ~]# cd /opt/kubernetes/software/
[root@master src]# wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
[root@master src]# wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
[root@master src]# wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
[root@master src]# chmod +x cfssl*
[root@master src]# mv cfssl-certinfo_linux-amd64 /opt/kubernetes/bin/cfssl-certinfo
[root@master src]# mv cfssljson_linux-amd64  /opt/kubernetes/bin/cfssljson
[root@master src]# mv cfssl_linux-amd64  /opt/kubernetes/bin/cfssl
复制cfssl命令文件到node01和node02节点。如果实际中多个节点
[root@master ~]# scp /opt/kubernetes/bin/cfssl* node01:/opt/kubernetes/bin
[root@master ~]# scp /opt/kubernetes/bin/cfssl* node02:/opt/kubernetes/bin
```

- ### 初始化cfssl

  ```
  [root@master src]# mkdir ssl && cd ssl
  [root@master ssl]# cfssl print-defaults config > config.json
  [root@master ssl]# cfssl print-defaults csr > csr.json
  ```

- ### 创建用来生成ca证书的json配置文件

  ```
  [root@master ssl]# vim ca-config.json
  {
    "signing": {
      "default": {
        "expiry": "8760h"
      },
      "profiles": {
        "kubernetes": {
          "usages": [
              "signing",
              "key encipherment",
              "server auth",
              "client auth"
          ],
          "expiry": "8760h"
        }
      }
    }
  }
  ```

- ### 创建用来生成ca证书签名请求的（csr）json配置文件

  ```linux
  [root@master ssl]# vim ca-csr.json
  {
    "CN": "kubernetes",
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

- ### 生成CA证书（ca.pem）和秘钥（ca-key.pem）

  ```
  [root@ master ssl]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca
  [root@ master ssl]# ls -l ca*
  -rw-r--r-- 1 root root  290 Mar  4 13:45 ca-config.json
  -rw-r--r-- 1 root root 1001 Mar  4 14:09 ca.csr
  -rw-r--r-- 1 root root  208 Mar  4 13:51 ca-csr.json
  -rw------- 1 root root 1679 Mar  4 14:09 ca-key.pem
  -rw-r--r-- 1 root root 1359 Mar  4 14:09 ca.pem
  ```

- ### 分发证书

  ```
  [root@ master ssl]# cp ca.csr ca.pem ca-key.pem ca-config.json /opt/kubernetes/ssl
  SCP证书node01和node02节点
  [root@ master ssl]# scp ca.csr ca.pem ca-key.pem ca-config.json node01:/opt/kubernetes/ssl 
  [root@ master ssl]# scp ca.csr ca.pem ca-key.pem ca-config.json node02:/opt/kubernetes/ssl
  ```