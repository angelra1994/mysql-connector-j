# MySQL Connector/J Maven 构建指南

> 本文档记录如何在 MySQL Connector/J 上游源码基础上，使用 Maven 替代官方 Ant 构建方式进行编译。

## 1. 背景

MySQL Connector/J 官方使用 **Ant**（`build.xml`）作为构建工具，不提供 `pom.xml`。这给使用 Maven 生态的开发者带来了不便：
- 依赖管理需要手动下载 jar 放到 `com.mysql.cj.extra.libs` 目录
- 无法直接 `mvn install` 部署到本地仓库
- IDE 集成不够友好

本项目通过自建 `pom.xml`，在保留上游源码不做任何修改的前提下，实现 Maven 构建。

## 2. 源码结构

上游采用多源码目录组织，不同于 Maven 默认的单一 `src/main/java`：

```
src/
  build/           → 构建期 instrumentation 工具（javassist 字节码后处理）
    java/
  demo/            → 示例代码
    java/
  generated/       → protobuf 生成的 X DevAPI 协议代码
    java/
  legacy/          → 旧版 com.mysql.jdbc.Driver 兼容层
    java/
  main/
    core-api/       → 核心公共 API
    core-impl/      → 核心实现
    protocol-impl/  → 通信协议实现（X Protocol / Classic Protocol）
    user-api/       → 用户面向 API
    user-impl/      → 用户面向实现（含 Driver 类）
    resources/      → 资源文件（配置 properties、错误消息等）
    doc/            → 文档资源
  test/            → 测试套件
    java/
```

## 3. pom.xml 关键设计

### 3.1 多源码目录注册

使用 `build-helper-maven-plugin` 的 `add-source` goal 将所有源码目录注册为 Maven 编译源：

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>build-helper-maven-plugin</artifactId>
    <version>3.6.1</version>
    <executions>
        <execution>
            <id>add-build</id>
            <phase>generate-sources</phase>
            <goals><goal>add-source</goal></goals>
            <configuration>
                <sources>
                    <source>src/build/java/</source>
                    <source>src/demo/java/</source>
                    <source>src/generated/java/</source>
                    <source>src/legacy/java/</source>
                    <source>src/main/core-api/java/</source>
                    <source>src/main/core-impl/java/</source>
                    <source>src/main/protocol-impl/java/</source>
                    <source>src/main/user-api/java/</source>
                    <source>src/main/user-impl/java/</source>
                </sources>
            </configuration>
        </execution>
        <execution>
            <id>add-test</id>
            <phase>generate-test-sources</phase>
            <goals><goal>add-test-source</goal></goals>
            <configuration>
                <sources>
                    <source>src/test/java/</source>
                </sources>
            </configuration>
        </execution>
    </executions>
</plugin>
```

这对应 Ant build.xml 中 `-copy-files-main` target 的行为（build.xml 第 577-592 行）。

### 3.2 依赖管理

| 依赖 | 版本 | 用途 | scope |
|---|---|---|---|
| `com.google.protobuf:protobuf-java` | 4.36.1 (26.10.0) / 4.31.1 (26.7.0, 9.7.0) | X DevAPI 协议 | optional |
| `org.javassist:javassist` | 3.30.2-GA | instrumentation 字节码后处理 | compile |
| `com.mchange:c3p0` | 0.11.0 | 连接池集成 | compile |
| `org.slf4j:slf4j-api` | 2.0.17 | 日志门面 | compile |
| `io.opentelemetry:opentelemetry-api` | 1.50.0 | 可观测性 | compile |
| `io.opentelemetry:opentelemetry-context` | 1.50.0 | 可观测性上下文 | compile |
| `com.oracle.oci.sdk:oci-java-sdk-common` | 3.66.0 | OCI 集成 | compile |
| `org.hamcrest:hamcrest` | 3.0 | 测试断言 | compile |
| JUnit 5 系列 | 5.12.2 / 1.12.2 | 测试框架 | test |

> **protobuf 版本是关键**：必须与 `src/generated/java/` 中代码声明的版本一致。26.10.0 的 generated 代码标记 `Protobuf Java Version: 4.36.1`，且类继承 `com.google.protobuf.GeneratedFile`（4.36+ 新 API）；26.7.0 和 9.7.0 使用 4.31.1 API。版本不匹配会导致编译报 `找不到符号` 错误。

### 3.3 编译器配置

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.14.1</version>
    <configuration>
        <source>21</source>
        <target>21</target>
        <encoding>UTF-8</encoding>
    </configuration>
</plugin>
```

