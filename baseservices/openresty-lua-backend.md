# `OpenResty`基于`Lua`/`Redis`依据`IP`简单灰度

需求: 根据不同IP返回不同后端，实现一个简单版的一个灰度发布功能

使用到的模块

- https://github.com/openresty/lua-nginx-module#synopsis

该模块已内置于`OpenResty`可以直接使用


## `OpenResty配置`

1. 先定义二个`upstream`

    ```conf      
    # 定义一个default,为默认后端
    upstream default {
        server 127.0.0.1:8080 weight=10;
        server 127.0.0.1:8081 weight=10;
    }

    # 定义一个灰度,匹配的IP由灰度提供服务
    upstream stage {
        server 127.0.0.1:8090 weight=10;
        server 127.0.0.1:8091 weight=10;
    }
    ```

2. 通过`rewrite_by_lua_file`指令来实现不同后端

    ```conf
    server {
        ...
        location / {
            ...
            set $backend "default";
            rewrite_by_lua_file conf/lua/set_upstream.lua;
            proxy_pass http://$backend;
        }
        ...
    }
    ```

3. `set_upstream.lua`

    ```lua
    #!/usr/bin/lua

    local redis_host = "127.0.0.1"
    local redis_port = 6379
    -- key 为Redis的Set类型
    local key = "td:stage:shops"


    local function close_redis(red)
        if not red then
            return
        end
        local pool_max_idle_time = 100
        local pool_size = 20
        local ok, err = red:set_keepalive(pool_max_idle_time, pool_size)
        if not ok then
            ngx.log(ngx.ERR, "set keepalive err ", err)
        end
    end


    local function get_ip()
        local client_ip = ngx.req.get_headers()["X-Real-IP"]
        if client_ip == nil then
            client_ip = ngx.req.get_headers()["x_forwarded_for"]
        end

        if client_ip == nil then
            client_ip = ngx.var.remote_addr
        end

        return client_ip
    end


    local function is_stage(ip)
        local result = 0
        local resty_redis = require "resty.redis"
        local redis = resty_redis:new()
        local ok, err = redis:connect(redis_host, redis_port)
        if err then
            colse_redis(redis)
            ngx.log(ngx.ERR, "Connect to redis failed ", err)
        else
            -- ngx.log(ngx.ERR, "Connect success")
            local _result, err = redis:sismember(key, ip)
            if err then
                ngx.log(ngx.ERR, "Get key failed", err)
            else
                result = _result
            end
        end
    
        if result == 1 then
            return true
        else
            return false
        end
    end


    local ip = get_ip()
    if is_stage(ip) then
        ngx.var.backend = "stage"
    else
        ngx.var.backend = "default"
    end
    ```

## 方法二


前面的方法虽然是实现了，但总感觉不够优雅。而且做完压力测试后发现，以100的并发来压测，虽然增加的响应时间忽略不计，但对网络IO增长还是蛮多，连接`Redis`后的`TIME_WAIT`状态达到`9000`多，想想也是有点恐怖。

在此方法中，优化连接`Redis`的连接，因为`IO`都集中在频繁连接/断开`Redis`。通过`lua_shared_dict`，`init_by_lua_file`与`set_by_lua_file`实现

## 1. 定义二个`upstream`，此处和上面一样，略

## 2. `nginx.conf`

```conf
...
http {
    ...
    # 共享内存区域，用于所有的work进程变量共享
    lua_shared_dict ips 20m;
    init_by_lua_file conf/lua/init.lua;
    ...
    server {
        ...
        location / {
            ...
            # 设置upstream
            set_by_lua_file $backend conf/lua/set_upstream.lua;
            proxy_pass http://$backend;
        }
        ...
    }
    ...
}
...

```

## `init.lua`

```lua
#!/usr/bin/lua

local host = "127.0.0.1"
local port = 6380
local key = "td:stage:shops"

local cmd = string.format("echo 'smembers %s' | redis-cli -h %s -p %d", key, host, port)
-- local cmd = "cat /usr/local/openresty/nginx/conf/lua/ip.txt"
local f = io.popen(cmd)
local output = f:read("*a")
local ips = ngx.shared.ips
-- local std = string.gsub(output, "\n", ",")
ips:set("ips", output)
-- ngx.log(ngx.ERR, output)
f:close()
```

## `set_upstream.lua`

```lua
local function get_ip()
    local client_ip = ngx.req.get_headers()["X-Real-IP"]
    if client_ip == nil then
        client_ip = ngx.req.get_headers()["x_forwarded_for"]
    end

    if client_ip == nil then
        client_ip = ngx.var.remote_addr
    end

    return client_ip
end


local remote_ip = get_ip()
local ngx_dic = ngx.shared.ips
local stage_ip = ngx_dic:get("ips")
local result = string.find(stage_ip, remote_ip)
if result == nil then
    return "default"
else
    return "stage"
end
```

方法二的缺点就是不能实时更新，因为只有在`init`阶段写入共享内存，因此当要更新`IP`时，只能`Reload`


## 测试

 往`Redis`中设置`Key`

 ```bash
 echo "sadd td:stage:shops 192.168.0.100" | redis-cli -x
 ```

通过启用`log_format`中的`request_time`时间对比,使用了`rewrite_by_lua_file`与未使用相比,时间上没差距.既使`Redis`挂了,也不会产生阻塞导致响应时间过长,而是会直接使用默认(`"default"`)的后端