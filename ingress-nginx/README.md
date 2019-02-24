# YAML部署ingress-nginx

- ### 部署

  ```
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
  ```

- ### 查看服务

  ```
  [root@master ~]# kubectl get pod --all-namespaces | grep ingress
  ingress-nginx   nginx-ingress-controller-68db76b4db-g7nxp   1/1     Running   0          6h27m 
  [root@master ~]# kubectl get deployment --all-namespaces | grep ingress   
  ingress-nginx   nginx-ingress-controller   1/1     1            1           6h27m
  ```

  