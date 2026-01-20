---
title: Docker 部署 Dubbo 微服务
date: 2026-01-03T22:30:00+08:00
tags:
- Docker
- 微服务
categories:
- Docker
ShowToc: true
preview: 200
---

## Docker 部署 Nacos 注册中心

首先需要部署 Nacos 注册中心，在开发环境中运行 Nacos 单机模式，执行以下 Docker 命令拉取镜像并运行容器：

```
docker run -d \
  --name nacos-server \
  -p 8848:8848 \
  -p 9848:9848 \
  -e MODE=standalone \
  nacos/nacos-server:2.0.2
```

需要注意的是，若在开发环境中使用模拟域名（配置在 /etc/hosts 中）访问 Docker 容器中的 Nacos 服务，当开启了 socks 等代理服务时，可能导致访问失败。若使用了如 clash 等代理服务，可以在配置文件中关闭其 dns 功能，并配置该模拟域名为直连：

```
dns:
  enable: false
  ...

rules:
  - DOMAIN-SUFFIX,live.zjxjwxk.com,🎯 全球直连
```

若部署成功，可以通过 localhost:8848/nacos/index.html 访问 Nacos 控制台页面。

## Maven 插件配置

在需要部署的微服务的 pom.xml 文件中配置如下插件，用于在 Maven 打包的同时构建 Docker 镜像：

```xml
<build>
  <finalName>${artifactId}-docker</finalName>
  <plugins>
    <plugin>
      <groupId>com.spotify</groupId>
      <artifactId>docker-maven-plugin</artifactId>
      <version>1.2.0</version>
      <executions>
        <!--当mvn执行install操作的时候，执行docker的build-->
        <execution>
          <id>build</id>
          <phase>install</phase>
          <goals>
            <goal>build</goal>
          </goals>
        </execution>
      </executions>
      <configuration>
        <imageTags>
          <imageTag>${project.version}</imageTag>
        </imageTags>
        <imageName>${project.build.finalName}</imageName>
        <!--指定Dockerfile文件的位置-->
        <dockerDirectory>${project.basedir}/docker</dockerDirectory>
        <!--指定jar包路径，这里对应Dockerfile中复制jar包
        到docker容器指定目录配置，也可以写到Dockerfile中-->
        <resources>
          <resource>
            <targetPath>/</targetPath>
            <!--将下边目录的内容，拷贝到docker镜像中-->
            <directory>${project.build.directory}</directory>
            <include>${project.build.finalName}.jar</include>
          </resource>
        </resources>
      </configuration>
    </plugin>
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
    </plugin>
  </plugins>
</build>
```

## 编写 Dockerfile 文件

在需要部署的微服务目录下创建 docker 文件夹，并在此文件夹下创建如下 Dockerfile 文件（路径与 pom.xml 中的 `${project.basedir}/docker` 一致）：

```dockerfile
FROM openjdk:17-jdk-alpine
VOLUME /tmp
ADD live-user-provider-docker.jar app.jar
ENV JAVA_OPTS="\
-server \
-Xmx1g \
-Xms1g \
-Xmn256m"
ENTRYPOINT java ${JAVA_OPTS} -Djava.security.egd=file:/dev/./urandom -jar app.jar
```

## 构建 Docker 镜像

执行如下命令，在 Maven 打包的同时构建 Docker 镜像：

```
mvn install
```

若构建成功，则可以在 `docker images` 命令中找到构建成功的 Docker 镜像：

```
❯ docker images                                                                        ─╯
REPOSITORY                          TAG             IMAGE ID       CREATED         SIZE
live-user-provider-docker           1.0.0           45a95bb45bd6   2 hours ago     470MB
```

## 运行 Docker 容器

-   开发环境中，为了避免本机 IP 变化造成的麻烦，可以在环境变量中（如 `~/.zshrc` 中）定义本机 IP 为 $HOST_IP。如 MacOS 可以定义如下变量，自动获取当前 IP：

```
export HOST_IP=$(ipconfig getifaddr en0)
```

-   由于该服务依赖的所有外部地址均使用模拟域名，并在开发环境中配置了 /etc/hosts 文件用于模拟真实域名，因此在运行 Docker 容器时，需要将域名和宿主机 IP 的映射通过 `--add-host`作为 host 传入容器内的 `/etc/hosts` 文件。

-   由于该 Dubbo 服务需要向 Nacos 注册当前服务 IP，因此需要配置环境变量 `-e DUBBO_IP_TO_REGISTRY` 用于配置向 Nacos 注册的真实 IP（此处为宿主机 IP），而不是容器内的 IP；并通过 `-p` 实现端口映射。（可参考：[Dubbo 部署到 Docker 环境](https://dubbo.apache.ac.cn/en/docs3-v2/java-sdk/advanced-features-and-usage/others/docker/)）

综上，执行以下 Docker 命令运行容器（其中 $HOST_IP 为宿主机 IP， "live.zjxjwxk.com" 为此服务依赖的模拟域名）：

```
docker run --name live-user-provider \
 -p "$HOST_IP":9090:9090 \
 -e DUBBO_IP_TO_REGISTRY="$HOST_IP" \
 --add-host "live.zjxjwxk.com:$HOST_IP" \
 -d live-user-provider-docker:latest
```
