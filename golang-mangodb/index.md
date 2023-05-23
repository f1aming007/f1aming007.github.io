# Golang MangoDB


# MongoDB

## MongoDB介绍
* mongoDB是基于分布式文件存储的数据库，是一个介于关系型数据库和非关系数据库之间的产品
* mongoDB将一条数据存储为一个文档， 数据结构由键值对组成， 其中文档类似我们编辑中用到的JSON对象，文档中的字段值可以包含其他文档/数组及文档数组


| MongoDB术语 | 说明                                 | SQL术语     |
|-------------|--------------------------------------|-------------|
| database    | 数据库                               | database    |
| collection  | 集合                                 | table       |
| document    | 文档                                 | row         |
| field       | 字段                                 | column      |
| index       | index                                | 索引        |
| primary key | 主键 MongoDB自动将_id字段设置为主键	 | primary key |


## mongoDB安装

```bash
docker pull mongo:latest

docker run -itd --name mongo -p 27017:27017 mongo --auth
```
* 参数说明
    * -p 27017:27017 ：映射容器服务的 27017 端口到宿主机的 27017 端口。外部可以直接通过 宿主机 ip:27017 访问到 mongo 的服务。
    * --auth：需要密码才能访问容器服务。

使用以下命令添加用户和设置密码，并且尝试连接
```bash
$ docker exec -it mongo mongosh admin
# 创建一个名为 admin，密码为 123456 的用户。
>  db.createUser({ user:'admin',pwd:'123456',roles:[ { role:'userAdminAnyDatabase', db: 'admin'},"readWriteAnyDatabase"]});
# 尝试使用上面创建的用户信息进行连接。
> db.auth('admin', '123456')
```

* 连接 mongo
```bash
docker exec -it mongo mongosh admin
```

## golang 操作mongo DB

* 安装mongoDB Go驱动包

```bash
go get github.com/mongodb/mongo-go-driver
```

* Go代码连接mongoDB
```go
package main

import (
	"context"
	"fmt"
	"log"

	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

func main() {
	// 设置客户端连接配置
	clientOptions := options.Client().ApplyURI("mongodb://192.168.0.9:27017")

	// 连接到MongoDB
	client, err := mongo.Connect(context.TODO(), clientOptions)
	if err != nil {
		log.Fatal(err)
	}

	// 检查连接
	err = client.Ping(context.TODO(), nil)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("Connected to MongoDB!")
}
```

* 处理数据集
```go
// 指定获取要操作的数据集
collection := client.Database("demo").Collection("student")
```
* 处理完任务之后可以通过下面的命令断开与MongoDB的连接
```go
// 断开连接
err = client.Disconnect(context.TODO())
if err != nil {
	log.Fatal(err)
}
fmt.Println("Connection to MongoDB closed.")
```

## 连接池模式
```go
import (
	"context"
	"time"

	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

func ConnectToDB(uri, name string, timeout time.Duration, num uint64) (*mongo.Database, error) {
	ctx, cancel := context.WithTimeout(context.Background(), timeout)
	defer cancel()
	o := options.Client().ApplyURI(uri)
	o.SetMaxPoolSize(num)
	client, err := mongo.Connect(ctx, o)
	if err != nil {
		return nil, err
	}

	return client.Database(name), nil
}
```

## BSON
* mongoDB中的JSON文档存储在BSON的二进制表示中
* BSON编码拓展了JSON表示，使其包含额外的类型如 int\long\date\浮点数和decimal128，
* 是应用程序更容易可靠的处理、排序和比较数据

要使用BSON 需要倒入包
```go
import "go.mongodb.org/mongo-driver/bson"
```

* 使用D类型构建的过滤文档的理智， 它可以用来查询name字段与’张三’或’李四’匹配的文档
```go
bson.D{{
	"name",
	bson.D{{
		"$in",
		bson.A{"张三", "李四"},
	}},
}}
```

## CURD

