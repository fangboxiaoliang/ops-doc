# Kubectl 多集群连接

1. 设置`KUBECONFIG`环境变量

    以下假设有两个配置`kubeconfig`。两个集群的`contexts[i].context.name`和`clusters[i].name`不要一样。

        ```bash
        vi /etc/profile
        # 如果是放在root用户，建议将${HOME}换成/root
        export KUBECONFIG=${HOME}/.kube/config:${HOME}/.kube/config-sz
        ```

2. 通过`kubectl config view --flatten`自动合并

    ```bash
    kubectl config view --flatten
    ```

3. 使用

    虽然有点坑，总算解决了燃眉之急。
    ```bash
    # 指定context与kubeconfig
    kubectl --context sz --kubeconfig=~/.kube/config-sz get nodes
    ```