---
layout:     post
title:      Kafka 使用示例
subtitle:   Kafka
date:       2018-10-24
author:     tanjiquan
header-img: img/in-post/kafka-bg.png    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 消息队列
---
  
&emsp;&emsp;  本章节将介绍一下 Kafka 中的序列化问题。
<br>&emsp;&emsp;  在这个例子中 [kafka-example](https://github.com/tanjiquan/kafka-application/tree/master/kafka-example/readme.md)
主要是简单操作kafka，其中包含对生产kafka数据、消费kafka数据的示例，主要是对一些参数的介绍和认识。有兴趣者可参考。

### （一）序列化
   序列化主要是用来解决数据在网络中传输的问题. 在网络中传输的数据必须全是字节,也称为字节流. 而文本数据到字节数据的这一步就是序列化
#### 1.1 StringSerializer 序列化，StringDeserializer 反序列化
&emsp;&emsp;  StringSerializer、StringDeserializer 类很简单，就是实现 Serializer 接口，但是注意这个接口是
org.apache.kafka.common.serialization包下面的，String序列化很简单，就是把字符串和byte[]数组进行
相互转换，编码默认为UTF8。其实到最后，kafka接收和发送的都是以字节数据从形式。
#### 1.2 自定义序列化类型
&emsp;&emsp;自定义序列化类其实比较简单，简单一句话就是实现org.apache.kafka.common.serialization中的Serializer和Deserializer接口。
并且序列化的类（MessageData）也要实现Serializable 接口，最好重写 toString 方法，便于打印日志。
<br>&emsp;&emsp;  MessageDataSerializer 就是将传入的 MessageData 对象数据转换为byte[]，这里采用的是  JSON.toJSONBytes(data);
<br>&emsp;&emsp;  MessageDataDeSerializer 就是将 byte[] 转化为MessageData对象，这里采用JSON.parseObject(data, MessageData.class);
<br>&emsp;&emsp;  上面的序列化和反序列化类和String 的序列换和反序列化及其相似。

#### 1.3 avro 序列化/反序列化
&emsp;&emsp;  Avro 是一个数据序列化的系统，它可以将数据结构或对象转化成便于存储或传输的格式。
Avro设计之初就用来支持数据密集型应用，适合于远程或本地大规模数据的存储和交换。
<br>&emsp;&emsp;  JSON是一种常用的数据交换格式，很多系统都会使用JSON作为数据接口返回的数据格式，然而，由于JSON数据中包含大量的字段名字，
导致空间的严重浪费，尤其是数据文件较大的时候，而AVRO是一种更加紧凑的数据序列化系统，占用空间相对较少，更利于数据在网络当中的传输.
首先我们要将嵌套的JSON数据与AVRO文件的相互转换，主要使用 [avro-tools](http://mirrors.hust.edu.cn/apache/avro/avro-1.8.2/java/)
工具对这两种文件格式进行转换。在Hadoop的生态中，一般使用avro传输数据较为常见。使用见 kafka-example 中的示例。

（1）定义json对应的avro数据结构，数据结构大致如下：
json数据:
```text
{
	"data": [{
		"dataCommit": "commit",
		"dataId": 1
	}],
	"recordID": "1059390768",
	"sendIp": "127.0.0.1",
	"sendTime": 1539877033494
}
```
对应avro数据格式:
```txt
{
  "namespace": "com.kafka.example.avro",
  "type": "record",
  "name": "AvroMessageData",
  "fields": [
    {"name": "sendTime", "type": "long"},
    {"name": "recordID",  "type": "string"},
    {"name": "sendIp",  "type": "string"},
    {"name": "datas",  "type": "array",
        "items":{
             "type":"record",
             "name" : "AvroInnerData",
             "fields":[
                    {"name":"dataId","type":"int"},
                    {"name":"dataCommit","type":"string"}
              ]}
    }
  ]
}
```
对于以上 MessageData.avsc 文件需要注意
* 类型为array，因为从前面的JSON数据上可以看到，整个JSON数据是一个数组，因此，这里的类型为array，当类型为array时，必须指定items。
* 由于数组内部是记录形式，因此，在items里面的type是record，这里必须指定name和fields。
* fields里的name必须与JSON数据里的字段名保持一致。
* 值得注意的是，从前面的JSON数据可以看到，data信息为数组字段，因此，必须先指定name为Data，
而且type是一个对象，并不能单纯地指定为array，而是需要在对象里再用一个type来指定array，然后再加上items。

（2）根据 MessageData.avsc 文件生成 .java 文件，会生成 InnerData.java 和 MessageData.java
<br>`java -jar /path/to/avro-tools-1.8.0.jar compile schema <schema file> <destination>`
<br> 例如：进入 avro-tools-1.8.2.jar 的目录
<br> `java -jar /../avro-tools-1.8.2.jar compile schema /../MessageData.avsc  /../main/java/`

（3）指定如下配置，并且ProducerRecord 和 ConsumerRecords 分别指定为 AvroMessageData 类型
详情请见 AvroSerializerProducer 和 AvroSerializerConsumer
```text
props.put("value.serializer", "com.tt.kafka.example.message.serializer.AvroSerializer");
props.put(AvroSerializer.SCHEMA_CONFIG, AvroMessageData.SCHEMA$);
```

#### 1.4 protobuf 序列化/反序列化 <br> 
   Google Protocol Buffer( 简称 Protobuf) 是 Google公司内部的混合语言数据标准，使用较为复杂点，但是传输效率比较高。

   Protocol Buffers 是一种轻便高效的结构化数据存储格式，可以用于结构化数据串行化，或者说序列化。它很适合做数据存储或 RPC
数据交换格式。可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。

   在这里不在介绍protobuf 相关的使用，