* 定义一个Student类型
```go
type Student struct {
	Name string
	Age int
}
```

* 准备数据

```go
s1 := Student{"张三", 12}
s2 := Student{"李四", 10}
s3 := Student{"王五", 11}
```

* 插入一条文档记录
```go
insertResult, err := collection.InsertOne(context.TODO(), s1)
if err != nil {
	log.Fatal(err)
}

fmt.Println("Inserted a single document: ", insertResult.InsertedID)
```

* 插入多条文档
```go
students := []interface{}{s2, s3}
insertManyResult, err := collection.InsertMany(context.TODO(), students)
if err != nil {
	log.Fatal(err)
}
fmt.Println("Inserted multiple documents: ", insertManyResult.InsertedIDs)
```

### 更新文档
* updateone() 方法允许你更新当个文档，需要一个筛选器文档来批评数据库中的文档， 并需要一个更新文档来描述更新操作， 使用bson.D 类型来构建筛选文档和更新文档

```go
filter := bson.D{{"name", "小兰"}}

update := bson.D{
	{"$inc", bson.D{
		{"age", 1},
	}},
}

updateResult, err := collection.UpdateOne(context.TODO(), filter, update)
if err != nil {
	log.Fatal(err)
}
fmt.Printf("Matched %v documents and updated %v documents.\n", updateResult.MatchedCount, updateResult.ModifiedCount)
```

## 查找文档
* 找到一个文档，需要一个filter文档， 以及一个指向可以将结构解码为其值的指针
* 要查找单个文档， 使用collection.FindOne()。这个方法返回一个可以解码为值的结果
```go
// 创建一个Student变量用来接收查询的结果
var result Student
err = collection.FindOne(context.TODO(), filter).Decode(&result)
if err != nil {
	log.Fatal(err)
}
fmt.Printf("Found a single document: %+v\n", result)
```

* 要查找多个文档，请使用collection.Find()。此方法返回一个游标。游标提供了一个文档流，你可以通过它一次迭代和解码一个文档。当游标用完之后，应该关闭游标。下面的示例将使用options包设置一个限制以便只返回两个文档。
```go
// 查询多个
// 将选项传递给Find()
findOptions := options.Find()
findOptions.SetLimit(2)

// 定义一个切片用来存储查询结果
var results []*Student

// 把bson.D{{}}作为一个filter来匹配所有文档
cur, err := collection.Find(context.TODO(), bson.D{{}}, findOptions)
if err != nil {
	log.Fatal(err)
}

// 查找多个文档返回一个光标
// 遍历游标允许我们一次解码一个文档
for cur.Next(context.TODO()) {
	// 创建一个值，将单个文档解码为该值
	var elem Student
	err := cur.Decode(&elem)
	if err != nil {
		log.Fatal(err)
	}
	results = append(results, &elem)
}

if err := cur.Err(); err != nil {
	log.Fatal(err)
}

// 完成后关闭游标
cur.Close(context.TODO())
fmt.Printf("Found multiple documents (array of pointers): %#v\n", results)
```

## 删除文档
使用collection.DeleteOne()或collection.DeleteMany()删除文档。如果你传递bson.D{{}}作为过滤器参数，它将匹配数据集中的所有文档。还可以使用collection. drop()删除整个数据集。
```go
// 删除名字是小黄的那个
deleteResult1, err := collection.DeleteOne(context.TODO(), bson.D{{"name","小黄"}})
if err != nil {
	log.Fatal(err)
}
fmt.Printf("Deleted %v documents in the trainers collection\n", deleteResult1.DeletedCount)
// 删除所有
deleteResult2, err := collection.DeleteMany(context.TODO(), bson.D{{}})
if err != nil {
	log.Fatal(err)
}
fmt.Printf("Deleted %v documents in the trainers collection\n", deleteResult2.DeletedCount)
```

* 参考链接
https://pkg.go.dev/go.mongodb.org/mongo-driver/mongo
