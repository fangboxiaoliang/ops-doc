# 本地`Yum`源脚本

## 安装工具

```bash
yum install nginx yum-utils -y
```

## 编写脚本
```bash
#!/bin/bash
export PATH

cd /data/yumdata
reposync -nd
dirs=$(ls -F |grep "/")
for d in ${dirs};
do
 cd /data/yumdata/${d}
 createrepo --update ./
done
```

## 提供对外服务

将通过本机`/etc/yum.repod.d/`中的`yum`源，同步至`/data/yumdata`目录下，将该目录通过`NGINX代理出去`

```conf
...
location / {
    autoindex on;
    root /data/yumdata;
}
```