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


## 测试

 往`Redis`中设置`Key`

 ```bash
 echo "sadd td:stage:shops 192.168.0.100" | redis-cli -x
 ```



通过启用`log_format`中的`request_time`时间对比,使用了`rewrite_by_lua_file`与未使用相比,时间上没差距.既使`Redis`挂了,也不会产生阻塞导致响应时间过长,而是会直接使用默认(`"default"`)的后端