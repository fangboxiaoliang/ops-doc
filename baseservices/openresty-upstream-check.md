# OpenResty 后端健康检查（HTTP方式）

[https://github.com/openresty/lua-resty-upstream-healthcheck](https://github.com/openresty/lua-resty-upstream-healthcheck)

`NGINX`默认自带的健康检查仅仅是检查后端服务器的端口（TCP层）是不是通的，应用层（HTTP）的异常并不能被检测到。

```lua
server {
    ...
    http {
        ...
        lua_shared_dict healthcheck 1m;

        lua_socket_log_errors off;

        init_worker_by_lua_block {
            local hc = require "resty.upstream.healthcheck"

            local ok, err = hc.spawn_checker{
                shm = "healthcheck",  -- defined by "lua_shared_dict"
                upstream = "foo.com", -- defined by "upstream"
                type = "http",

                http_req = "GET /status HTTP/1.0\r\nHost: foo.com\r\n\r\n",
                -- run the check cycle every 2 sec
                interval = 2000,
                -- 1 sec is the timeout for network operations
                timeout = 1000,
                -- of successive failures before turning a peer down
                fall = 3,
                -- of successive successes before turning a peer up
                rise = 2,
                -- a list valid HTTP status code
                valid_statuses = {200, 302},
                -- concurrency level for test requests
                concurrency = 10, 
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
    ...

        location = /status {
            access_log off;
            allow 127.0.0.1;
            deny all;

            default_type text/plain;
            content_by_lua_block {
                local hc = require "resty.upstream.healthcheck"
                ngx.say("Nginx Worker PID: ", ngx.worker.pid())
                ngx.print(hc.status_page())
            }
        }
    }
    ...
}
```