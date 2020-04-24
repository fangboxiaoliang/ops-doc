# MySQL备份恢复

## `xtrabackup`安装

```bash
# 安装Repo库
yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm

# 如果MySQL 8.0以上的安装此包
yum install percona-xtrabackup-80 -y

# MySQL5.1/5.5/5.6/5.7
yum install percona-xtrabackup-24 -y
```

## 基于`xtrabackup`整库备份

```bash
# 如果不加--database就备份所有数据库
xtrabackup  --host=127.0.0.1 --port=3306 --user=USERNAME --password=PASSWORD \
            --backup  --parallel=4 --compress --compress-threads=4 \
            --datadir=/var/lib/mysql --database=DBNAME --target-dir=/data/backup
```

## 基于xtrabackup恢复

因为`xtrabackup`执行的是物理备份，所以恢复时只需要将备份文件复制回去即可:

```bash
# 先将压缩文件解压,解压命令需要qpress支持.xtrabackup源中有该命令
yum install -y qpress
xtrabackup --decompress --target-dir=/data/backup
xtrabackup --prepare --target-dir=/data/backup/
# 将解压来的文件还原到MySQL数据目录内。会自动读取/etc/my.cnf文件内的内容，找到MySQL的数据存放目录。还原之前，该目录必须 为空。不然会提示文件不为空的失败信息
xtrabackup  --copy-back  --target-dir=/data/backup
# 更改所有者，不然会启动失败
chown -R mysql:mysql /data/mysql
# 启动服务
systemctl start mysqld
```
## 单表备份还原

单表备份依赖独立表空间，在`MySQL5.7`之后，该值默认开启

```SQL
SHOW VARIABLES LIKE 'innodb_file_per_table';
```


```bash
# 方式一: 使用innobackupex
innobackupex --user=backup --password=123456 --include="cumcm.cumcm_sys_country" /home/backup

# 完整备份完之后，再应用Redo日志，怕备份期间有事务产生
innobackupex --apply-log --export /home/backup/2019-0o-10_13-17-57/

# 方式二: 使用xtrabackup。--tables是基于正则表达式的。
xtrabackup --host=127.0.0.1 --port=3306 --user=USERNAME --password=PASSWORD \
           --backup  --parallel=4 --compress --compress-threads=4 \
           --datadir=/var/lib/mysql --tables=tableName1,tableName2 \
           --target-dir=/data/backup/tables
           
# 在要还原的DB上，需要先创建表结构
mysql>CREATE TABLE cumcm.cumcm_sys_country LIKE cumcm.cumcm_sys_country_20190101;

# 取消原来的表空间
mysql>ALTER TABLE cumcm.cumcm_sys_country DISCARD TABLESPACE;

# 停止MySQL服务器
systemctl stop mysqld

# 复制备份出来的ibd文件至MySQL数据目录下, .frm文件不复制
cp /home/backup/2019-0o-10_13-17-57/cumcm/cumcm_sys_country.ibd /var/lib/mysql/cumcm

# 更改文件所有者为MySQL
chown -R mysql:mysql /var/lib/mysql/

# 启动MySQL
systemctl start mysqld

# 导入表空间,如果在该步提示导入失败，那在复制备份文件时，只复制ibd文件
mysql>ALTER TABLE cumcm.cumcm_sys_country IMPORT TABLESPACE;

# 检查, 计数应和备份前的一致
SELECT COUNT(*) FROM cumcm.cumcm_sys_country;
```