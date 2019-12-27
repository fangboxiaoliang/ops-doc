# RabbitMQ 生产环境推荐配置

主要分为操作系统优化与`RabbitMQ`自身的优化二项

## 操作系统配置项

### 最大文件打开描述符

官方要求最少`50K`，并同时不建议超过`500K`允许打开的文件描述符

```bash
vi /etc/security/limits.conf
root  -  nofile  65536
*  -  nofile  65536
```

### 内核参数调整

```conf
# vim /etc/sysctl.d/90-rabbitmq.conf

net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time=30
net.ipv4.tcp_keepalive_intvl=10
net.ipv4.tcp_keepalive_probes=4
net.ipv4.tcp_tw_reuse = 1
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog=4096
# TCP Keppalive优化，用于心跳
net.ipv4.tcp_keepalive_time=30
net.ipv4.tcp_keepalive_intvl=10
net.ipv4.tcp_keepalive_probes=4
net.ipv4.conf.default.rp_filter = 0
```

## RabbitMQ 配置项

### Memory

默认为当使用物理服务器的40%的内存时，会进行`GC`回收。这个值一般无需要修改。

### Disk

磁盘空间，默认最小`50MB`就可以启动`RabitMQ`,但是在生产环境中，建设设置为物理内存的`1.5`倍即可

```bash
{disk_free_limit, {mem_relative, 1.5}}
```

### Networking

面对具有很多客户端连接的场景，`RabbitMQ`的网络调优是很重要的一件事情。

```bash
# RabbitMQ config
tcp_listen_options.backlog = 4096
tcp_listen_options.nodelay = true
# Erlang VM I/O Thread Pool()32-128。4 per CPU
RABBITMQ_IO_THREAD_POOL_SIZE = 32
RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS = "+A 128"
rabbit.reverse_dns_lookups = false
# 发送缓存与接受缓存的值需要一致。TCP/MQTT/STMP配置项保持一致
# TCP
tcp_listen_options.backlog = 128
tcp_listen_options.nodelay = true
tcp_listen_options.linger.on      = true
tcp_listen_options.linger.timeout = 0
tcp_listen_options.sndbuf  = 32768
tcp_listen_options.recbuf  = 32768

# MQTT
mqtt.tcp_listen_options.backlog = 128
mqtt.tcp_listen_options.nodelay = true
mqtt.tcp_listen_options.linger.on      = true
mqtt.tcp_listen_options.linger.timeout = 0
mqtt.tcp_listen_options.sndbuf  = 32768
mqtt.tcp_listen_options.recbuf  = 32768

#STOMP
stomp.tcp_listen_options.backlog = 128
stomp.tcp_listen_options.nodelay = true
stomp.tcp_listen_options.linger.on      = true
stomp.tcp_listen_options.linger.timeout = 0
stomp.tcp_listen_options.sndbuf  = 32768
stomp.tcp_listen_options.recbuf  = 32768

# 每个连接的通道最大值,官方建议不要超过200.如果超过200，客户端需要优化了
channel_max = 200
```

### 查看当前配置

```bash
rabbitmqctl environment
```
