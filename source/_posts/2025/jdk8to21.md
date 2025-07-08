---
title: 'JDK从8升级到21的问题集'
date: 2025-02-21
description: 'JDK从8升级到21的问题集'
# topic: leetcode
author: hzlei
# banner: https://cdn.jsdelivr.net/gh/hzleii/imgs/stellar/post/2024/ruoyi-rce/index.webp
# cover: https://cdn.jsdelivr.net/gh/hzleii/imgs/stellar/post/2024/ruoyi-rce/index.webp
nav_tabs: true
article:
  type: # tech/story
poster: # 海报（可选，全图封面卡片）
  # topic: 标题上方的小字 # 标题上方的小字，可选
  headline: JDK从8升级到21的问题集 # 必选
  caption:  # 标题下方的小字，可选
  color: hsl(154deg 100% 61%) # 标题颜色，可选，默认为跟随主题的动态颜色 # white,red...
---


## 背景与挑战

### 升级原因

- Oracle长期支持策略

- 现代特性需求：协程、模式匹配、ZGC等

- 安全性与性能的需求

- AI新技术引入的版本要求


### 项目情况

- 多项目并行升级的协同作战

- 多技术栈并存

- 持续集成体系的适配挑战


## 主要问题域与解决方案

### 依赖管理的“蝴蝶效应”


- sun.misc.BASE64Encoder等内部API废弃 → 引发编译错误
- JAXB/JAX-WS从JDK核心剥离 → XML处理链断裂
- Lombok与新版编译器兼容性问题（尤其record类型）
核心原因在于JEP320提案：[openjdk.org/jeps/320](https://openjdk.org/jeps/320)


{% box child:tabs %}
{% tabs %}

<!-- tab 案例1：{% mark 历史SDK的编译陷阱 color:red %} -->

```less
Compilation failure: Compilation failure:
#14 4.173 [ERROR] 不再支持源选项 6。请使用 8 或更高版本。
#14 4.173 [ERROR] 不再支持目标选项 6。请使用 8 或更高版本。
```

```xml
<!-- 旧版本编译器配置导致构建失败 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.5</version>
    <configuration>
        <source>1.6</source>
        <target>1.6</target>
    </configuration>
</plugin>
```

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.13.0</version>
    <configuration>
        <release>8</release><!-- 统一使用release参数 -->
    </configuration>
</plugin>
```

<!-- tab 案例2：{% mark JAXB的模块化剥离 color:red %} -->

```less
javax.xml.bind.JAXBException:Implementation of JAXB-API has not been found
```

```xml
<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
    <version>4.0.5</version>
</dependency>
```

{% endtabs %}
{% endbox %}

{% box child:tabs %}
{% tabs %}

<!-- tab 案例3：{% mark Lombok与新版编译器兼容性问题 color:red %} -->

```less
java: java.lang.NoSuchFieldError
```

```xml
<dependency>
 <groupId>org.projectlombok</groupId>
 <artifactId>lombok</artifactId>
 <version>1.18.30</version>
</dependency>
```

<!-- tab 案例4：{% mark Resource注解找不到 color:red %} -->

```less
Caused by: java.lang.NoSuchMethodError: 'java.lang.String javax.annotation.Resource.lookup()'
at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor$ResourceElement.<init>(CommonAnnotationBeanPostProcessor.java:664)
at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.lambda$buildResourceMetadata$0(CommonAnnotationBeanPostProcessor.java:395)
at org.springframework.util.ReflectionUtils.doWithLocalFields(ReflectionUtils.java:669)
at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.buildResourceMetadata(CommonAnnotationBeanPostProcessor.java:377)
at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.findResourceMetadata(CommonAnnotationBeanPostProcessor.java:358)
at org.springframework.context.annotation.CommonAnnotationBeanPostProcessor.postProcessMergedBeanDefinition(CommonAnnotationBeanPostProcessor.java:306)
at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.applyMergedBeanDefinitionPostProcessors(AbstractAutowireCapableBeanFactory.java:1116)
at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:594)
... 37 more
```

```xml
<dependency>
    <groupId>jakarta.annotation</groupId>
    <artifactId>jakarta.annotation-api</artifactId>
    <version>1.3.5</version>
</dependency>