- source/target 设为 **21**，可在 JDK 21 或 JDK 25 下编译通过
- 上游 build.xml 声称 "requires Java 8"，但实际源码在 JDK 21+ 下完全兼容

### 3.4 JAR 打包

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.5.0</version>
    <configuration>
        <archive>
            <addMavenDescriptor>false</addMavenDescriptor>
        </archive>
    </configuration>
</plugin>
```

`addMavenDescriptor=false` 避免在 jar 中生成 `pom.properties` 和 `pom.xml`，保持与官方 jar 接近的结构。

### 3.5 测试配置

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.5.4</version>
    <dependencies>
        <dependency>
            <groupId>org.apache.maven.surefire</groupId>
            <artifactId>surefire-junit-platform</artifactId>
            <version>3.5.4</version>
        </dependency>
    </dependencies>
    <configuration>
        <systemPropertyVariables>
            <com.mysql.cj.testsuite.url>jdbc:mysql://127.0.0.1:3093/test?user=root&amp;password=...</com.mysql.cj.testsuite.url>
            <com.mysql.cj.testsuite.mysqlx.url>jdbc:mysql://root:...@127.0.0.1:33060/test</com.mysql.cj.testsuite.mysqlx.url>
        </systemPropertyVariables>
    </configuration>
</plugin>
```

- 显式指定 `surefire-junit-platform` provider 以支持 JUnit 5
- 通过 `systemPropertyVariables` 传入测试数据库连接 URL（对应 Ant 的 `com.mysql.cj.testsuite.url` 属性）

## 4. JDBC SPI 自动注册

Java JDBC 4.0+ 引入了 `ServiceLoader` 机制，Driver 会通过 `META-INF/services/java.sql.Driver` 文件自动注册。

本工程在 `src/main/resources/META-INF/services/java.sql.Driver` 中手动创建该文件：

```
com.mysql.cj.jdbc.Driver
com.mysql.jdbc.Driver
```

- `com.mysql.cj.jdbc.Driver`：新版驱动（cj 包），主要入口
- `com.mysql.jdbc.Driver`：legacy 兼容驱动，让旧代码 `Class.forName("com.mysql.jdbc.Driver")` 仍能工作

> **注意**：官方 Ant 构建通过 `<service>` 标签在打包 jar 时自动生成此文件，且只注册 `com.mysql.cj.jdbc.Driver`。本工程额外注册了 legacy Driver 以保证向后兼容。

## 5. 构建命令

```bash
# 仅编译
mvn clean compile

# 打包 jar（不跑测试）
mvn clean package -DskipTests

# 打包 jar（跑测试，需要 MySQL 实例）
mvn clean package

# 安装到本地 Maven 仓库
mvn clean install -DskipTests
```

## 6. 多版本管理

本项目为三个版本分别创建了分支，每个分支的 pom.xml 仅 `<version>` 和 protobuf 版本不同：

| 分支 | 版本 | protobuf | 说明 |
|---|---|---|---|
| `release/26.10.0` | 26.10.0 | 4.36.1 | 最新 26.x |
| `release/26.7.0` | 26.7.0 | 4.31.1 | 26.x |
| `release/9.7.0` | 9.7.0 | 4.31.1 | 9.x 最新 |

所有分支均基于上游 tag 的 depth=2 浅 clone。
