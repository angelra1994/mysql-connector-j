# Ant vs Maven 构建方式对比

> 本文档对比 MySQL Connector/J 官方 Ant 构建方式与本项目 Maven 构建方式的异同。

## 1. 总览

| 维度 | 官方 Ant (`build.xml`) | 本项目 Maven (`pom.xml`) |
|---|---|---|
| 构建工具 | Apache Ant 1.10+ | Apache Maven 3.x+ |
| 构建文件 | `build.xml` (1500+ 行) | `pom.xml` (205 行) |
| 依赖管理 | 手动下载 jar 放到 `extra.libs` 目录 | Maven 中央仓库自动解析 |
| 源码目录 | Ant `<copy>` 合并到临时目录后编译 | `build-helper-maven-plugin` 注册多源码目录 |
| 编译目标 | Java 8（声称） | Java 21（实际） |
| 字节码后处理 | instrumentation 三步（javassist） | 不执行 |
| SPI 注册 | `<service>` 标签自动生成 | 手动创建 resources 文件 |
| OSGi 元数据 | 完整 MANIFEST.MF（Export/Import-Package） | 不生成（简化） |
| 产物 | `mysql-connector-j-*.jar` + sources + javadoc + Maven bundle | `mysql-connector-j-*.jar` |
| 测试 | JUnit 4（build.xml 集成） | JUnit 5（surefire + junit-platform） |

## 2. 源码目录处理

### Ant 方式

Ant 不直接编译原始源码目录。`-copy-files-main` target（build.xml 第 506-593 行）将所有源码目录复制到临时构建目录 `${com.mysql.cj.build.dir.driver}`，同时做版本号/许可证的 token 替换：

```xml
<copy todir="${com.mysql.cj.build.dir.driver}" filtering="true" encoding="UTF-8">
    <fileset dir="src/build/java" />
    <fileset dir="src/build/misc" />
    <fileset dir="src/generated/java" />
    <fileset dir="src/main/user-api/java" />
    <fileset dir="src/main/user-impl/java" />
    <fileset dir="src/main/core-api/java" />
    <fileset dir="src/main/core-impl/java" />
    <fileset dir="src/main/protocol-impl/java" />
    <fileset dir="src/main/resources" />
    <fileset dir="src/legacy/java" />
    <fileset dir="src/demo/java" />
    <filterset refid="versionFilterset" />
    <filterset refid="licenseFilterset" />
</copy>
```

然后 `javac` 的 `srcdir` 指向这个合并后的临时目录。

### Maven 方式

Maven 不复制源码，而是通过 `build-helper-maven-plugin` 的 `add-source` goal 告诉编译器"这些目录都是源码目录"，直接原地编译：

```xml
<execution>
    <id>add-build</id>
    <phase>generate-sources</phase>
    <goals><goal>add-source</goal></goals>
    <configuration>
        <sources>
            <source>src/build/java/</source>
            <source>src/generated/java/</source>
            <source>src/legacy/java/</source>
            <source>src/main/core-api/java/</source>
            <!-- ... 其余目录 -->
        </sources>
    </configuration>
</execution>
```

**差异**：
- Ant 做 token 替换（如 `@MYSQL_CJ_VERSION@` → 实际版本号），Maven 不做（源码中不依赖这种 token）
- Ant 排除了 `src/build/misc` 目录中的非 Java 文件，Maven 通过 `add-source` 只添加 Java 源码目录
- Ant 的临时目录方式更灵活（可以做过滤、替换），Maven 方式更简单直接

## 3. 依赖管理

### Ant 方式

Ant 不自动下载依赖。需要开发者手动准备 `com.mysql.cj.extra.libs` 目录（build.xml 第 284-293 行）：

```xml
<path id="com.mysql.cj.extra.libs.classpath" cache="true">
    <fileset dir="${com.mysql.cj.extra.libs}">
        <include name="**/*.jar" />
        <exclude name="**/*-sources.jar" />
        <exclude name="**/*-javadoc.jar" />
        <exclude name="ant/**" />
        <exclude name="jacoco/**" />
        <exclude name="src/**" />
    </fileset>
</path>
```

所有 `*.jar` 都会被加入 classpath，不区分 scope。

### Maven 方式

Maven 通过 `<dependencies>` 声明，自动从中央仓库下载，并区分 scope：

```xml
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>4.36.1</version>
    <optional>true</optional>  <!-- 对应 Ant 中 OSGi 的 resolution:=optional -->
</dependency>
```

**差异**：
- Ant 粗粒度（一个目录全部 jar），Maven 细粒度（精确到每个坐标+版本）
- Ant 无 scope 区分，Maven 区分 compile/test/provided/optional
- Ant 的 `<optional>` 体现在 OSGi Import-Package 的 `resolution:=optional`，Maven 用 `<optional>true</optional>`

## 4. 字节码后处理（instrumentation）

这是**最大的差异**。

### Ant 方式

Ant 在编译后执行三个 instrumentation 步骤（build.xml 第 1067-1090 行）：

1. **`CommonChecks`** — 使用 javassist 检查关键类的方法签名一致性
2. **`TranslateExceptions`** — 使用 javassist 将 `SQLException` 包装为 `CJException`（运行时异常），实现 `assertThrows` 等测试辅助方法
3. **`AddMethods`** — 使用 javassist 向 `MysqlDataSource` 等类动态添加便捷方法

这三个步骤通过 `java` task 调用 `instrumentation.*` 类，fork 新 JVM 执行。

### Maven 方式

