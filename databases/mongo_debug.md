# MongoDB Debug

-  当前服务器正在进行的操作

    ```bash
    # 查询服务器所有正在Running的操作
    db.currentOp();
    # 只查找对dbName.collectionName的操作
    db.currentOp({"ns":"dbName.collectionName"})
    ```

- 终止操作的执行

    ```bash
    db.killOp(openId)
    ```

- 服务器分析器，类似`MySQL`慢查询

    默认分析器记录集合是一个很小的固定集合。如果需要更大的数据需求，需要手动创建一个满足需求的固定集合。其默认名为`system.profile`.执行`db.setProfilingLevel`在那个数据库下，集合就会建立在该数据库下

    ```bash
    # 默认会记录操作大于100ms的操作.这里设置大于500ms记录
    db.setProfilingLevel(1, 500)
    # 查看查操作记录
    db.system.profile.findOne().pretty()
    ```
- 查询文档大小
  ```bash
  Object.bsonsize(db.users.findOne());
  ```
- 查看集合信息
  ```bash
  db.collectName.stats()
  ```

- 重命名集合
  ```bash
  db.collectionName.renameCollection("newName")
  ```

  - 不同`MongoDB`服务器之间克隆数据
    
    不能在相同服务器之间克隆数据
    ```bash
    db.runCommand({"colneCollection": "collName", "from": "host:27017"})
    ```