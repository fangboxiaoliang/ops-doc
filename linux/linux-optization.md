# Linux系统调优

## 基础调优

- 资源限制修改，主要修改`nofile`可以打开的文件描述符

  ```bash
   # 设置之前查看当前系统可以打开的最大数,不建设置值超过500K
   # cat /proc/sys/fs/file-max
   /etc/security/limits.conf
   # root需要单独列出来
   root   -   nofile  $value
   *      -  nofile   $value
  ```

- 内核优化

  ```bash
  #vim /etc/sysctl.d/add.conf

  # 开启恶意icmp错误消息保护
  net.ipv4.icmp_ignore_bogus_error_responses = 1
  # 定义UDP和TCP链接的本地端口的取值范围
  net.ipv4.ip_local_port_range = 1024 65000
  # 表示操作系统允许保持TIME_WAIT套接字数量的最大值，如超过此值，TIME_WAIT套接字将立刻被清除并打印警告信息,默认为8000。
  net.ipv4.tcp_max_tw_buckets = 5000
  # 主动关闭连接时，FIN_TIMEOUT状态时间
  net.ipv4.tcp_fin_timeout = 30
  # 当keepalive启用时，TCP发送keepalive消息的频度
  net.ipv4.tcp_keepalive_time = 600
  # 默认值128，表示socket监听的backlog(监听队列)上限。这个参数用于调节系统同时发起的TCP连接数
  net.core.somaxconn = 256
  
  # 慎用！！
  net.ipv4.tcp_tw_reuse = 1
  ```

- 文件系统优化，主要调整数据盘目录的挂载参数

  ```bash
  /etc/fstab
  ```

## 其他建议

- 应用数据目录设置成`LVM`格式，并单独挂载出来
