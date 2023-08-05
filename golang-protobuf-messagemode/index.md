# Golang Protobuf MessageMode

# ![Minion](https://images.unsplash.com/photo-1661956603025-8310b2e3036d?ixlib=rb-4.0.3&ixid=M3wxMjA3fDF8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1740&q=80)
# 🔥 gRpc 教程- protobuf 通信模式
**四种通信模式**
1. Simple RPC
2. Server-Streaming RPC
3. Client-Streaming RPC
4. Bidirectionnal-Streaming RPC

<!--more-->

## Simple RPC
```protobuf
syntax = "proto3";

package ecommerce;

import "google/protobuf/wrappers.proto";

option go_package = "ecommerce/";

message Order {
  string id = 1;
  repeated string items = 2;
  string description = 3;
  float price = 4;
  string destination = 5;
}

service OrderManagement {
  rpc getOrder(google.protobuf.StringValue) returns (Order);
}
```
注意事项
* 使用protobuf最新版本syntax = "proto3";
* protoc-gen-go要求 pb 文件必须指定 go 包的路径。即option go_package = "ecommerce/";
* 定义的method仅能有一个入参和出参数。如果需要传递多个参数需要定义成message
* 使用import引用另外一个文件的 pb。google/protobuf/wrappers.proto是 google 内置的类型

生成 go 和 grpc 的代码
```bash
$ protoc -I ./pb \
  --go_out ./ecommerce --go_opt paths=source_relative \
  --go-grpc_out ./ecommerce --go-grpc_opt paths=source_relative \
  ./pb/product.proto
```

### server实现

