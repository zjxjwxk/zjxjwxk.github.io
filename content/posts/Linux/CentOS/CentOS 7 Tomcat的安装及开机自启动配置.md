---
title: CentOS 7 Tomcat开机自启动配置
date: 2019-01-13 00:04:00
tags: 
- Linux
- CentOS
- Tomcat
categories: 
- Linux
preview: 300
---



# CentOS 7 Tomcat开机自启动配置

## 配置开机运行

### Tomcat增加启动参数

Tomcat需要增加一个pid文件，在 `$CATALINA_HOME/bin` 目录下面，增加 `setenv.sh` 配置，`catalina.sh`启动的时候会调用，同时配置Java内存参数。添加如下命令：
`[root@vps bin]# vim setenv.sh`

```shell
#Tomcat startup pid

#set Java runtime environment variable 
export JAVA_HOME=/usr/java/jdk1.8.0_191-amd64
export PATH=$PATH:$JAVA_HOME/bin
export CATALINA_HOME=/developer/apache-tomcat-7.0.91
export CATALINA_BASE=/developer/apache-tomcat-7.0.91

#add Tomcat pid
CATALINA_PID="$CATALINA_BASE/tomcat.pid"

#add Java opts
JAVA_OPTS="-server -XX:PermSize=256M -XX:MaxPermSize=1024m -Xms512M -Xmx1024M -XX:MaxNewSize=256m"
```

**注意:** 配置开机运行时,需要再次添加 JAVA_HOME

### 增加 tomcat.service

在/usr/lib/systemd/system目录下增加tomcat.service，目录必须是绝对目录，添加如下命令：
`[root@vps bin]# vim /usr/lib/systemd/system/tomcat.service`

```shell
# conf service desc ,set do this after network started
[Unit]
Description=tomcat 
After=syslog.target network.target remote-fs.target nss-lookup.target

# conf service pid, start,stop and restart
[Service]
Type=forking
PIDFile=/developer/apache-tomcat-7.0.91/tomcat.pid
ExecStart=/developer/apache-tomcat-7.0.91/bin/startup.sh
ExecStop=/bin/kill -s QUIT $MAINPID
ExecReload=/bin/kill -s HUP $MAINPID
PrivateTmp=true

# conf user 
[Install]
WantedBy=multi-user.target
```

*[unit]:* 配置了服务的描述，规定了在network启动之后执行，

*[service]:* 配置服务的pid，服务的启动，停止，重启

*[install]:* 配置了使用用户

### 使用tomcat.service

centos7使用systemctl替换了service命令，如需设置其他服务，替换此处的tomcat即可，如:`systemctl start vsftp.service`

- 启动服务
  systemctl start tomcat.service
- 停止服务
  systemctl stop tomcat.service
- 重启服务
  systemctl restart tomcat.service
- 增加开机启动
  systemctl enable tomcat.service
- 删除开机启动
  systemctl disable tomcat.service

因为配置pid，在启动的时候会在Tomcat的根目录下生产tomcat.pid文件,服务停止后删除。
同时Tomcat在启动时，执行start不会启动两个Tomcat，保证始终只有一个Tomcat服务在运行。多个Tomcat可以配置在多个目录下，互不影响。

## 查看效果

重启服务器后,通过wget访问，终端输出如下所示，配置tomcat开机自启动成功！

```shellell
[root@vps ~]# wget 35.234.8.23:8080
--2019-01-12 16:02:41--  http://35.234.8.23:8080/
Connecting to 35.234.8.23:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 909 [text/html]
Saving to: ‘index.html’

100%[===========================================================>] 909         --.-K/s   in 0s      

2019-01-12 16:02:41 (157 MB/s) - ‘index.html’ saved [909/909]
```



同时,客户端浏览器也能成功访问。