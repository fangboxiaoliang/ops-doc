# Mongo 复制集

## 配置

不同机器的`MongoDB`服务进程使用相同启动参数运行:
```bash
    mongod --replSet rs0 -f /etc/mongod.conf
```
三台机器运行起来后，再在三台当中任何一台中执行:

```bash
# 最后一个节点为仲裁节点，仲裁节点只进行选举投票，不保存数据。一个复制集中，最多只能有一个仲裁节点

# 优先级为0时，永远都不能成为主节点
config = {
    "_id": "rs0",
    "members": [
        {"_id": 0, "host": "128.0.100.172:27017", "priority": 2},
        {"_id": 1, "host": "128.0.100.173:27017"},
        {"_id": 2, "host": "128.0.100.174:27017", "arbiterOnly": true}
    ]
};
rs.initiate(config);
# 查看复制集详情
rs.status();
```

## 管理

- 重新配置复制集

```bash
# 假设复制集的第一个节点的IP变更了
config = rs.config();
config.members[0].host = "128.0.100.175:27017";
rs.reconfig(config);
```

- 延迟备份节点

   主动设置备份节点从主节点同步数据的延时。在初始化复制集时，通过`slaveDelay`参数指定。

- 索引忽略

    备节点可能只是备份作用，并不提供查询。因此为节约磁盘空间，不需要在备节点上创建索引，那么在初始化时，通过`buildIndexs: false`指定不创建索引。该选项是永久选项，并要求主机节点的优先级(`priority`)为0.

- 隐藏节点

   节点不希望被客户端连接并查询。比如因硬件性能差，只是起备份作用。通过参数`hidden: true`。只有`priority`为0的节点才能隐藏。

- 主节点降级

        /* 主节点降为备节点，并等待120秒选举。如果120秒后未选举出新主节点，重新成为主节点 */
        rs.stepDown(120);
    
- 单机模式运行

  因为在复制集模式下，不能修改备份节点的相关参数。如果要修改备份节点相关参数，则需要通过单机模式下启动服务

  ```bash
  # 先获取当前进程的启动参数，主要关注dbpath选项以及复制集的名称
  db.serverCmdLineOpts();

  # 停止掉备份进程，并以相同dbpath，不同端口。以单机模式启动服务.假设之前的dbpath为/data/db
  mongod --port 3001 --dbpath /data/db

  # 相关执行操作完成后，继续以之前的命令行参数启动即可。会自动从停止前的位置进行同步
  ```

- 备节点进入维护模式，对客户端不可见

    ```bash
    # true为维护模式，恢复时修改为false即可
    db.adminCommand({"replSetMaintenanceModel": true})
    ```
- 查看`oplog`使用情况

   ```bash
   db.printReplicationInfo()
   ```