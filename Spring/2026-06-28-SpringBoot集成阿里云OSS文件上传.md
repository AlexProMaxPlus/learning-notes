# Spring Boot 集成阿里云 OSS 文件上传

> 学习日期：2026-06-28
> 分类：Spring

---

## 第一点：前提与使用场景

### 为什么需要云存储？

本地存储文件存在三大问题：
- 本地存储无法直接从外部访问
- 本地磁盘空间有限
- 本地磁盘损坏会导致数据丢失

### 阿里云 OSS 的优势

阿里云对象存储（Object Storage Service，简称 OSS）是阿里云提供的海量、安全、低成本、高可靠的云存储服务，具有以下优势：

- **高可靠性**：多副本存储和容灾备份机制
- **安全性**：支持访问控制、加密传输等策略
- **弹性扩展**：按需存储，灵活调整容量
- **低成本**：按量付费模式

### 典型使用场景

- 用户头像上传
- 商品图片存储
- 文档资料上传
- 音视频文件存储

### 前提条件

- 已注册阿里云账号并完成实名认证
- 已开通 OSS 服务
- 已创建 Bucket（存储空间）
- 已获取 AccessKey ID 和 AccessKey Secret

---

## 第二点：具体构成与使用方法

在 Spring Boot 中实现阿里云 OSS 文件上传，整体流程分为四个部分。

### 2.1 引入 Maven 依赖

#### 为什么需要引入依赖？

阿里云提供了官方的 Java SDK，封装了与 OSS 服务交互的底层 API，开发者无需手动处理 HTTP 请求、签名认证等复杂逻辑，只需调用 SDK 提供的方法即可完成文件上传、下载、删除等操作。

#### 基础依赖

```xml
<!-- 阿里云 OSS SDK -->
<dependency>
    <groupId>com.aliyun.oss</groupId>
    <artifactId>aliyun-sdk-oss</artifactId>
    <version>3.15.1</version>
</dependency>
```

#### Java 9+ 额外依赖

如果使用 Java 9 及以上版本，需要额外添加 JAXB 相关依赖：

```xml
<dependency>
    <groupId>javax.xml.bind</groupId>
    <artifactId>jaxb-api</artifactId>
    <version>2.3.1</version>
</dependency>
<dependency>
    <groupId>javax.activation</groupId>
    <artifactId>activation</artifactId>
    <version>1.1.1</version>
</dependency>
<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
    <version>2.3.3</version>
</dependency>
```

> **原因**：Java 9 之后移除了 JAXB 模块，需要手动引入。

---

### 2.2 配置 OSS 参数

#### 需要配置哪些信息？

要连接阿里云 OSS，需要四个核心参数：

| 参数 | 说明 | 示例 |
|-----|------|------|
| endpoint | 地域节点 | oss-cn-hangzhou.aliyuncs.com |
| accessKeyId | 访问密钥 ID | - |
| accessKeySecret | 访问密钥密码 | - |
| bucketName | 存储空间名称 | - |

> **安全注意**：敏感信息不应硬编码，推荐使用配置文件或环境变量。

#### 方式一：直接在工具类中定义（适合学习阶段）

```java
public class AliOSSUtils {
    private String endpoint = "oss-cn-hangzhou.aliyuncs.com";
    private String accessKeyId = "你的AccessKeyId";
    private String accessKeySecret = "你的AccessKeySecret";
    private String bucketName = "你的Bucket名称";
}
```

#### 方式二：使用配置文件（推荐生产环境）

在 `application.yml` 中配置：

```yaml
aliyun:
  oss:
    endpoint: oss-cn-hangzhou.aliyuncs.com
    access-key-id: 你的AccessKeyId
    access-key-secret: 你的AccessKeySecret
    bucket-name: 你的Bucket名称
```

创建配置类读取：

```java
@Component
@ConfigurationProperties(prefix = "aliyun.oss")
@Data
public class AliOSSProperties {
    private String endpoint;
    private String accessKeyId;
    private String accessKeySecret;
    private String bucketName;
}
```

---

### 2.3 编写 OSS 工具类

#### 为什么需要工具类？

文件上传涉及多个步骤：获取文件流、生成唯一文件名、调用 SDK 上传、拼接访问 URL、关闭客户端。将这些逻辑封装在工具类中，可以：

- 实现代码复用
- 简化 Controller 层代码
- 便于统一管理 OSS 操作

#### 完整工具类代码

