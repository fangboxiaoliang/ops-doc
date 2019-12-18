# MongoDB

###  目录

- [特点](#td)
- [CRUD](#crud)
- [修改器](#xgq)
- [查询](#query)
- [管理](#manager)
- [索引](#index)
- [索引管理](#index-manager)
- [特殊集合与索引](#collection)
- [聚合](#jh)
### 特点<span id="td"/>

- 索引

        支持通用二级索引、唯一索引、复合索引、地理空间索引、全文索引

- 聚合

        支持聚合管道(aggreation pipeline)
- 特殊集合类型

        基于时间失效的集合，适用于像日志,session场景

- 文件存储
        
        GridFS

### CRUD<span id="crud"/>

-  插入
        
        /* 单条记录 */
        doc = {"username": "username", "from": "china", "birth": 1990};
        db.users.insert(doc);
        /* 多条记录 */
        docs = [{"user": "user1"}, {"user":"user2"}, {"user": "user3"}]
        db.users.insertMany(docs);

- 删除

        /* 删除user集合的所有内容,但保留user集合本身 */
        db.user.remove();
        /* 整个集合都删除，包括集合本身 */
        db.user.drop();
        /* 删除user集合中name为unknow的用户 */
        db.user.remove({"name":"unknow"});
- 更新

        /* 先从集合中读取一个用户，修改属性后再更新 */
        old_user = db.user.findOne({"name":"name1"});
        old_user.username = "newname";
        db.user.update({"name":"name1"}, old_user);
        /* 如果存在多条如name:name1的文档时，则需要通过_id键来进行更新 */

### 修改器<span id="xgq"/>

- 更新修改器

    - `$inc`  指定的KEY键自增值V,如果键不存在，创建它，并赋值

            /* 更新name:name1的age 加1。第三个参数是如果文档不存在就创建*/ 
            db.user.update({"name":"name1"}, {"$inc": {"age": 1}}, true});
    
    - `$set` 修改指定字段的值，如果字段不存在则创建并赋值

            db.user.update({"name":"name1"}, {"$set": {"country": "china"}});

    - `$unset` 如果字段存在，移除该字段，如果不存在，什么也不做
    
    - `$push` 向数组末尾添加元素，如果数组不存在，则创建它

            db.user.update({"name":"name1"}, {"$push": {"favirute": "swimming"}});
    - `$pull` 匹配所有文档中的数组，并删除数组中给定的KEY?测试下来是删除一个？

            docs = [
                {"name":"name1", "favirute": ["swimming","football"]},
                {"name":"name2", "favirute": ["swimming","football2"]},
                {"name":"name3", "favirute": ["swimming","football3"]},
            ];
            db.users.insertMany(docs);
            db.users.update({}, {"$pull": {"favirute": "swimming"}});

### 查询<span id ="query"/>

- `find`

        /* 查询所有文档 */
        db.collection.find();

        /* 查询指定KEY具有指定值的文档 */
        db.collection.find({"name":"name1", "country": "china"})

        /* 指定需要返回的键 */
        db.collection.find({}, {"name":1})

        /* 指定不需要返回的键 */
        db.collection.find({}, {"_id": 0})

        /* 条件查询，查询年龄大于18，小于40的文档 */
        db.collection.find({"age": {"$gt": 18, "$lt": 40}})

        /* 包含查询,$in包含, $nin不包含 */
        db.collection.find({"number": {"$in": [75, 28.87]}})

        /* $or查询,接受一个列表 */
        db.collection.find(
            "$or": [
                {"number": {"$in": [987,"234"]}},
                {"winner": true}
            ]
        )

        /* not,取反 */
        db.collection.find({"$not": {"number":{"$in": [2, 5, "9"]}}})

        /* $exists 判断键是存在与否 */
        db.collection.find({"key": {"$exists": true}})

        /* 正则表达式,注意没有使用双引号 */
        db.collection.find({"name": /abc/})

- 数组查询

        /* likes是数组，则匹配数组中既有swimming又有reading的文档 */
        db.collection.find(
            {"likes": {"$all": ["swimming", "reading"]}}
        )

        /* size 查询数组长度 */
        db.collection.find({"likes": {"$size": 3}})

        /* slice 返回匹配文档的数组的切片 */
        db.collection.find(
            {"name":"name1"},
            {"likes": {"$slice":[2, 4]}}
        )

        /* 数组最后的值 */
        db.collection.find(
            {"name":"name1"},
            {"likes": {"$slice": -1}}
        )

        /* 只匹配数组范围查询.数组内的所有数都需要满足下面的条件 */
        db.collection.find({"likes":
            {"$elemMatch": {"$gt": 10, "$lt": 20}}
        })

- 内联文档查询

        
        /* 内联文档查询使用.运算符 */
        {  
            "name": {"first_name": "cheng", "last_name": "name2"},
            "age": 30
        }
        db.collection.find({"name.first_name": "cheng"})

        /* 多条件查询内联文档时，也可以使用$elemMatch
        db.collection.find({
            "name": {
                "$elemMatch: {
                    "fist_name": "cheng", "age":{"$gt": 30}
                }
            }
        })

        /* $where 官方建议禁用该功能，可以执行任何JavaScript语句 */

- 查询选项

    - `limit` 限制返回的数目

            db.collection.find().limit(5)
    - `skip` 跳过前面指定的文档

            db.collection.find().skip(3)
    - `sort` 接受一个对象作为参数，作为排序的键

            /* 1为升序，-1为降序 */
            db.collecton.find().sort({"age": 1, "username": -1})

            /* 以上三个参数组合使用 */
            db.collection.find().skip(50).limit(50).sort({"age": 1})


### 管理<span od="manager"/>

    /* 查询当前DB执行中的任务 */
    db.currentOp()

### 索引<span id="index"/>

- 索引覆盖

       如果查询只需要查询索引中包含的字段，而无需返回整个文档。当一个索引包含用户请求的所有字段，就称为索引覆盖。
       应该优先使用索引覆盖。在查询中指定不需要返回的列。

- 多键索引

        如果索引键在文档中是一个数组，那么这个索引就会被标记为多键索引。被标记为多键索引后，就无法再变成非多键索引。除非删除索引后重建索引。

- 索引类型

    - 唯一索引： 索引键的值不能有重复，必须唯一

            db.collection.createIndex({"name": 1}, {"uniq": true})
    - 复合唯一索引
    - 稀疏索引：  该键可以不存在，但是存在时，其值就必须唯一
            
            db.collection.createIndex({"name": 1}, {"sparse": true});

- 索引管理<span id="index-manager"/>

    - 创建
        
        索引创建很费资源，默认它为阻塞所有读/写请求，直到索引创建完毕。如果在运行的服务器中创建索引，使用`background`选项，让`MongoDB`在后台创建索引。

            db.collection.createIndex();
    
    - 查询 

            db.collection.getIndexes();

    - 删除

            db.collection.dropIndex(IndexName);

### 特殊集合与索引<span id="collection"/>

-  固定集合

    类似循环队列，设置大小后当集合达到指定大小后，新数据会覆盖老数据。固定集合在使用之前需要显式创建。

    如果要限定固定集合的文档数量，`size`与`max`都需要指定。最终先达到限制的条件来决定文档的删除。
        
        /* 创建大小固定为100 000 Byte大小,最大文档数为1000的集合 */
        db.createCollection("name", {"capped": true, "size": 100000, "max": 1000})

        /* 将已存在的集合转换为固定集合 */
        db.runCommand({"convertToCapped": "collectName", "size": 10000, "max": 1000});

- TTL索引

    创建了该索引的文档，到达老化时间后，会被`MongoDB`自动删除。

        db.collection.createIndex({"lastUpdate": 1}, {"expireAfterSecs": 60 * 60 * 24})

- 全文本索引

    一个文档只能包含一个全文本索引，但全文本索引可以包含多个字段

        /* 将文档的title设置为全文本索引 */
        db.collection.createIndex({"title": "text"});

        /* 通过全文本索引来搜索文档 */
        db.runCommand({"text": "collectionName", "search": "ask hn"})

- 地理空间索引
- 2d索引

### 聚合<span id="jh"/>

- 操作符简介

    - `$match`
      
      用于对文档进行筛选，匹配符合条件的文档
        
            {"$match": {"name": "name1"}}

    - `$project`

      可以从文档中提取字段，重命名字段，指定要返回的字段

            /* 指定返回的字段 */
            {"$project": {"name": 1, "_id": 0}}

            /* 将结果返回并重命名_id为userId */
            {"$project": {"userId": "_id",  "_id": 0}}

    - `$project`表达式

        - `$add` 给定的列相加
        - `$subtract` 第一列减去第二列
        - `$multipy` 两个或者多个表达式相乘，积为结果
        - `$divde` 接受两个表达式，第一个除以第二个的商作为结果
        - `$mod` 接受两个表达式，第一个除以第二个的余数作为结果
        - `$substr` 子串字节
        - `$concat` 两个或者多个表达式，连接字符串
        - `toLower` 参数为字符串，返回小写形式
        - `toUpper` 参数为字符串，返回大写形式
        - `$cmp` 接受两个表达，判断大小。返回0/1/-1
        - `strcasecmp` 字符串比较，区分大小写
        - `$cond` 三元操作符
        - `ifNull` [`expr`, `replaeExpr`] 如果`expr`为`null`，返回`repleExpr`.否则返回`expr`
    
    - `$group`

        将文档根据不同字段的值进行分组。要分组的字段传给`$group`的`_id`

            /* 根据name字段进行分组 */
            db.collection.aggregate({"group": {"_id": "$name"}})

        - `$group`操作符

            - `$sum`

                    db.collection.aggregate(
                        "$group": {
                            "_id": "$age",
                            "total": {"$sum": "$age"}
                        }
                    )
            - `$avg`
            - `$max`
            - `$min`
            - `$first`
            - `$last`

    - `$unwind`

        将内联文档里面的文档进行拆分

    - `$sort`
        
        可以根据任何字段进行排序。如果要对大量文档进行排序，建议在管道第一阶段进行排序

    - `$limit`

    - `$skip`

    - `$distinct`

        给定键的不同值
            
            db.runCommand({"$distinct": "collectName", "key":　"name"})