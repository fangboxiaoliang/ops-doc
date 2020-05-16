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
read only = true
hosts allow = 128.0.255.0/24
auth users = mindoc
secrets file = /etc/rsyncd.secrets

# 更多可用参数: man rsyncd.conf 查看说明 
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
# 将服务端的/var/www/mindoc目录下所有文件同步至当前目录下.当源服务器中的文件被删除时，目标服务器中的文件也删除
rsync -aPvz --delete --password-file=rsync.pass mindoc@服务端IP::mindoc   ./
# -a 归档模式
# -P 断点续传功能
# -v 输出详细信息
# -z 启用压缩
# -n 模拟执行，但并不会实际执行。一般配合-v使用
# --delete 在目标端删除源不存在的文件，要配合-r参数选项。
```

## Debug

如果有任何异常，可以查看服务端的`/var/log/messages`文件，错误提示信息默认记录在该文件中