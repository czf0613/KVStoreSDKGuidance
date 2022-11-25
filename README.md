# 键值对存储

## 历史渊源

对于任何的程序来说，都有存储配置文件的需求，比如说安卓里面的SharedPreferences，iOS里面的UserDefaults，Windows里面的ini，浏览器里面的xxxStorage等等。

但是当现在的app开始逐渐走向“跨平台”，多用户的趋势的时候，传统的配置文件会越来越成为瓶颈，主要是以下的几个问题。

### 1，类型不安全

大部分的配置文件，本质上都是文本文件或者xml之类的标记语言。因此无论是什么数据类型，都会被当成“字符串”进行处理，因而在某些环境下可能会导致类型不安全的问题。

举个例子来说，app_version = 1;你觉得这个1是int32呢，还是uint32呢还是string呢？

### 2，无法同步

举个例子，比如说app卸载重装，或者说iOS端和安卓双栈的app，同步设置是一个非常常见的功能。如果按照传统的配置文件格式的话，安卓的SharedPreferences对应的xml语法和iOS的UserDefaults对应的plist语法是不兼容的，如果要进行同步，就必须要自己手动实现关于同步的转换功能。

同时，也有可能因为设备突然间crash或者遭遇故障，导致配置文件没来得及写入磁盘，又引发数据丢失问题。

### 3，储存数量有限

毕竟手机内的存储是寸土寸金的，保存的配置文件必然不能过于庞大，也不能读写某些相对较大的二进制内容。

### 4，存在破解的风险

如果配置文件不进行加密，那么被root过后的安卓手机，就可以随意修改app的配置信息，从而导致无法预料的问题。但是如果要对整个配置文件进行加密，又会产生别的问题，比如说安全的加密算法，加密的密钥存去哪里等等的问题。

### 5，并发&性能问题

基于配置文件的读写，是没有办法（准确来说应该是很难，甚至还得祭出MVCC来实现）进行并发操作的，即便我修改了不一样的条目，也不能同时执行。

同时，当配置文件非常庞大，条目非常多的时候，配置文件的解析、读取也是十分消耗性能的，因为它们都需要经过类似xml或者json之类的反序列化之后，才能为app所用。

### 6，难以追踪版本变化&分析

比如说这个配置的条目，什么时候，被谁改过；以及这些记录读取的频次、热点高不高。这些都是很难单靠配置文件来实现的。

综上所述，我们决定开发一套基于云储存的KV键值对保存工具。目前而言，这个项目的云端存储具体实现暂未开放源代码授权。

## 项目结构

``` mermaid
graph LR;
getOrCreateApp[申请AppId] --> appKey(服务端下发appId与appKey)
appKey --调用REST接口--> manage[即可进行增删改查数据]
```

## 接口文档

目前开放大众使用的接口只有四个，分别对应获取AppId、数据的GET、PUT、DELETE操作。

在所有接口中，我们都遵循以下的约定：

1，编码方式使用utf-8，不然会搞出来乱码。

2，使用json传递int64或者uint64的时候，javaScript可能会出现越界问题，需要前端手动进行处理，市面上有很多成熟的解决方案。

3，每个键值对的KeyIdentifier与UserIdentifier尽可能不要使用非ASCII字符和控制字符（我只是说尽可能不要这么用，因为我也没有办法保证在我们的服务端、数据库之间的任何一个环节会不会出现问题，用英文字母+半角符号肯定是最保险的）

4，目前这个版本仍在迭代过程中，并没有办法防范一些潜在的攻击行为，日后可能在软件的使用流程上进行改进。

5，由于键值对的特性，出现同Key的情况下，会发生覆盖并且不会有任何的异常提示。

## 键值对的REST接口

路径都是一样的，仅仅只有GET、PUT、DELETE方法的区别，参数如下：

### 获取值

#### 请求参数：

```http
GET https://https://kv.kevinc.ltd/data/manageData?key=yourKey&user=yourUser

X-AppId: xxxxxxxxxxx
X-AppKey: xxxxxxxxxxxx
```

需要在Header中提供AppId与AppKey。

有两个Query Parameter。键值对在App内的每一个用户下唯一，因此键值对的Key既包含KeyIdentifier和UserIdentifier两部分。

#### 返回参数：

``` json
{
    "valueTypeIndicator": 3,
    "isArray": true,
    "keyIdentifier": "test_uint64_arr",
    "userIdentifier": "czf",
    "int32Value": 0,
    "uint32Value": 0,
    "int64Value": 0,
    "uint64Value": 0,
    "booleanValue": false,
    "floatValue": 0,
    "doubleValue": 0,
    "stringValue": "",
    "byteStringValue": "",
    "int32Values": [],
    "uint32Values": [],
    "int64Values": [],
    "uint64Values": [
        123457895,
        5613165132,
        121354846513,
        789545612,
        852741963
    ],
    "booleanValues": [],
    "floatValues": [],
    "doubleValues": [],
    "stringValues": []
}
```

读取数据时，需要关注三个地方：valueTypeIndicator、isArray与对应的field。

valueTypeIndicator的取值范围如下：

```csharp
public enum DataType
{
    Int32 = 0,
    Int64 = 1,
    UInt32 = 2,
    UInt64 = 3,
    Boolean = 4,
    Float = 5,
    Double = 6,
    String = 7,
    /// <summary>
    /// 是bytes，不是一个字节，这里本质上相当于byte array
    /// </summary>
    Bytes = 8
}
```

然后就是isArray字段，如果isArray为真的时候，则需要从对应数据类型的数组进行读取。

以我们上面的返回做例子，valueTypeIndicator是3，表示是一个UInt64类型，同时isArray为true，因此，只需要读取json里面的"uint64Values"字段就可以获得对应的值了。

### 更新值

```http
PUT https://https://kv.kevinc.ltd/data/manageData

X-AppId: xxxxxxxxxxx
X-AppKey: xxxxxxxxxxxx
```

#### 请求参数：

需要在header中附带AppId与AppKey，剩下的参数在body中进行传输

```json
{
    "valueTypeIndicator": 3,
    "isArray": true,
    "keyIdentifier": "test_uint64_arr",
    "userIdentifier": "czf",
    "uint64Values": [
        123457895,
        5613165132,
        121354846513,
        789545612,
        852741963
    ]
}
```

同样的道理，看看GET接口那里约定的valueTypeIndicator和isArray就能明白了。

需要说明一下的是KeyIdentifier与UserIdentifier。我们认为的键值对，是在每个app，每个人下唯一的。因此，不同的用户，用相同的KeyIdentifier是没有问题的；不同的appId下，出现重名的UserIdentifier也是没有问题的。

#### 返回值：

没有返回值，只用看状态码就行。

状态码为200表示写入成功。

状态码为400可能是因为写入的数据过大，我们限制写入的数据量不能超过16MB（这个值不精确，因为可能会有padding之类的问题，实际的能写入的值应该只有15MB多一些）

### 删除值

#### 请求参数：

```http
DELETE https://https://kv.kevinc.ltd/data/manageData?key=yourKey&user=yourUser

X-AppId: xxxxxxxxxxx
X-AppKey: xxxxxxxxxxxx
```

需要在Header中提供AppId与AppKey。

#### 返回值：

永远都是返回成功，不管给出的Key有没有意义。
