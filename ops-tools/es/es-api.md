# Elasticsearch常用API

## Index API

- 查看所有索引
  
        curl -XGET http://es:9200/_cat/indices?pretty

- 删除指定索引
  
        curl -XDELETE http://es:9200/IndexName

- 索引配置信息
  
        curl -XGET http://es:9200/IndexName/_settings?pretty

- 取消所有索引只读模式

        curl -XPUT -H "Content-type: application/json" http://172.18.61.78:9200/_settings -d '{
                "index.blocks.read_only_allow_delete":"false"
        }'

## 集群API

- 修改默认密码

        curl -XPUT -H "Content-type: application/json" -u elastic  'http://es:9200/_xpack/security/user/elastic/_password'  -d '{
                "password" : "NewPassword"
        }'

- 查看集群健康状况

        curl -XGET http://es:9200/_cluster/health?pretty

- 查询集群详细信息
  
        curl -XGET http://es:9200/_cluster/state?pretty

- 查看集群的节点详细信息

        curl -XGET http://es:9200/_nodes/NodeName?pretty

- 查看阻塞的任务
  
        curl -XGET http://es:9200/_cluster/pending_tasks?pretty

## 模板API

- 查看所有模板
  
        curl -XGET http://es:9200/_template?pretty

- 查看模板详细信息

        curl -XGET http://es:9200/_template/TemplateName?pretty

- 修改模板。 

  一般修改模板时，使用原来的模板创建新的，并修改优先值。先通过GET获取到模板，然后将所有内容复制出来，将修改的东西增加进去，修改新模板的order值，值越高，越优先。

        curl -XPUT -H "Content-type: application/json" http://es:9200/_template/TemplateName -d "{
                "order": 2,
                "template": "graylog_*",
                "settings": {
                  "index": {
            	     "max_result_window": "2000000",
                  },
                  ...
                }

        }"

## 搜索 API

- 检索信息

        curl -XGET http://es:9200/_search?pretty

- 检索指定索引的信息

        curl -XGET http://es:9200/IndexName/_search?pretty

- 检索指定索引指定Type的信息

        curl -XGET http://es:9200/IndexName/TypeName/_search?pretty

- 获取指定ID的文档
        
        curl -XGET http://es:9200/IndexName/TypeName/ID