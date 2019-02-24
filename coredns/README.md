# coredns部署

- ### 下载软件包

  ```
  [root@master coredns]# cd /opt/kubernetes/software/
  [root@master software]# wget https://dl.k8s.io/v1.13.2/kubernetes.tar.gz
  
  ```

- ### 修改yaml文件

  ```
  [root@master software]# cd kubernetes/cluster/addons/dns/coredns/
  [root@master coredns]# ls
  coredns.yaml.base  coredns.yaml.in  coredns.yaml.sed  Makefile  transforms2salt.sed  transforms2sed.sed
  [root@master coredns]# cp /root/coredns.yaml .
  [root@master coredns]# ls
  coredns.yaml  coredns.yaml.base  coredns.yaml.in  coredns.yaml.sed  Makefile  transforms2salt.sed  transforms2sed.sed
  [root@master coredns]# diff coredns.yaml coredns.yaml.base 
  67c67
  <         kubernetes cluster.local. in-addr.arpa ip6.arpa {
  ---
  >         kubernetes __PILLAR__DNS__DOMAIN__ in-addr.arpa ip6.arpa {
  180c180
  <   clusterIP: 10.1.0.2
  ---
  >   clusterIP: __PILLAR__DNS__SERVER__
  ```

  

- ### 部署coredns       

  ```
  [root@master coredns]# kubectl apply -f coredns.yaml
  ```

  

