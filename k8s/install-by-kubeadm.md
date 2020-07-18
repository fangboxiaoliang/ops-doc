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

    # 安装相关包
    yum install docker-ce kubeadm kubectl kubelet -y

    # 启动Dokcer
    systemctl enable docker && systemctl start docker


    # 增加docker参数文件
     cat << EOF > /etc/docker/daemon.json
     {
         "registry-mirrors": ["https://o9a5ub50.mirror.aliyuncs.com"],
         "log-opts": {
             "max-file": "3",
             "max-size": "10m"
         },
         "insecure-registries": ["172.30.0.0/16", "docker.hub.com:80"],
         "add-registry": ["docker.hub.com:80"]
     }
     EOF


    # 重启docker
    systemctl restart docker
    ```

2. 创建`ETCD`集群

    ```bash
    # 修改默认kubelt文件
    cat << EOF > /etc/systemd/system/kubelet.service.d/20-etcd-service-manager.conf
    [Service]
    ExecStart=
    #  Replace "systemd" with the cgroup driver of your container runtime. The default value in the kubelet is "cgroupfs".
    ExecStart=/usr/bin/kubelet --address=127.0.0.1 --pod-manifest-path=/etc/kubernetes/manifests --cgroup-driver=systemd
    Restart=always
    EOF

    systemctl daemon-reload
    systemctl restart kubelet

    # 2. 准备etcd的kubeadmin配置文件以及证书相关
    # 使用 IP 或可解析的主机名替换 HOST0、HOST1 和 HOST2
    export HOST0=10.0.0.6
    export HOST1=10.0.0.7
    export HOST2=10.0.0.8

    # 创建临时目录来存储将被分发到其它主机上的文件
    mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/

    ETCDHOSTS=(${HOST0} ${HOST1} ${HOST2})
    NAMES=("infra0" "infra1" "infra2")

    for i in "${!ETCDHOSTS[@]}"; do
    HOST=${ETCDHOSTS[$i]}
    NAME=${NAMES[$i]}
    cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml
    apiVersion: "kubeadm.k8s.io/v1beta2"
    kind: ClusterConfiguration
    etcd:
        local:
            serverCertSANs:
            - "${HOST}"
            peerCertSANs:
            - "${HOST}"
            extraArgs:
                initial-cluster: infra0=https://${ETCDHOSTS[0]}:2380,infra1=https://${ETCDHOSTS[1]}:2380,infra2=https://${ETCDHOSTS[2]}:2380
                initial-cluster-state: new
                name: ${NAME}
                listen-peer-urls: https://${HOST}:2380
                listen-client-urls: https://${HOST}:2379
                advertise-client-urls: https://${HOST}:2379
                initial-advertise-peer-urls: https://${HOST}:2380
    EOF
    done

    # 创建证书CA机构，会生成以下两个文件
    # - /etc/kubernetes/pki/etcd/ca.crt
    # - /etc/kubernetes/pki/etcd/ca.key
    kubeadm init phase certs etcd-ca

    ```

3. 初始化第一个控制节点

    ```bash
    # 第一台控制节点
    # 10.0.0.100是内部负载均衡的IP地址
    kubeadm init    --control-plane-endpoint "10.0.0.100:6443" \
                    --kubernetes-version "1.18.5" \
                    --service-cidr "172.16.0.0/16" \
                    --pod-network-cidr "10.1.0.0/16" \
                    --image-repository "hub.docker.local:5000"
    ```

2. 控制节点加入

## 三、 后续操作

1. 更新证书，可以写入`crontab`默认的证书有效期为1年

    ```bash
    # 查看当前证书有效期
    kubeadm alpha certs check-expiration

    # 更新所有证书
    kubeadm alpha certs renew all
    ```
