---
title: Docker 容器下的 MySQL 主从同步
date: 2025-12-30T22:19:00+08:00
tags:
- Database
- MySQL
- Docker
categories:
- 数据库
ShowToc: true
preview: 200

---

## 创建宿主机 Docker 容器挂载目录

```bash
mkdir -p /var/lib/mysql/master1/conf
mkdir -p /var/lib/mysql/master1/data
mkdir -p /var/lib/mysql/slave1/conf
mkdir -p /var/lib/mysql/slave1/data
```

## 创建 MySQL 主从配置文件

创建主库配置文件（宿主机路径为 /var/lib/mysql/master1/conf/my.cnf）：

```
[mysqld]
character-set-server = utf8
lower-case-table-names = 1
# 主从复制,主机配置,主服务器唯一ID
server-id = 1
# 启用二进制日志
log-bin=mysql-bin
# 设置logbin格式
binlog_format = STATEMENT
```

创建从库配置文件（宿主机路径：/var/lib/mysql/slave1/conf/my.cnf）：

```
[mysqld]
character-set-server = utf8
lower-case-table-names = 1
# 主从复制,从机配置,从服务器唯一ID
server-id = 2
# 启用中继日志
relay-log = mysql-relay
```

授权文件夹：

```bash
chmod -R 777 /var/lib/mysql
```

需要注意的是，主从的 my.cnf 文件权限不能为 777，否则 my.cnf 文件无法在容器内生效，在登陆 Docker 容器内的 MySQL 时会提示：`mysql: [Warning] World-writable config file '/etc/mysql/my.cnf' is ignored`。解决办法是修改宿主机上主从的 my.cnf 权限为更低的 644：

```bash
chmod 644 /var/lib/mysql/master1/conf/my.cnf
chmod 644 /var/lib/mysql/slave1/conf/my.cnf
```

## Docker 容器启动命令

由于本人 MySQL 的 Docker 镜像采用的是 mysql:8.0，默认的 data 路径为 /var/lib/mysql，默认的 my.cnf 文件路径为 /etc/mysql/my.cnf，故配置如下 docker 启动命令。

MySQL 主库：

```bash
docker run --name=mysql-master-1 \
--privileged=true \
-p 8808:3306 \
-v /var/lib/mysql/master1/data/:/var/lib/mysql \
-v /var/lib/mysql/master1/conf/my.cnf:/etc/mysql/my.cnf \
-v /var/lib/mysql/master1/mysql-files/:/var/lib/mysql-files/ \
-e MYSQL_ROOT_PASSWORD=xxxxxx \
-d mysql:8.0 --lower_case_table_names=1
```

MySQL 从库：

```bash
docker run --name=mysql-slave-1 \
--privileged=true \
-p 8809:3306 \
-v /var/lib/mysql/slave1/data/:/var/lib/mysql \
-v /var/lib/mysql/slave1/conf/my.cnf:/etc/mysql/my.cnf \
-v /var/lib/mysql/slave1/mysql-files/:/var/lib/mysql-files/ \
-e MYSQL_ROOT_PASSWORD=xxxxxx \
-d mysql:8.0 --lower_case_table_names=1
```

## MySQL 主从同步脚本

MySQL 主库执行脚本：

```
# 创建用户,设置主从同步的账户名
create user 'live-slave'@'%' identified with mysql_native_password by '12345678';
# 授权
grant replication slave on *.* to 'live-slave'@'%';
# 刷新权限
flush privileges;
# 查询server_id值
show variables like 'server_id';
# 也可临时（重启后失效）指定server_id的值（主从数据库的server_id不能相同）
set global server_id = 1;
# 查询Master状态，并记录File和Position的值，这两个值用于和下边的从数据库中的change那条sql中的master_log_file，master_log_pos参数对齐使用
show master status;
# 重置下master的binlog位点
reset master;
```

可以从 `show master status` 获得主库当前的 File 和 Position 信息，例如 File 为 mysql-bin.000001，Position 为 157，用于从库 master_log_file 和 master_log_pos 的设置。

MySQL 从库执行脚本：

```
# 查询server_id值
show variables like 'server_id';

# 也可临时（重启后失效）指定server_id的值（主从数据库的server_id不能相同）
set global server_id = 2;

# 若之前设置过同步，请先重置
stop slave;
reset slave;

# 设置主数据库
change master to master_host='172.20.10.5',master_port=8808,master_user='live-slave',master_password='12345678',master_log_file='mysql-bin.000001',master_log_pos=157;

# 开始同步
start slave;

# 查询Slave状态
show slave status;

# 若出现错误，则停止同步，重置后再次启动
stop slave;
reset slave;
start slave;
```

根据主库的 IP、端口等信息，设置从库的主数据库，同时设置 master_log_file 和 master_log_pos 为上一步从主库获取的 File 和 Position 值。

## MySQL 主从同步状态判断

开始同步后，若执行 `show slave status;` 查询从库状态时，若满足以下条件，则表示主从同步正常：

-   Slave_IO_Running 显示为 Yes
-   Slave_SQL_Running 显示为 Yes

在此之上，比较以下两对值，若均一致，则表示主从同步完成且一致：

-   主库 ( File , Position ) = 从库 ( Master_Log_File , Read_Master_Log_Pos )
-   从库 ( Master_Log_File , Read_Master_Log_Pos ) = 从库 ( Relay_Master_Log_File , Exec_Master_Log_Pos )

其中：

-   主库 ( File , Position ) 记录了主库 binlog 的位置。
-   从库 ( Master_Log_File , Read_Master_Log_Pos ) 记录了从库 IO 线程当前正在接收的二进制日志事件在主库 binlog 中的位置，如果小于 ( File , Position )，则表示 IO 线程存在延迟。
-   从库 ( Relay_Master_Log_File, Exec_Master_Log_Pos ) 记录了从库 SQL 线程当前正在执行的二进制日志事件在主库 binlog 的位置，如果小于 ( Master_Log_File, Read_Master_Log_Pos )，则表示 SQL 线程存在延迟。