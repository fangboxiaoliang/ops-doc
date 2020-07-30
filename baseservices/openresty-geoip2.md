# OpenResty GeoIP2 模块识别城市

`GeoIP`数据太老，不能误别城市了，新项目升级为`GeoIP2`。要下载免费数据库文件，需要在官网进行注册。

用到的开源项目

- https://github.com/maxmind/libmaxminddb
- https://github.com/leev/ngx_http_geoip2_module 

免费的城市与国家数据库下载
- https://dev.maxmind.com/geoip/geoip2/geolite2/  


## 安装

安装编译工具

```bash
yum install -y gcc gcc+ openssl-devel perl-devel perl-ExtUtils-Embed pcre-devel zlib-devel automake make 
```

编译安装`libmaxminddb`

```bash
# 下载最新的release包
wget https://github.com/maxmind/libmaxminddb/releases/download/1.4.2/libmaxminddb-1.4.2.tar.gz

# 解压编译安装
tar -zxvf libmaxminddb-1.4.2.tar.gz
cd libmaxminddb-1.4.2
./configure && make && make install

# 做一个软连接.OpenResty/NGINX默认会从lib64里面加载库文件
ln -s /opt/maxminddb/lib/libmaxminddb.so.0.0.7 /lib64/libmaxminddb.so.0
```

编译安装`OpenResty`

```bash
# 下载ngx_geoip2_module模块
wget https://github.com/leev/ngx_http_geoip2_module/archive/3.3.tar.gz

# 解压出来待用
tar -zxvf 3.3.tar.gz

# 下载最新OpenResty release包
wget https://openresty.org/download/openresty-1.17.8.2.tar.gz

# 解压编译安装
cd openresty-1.17.8.2
./configure --prefix=/opt/openresty \
            --with-pcre --with-http_realip_module \
            --with-file-aio --with-threads \
            --with-stream --with-stream_realip_module \
            --without-stream_limit_conn_module \
            --with-http_perl_module \
            --add-module=/root/ngx_http_geoip2_module-3.3 \
            && gmake && gmake install
```

## 配置

`nginx.conf`

`GeoLit32-City.mmdb`需要从官网下载下来，解压后放入`conf`根目录

```conf
...
http {
    ...
    geoip2 conf/GeoLite2-City.mmdb {
        $city_name default=London source=$remote_addr city names en;
    }
    ...
}
... 
```

`city_name`为城市英文代码

变量`city_name`是根据`remote_addr`来解析的，如果前面还有一层代理，则需要将该变量设置为正确的外网IP，或者替换变量名。不然可能会解析错误。如果解析不出来，则`city_name`的值会使用默认值`London`。