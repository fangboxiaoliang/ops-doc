# CoreDNS 配置示例

```conf
.:53 {
        hosts {
          172.18.171.110  host1
          172.18.203.237  host2
          172.18.203.240  test.testdomain.cn
          cache 60
          reload 1m
          fallthrough
        }
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           upstream
           fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
  # test.io将由内部DNS解析
  test.io:53 {
    errors
    cache 30
    reload
    forward . 173.18.171.109
  }
```
注意点:

-  不要绑定已存在的域名的A记录，比如上面的配置文件中，`test.io`将由173.18.171.109外部的DNS解析，则不能在`hosts`字段中出现以`test.io`域的主机名。例如`hostname.test.io`。这样，`CoreDNS`将始终交由外部DNS解析

- 有些版本的`CoreDNS`修改`corefile`后重载会报错，需要手动删除掉老的`Pod`。