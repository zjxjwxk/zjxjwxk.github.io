---
title: Dubbo微服务影院系列（4）：业务基础环境构建
date: 2020-03-09 22:25:00
tags:
- Dubbo
- 微服务架构
categories:
- Dubbo
preview: 300
---

# 章节概要

- 构建基于 Guns + SpringBoot + Dubbo 的框架
- 学会抽离业务接口
- 学会 API 网关变形应用

# API 网关介绍

- API 网关有点类似于设计模式中的 Facade 模式
- API 网关一般都是微服务系统中的门面
- **API  网关是微服务的重要组成部分**

![API网关示意图](images/image-20200309195624129.png)

## API 网关的常见作用

- 身份验证和安全
- 审查和检测
- 动态路由
- 压力测试
- 负载均衡（Dubbo 自带）
- 静态相应处理

![API网关作用](images/image-20200309200432132.png)

# Guns 环境构建

## 导入 guns 项目

![guns项目结构](images/image-20200309203515871.png)

## 创建 guns_rest 数据库

![guns_rest数据库](images/image-20200309203649262.png)

## 创建 user 表

```sql
-- ----------------------------
-- Table structure for user
-- ----------------------------
DROP TABLE IF EXISTS `user`;
CREATE TABLE `user`  (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `userName` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 2 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of user
-- ----------------------------
INSERT INTO `user` VALUES (1, 'admin');

SET FOREIGN_KEY_CHECKS = 1;
```

## 添加 log4j 依赖

```xml
<dependency>
  <groupId>log4j</groupId>
  <artifactId>log4j</artifactId>
  <version>1.2.17</version>
</dependency>
```

## 修改 datasource 的 url

修改 application.yml 中 spring 的 datasource 的 url 为

```yaml
url: jdbc:mysql://127.0.0.1:3306/guns_rest?autoReconnect=true&useUnicode=true&characterEncoding=utf8&serverTimezone=GMT%2B8
```

## 验证环境

启动 GunsRestApplication，访问 url http://localhost/auth?userName=admin&password=admin，若返回如下 token 则环境搭建成功。

```json
{
  "randomKey": "zd8rb8",
  "token": "eyJhbGciOiJIUzUxMiJ9.eyJyYW5kb21LZXkiOiJ6ZDhyYjgiLCJzdWIiOiJhZG1pbiIsImV4cCI6MTU4NDM2MjkxOCwiaWF0IjoxNTgzNzU4MTE4fQ.mR7Elro8oK8T2Rx6FE02FWTvORJnUAcFfPka4-KbQYsSpoMQQl5iUgd-wd9wYIw_5wbXpn3TFoii72suPn0b4A"
}
```

# API 网关模块构建测试

在 guns 项目中复制一份 guns-rest 模块并重命名为 guns-gateway，在主模块 pom.xml 中添加 <module>guns-gateway</module>

```xml
    <modules>
        <module>guns-admin</module>
        <module>guns-core</module>
        <module>guns-rest</module>
        <module>guns-generator</module>
        <module>guns-gateway</module>
    </modules>
```

修改其 pom.xml 中有关名称为 guns-gateway，并在 Project Structure 中将该模块重命名为 guns-gateway，同时删除原来 guns-rest 中多余的 Content Root（从 guns-gateway 加入的）

![需删除的Content Root](images/image-20200309210548081.png)

这样 guns-gateway 就构建好了（测试同 guns-rest）。

# API 网关集成 Dubbo

在 guns-gateway 项目的 pom.xml 文件中添加 Dubbo 和 ZooKeeper 的依赖

```xml
<dependency>
  <groupId>com.alibaba.spring.boot</groupId>
  <artifactId>dubbo-spring-boot-starter</artifactId>
  <version>2.0.0</version>
</dependency>
<dependency>
  <groupId>com.101tec</groupId>
  <artifactId>zkclient</artifactId>
  <version>0.10</version>
</dependency>
```

在 application.yml 中添加

```yaml
spring:
  application:
    name: cinema-gateway
  dubbo:
    server: true
    registry: zookeeper://localhost:2181
```

- 在 Application 类上添加注解 @EnableDubboConfiguration
- 在 Service 实现类上添加注解 @Component 和 @Service(interfaceClass = UserAPI.class)，其中 @Service 来自Dubbo

# 抽离业务 API

- 在 guns 项目中复制一份 guns-core 模块并重命名为 guns-api
- 修改其 pom.xml 中各名称后，删除其所有依赖
- 删除所有 java 下所有包，创建 api 包，其中可创建公用接口类
- 使用 Maven 的 install 指令将其放入库中

![maven install](images/image-20200309215810417.png)

- 在主模块的 pom.xml 中添加依赖

```xml
<dependency>
  <groupId>com.stylefeng</groupId>
  <artifactId>guns-api</artifactId>
  <version>${guns.version}</version>
</dependency>
```

- 在 guns-gateway 的 pom.xml 中添加依赖

```xml
<dependency>
  <groupId>com.stylefeng</groupId>
  <artifactId>guns-api</artifactId>
</dependency>
```

之后，就可以在 guns-gateway 模块中，引入 guns-api 中的接口来使用了。这样接口类只需要在 guns-api 中写一份就够了，避免了冗余，达到了抽离业务 API 的效果。