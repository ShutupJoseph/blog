# 第二节：Protocol Buffers

## 一、Protocol Buffers介绍
Protocol Buffers（简称 Protobuf）是由Google开发的一种轻巧高效的结构化数据存储格式（IDL，Interface Definition Language），用于序列化结构化数据。它类似于XML、JSON等数据交换格式，但比它们更小、更快、更简单。Protocol Buffers是Google公司专为gRPC协议开发的一种数据交换格式。

Protocol Buffers的主要特点包括：

1. **高效率**：Protocol Buffers生成的数据非常紧凑，序列化和反序列化过程速度快，适合对性能要求较高的场景。
2. **跨语言**：Protocol Buffers支持多种编程语言，如Java、C++、Python、Go等，方便不同语言之间的数据交换。
3. **易于使用**：Protocol Buffers提供了简单的接口定义语言（IDL），可以方便地定义数据结构，自动生成对应的序列化和反序列化代码。
4. **强类型**：Protocol Buffers定义的数据结构具有明确的类型，保证了数据的一致性和正确性。
5. **可扩展性**：Protocol Buffers允许在不破坏旧数据的基础上，扩展新的字段和数据结构，方便后续版本升级。
6. **丰富的数据类型**：Protocol Buffers支持多种数据类型，包括基本数据类型（如int32、float、double等）和复杂数据类型（如map、enum等）。

现在有许多框架等在使用Protocol Buffers。gRPC也是基于Protocol Buffers。 Protocol Buffers 目前有2和3两个版本号。本课程使用proto3版本。

## 二、安装依赖包
要在Python中使用`gRPC` ，我们首先要安装`grpcio` 和`grpcio-tools` ，`grpcio` 是用于运行`gRPC` 协议（服务端和客户端）的，而`grpcio-tools` 是用于将`Protocol Buffers` 语法代码转换为Python代码。安装命令如下：

```bash
$ pip install grpcio==1.64.1
$ pip install grpcio-tools==1.64.1
```

## 三、Protocol Buffers编写
### 1. 在Pycharm中安装插件
在Pycharm的插件配置中，在Marketplace中搜索Prottocol Buffers插件（注意是JetBrains官方出品的），然后点击安装，安装完后重启pycharm。

### 2. Protocol Buffers代码文件
在项目指定路径下，创建一个以`.proto` 结尾的文件，这个里面就是写Protocol Buffers代码的文件。

## 四、Protocol Buffers语法（proto 3）
### 1. 版本
Protocol Buffers目前有2和3两个版本，默认用的是2的版本，课程中用3的版本，所以需要在`.proto` 文件开头，添加以下代码即可申明版本号：

```protobuf
syntax = "proto3";
```

### 2. 注释
在Protocol Buffers中，提供以下两种注释方式：

```protobuf
// 单行注释
/*
多行注释
*/
```

### 3. 定义消息
消息可以说是定义一种数据结构，用于微服务间通信的约束。比如某个微服务定义了一个服务，这个服务接受什么格式的参数，返回的数据又是什么格式的，就是通过消息来定义的。以下是定义一个文章的消息：

```protobuf
message Article{
  // 消息字段格式：类型 字段名 参数位置
  int32 id = 1;
  string title = 2;
  string content = 3;
  string create_time = 4;
}
```

其中message是固定写法，Article是消息名称，Article中每行代表该消息中的一个字段，字段定义格式为：`type field_name = position` 。

字段位置范围为`[<font style="color:rgb(34, 34, 34);">1</font><font style="color:rgb(34, 34, 34);">,</font><font style="color:rgb(34, 34, 34);">536870911]</font>`之间，其中`[<font style="color:rgb(34, 34, 34);">19000</font><font style="color:rgb(34, 34, 34);">,</font><font style="color:rgb(34, 34, 34);">19999</font>]`是Protocol Buffers保留位置，我们不能使用。另外我们应该尽量将常用的字段的位置设置在`[1,15]`之间，因为`[1,15]`之间的位置字段只占用一个字节，超过15位置后的字段将占用2个字节。

### 4. 基本数据类型
| protobuf类型 | Python类型 | 说明 |
| --- | --- | --- |
| int32 | int | 对负数编码效率较低，如果出现附属，请使用sint32 |
| int64 | int/long | 对负数编码效率较低，如果出现附属，请使用sint64 |
| float | float |  |
| double | float |  |
| uint32 | int/long |  |
| uint64 | int/long |  |
| sint32 | int | 带符号32位整形，存放负数比int32更高效 |
| sint64 | int/long | 带符号32位整形，存放负数比int64更高效 |
| fixed32 | int | 4字节编码，如果值大于2^28，会比uint32高效 |
| fixed64 | int/long | 8字节编码，如果值大于2^56，会比uint64高效 |
| sfixed32 | int | 4字节编码 |
| sfixed64 | int/long | 8字节编码 |
| bool | bool |  |
| string | str/unicode | 必须采用utf-8编码，或者是7比特的ASCII文本，长度不能超过2^32 |
| bytes | bytes | Python3中的bytes类型，Python2中的str类型，长度不能超过2^32 |


