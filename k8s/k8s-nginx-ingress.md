# `Nginx Ingress`安装以及优化


  `NGINX Ingress`官方的`YAML`文件是以`Deployment`方式运行的，这样就会有坑，`Pod`会到处跑，即使加上`nodeSecetor`，也是会有可能一个主机跑两个，而另外的主机不跑。实际需求中，我们希望在固定的主机上运行它，再在它的前端加一层代理，将外部流量引入进来。

  示意拓朴图:

  ```bash
           |------NGINX----|----K8S-Master03
   ---->SLB|               |----K8S-Master02    
           |------NGINX----|----K8S-Master01

  ```

  ## 准备工作

  ```bash
  # 打上label
  kubectl label node NodeName ingress=true

  # 获取部署YAML
  wget https://github.com/kubernetes/ingress-nginx/blob/master/deploy/static/provider/baremetal/deploy.yaml
  ```

  ## 修改`deploy.yaml`文件

  
  ```yaml
  data:
    # 增加以下
    disable-access-log: "true"
    worker-cpu-affinity: "auto"
```

  修改`Deployment`为`DaemonSet`

  ```yaml
  # 约306行
  ...
  kind: DaemonSet
    ....
    #约359行
    env:
    - name: POD_NAME
      value: ingress-nginx
    - name: POD_NAMESPACE
      value: ingress-nginx 
  ```

  增加节点容忍，不然`Master`节点不会被调度
  ```yaml
  # 约401行
  ...
  serviceAccountName: ingress-nginx
  # 加入
  nodeSelector:
    ingress: "true"
  tolerations:
    - key: "node-role.kubernetes.io/master"
      operator: "Equal"
      value: ""
      effect: "NoSchedule"
  ...
  ```

## 应用就OK了

```bash
kubectl apply -f deploy.yaml

# 验证
kubectl get ds -n ingress-nginx
kubectl get pod -n ingress-nginx -o wide
```