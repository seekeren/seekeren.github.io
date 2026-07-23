---
date: '2024-03-31T12:01:56+08:00'
draft: false
title: 'maven项目启动失败排查'
tags: ["maven", "springboot"]
categories: ["work-eval"]
---

# 背景

从远程仓库克隆了系统代码到本地，但是在vscode种右键运行失败。

# 解决

项目是一个多模块的项目，其中的一个模块下载了第三方的jar包，需要把这些jar包安装到本地仓库，然后才能在pom中引入使用。

直接在终端中cd到对应模块执行

```bash
mvn install:install-file -DgroupId=onbon -DartifactId=bx06.message -Dversion=0.6.0-SNAPSHOT
# 或者
mvn install
```

然后cd到项目根目录，构建依赖的模块

```bash
mvn clean install -DskipTests
```

安装完依赖之后，AI的意思是可以通过以下指令执行代码：

```bash
cd xx-xxxxxxxxx-admin
mvn spring-boot:run -Dspring-boot.run.profiles=local
```

但是我想要通过vscode的run java执行，因为我举得这个功能很方便使用。但是失败了。

原因是vscode的收集classpath过程失败：

- 对于单模块项目，直接扫描 `pom.xml` 中的依赖，一般正常
- 对于多模块项目，可能只解析当前模块，忽略模块间依赖

vscode 需要正确识别 `xx-xxxxxxxxx-admin` 依赖了 `gq-test-admin`，需要将其 classpath 加入。

通过上述mvn启动却是可以的，然后我就好奇：

**vscode中的run java，idea中的run和maven中的run有什么异同。**

以下是AI总结的对比：

其实主要是第5行多模块支持，VS Code Run Java可能存在问题

| 维度           | VS Code Run Java    | IDEA Run            | Maven spring-boot:run                             |
| -------------- | ------------------- | ------------------- | ------------------------------------------------- |
| **启动方式**   | `java -cp` 直接运行 | `java -cp` 直接运行 | `mvn spring-boot:run`<br />底层本质还是`java -cp` |
| **编译方式**   | Eclipse JDT 编译    | IDEA Make 编译      | Maven 编译                                        |
| **Classpath**  | Eclipse JDT 解析    | IDEA 依赖解析       | Maven 依赖管理                                    |
| **多模块支持** | ⚠️ 可能有问题        | ✅ 完美支持          | ✅ 完美支持                                        |
| **启动速度**   | 快（增量）          | 快（增量）          | 慢（需编译所有依赖）                              |
| **配置复杂度** | 低（零配置）        | 低（零配置）        | 中（命令行参数）                                  |
| **依赖一致性** | ⚠️ 可能不一致        | ⚠️ 可能不一致        | ✅ 与打包一致                                      |
| **推荐场景**   | 简单项目            | 开发调试            | 生产前验证                                        |

所以相对好的办法就是通过maven启动项目，完美支持多模块项目启动

当然AI还建议了以下方式来运行VS Code Run Java，但是我试了是行不通的

创建 `.vscode/launch.json`：

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "java",
      "name": "Launch RuoYiApplication",
      "request": "launch",
      "mainClass": "com.ruoyi.RuoYiApplication",
      "projectName": "xx-xxxxxxxxx-admin",
      "vmArgs": "-Dspring.profiles.active=local",
       "classPaths": [
        "${workspaceFolder}/xx-xxxxxxxxx-admin/target/classes",
        "${workspaceFolder}/xx-test-admin/target/classes",
        "${workspaceFolder}/xx-algorithm/target/classes",
        "${workspaceFolder}/xx-xxxxxxxxx-transformer/target/classes",
        "${workspaceFolder}/xx-xxxxxxxxx-framework/target/classes",
        "${workspaceFolder}/xx-xxxxxxxxx-system/target/classes",
        "${workspaceFolder}/xx-xxxxxxxxx-common/target/classes",
        "${workspaceFolder}/xx-xxxxxxxxx-quartz/target/classes",
        "${workspaceFolder}/xx-xxxxxxxxx-generator/target/classes"
      ],
    }
  ]
}
```

先开始我用上述的方式启动，但是报错。后来我AI建议取消classpath。以下这种方式是没有写classPath：让vscode自己构建

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "java",
      "name": "Launch RuoYiApplication",
      "request": "launch",
      "mainClass": "com.ruoyi.RuoYiApplication",
      "projectName": "xx-xxxxxxxxx-admin",
      "vmArgs": "-Dspring.profiles.active=local"
    }
  ]
}
```

## 

## Maven spring-boot:run 的实际流程

```
mvn spring-boot:run
        ↓
1. 编译源代码 → target/classes
        ↓
2. 解析 pom.xml 依赖树
        ↓
3. 构建完整 classpath（包含所有模块 classes + 依赖 jar）
        ↓
4. Maven 插件调用 java -cp 运行（不是运行 jar）
```

Maven spring-boot:run 会编译 class、**不打包**、用 java -cp 运行（但 classpath 由 Maven 构建）

**实际执行的命令（简化版）：**

```bash
java -cp "xx-xxxxxxxxx-admin\target\classes;
         xx-test-admin\target\classes;
         D:\...\.m2\repository\spring\...\spring-boot-2.5.15.jar;
         ..."
     -Dspring.profiles.active=local
     com.ruoyi.RuoYiApplication
```

如果是运行打包之后的jar，则使用的命令是`java -jar xxx.jar`

# 知识点

## 依赖 jar 是什么

**依赖 jar = 已经编译好的第三方库的 class 文件集合**

