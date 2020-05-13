# OpenResty 后端健康检查（HTTP方式）

[https://github.com/openresty/lua-resty-upstream-healthcheck](https://github.com/openresty/lua-resty-upstream-healthcheck)

`NGINX`默认自带的健康检查仅仅是检查后端服务器的端口（TCP层）是不是通的，应用层（HTTP）的异常并不能被检测到。

```lua
...
http {
    ...
    upstream test {
        server 127.0.0.1:8080;
        server 127.0.0.1:8082;
    }
    # the size depends on the number of servers in upstream {}:
    lua_shared_dict healthcheck 1m;
    lua_socket_log_errors off;
    init_worker_by_lua_block {
    local hc = require "resty.upstream.healthcheck"
    local ok, err = hc.spawn_checker {
        shm = "healthcheck",  -- defined by "lua_shared_dict"
        upstream = "test", -- defined by "upstream"
        type = "http",
        http_req = "GET / HTTP/1.0\r\nHost: dev.siss.io\r\n\r\n",
        -- run the check cycle every 2 sec
        interval = 2000,
        -- 10 sec is the timeout for network operations
        timeout = 10000,
        -- of successive failures before turning a peer down
        fall = 3,
        -- of successive successes before turning a peer up
        rise = 2,
        -- a list valid HTTP status code
        valid_statuses = {200, 302, 301, 204},
        -- concurrency level for test requests
        concurrency = 2, 
    }
    if not ok then
        ngx.log(ngx.ERR, "failed to spawn health checker: ", err)
        return
    end
    -- Just call hc.spawn_checker() for more times here if you have
    -- more upstream groups to monitor. One call for one upstream   group.
    -- They can all share the same shm zone without conflicts but they
    -- need a bigger shm zone for obvious reasons.
    }
    server {
        ...
        
        location /  {
            proxy_pass http://test;
        }
        
        ...
    }
    ...
}
```