<dependency>
    <groupId>javax.annotation</groupId>
    <artifactId>javax.annotation-api</artifactId>
    <version>1.3.2</version>
</dependency>

# 上述两个依赖代码基本一样，推荐使用该版本：
# jakarta.annotation:jakarta.annotation-api。
```

{% endtabs %}
{% endbox %}


### 模块化的破与立

1. 反射访问的模块墙

```less
[ERROR] Unable to make field private int java.text.SimpleDateFormat.serialVersionOnStream accessible
```

```java
# 启动参数添加模块开放配置
--add-opens java.base/java.text=ALL-UNNAMED
--add-opens java.base/java.lang.reflect=ALL-UNNAMED
```

2. 完整模块开放配置模板

```java
export JAVA_OPTS="-Djava.library.path=/usr/local/lib -server -Xmx4096m --add-opens java.base/sun.security.action=ALL-UNNAMED
--add-opens java.base/java.lang=ALL-UNNAMED
--add-opens java.base/java.math=ALL-UNNAMED
--add-opens java.base/java.util=ALL-UNNAMED
--add-opens java.base/sun.util.calendar=ALL-UNNAMED
--add-opens java.base/java.util.concurrent=ALL-UNNAMED
--add-opens java.base/java.util.concurrent.locks=ALL-UNNAMED
--add-opens java.base/java.security=ALL-UNNAMED
--add-opens java.base/jdk.internal.loader=ALL-UNNAMED
--add-opens java.management/com.sun.jmx.mbeanserver=ALL-UNNAMED
--add-opens java.base/java.net=ALL-UNNAMED
--add-opens java.base/sun.nio.ch=ALL-UNNAMED
--add-opens java.management/java.lang.management=ALL-UNNAMED
--add-opens jdk.management/com.sun.management.internal=ALL-UNNAMED
--add-opens java.management/sun.management=ALL-UNNAMED
--add-opens java.base/sun.security.action=ALL-UNNAMED
--add-opens java.base/sun.net.util=ALL-UNNAMED
--add-opens java.base/java.time=ALL-UNNAMED
--add-opens java.base/java.lang.reflect=ALL-UNNAMED
--add-opens java.base/java.io=ALL-UNNAMED"
```



### 语法层面的"时空穿越"

{% box child:tabs %}
{% tabs %}

<!-- tab 案例1：Base64编解码改造 -->


<!-- cell -->

**JDK8写法（已废弃）**
{% box child:codeblock color:red %}

```java
BASE64Encoder encoder = newBASE64Encoder();
String encoded = encoder.encode(data);
```

{% endbox %}

<!-- cell -->

**JDK21规范写法**
{% box child:codeblock color:green %}

```java
Base64.Encoder encoder = Base64.getEncoder();
String encoded = encoder.encodeToString(data);
```

{% endbox %}


<!-- tab 案例2：日期序列化问题 -->

```less
Caused by:java.lang.reflect.InaccessibleObjectException: 
Unable to make field private int java.text.SimpleDateFormat.serialVersionOnStream accessible
```

解决方案：

1. 使用`DateTimeFormatter`替代`SimpleDateFormat`

2. 或添加模块开放参数：`--add-opens java.base/java.text=ALL-UNNAMED`

{% endtabs %}
{% endbox %}

### 隐秘的"依赖战争"

注解包冲突典型案例：

```less
[ERROR] javax.annotation.Resource exists in both 
jsr250-api-1.0.jar and jakarta.annotation-api-1.3.5.jar
```

```xml
<!-- 统一使用Jakarta标准 -->
<dependency>
    <groupId>jakarta.annotation</groupId>
    <artifactId>jakarta.annotation-api</artifactId>
    <version>2.1.1</version>
</dependency>
<!-- 排除旧版本依赖 -->
<exclusions>
    <exclusion>
        <groupId>javax.annotation</groupId>
        <artifactId>jsr250-api</artifactId>
    </exclusion>
</exclusions>
```

### 构建体系的改造

Maven插件兼容性问题：

```less
[ERROR] The plugin org.apache.maven.plugins:maven-compiler-plugin:3.13.0 
requires Maven version 3.6.3
```

`升级策略：`

1. 升级Maven版本

2. 统一插件版本

```xml
<build>
    <pluginManagement>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <version>3.4.0</version>
            </plugin>
        </plugins>
    </pluginManagement>
