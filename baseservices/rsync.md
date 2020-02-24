# `Rsync`同步文件

`rsync`同步不同主机的文件时，一个作为服务端，另一个作为客户端。在服务端，一般称为`rsyncd`

## 服务端

安装

```bash
yum install rsync -y
```

准备配置文件`/etc/rsyncd.conf`

```conf
[mindoc]
path = /var/www/mindoc
comment = Java mindoc
auth users = mindoc
secrets file = /etc/rsyncd.secrets
```

创建引用的`/etc/rsyncd.secrets`，格式为`用户名:密码`

```txt
mindoc:Password
```

更改密码文件的权限

```bash
chmod 400 /etc/rsyncd.secrets
```

启动服务端

```bash
systemctl enable rsyncd 
systemctl start rsyncd
```

## 客户端

安装

```bash
yum install rsync -y
```

准备密码文件并更改为`400`权限

```bash
# 这里的内容就是服务端的/etc/rsyncd.secrets里面的密码部分
echo "Password" > rsync.pass
chmod 400 rsync.pass
```

启动

```bash
cd /backups/
# 将服务端的/var/www/mindoc目录下所有文件同步至当前目录下
rsync -avz --password-file=rsync.pass mindoc@服务端IP::mindoc   ./
```

## Debug

如果有任何异常，可以查看服务端的`/var/log/messages`文件，错误提示信息默认记录在该文件中