```
spring-boot-2.5.15.jar
├── org/springframework/boot/SpringApplication.class  ← 已编译的 class 文件
├── org/springframework/boot/context/ApplicationContext.class
├── META-INF/MANIFEST.MF
└── ... 更多 class 文件
```

### 是否需要编译？

| 对象                         | 是否已编译 | 是否需要再次编译 |
| ---------------------------- | ---------- | ---------------- |
| **你的源代码** (`*.java`)    | ❌          | ✅ 需要编译       |
| **第三方依赖 jar** (`*.jar`) | ✅          | ❌ 不需要编译     |

**原因：** 依赖 jar 包已经是**预编译的二进制文件**，包含 `.class` 文件，下载后直接可用。

### 示例流程

```
项目依赖 spring-boot-2.5.15.jar

[你的代码]          [第三方依赖]
  ↓                    ↓
MyService.java   →  spring-boot-2.5.15.jar (已包含 .class)
  ↓ 编译              ↓ 直接使用
MyService.class        org/.../SpringApplication.class
  ↓                    ↓
  └───────── 一起用 java -cp 运行 ─────────┘
```

## Classpath 是什么

### 定义

**Classpath = Java 虚拟机 (JVM) 查找 class 文件的位置列表**

类比：

- **Java 的 classpath** = Windows 的 **PATH 环境变量**
- **PATH** → 操作系统查找 `.exe` 文件的位置
- **CLASSPATH** → JVM 查找 `.class` 文件的位置

### 格式



```
classpath = 路径1;路径2;路径3;...  (Windows 用 ; 分隔)
          = 路径1:路径2:路径3:...  (Linux/Mac 用 : 分隔)
```

### classpath 包含的内容

| 类型         | 示例                                                      |
| ------------ | --------------------------------------------------------- |
| **目录**     | `target/classes` (你的编译结果)                           |
| **jar 文件** | `D:\...\.m2\repository\spring\...\spring-boot-2.5.15.jar` |
| **zip 文件** | (较少使用，类似 jar)                                      |

### 实际运行时的 classpath 示例

```bash
java -cp "
  gq-evaluation-admin\target\classes;              ← 你的代码
  gq-test-admin\target\classes;                    ← 模块依赖代码
  D:\...\.m2\repository\org\springframework\...\spring-boot-2.5.15.jar;  ← 第三方依赖
  D:\...\.m2\repository\org\springframework\...\spring-core-5.3.39.jar;
  ...
" com.ruoyi.RuoYiApplication
```

### JVM 如何使用 classpath

```
JVM 启动，需要加载 com.ruoyi.RuoYiApplication.class
        ↓
在 classpath 中按顺序查找：
1. gq-evaluation-admin\target\classes → 找到！加载 ✅

JVM 运行中，需要加载 org.springframework.boot.SpringApplication.class
        ↓
在 classpath 中按顺序查找：
1. gq-evaluation-admin\target\classes → 没找到
2. gq-test-admin\target\classes → 没找到
3. spring-boot-2.5.15.jar → 在 jar 内找到！加载 ✅
```

### classpath 的作用

```
┌─────────────────────────────────────────────────────────────┐
│                    JVM 启动                                  │
└─────────────────────────────────────────────────────────────┘
                            │
                    设置 classpath
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
   你的代码目录         模块依赖目录          第三方 jar
   target/classes     gq-test-admin/...      spring-boot.jar
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
                   JVM 统一查找和加载 class 文件
```

------

## 总结对比

| 概念           | 说明                              | 类比           |
| -------------- | --------------------------------- | -------------- |
| **依赖 jar**   | 第三方库预编译好的 class 文件集合 | 厂家做好的零件 |
| **class 文件** | Java 源代码编译后的字节码         | 自己加工的零件 |
| **classpath**  | JVM 查找 class 文件的位置列表     | 零件仓库的索引 |

------

## 运行流程完整图



```
项目结构
├── src/main/java/MyService.java          ← 你的源代码
├── pom.xml                              ← Maven 配置（声明依赖）
└── [Maven 下载的 jar 仓库]
    └── spring-boot-2.5.15.jar           ← 第三方依赖（已编译）

启动流程
┌──────────────────────────────────────────────────────────────┐
│ 1. 编译                                                        │
│    MyService.java  →  MyService.class (到 target/classes)    │
├──────────────────────────────────────────────────────────────┤
│ 2. 构建 classpath                                              │
│    classpath =                                                │
│      target/classes              (你的代码)                  │
│      gq-test-admin/target/classes (模块依赖)                  │
│      D:\...\.m2\repository\...\spring-boot-2.5.15.jar (第三方)│
├──────────────────────────────────────────────────────────────┤
│ 3. JVM 启动                                                    │
│    java -cp "<classpath>" com.ruoyi.RuoYiApplication          │
│    │                                                           │
│    └─ JVM 根据 classpath 找到所有需要的 .class 文件           │
│       并加载执行                                               │
└──────────────────────────────────────────────────────────────┘
```

------

## 简单回答

| 问题                      | 答案                                                         |
| ------------------------- | ------------------------------------------------------------ |
| **依赖 jar 是什么？**     | 第三方库已经编译好的 `.class` 文件集合（压缩包）             |
| **依赖 jar 需要编译吗？** | 不需要，它已经是编译后的二进制文件                           |
| **classpath 是什么？**    | JVM 查找 class 文件的位置列表（类似 Windows 的 PATH）        |
| **classpath 包含什么？**  | 1. 你的 classes 目录 2. 模块依赖的 classes 目录 3. 第三方 jar 文件 |

