# 目的

在一个成熟的软件系统中, 日志系统是一个必不可少的部分. 当你的项目处于上升阶段, 必然会遇到搭建完善的日志服务系统的需求. 而日志中心化, 则是在服务器集群中管理日志的一种必然结果. 目前比较流行的是使用开源软件 ELK (Elasticsearch+logstash+kibana) 统一管理日志. 

不得不承认, ELK 是一个成熟的解决方案, 其配置简单, 成熟的数据处理, 还有拥有强大的社区支持. 而接下来我将介绍另一种日志中心服务器的搭建思路, 即使用 Linux 系统的强大工具 syslog-ng 为核心, 更为简单, 更为高效, 稳定性更强.

# 背景

syslog-ng 提供了一个灵活的扩展性高的日志中心化解决方案. 通过 syslog-ng, 你可以收集/解析/分类来自任何来源的日志数据, 并且把日志分发到属于你的解析系统里, 是不是很灵活很cool? (笑)

# 方法

我先假设你有一个小型的服务器集群, 即一台业务服务器 (BS) 和一台日志服务器 (LS). 

### 安装 syslog-ng

我的测试服务器是 CentOS 系统, 通过 `yum` 工具在 BS 机器上安装.

```
yum install syslog-ng
```

其他系统可以参考 [https://github.com/balabit/syslog-ng](https://github.com/balabit/syslog-ng) 提供的安装方法

### 配置 BS 机器 syslog-ng

在 BS 机器的 syslog-ng 的指定配置文件 (通常路径为 `/etc/syslog-ng/syslog.conf`) 中添加以下配置, 用于接收日志并将日志转发到中心服务器.

```
# 设置通过unix-stream的方式接收日志数据
source s_test {
	unix-stream("/tmp/test.sock");
};

# 设置通过syslog方式将日志推送到中心服务器
destination d_test_syslog {
	syslog(
	"192.168.33.11"
	# 设置发送目的地的端口为1999
	port(1999)
	# 设置包传输的协议为TCP
	transport(tcp)
	);
};

# 绑定来源与目标
log { source(s_test); destination(d_test_syslog); };
```

### 配置 LS 机器 syslog-ng

在 LS 机器的 syslog-ng 的指定配置文件中添加以下配置, 用于接收客户端日志并且保存到文件.

```
# 设置通过syslog方式接收日志
source s_test_syslog {
    syslog(
    ip(192.168.33.11)
    # 设置接收包的协议为TCP
    transport(tcp)
    # 设置接收的端口号1999, 与日志转发的端口号一致
    port(1999)
    );
};

destination d_test {
    file(
    # 使用 PROGRAM 变量作为文件路径, 使客户端可以动态指定保存文件路径
    "${PROGRAM}"
    template("${MSG}\n")
    );
};

log { source(s_test_syslog); destination(d_test); };
```

# 测试

编写一个PHP脚本向 syslog-ng 指定的 socket 发送数据.

```
<?php

$fs = fsockopen('unix:///tmp/test.sock');
// :前的变量字符会解析为 syslog-ng 的 $PROGRAM 变量, 利用这一点可以动态指定中心服务器接收日志的路径
fwrite($fs, '/tmp/syslog-ng.log:测试日志数据');
fclose($fs);
```

通过`tcpdump`捕获 BS 机器发送的 tcp 数据包.

```
02:11:17.143119 IP 192.168.33.12.56755 > 192.168.33.11.tcp-id-port: Flags [P.], seq 1:4, ack 1, win 229, options [nop,nop,TS val 104554461 ecr 104475228], length 3
02:11:17.144102 IP 192.168.33.11.tcp-id-port > 192.168.33.12.56755: Flags [.], ack 4, win 227, options [nop,nop,TS val 104537916 ecr 104554461], length 0
02:11:17.144118 IP 192.168.33.12.56755 > 192.168.33.11.tcp-id-port: Flags [P.], seq 4:97, ack 1, win 229, options [nop,nop,TS val 104554462 ecr 104537916], length 93
02:11:17.144903 IP 192.168.33.11.tcp-id-port > 192.168.33.12.56755: Flags [.], ack 97, win 227, options [nop,nop,TS val 104537917 ecr 104554462], length 0
```

查看保存的日志

```
[root@vagrant-centos65 tmp]# tail -f /tmp/syslog-ng.log
测试日志数据
```
