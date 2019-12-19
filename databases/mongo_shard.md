# MongoDB 分片

## 速览

官方建议至少三台以上,分片的前提需要副本集.

- 分片解决的问题
    - 增加可用RAM
    - 增加可用磁盘空间
    - 减轻单台服务器压力
    - 处理单个`mongod`无法承受的吞吐量

- 分片的对象是数据库中的集合，分片的键是集合的键

- 复制集设置成分片后,客户端需要连接`mongos`进程

- 分片的键不能是数组

## 分片键的确定

- 数据分发

   - 升序片键
     
     随着时间稳定增长的字段,如`data`, `objectID`

   - 随机分发的片键

      不同分片随机分配,各分片数据会比较均衡.随机分发的键可是用户名,邮件地址,`UUID`,散列值等实现

    - 基于位置的片键

      根据位置进行分片,如IP地址,经纬度,地址等

## 片键策略

- 散列片键
  
   如果追求的是数据的加载的极速,散列是最佳选择.

   散列片键不支持浮点数,浮点数被取整数进行散列.`1.99`与`1`散列结果是一样的
   
   缺点: 
   
   - 无法做范围查询
   - 不支持`uniq`索引
   - 不能使用数组字段

   ```bash
   # 创建一个散列索引
   db.collection.createIndex({"KeyName": "hashed"});

   # 对集合进行分片
   sh.shardCollection("dbName.collectionName", {"KeyName": "hashed"})
   ```

- `GridFS`散列片键

    需要配合`GridFS`文件系统,此处略

- 流水策略

    如果有些服务器性能比较强悍,希望性能强大的机器处理更多负载.流水策略一般配合`tag`处理.


## 配置

分片需要`mongod`进程运行在分片模式下

### `MongoDB`配置

- 启动配置服务器

    ```bash
    # 先启动配置服务器.生产环境时,就运行在三台不同的主机上,启动参数一致
    mongod --configsvr --dbpath /data/db
    ```
- 启动`mongos`进程

    `mongos`进程可以运行任意个,但是它们的启动参数列表必须一致

    ```bash
    # mongos进程不需要指定数据目录,它会从配置服务器获取数据加载到本地
    mongos --configdb config-1:27019,config-2:27019,config-3:27019
    ```

- 将复制集转换为分片

    连入`mongos`进程,将复制集添加进来

    ```bash
    # ReplicationName作为分片的名称
    sh.addShard("ReplicationName/server-1:27017,server-2:27017,server-3:27017");
    ```

### 集合配置

```bash

# 先说明那个数据库要启用分片
sh.enableSharding("DBName");
# 在要分片集合上面创建索引
use DBName;
db.users.createIndex({"username": 1});
# 在对集合进行进行分片,指定分片键为索引键
 sh.shardCollection("test.users", {"username": 1})
# 查看状态
sh.status()
```

## 分片管理

- 分片配置信息

    分片的所有配置信息,全部存储在配置服务器的的`config`数据库中.永远都不要边入配置服务器中修改`config`数据库,应该连入`mongos`进程,再通过`use config`来查找或者修改里面的内容

    - `config.shards` 记录所有分片信息
    - `config.databases` 记录集群中所有数据库信息
    - `config.collections` 记录所有分片集合信息
    - `config.chunks` 记录集合中所有块的信息
    - `config.changelog` 记录迁移记录,可以用于性能诊断
    - `config.settings` 记录当前负载均衡器与块大小设置信息

- 分片服务器之间连接信息

    ```bash
        # 只有在分片的mongos和mongod上此命令才有有效
        db.adminCommand({"connPoolStats": 1})
    ```
- 增加服务器

    - 增加`mongos`
      
      `mongos`增加很简单，只要启动命令`--configdb`参数和之前一样即可，它就会自动加入到现有集群中

    - 增加分片服务器

        新新的复制集添加即可,通过`sh.addShard()`

    - 指定负载均衡的操作时间
     
      默认系统时刻都在负载均衡，有时会影响性能。需要指定负载均衡的时间。可以通过`config.settings`设置时间
      
      ```bash
      use config;
      db.settings.update(
          {"_id": "balancer"}, 
          {"$set": {"activeWindow": {"start": "23:00", "stop":" 01:00"}}}, true)
      ```

    - 修改块大小

      默认`MongoDB`的分片块大小是`64MB`.

      ```bash
      use config;
      # 查看当前配置
      db.settings.findOne();
      # 修改为32
      db.settings.save({"_id": "chunksize", "value": 32});
      ```

    - 刷新配置
     
      如果`mongos`没有从配置服务器中读到最新的配置，可以通过刷新配置命令更新配置

      ```bash
      # 如果刷新后仍然配置没更新，则需要重启所有的mongos或mongod进程
      db.adminCommand({"flushRouterConfig": 1})
      ```