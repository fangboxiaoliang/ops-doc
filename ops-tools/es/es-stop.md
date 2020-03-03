# Elasticsearch正确重启维护

## 加大延迟索引复制时间

```bash
curl -XPUT http://es:9200/_all/_settings -d '{
  "settings": {
    "index.unassigned.node_left.delayed_timeout": "5m"
  }
}'
```

## 通过API，临时禁用索引分片

```bash
curl -XPUT http://es:9200/_cluster/settings -d '{
    "transient" : {
        "cluster.routing.allocation.enable" : "none"
    }
 }'
```

## 停止`Elasticsearch`服务或者进程

```bash
systemctl stop elasticsearch
```

### 重新打开全局负载均衡

```bash
curl -XPUT http://es:9200/_cluster/settings -d '{
    "transient" : {
        "cluster.routing.allocation.enable" : "all"
    }
 }'
```

集群重启的时候关闭`allocation`，是为了防止节点重启时，节点上的分片被重新分配，这个过程是很耗资源的，而且重启一般是很快完成的，当集群快速重启后，其上的分片大部分是可以直接分配来用的，这样也可以加速集群完成分片分配的速度，从而加速重启的速度。

`cluster.routing.allocation.enable`为`none`

- 创建新索引时会成功，但索引状态为`red`，因为无法分配到节点上

`cluster.routing.rebalance.enable`为`none`。

- 索引不会重新负载均衡到其他节点上

- 索引在创建的过程中在哪台主机上，就会一直在该台主机上

- 如果某台主机重启或者`DOWN`，在该主机上的主索引的索引状态将为`red`