```java
package com.itheima.utils;

import com.aliyun.oss.OSS;
import com.aliyun.oss.OSSClientBuilder;
import org.springframework.stereotype.Component;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.io.InputStream;
import java.util.UUID;

@Component  // 交给 Spring 容器管理
public class AliOSSUtils {

    // OSS 配置参数
    private String endpoint = "oss-cn-hangzhou.aliyuncs.com";
    private String accessKeyId = "你的AccessKeyId";
    private String accessKeySecret = "你的AccessKeySecret";
    private String bucketName = "你的Bucket名称";

    /**
     * 实现上传文件到 OSS
     * @param file 上传的文件
     * @return 文件访问路径
     */
    public String upload(MultipartFile file) throws IOException {
        // 1. 获取上传文件的输入流
        InputStream inputStream = file.getInputStream();

        // 2. 生成唯一文件名，避免文件覆盖
        String originalFilename = file.getOriginalFilename();
        // 截取文件扩展名
        String extension = originalFilename.substring(originalFilename.lastIndexOf("."));
        // 使用 UUID 生成唯一文件名
        String fileName = UUID.randomUUID().toString() + extension;

        // 3. 创建 OSS 客户端实例
        OSS ossClient = new OSSClientBuilder().build(endpoint, accessKeyId, accessKeySecret);

        // 4. 上传文件到 OSS
        ossClient.putObject(bucketName, fileName, inputStream);

        // 5. 拼接文件访问路径
        // 格式：https://bucketName.endpoint/fileName
        String url = "https://" + bucketName + "." + endpoint + "/" + fileName;

        // 6. 关闭 OSS 客户端
        ossClient.shutdown();

        // 7. 返回文件访问路径
        return url;
    }
}
```

#### 代码逻辑解析

| 步骤 | 说明 |
|-----|------|
| 获取输入流 | 从 MultipartFile 获取文件流 |
| 生成唯一文件名 | UUID + 原文件扩展名，避免重名覆盖 |
| 创建 OSS 客户端 | 使用 endpoint、accessKeyId、accessKeySecret 构建 |
| 上传文件 | 调用 putObject 方法上传 |
| 拼接 URL | 组成完整的文件访问地址 |
| 关闭客户端 | 释放资源 |
| 返回 URL | 供前端或其他服务使用 |

---

### 2.4 编写 Controller 层

#### Controller 的职责

Controller 层负责接收前端请求，调用业务逻辑，返回响应结果。在文件上传场景中：

- 接收前端通过 multipart/form-data 提交的文件
- 调用工具类完成上传
- 返回上传后的文件访问地址

#### Controller 代码

```java
package com.itheima.controller;

import com.itheima.pojo.Result;
import com.itheima.utils.AliOSSUtils;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;

@Slf4j
@RestController
public class UploadController {

    @Autowired
    private AliOSSUtils aliOSSUtils;

    /**
     * 文件上传接口
     * @param image 上传的文件，参数名需与前端一致
     * @return 文件访问地址
     */
    @PostMapping("/upload")
    public Result upload(MultipartFile image) throws IOException {
        log.info("文件上传，文件名：{}", image.getOriginalFilename());

        // 调用阿里云 OSS 工具类进行文件上传
        String url = aliOSSUtils.upload(image);

        log.info("文件上传完成，访问地址：{}", url);

        return Result.success(url);
    }
}
```

#### 关键点说明

| 要点 | 说明 |
|-----|------|
| @RestController | 表示这是一个 RESTful 接口 |
| @PostMapping("/upload") | 定义上传接口路径 |
| MultipartFile image | Spring 提供的文件上传接收类，参数名需与前端表单字段名一致 |
| throws IOException | 文件操作可能抛出 IO 异常 |
| Result.success(url) | 返回统一格式的响应结果 |

---

## 第三点：整体作用

Spring Boot 集成阿里云 OSS 文件上传的整体作用：

1. **解决本地存储弊端**：文件存储在云端，不占用本地服务器空间，支持外网直接访问
2. **提升开发效率**：通过工具类封装复杂逻辑，Controller 层代码简洁明了
3. **保证数据安全可靠**：利用阿里云 OSS 的多副本存储机制，避免单点故障导致数据丢失
4. **支持弹性扩展**：随着业务增长，存储空间可弹性扩容，无需担心容量限制
5. **降低运维成本**：无需搭建和维护文件服务器，按量付费，成本可控

---

## 完整流程总结

```
前端上传文件 → Controller 接收 MultipartFile → 
工具类生成唯一文件名 → 创建 OSS 客户端 → 
上传到 OSS → 返回文件访问 URL → 前端展示或保存
```

---

## 学习心得

今天学习了 Spring Boot 集成阿里云 OSS 实现文件上传的完整流程，掌握了：
- Maven 依赖的引入方式
- OSS 配置参数的管理方法
- 工具类的封装技巧
- Controller 层的接口设计

后续可以扩展：文件大小校验、文件类型校验、下载、删除等功能。

---

*学习笔记 - 2026-06-28*