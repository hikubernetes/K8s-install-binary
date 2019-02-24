# 验证集群

- 启动

  ```
  kubectl run nginx --replicas=2 --image=nginx:alpine --port=80
  kubectl expose deployment nginx --type=NodePort --name=example-service-nodeport
  kubectl expose deployment nginx --name=example-service
  kubectl scale --replicas=3 deployment/nginx
  ```

- 查看状态

  ```
  kubectl get deploy -o wide
  kubectl get pods -o wide
  kubectl get svc -o wide
  kubectl describe svc example-service
  ```

- dns解析

  ```
  kubectl run curl --image=/busybox:latest -i -t
  nslookup kubernetes
  nslookup example-service
  curl example-service
  ```

  