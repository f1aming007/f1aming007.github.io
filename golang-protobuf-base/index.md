# Golang Protobuf Base

![Minion](https://images.unsplash.com/photo-1661956601349-f61c959a8fd4?ixlib=rb-4.0.3&ixid=M3wxMjA3fDF8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1742&q=80)

# 🔥 gRpc 教程- protobuf 基础

* **序列化协议**： `grpc` 使用 `protobuf`, 首先使用protobuf 定义服务，然后使用这个文件来生成客户端和服务端的代码， 因为pb是跨语言的， 因此使用服务端和客户端语言并不一致也可以相互序列化和反序列化
* **网络传输层**： grpc使用的是`http2.0协议`，

<!--more-->

## Protobuf IDL
序列化通俗来说就是把内存的一段数据转化为二进制并存储或者通过网络传输， 读取磁盘或另一端收到后可以在内存中重建这段数据

1. `protobuf` 协议是跨语言跨平台的序列化协议
2. `protobuf` 本身并不是和`gPRC`绑定的， 它可以被用于非RPC场景， 如内存等
3. `json`、`xml` 都是一种序列化方式， 只是他们不需要提前预定义 idl， 且具备可读性，

https://protobuf.dev/programming-guides/proto3/

## 定义消息类型
`protobuf`里最基础的类型就是 `message`, 每一个`message`都会有一个或者多个字段（`field`）， 其中字段包含如下元素

* **类型**： 类型不仅可以是标量类型（int、string等），也可以是复合类型（enum等），也可以是其他message
* **字段名**： 字段名比较推荐的是使用下划线/分隔名称
* **字段编号**： 一个message内每一个字段编号都必须唯一的，在编码后其实传递的是这个编号而不是字段名
* **字段规则**：消息字段可以是以下字段之一
    * singular：格式正确的消息可以有零个或一个字段（但不能超过一个）。使用 proto3 语法时，如果未为给定字段指定其他字段规则，则这是默认字段规则
    * optional：与 singular 相同，不过您可以检查该值是否明确设置
    * repeated：在格式正确的消息中，此字段类型可以重复零次或多次。系统会保留重复值的顺序
    * map：这是一个成对的键值对字段
* **保留字段**：为了避免再次使用到已移除的字段可以设定保留字段。如果任何未来用户尝试使用这些字段标识符，协议缓冲区编译器就会报错

## 复合类型

### 数组
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

### 枚举
```protobuf
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
  enum Corpus {
    UNIVERSAL = 0;
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
  Corpus corpus = 4;
}
```

### 服务
定义的method仅能有一个入参和出参数。如果需要传递多个参数需要定义成message
```protobuf
service SearchService {
  rpc Search(SearchRequest) returns (SearchResponse);
}
```

### 使用其他消息类型
使用import引用另外一个文件的pb
```protobuf
syntax = "proto3";

import "google/protobuf/wrappers.proto";

package ecommerce;

message Order {
  string id = 1;
  repeated string items = 2;
  string description = 3;
  float price = 4;
  google.protobuf.StringValue destination = 5;
}
```

## protoc使用
protoc就是protobuf的编译器，它把proto文件编译成不同的语言

## 安装
* Linux, using apt or apt-get, for example:
```bash
$ apt install -y protobuf-compiler
$ protoc --version  # Ensure compiler version is 3+
```

* MacOS, using Homebrew[1]:
```bash
$ brew install protobuf
$ protoc --version  # Ensure compiler version is 3+
```

### 使用
```bash
$ protoc --help
Usage: protoc [OPTION] PROTO_FILES

  -IPATH, --proto_path=PATH   指定搜索路径
  --plugin=EXECUTABLE:
  
  ....
 
  --cpp_out=OUT_DIR           Generate C++ header and source.
  --csharp_out=OUT_DIR        Generate C# source file.
  --java_out=OUT_DIR          Generate Java source file.
  --js_out=OUT_DIR            Generate JavaScript source.
  --objc_out=OUT_DIR          Generate Objective C header and source.
  --php_out=OUT_DIR           Generate PHP source file.
  --python_out=OUT_DIR        Generate Python source file.
  --ruby_out=OUT_DIR          Generate Ruby source file
  
   @<filename>                proto文件的具体位置
```

### 搜索路径参数
* 第一个比较重要的参数就是搜索路径参数，即上述展示的-IPATH, --proto_path=PATH。它表示的是我们要在哪个路径下搜索.proto文件，这个参数既可以用-I指定，也可以使用--proto_path=指定

### 语言插件参数
Language
C++ (include C++ runtime and protoc)
Java
Python
Objective-C
C#
Ruby
PHP

### proto文件位置参数

proto文件位置参数即上述的@<filename>参数，指定了我们proto文件的具体位置，如proto1/greeter/greeter.proto

## golang插件
**安装**
非内置的语言支持就得自己单独安装语言插件，比如--go_out=对应的是protoc-gen-go，安装命令如下
```bash
# 最新版
$ go install google.golang.org/protobuf/cmd/protoc-gen-go@latest

# 指定版本
$ go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.3.0
```

生成代码
```bash
$ protoc --proto_path=src --go_out=out --go_opt=paths=source_relative foo.proto bar/baz.proto
```