### 5. 枚举
枚举类型通过`enum` 关键字来定义，比如定义一个名为`Corpus` 的枚举，并且在消息中使用的代码如下：

```protobuf
enum Corpus {
  CORPUS_UNSPECIFIED = 0;
  CORPUS_UNIVERSAL = 1;
  CORPUS_WEB = 2;
  CORPUS_IMAGES = 3;
  CORPUS_LOCAL = 4;
  CORPUS_NEWS = 5;
  CORPUS_PRODUCTS = 6;
  CORPUS_VIDEO = 7;
}

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 results_per_page = 3;
  Corpus corpus = 4;
}
```

枚举的第一个值必须从0开始，并且值不能重复，否则将会报错。（可以通过使用option allow_alias=true来允许存在相同的值。）

### 6. 列表
如果想要在消息中定义一个列表，那么可以通过`repeated` 来实现，示例代码如下：

```protobuf
message SearchResponse {
  repeated Result results = 1;
}

message Result {
  string url = 1;
  string title = 2;
  repeated string snippets = 3;
}
```

### 7. map类型
map相当于Python中的字典，但是key和value的字段类型需要做好限定。定义map类型的语法如下：

```protobuf
map<key_type, value_type> map_field = N;
```

其中`key_type` 可以是任何整数或字符串类型，`value_type` 可以是除`map` 类型以外的其他任何类型。比如想要根据名称返回项目数据，那么可以定义如下：

```protobuf
map<string, Project> projects = 3;
```

map使用注意事项：

+ map字段不能使用`repeated`。
+ map中的数据是无序的，不能依赖创建该map对象元素的特定顺序。
+ 如果出现重复的key，那么以最后一个为准。
+ 如果为map类型的字段只提供key，但是没有提供value，则该字段序列化的行为与语言有关（Python中使用默认值）。

### 8. OneOf
如果在一个消息中，定义了多个字段，但是同一时刻最多只会设置其中一个值，则可以使用OneOf来节省内存，示例代码如下：

```protobuf
message SampleMessage {
  oneof test_oneof {
    string name = 4;
    SubMessage sub_message = 9;
  }
}
```

在以上`SampleMessage` 中，你可以只设置`name` ，或者`sub_message` ，如果两个同时设置了，那么只有最后设置了值的字段有效。其中OneOf不能使用`repeated` 关键字。

### 9. 保留字段
在消息中有些字段在版本更新后可能不再需要了，这时候如果直接删除，那么调用方如果用的还是老版本的消息，那么可能会导致严重的错误，包括数据损坏，隐私泄露等问题。为了解决这个问题，我们可以通过`reversed` 关键字来将某些字段进行保留（指定该字段已删除）。如果将来任何用户尝试使用这些字段，protobuf编译器将报错。示例代码如下：

```protobuf
enum Foo {
  reserved 2, 15, 9 to 11, 40 to max;
  reserved "FOO", "BAR";
}
```

以上消息中，指定`2,15,9-11,40-max` 这些位置的字段将不能使用，且名为`FOO,BAR` 的字段也不能使用。

### 10. 默认值
解析消息时，如果编码消息不包含特定的隐式存在元素，则访问解析对象中的相应字段将返回该字段的默认值。这些默认值是特定于类型的：

+ 对于字符串，默认值是空字符串。
+ 对于字节，默认值是空字节。
+ 对于布尔值，默认值为 false。
+ 对于数字类型，默认值为零。
+ 对于枚举，默认值是**第一个定义的枚举值**，必须为 0。
+ 对于消息字段，该字段未设置。其确切值取决于语言。
+ 重复字段的默认值为空（对应Python中的空列表）

### 11. 更多
关于ProtocolBuffers语法更多请参考：[https://protobuf.dev/programming-guides/proto3/](https://protobuf.dev/programming-guides/proto3/)

## 五、定义服务
一个服务相当于一个类，在一个服务中可以定义多个`rpc` ，而`rpc` 则相当于该类中的方法。定义服务示例代码如下：

```protobuf
message SearchRequest{
    string query=1;
}

message SearchResponse{
    string result=1;
}

service SearchService {
  rpc Search(SearchRequest) returns (SearchResponse);
}
```

## 六、生成代码
Protobuf是一个独立于任何编程语言的IDL语言，他创建的代码并不能直接在Python中使用，需要借助Google提供的`grpcio-tools` 工具来将Protobuf代码转换为Python代码。

### 1. 安装grpcio-tools
首先确保grpcio-tools安装成功。安装命令如下：

```bash
$ pip install grpcio-tools==1.64.1
```

### 2. 生成Python代码
使用以下命令即可生成Python代码：

```bash
$ python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. [proto文件]
```

相关参数解释如下：

+ `-I`：表示“proto文件”所在的路径。
+ `--python_out`：表示生成包含数据结构的Python文件的路径。
+ `--grpc_python_out`：表示生成包含服务以及接口的Python文件的路径。

### 3. grpcio-tools更多
`grpcio-tools` 更多请参考：[https://grpc.io/docs/languages/python/quickstart/](https://grpc.io/docs/languages/python/quickstart/)



> 原文: <https://www.yuque.com/hynever/micro-service/hvrcgd5e11s5ybxd>