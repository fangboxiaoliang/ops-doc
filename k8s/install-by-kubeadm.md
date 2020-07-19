# `Kubernetes`集群版基于`Kubeadm`安装

`kubeadm`是官方提供的安装工具，自1.18.5以来，用它来安装非常简单。但官方的指导文档主要是基于单机版的，这里记录下集群版安装思路。

## 一、 集群规划

`ETCD`节点与`Master`节点在同一台主机上。在云上，因为不能自建`Keepalived`+`LVS`实现虚拟IP。所以为了高可用，需要借助云平台的负载均衡产品，但好在现在一般云平台的负载均衡内网流量都不收费。

架构示意图:

```bash
            Master1
    slb-----Master2
            Master3

# K8S控制节点的API地址指向SLB，在SLB上代理后端三台控制节点
```

## 二、 安装

1. 前期准备

    ```bash
    # 设置K8S Repo源
    cat <<EOF > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
    EOF

    # 安装Docker-ce源
    yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

    # 安装相关包,建议指定包版本，不然默认会使用最新稳定版-1
    yum install docker-ce kubeadm-1.18.5 kubectl-1.18.5 kubelet-1.18.5 -y

    # 启动Dokcer
    systemctl enable docker && systemctl start docker


    # 增加docker参数文件
     cat << EOF > /etc/docker/daemon.json
     {
        "exec-opts": ["native.cgroupdriver=systemd"],
         "registry-mirrors": ["https://o9a5ub50.mirror.aliyuncs.com"],
         "log-opts": {
             "max-file": "3",
             "max-size": "10m"
         },
         "insecure-registries": ["docker.hub.com:80"],
         "add-registry": ["docker.hub.com:80"],
         "data-root": "/docker"
     }
     EOF


    # 重启docker
    systemctl restart docker
    ```

2. 初始化第一个控制节点

    ```bash
    # 第一台控制节点
    # 10.0.0.100是内部负载均衡的IP地址
    # 加上--dry-run不实际执行
    # --servcie-cidr与--pod-network-cidr不要和主机网络重叠了
    kubeadm init    --control-plane-endpoint "10.0.0.100:6443" \
                    --kubernetes-version "1.18.5" \
                    --verbose=5 \
                    --image-repository "hub.docker.local:5000" > out.txt
    ```
    以上执行完成后，第一个节点已好，此时只有一个`ETCD`的`Pod`运行,待后续控制节点加入后，`ETCD`将以集群方式运行，这个转换步骤将由`kubeadm`自动完成。

3. 将`SSL`证书分发到待加入的控制节点

    ```bash
    # 1.先在待加入节点创建后pki目录
    mkdir -p /etc/kubernetes/pki/etcd

    # 2. 在第一台控制节点上，将证书复制到待加入的控制节点。其他证书不要复用，kubeadm会自动创建
    scp /etc/kubernetes/pki/ca.*  root@10.0.0.102:/etc/kubernetes/pki
    scp /etc/kubernetes/pki/sa.*  root@10.0.0.102:/etc/kubernetes/pki
    scp /etc/kubernetes/pki/front*  root@10.0.0.102:/etc/kubernetes/pki
    scp /etc/kubernetes/pki/etcd/ca.*  root@10.0.0.102:/etc/kubernetes/pki/etcd

    # 另外一台复制也一样，此处略...
    ```

4. 加入另外的控制节点

    ```bash
    # 在第一台控制节点中的out.txt文件中，有加入控制节点与工作节点的命令，复制在新控制节点上运行即可
    kubeadm join 10.0.0.100:6443 --token 6t2uid.s0vqohvf976nnib8 \
    --discovery-token-ca-cert-hash sha256:9314daf259b95b133a254fd0a8dc41887bc99055c5dfd082a8d41591b18cc98b \
    --control-plane
    ```

5. 创建网络与测试验证

    ```bash
    # 在第一台控制节点上执行
    mkdir -p $HOME/.kube
    cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=v1.18.5"
    kubectl get nodes --all-namespaces -o wide
    kubectl get pods -n kube-system |grep etcd
    # 验证etcd三个节点都是正常运行，不在本机的coredns Pod的IP在本机可以PING通
    ```

## 三、 后续操作

1. 更新证书，可以写入`crontab`默认的证书有效期为1年，`CA`证书10年，下面的命令不能更新`CA`证书

    ```bash
    # 查看当前证书有效期
    kubeadm alpha certs check-expiration

    # 更新所有证书
    kubeadm alpha certs renew all
    ```
