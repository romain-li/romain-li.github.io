---
title: 通过Flume Avro Event向Kafka+Flume推送自定义消息
tags:
  - Python
  - Kafka
  - Flume
  - Avro
date: 2017-04-24 16:14:48
---


## 背景

目前在项目中使用了Kafka+Flume的组合将消息落地到HDFS中。
由于某些业务相关的逻辑，在Headers中加入了参数并在HDFS落地时使用这些参数分目录进行持久化。使用HTTP Source配合JSONHandler测试通过，转到Kafka Source时却发现，找不到相关的解决方案。

<!-- more -->

示意图：
```
Tornado -> Kafka -> | Kafka Source -> Channel -> HDFS Sink | -> HDFS
                    | (Flume Agent)                        |
```

Flume的相关配置网上已经非常多了，也可以参考[Flume配置文档](https://flume.apache.org/FlumeUserGuide.html)，关键配置如下：
```
event.sinks.s_hdfs.hdfs.path = hdfs://localhost:8022/%{project_id}/%y-%m-%d/%H%M
```


## 过程

翻阅官方文档，发现可以通过Kafka Source中的`useFlumeEventFormat`参数解析相关头信息，但是官方及网上所能找到的示例均是配合Kafka Sink使用的，并未提及此处的Flume Avro binary format。

| Property Name | Default | Description |
|:---|:---|:---|
| useFlumeEventFormat | false | By default events are put as bytes onto the Kafka topic directly from the event body. Set to true to store events as the Flume Avro binary format. Used in conjunction with the same property on the KafkaSource or with the parseAsFlumeEvent property on the Kafka Channel this will preserve any Flume headers for the producing side. |

在[Flume源码](https://github.com/apache/flume/blob/release-1.6.0/flume-ng-core/src/main/java/org/apache/flume/serialization/FlumeEventAvroEventSerializer.java#L30)中可以找到相关的Schema。

```javascript
{
  "type": "record",
  "name": "Event",
  "fields": [
    {
      "name": "headers",
      "type": {
        "type": "map",
        "values": "string"
      }
    },
    {
      "name": "body",
      "type": "bytes"
    }
  ]
}
```

有了Schema，那么接下来只要在Python中实现Avro的序列化即可，使用其他语言也是类似的思路。
无奈Avro几乎没有什么Python的文档，只能自己去看源码。
用StringIO配合DataFileWriter尝试了一下，Flume后台解析报错，原因是DataFileWriter会写入额外的头信息或进行额外压缩：

```
KafkaSource EXCEPTION, {}
java.lang.IndexOutOfBoundsException
	at java.io.ByteArrayInputStream.read(ByteArrayInputStream.java:180)
	at org.apache.avro.io.DirectBinaryDecoder.doReadBytes(DirectBinaryDecoder.java:184)
	at org.apache.avro.io.BinaryDecoder.readString(BinaryDecoder.java:263)
	at org.apache.avro.io.ResolvingDecoder.readString(ResolvingDecoder.java:201)
	at org.apache.avro.generic.GenericDatumReader.readString(GenericDatumReader.java:430)
	...
```

根据[Flume源码](https://github.com/apache/flume/blob/trunk/flume-ng-sources/flume-kafka-source/src/main/java/org/apache/flume/source/kafka/KafkaSource.java#L231)来看，需要使用Python使用Avro实现一个等价于Java中`DirectBinaryEncoder`的编码逻辑，由于Python的Avro源码相对比较简单，只能使用`BinaryEncoder`与`DatumWriter`自行实现相关的逻辑了。


## 结论

使用Python + Kafka + Flume，通过指定Header内参数的方式，控制HDFS的持久化目录。

1. 配置Flume的Kafka Sink：
```
event.sinks.s_hdfs.hdfs.useFlumeEventFormat = true
event.sinks.s_hdfs.hdfs.path = hdfs://localhost:8022/%{project_id}/%y-%m-%d/%H%M
```

2. 安装Python的Avro相关依赖：
```sh
sudo apt-get install python-dev libsnappy-dev
sudo pip install python-snappy avro
```

3. Python端测试代码：
```python
import avro.schema
from avro.io import BinaryEncoder, DatumWriter
from StringIO import StringIO
from kafka import KafkaProducer

SCHEMA = avro.schema.parse("""
{
  "type": "record",
  "name": "Event",
  "fields": [
    {
      "name": "headers",
      "type": {
        "type": "map",
        "values": "string"
      }
    },
    {
      "name": "body",
      "type": "bytes"
    }
  ]
}
""")

def encode(data):
    buf = StringIO()
    encoder = BinaryEncoder(buf)
    writer = DatumWriter(SCHEMA)
    writer.write(data, encoder)
    output = buf.getvalue()
    return output

value = encode({"headers": {"project_id": "0"}, "body": "body"})
client = KafkaProducer(bootstrap_servers='localhost:9092')
client.send('event', value)
client.flush()
```

## 参考资料

- [Flume配置文档](https://flume.apache.org/FlumeUserGuide.html)
- [Flume源码](https://github.com/apache/flume)
- [Avro源码](https://github.com/apache/avro)
- [Python Avro Demo](https://github.com/phunt/avro-rpc-quickstart#python)