**不执行**任何字节码后处理。`src/build/java/instrumentation/` 下的代码只参与编译（作为源码目录注册），不作为构建步骤运行。

**影响**：
- 官方 jar 中的异常处理经过了 `TranslateExceptions` 改造，Maven 构建的 jar 没有
- 官方 jar 中 `MysqlDataSource` 有额外的方法，Maven 构建的 jar 没有
- 对基本 JDBC 功能无影响，但测试套件中部分 assert 可能行为不同

## 5. JDBC SPI 注册

### Ant 方式

Ant 在打 jar 时通过 `<service>` 标签自动生成（build.xml 第 803 行）：

```xml
<jar jarfile="${com.mysql.cj.build.dir.driver}/${com.mysql.cj.build.driver.fullName}.jar"
     manifest="${com.mysql.cj.build.dir.driver}/META-INF/MANIFEST.MF"
     index="true">
    <!-- ... -->
    <service type="java.sql.Driver" provider="com.mysql.cj.jdbc.Driver" />
</jar>
```

只在 `META-INF/services/java.sql.Driver` 中注册 **一个** provider：`com.mysql.cj.jdbc.Driver`。

### Maven 方式

手动创建 `src/main/resources/META-INF/services/java.sql.Driver` 文件，内容为：

```
com.mysql.cj.jdbc.Driver
com.mysql.jdbc.Driver
```

注册了 **两个** provider，额外包含 legacy `com.mysql.jdbc.Driver`。

**差异**：
- Maven 版本多了 legacy Driver 注册，兼容旧代码 `Class.forName("com.mysql.jdbc.Driver")`
- 如果只需要新 API，可以删掉第二行

## 6. JAR 元数据

### Ant 方式

Ant 生成完整的 OSGi MANIFEST.MF（build.xml 第 772-793 行）：

```
Bundle-ManifestVersion: 2
Bundle-SymbolicName: com.mysql.cj
Bundle-Name: Oracle Corporation's JDBC and XDevAPI Driver for MySQL
Export-Package: com.mysql.cj.jdbc;version="26.10.0";uses:="...",com.mysql.cj;version="26.10.0",...
Import-Package: javax.net.ssl;resolution:=optional,javax.crypto;resolution:=optional,...
Specification-Title: JDBC
Specification-Version: 4.2
Implementation-Version: 26.10.0
```

还生成 `INFO_BIN`、`INFO_SRC`（构建信息）、checksum 文件（md5/sha1）。

### Maven 方式

通过 `maven-jar-plugin` 简化配置：

```xml
<configuration>
    <archive>
        <addMavenDescriptor>false</addMavenDescriptor>
    </archive>
</configuration>
```

- 不生成 OSGi 元数据（无 Export/Import-Package）
- 不生成 Maven 描述符（`pom.properties`/`pom.xml`）
- 不生成 INFO_BIN/INFO_SRC

**影响**：Maven 构建的 jar 不适合直接用于 OSGi 容器（如 Eclipse），但完全适用于普通 Java SE / Spring Boot / Tomcat 等场景。

## 7. 产物差异

| 产物 | Ant | Maven |
|---|---|---|
| 主 jar | `mysql-connector-j-26.10.0.jar` | `mysql-connector-j-26.10.0.jar` |
| sources jar | `mysql-connector-j-26.10.0-sources.jar` | 默认不生成 |
| javadoc jar | `mysql-connector-j-26.10.0-javadoc.jar` | 默认不生成 |
| Maven bundle | `mysql-connector-j-26.10.0_maven.tar.gz`（含 pom + checksum） | N/A（直接 mvn install） |
| 分发包 | `.zip` + `.tar.gz` | N/A |
| INFO_BIN/INFO_SRC | 有 | 无 |
| checksum | md5 + sha1 | 无 |

## 8. 测试

### Ant 方式

- 使用 JUnit 4
- 通过 `-compile-testsuite` target 编译测试
- 通过 `junit` task 运行，需要 `com.mysql.cj.testsuite.url` 属性指定数据库

### Maven 方式

- 使用 JUnit 5（5.12.2）
- 通过 `maven-surefire-plugin` 运行，需显式指定 `surefire-junit-platform` provider
- 通过 `systemPropertyVariables` 传入测试数据库 URL

**注意**：上游测试套件可能依赖 JUnit 4 API，切到 JUnit 5 后部分测试可能需要适配（JUnit 5 提供 `junit-vintage-engine` 兼容 JUnit 4，当前 pom 未引入）。

## 9. 构建流程对比

```
Ant 流程:
  -init → -copy-files-main (token 替换) → -compile-driver
    → CommonChecks → TranslateExceptions → AddMethods
    → build (生成 MANIFEST.MF + service + jar)
    → package (sources + javadoc + zip/tar.gz + Maven bundle)

Maven 流程:
  generate-sources (add-source 注册多目录)
    → compile (maven-compiler-plugin)
    → process-resources (复制 resources，含 SPI service 文件)
    → package (maven-jar-plugin，无元数据)
    → (可选) test (maven-surefire-plugin)
```

## 10. 何时用哪个

| 场景 | 推荐 |
|---|---|
| 需要官方完全一致的产物（OSGi、instrumentation） | Ant |
| 快速开发调试、IDE 集成 | Maven |
| 需要部署到 Maven 仓库 | Maven |
| 需要 sources/javadoc 分发包 | Ant |
| Spring Boot / 普通应用使用 | Maven 足够 |
| OSGi 容器（Eclipse 插件） | Ant（需要 OSGi 元数据） |
