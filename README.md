# 目录

* [前言](README.md)
* [Linux](linux/README.md)
  * [系统调优](linux/linux-optization.md)
  * [Shell技巧](linux/shell.md)
  
* [基础服务](baseservices/README.md)
  * [本地`Repo`源](baseservices/local-repo.md)
  * [DNS](baseservices/named.md)
  * [Logratate](baseservices/lograte.md)
  * [Rsync](baseservices/rsync.md)
  * NGINX/OpenResty
    * [NGINX常用内置变量](baseservices/nginx-variables.md)
    * [NGINX配置文件解读](baseservices/nginx-conf.md)
    * [NGINX FAQ](baseservices/nginx-faq.md)
    * [OpenRestry灵活限速](baseservices/openresty-lua-limit.md)
    * [OpenResty后端健康检查](baseservices/openresty-upstream-check.md)

* 消息队列
  * [RabbitMQ](queue/rabbitmq-prod.md)

* [数据库](database/README.md)
  * [MySQL Debug](databases/mysql_status.md)
  * [MySQL备份恢复](databases/mysql_xtrabackup.md)
  * [SQLServer系统函数](databases/mssql_sp.md)
  * MongoDB
    * [Mongo基础信息汇总](databases/mongo.md)
    * [MongoDB复制集](databases/mongo_repl.md)
    * [MongoDB分片](databases/mongo_shard.md)
    * [MongoDB Debug](databases/mongo_debug.md)

* [生态工具](ops-tools/README.md)
  * Zabbix监控模板
    * [MSSQLServer](https://github.com/MantasTumenas/Zabbix-template-for-Microsoft-SQL-Server)
    * [Reids](https://github.com/oscm/zabbix/tree/master/redis)
    * [MongoDB](https://github.com/omni-lchen/zabbix-mongodb) 不支持4.0
    * [RabbitMQ](https://github.com/jasonmcintosh/rabbitmq-zabbix)
  * [GitLab CI/CD](ops-tools/gitlab-ci.md)
  * GrayLog日志收集
    * [介绍](ops-tools/graylog/README.md)
    * [横向扩展节点](ops-tools/graylog/add-node.md)
    * [实践-收集Linux系统日志](ops-tools/graylog/linux.md)
    * [实践-收集Nginx访问日志](ops-tools/graylog/nginx.md)
    * [实践-基于Stream与Rule实现日志告警](ops-tools/graylog/alert.md)
    * [实践-整合Kibana](ops-tools/graylog/kibana.md)
  * Ceph分布式文件系统
    * [介绍](ops-tools/ceph/README.md)
    * [简单调试](ops-tools/ceph/ceph-debug.md)
    * [实践-K8S基于Rook部署Ceph](ops-tools/ceph/k8s-install-ceph.md)
  * [Goreplay流量镜像](https://github.com/buger/goreplay)
  * Elasticsearch全文检索
    * [Elasticsearch重要配置项](ops-tools/es/es-conf.md)
    * [Elasticsearch常用API](ops-tools/es/es-api.md)
    * [Elasticsearch正常关闭/维护](ops-tools/es/es-stop.md)
* Linux开发
  * [VIM技巧](dev/vim.md)
  * [常用开发网址](dev/README.md)
  * [Chrome允许自签名证书](dev/chrome.md)
* Kubernetes
  * [使用Kubeadm安装](k8s/install-by-kubeadm.md)
  * [kUbernetes使用NFS类型的StorageClass](k8s/k8s-nfs-pv.md)
  * [Kubectl多集群](k8s/kubectl.md)
  * [RBAC](k8s/k8s-rbac.md)
  * [CoreDNS配置示例](k8s/coredns.md)
  