</build>
```


## 最佳实践总结

### 本地编译

在本地进行编译，提前识别出语法错误、版本冲突及不兼容问题。

主要有以下几种场景：
1. Base64：参照 【Base64编解码改造】
2. lombok：升级版本
3. jsr250、jaxb-runtime、jakarta.annotation-api：参照 【注解包冲突典型案例】
4. maven-compiler-plugin：升级版本
5. maven-resources-plugin：升级版本
6. maven-war-plugin：升级版本


### 行云构建

同【本地编译】

### 行云部署

1. 镜像不匹配：自定义镜像或者使用已申请的jdk21镜像
2. module权限不够：参照【完整模块开放配置模板】
3. JDSecurity加解密

所有数据库操作：
`important.properties`配置文件的处理方式
`classpath:important.properties`
使用{% mark PropertyPlaceholderConfigurer color:green %}进行处理
不要用{% mark JDSecurityPropertyFactoryBean color:red %}。

```xml
<!--    <bean id ="secApplicationProperties" class="com.jd.security.configsec.spring.config.JDSecurityPropertyFactoryBean">-->
<!--       <property name="ignoreResourceNotFound" value="true" />-->
<!--       <property name="secLocation" value="classpath:important.properties"/>-->
<!--    </bean>-->
```

### 运行

1. 序列化异常

jdk21使用列表视图作为入参，导致jsf接口进行反序列化报错。报错代码如下：


**报错代码如下：**
{% box child:codeblock color:red %}
```java
List<String> subList = venderCodes.subList(i * batchSize, Math.min(venderCodes.size(), (i + 1) * batchSize));
VendorQueryVo vendorQueryVo = new VendorQueryVo();
vendorQueryVo.setVendorCodes(subList);
// 该接口最多支持100条调用
List<VendorVo> batchVendorNameByVendorCode = vendorBaseInfoService.getBatchVendorNameByVendorCode(vendorQueryVo, I18NParamFactory.getJDI18nParam());
```
{% endbox %}


将
vendorQueryVo.setVendorCodes(subList)
修改为 
vendorQueryVo.setVendorCodes(new ArrayList<>(subList)) 
即可解决问题


{% box child:codeblock color:green %}
```java
List<String> subList = venderCodes.subList(i * batchSize, Math.min(venderCodes.size(), (i + 1) * batchSize));
VendorQueryVo vendorQueryVo = new VendorQueryVo();
vendorQueryVo.setVendorCodes(new ArrayList<>(subList));
// 该接口最多支持100条调用
List<VendorVo> batchVendorNameByVendorCode = vendorBaseInfoService.getBatchVendorNameByVendorCode(vendorQueryVo, I18NParamFactory.getJDI18nParam());
```
{% endbox %}


2. 线程上下文类找不到：使用多线程场景下尽可能使用显式指定线程池【默认情况下 不同运行环境的处理机制不同】


### JVM调优

垃圾回收调优：

1. UseParallelGC
2. UseG1GC
3. UseZGC

是 Java 虚拟机（JVM）中三种不同的垃圾回收器（Garbage Collector, GC），它们的设计目标和使用场景有所不同。以下是它们的区别：





 | 特性 | UseParallelGC | UseG1GC | UseZGC | 
 | ----- | ----- | ----- | ---- | 
 | 设计目标 | 高吞吐量 | 平衡吞吐量和延迟 | 极低延迟 | 
 | 暂停时间 | 较长 | 较短 | 极短 | 
 | 适用堆大小 | 中小堆（几 GB 到几十 GB） | 大堆（几十 GB 到几百 GB） | 超大堆（TB 级别） | 
 | CPU 消耗 | 中等 | 中等 | 较高 | 
 | 适用场景 | 批处理、计算密集型任务 | 对延迟有一定要求的应用 | 对延迟极其敏感的应用 | 




1. 如果你的应用对吞吐量要求高，且可以接受较长的暂停时间，选择UseParallelGC。
2. 如果你的应用对延迟有一定要求，且堆内存较大，选择UseG1GC。
3. 如果你的应用对延迟极其敏感，且堆内存非常大，选择UseZGC。

{% quot 仅供参考，具体请按照实际情况来进行调整。 %}





