# OpenResty + Lua 智能限速

## OpenResty简要介绍

引用官方一段介绍:

>OpenResty® 是一个基于 Nginx 与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。</br>

个人最看重的还是默认集成的`Lua`能力，`NGINX`默认并不集成它，要启用此功能还需要重新编译。

## NGINX Lua简要介绍

官网是最好的教材，没有之一。[官网介绍](https://www.nginx.com/resources/wiki/modules/lua/)，下面是常用的命令与作用.

- `init_by_lua_block/init_by_lua_file`

        nginx Master进程加载配置时执行,通常用于初始化全局配置/预加载Lua模块

- `set_by_lua_block/set_by_lua_file`

        设置nginx变量，可以实现复杂的赋值逻辑,此处是阻塞的，Lua代码要做到非常快；

- `rewrite_by_lua_block/rewrite_by_lua_file`

        rrewrite阶段处理，可以实现复杂的转发/重定向逻辑；

- `access_by_lua_block/access_by_lua_file`

        请求访问阶段处理，用于访问控制

- `content_by_lua_block/content_by_lua_file`

        内容处理器，接收请求处理并输出响应

- `body_filter_by_lua_block/body_filter_by_lua_file`

        对响应数据进行过滤，比如截断、替换。

- `header_filter_by_lua_block/header_filter_by_lua_file`

        定义和输出过滤头



## 实战

### 需求

做云服务，经常有遇到某些大客户做活动导致整个平台的其他客户都受影响。简单实现对具体商户的资源请求限制（具体就是API的请求频率）。
 
开发前期准备工作已做好充分准备，不同商家请求公共资源时，都会在`HTTP`头里面新增一个假设`shopid`的值。

### 配置

`nginx.conf`
```conf
 ...
 http {
     ...
     # 关键一：通过HTTP头里面的shopid为KEY来做限速
     limit_req_zone $shop_id zone=one:10m rate=1r/s;
     ...
     server {
         ....
         location /lua {
             # 关键点二: 通过Lua获取自定义的HTTP头
             set_by_lua_file $shop_id "lua_files/access.lua";
             # 关键点三: 引用限速。只限制了并发二个，实际需要修改
             limit_req zone=one burst=2 nodelay;
             proxy_pass http://someserver;
         }
     }
 }
```
`access.lua`

```lua
function get_header(key)
  local headers = ngx.req.get_headers();
  for k, v in pairs(headers) do
    if(k == key) then
        return v
    end
  end
  return "nil"
end

local shop_id = get_header("shopid");
return shop_id;
```