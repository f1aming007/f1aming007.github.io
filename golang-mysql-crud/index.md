# Golang Mysql CRUD


# Go 操作 sql server
* docker 安装 sqlserver
```bash
#拉取镜像
docker pull mcr.microsoft.com/azure-sql-edge
#启动容器
docker run --name azuresqledge -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=yourStrong(!)Password' -d -p 1433:1433 mcr.microsoft.com/azure-sql-edge

```

* 连接SQL数据库，要加载目标数据库的驱动，驱动里面包含了与数据库交互的逻辑
* sql.Open()
    * 数据库驱动的名称
    * 数据源名称
    * 得到一个指向sql.DB这个struct的指针
* sql.DB 时用来操作数据库的，它代码0个或多个底层连接的池，这些连接由sql包来维护，sql包自动的创建和释放这些连接
    * 它对于多个goroutine并发的使用时安全的
```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"

	_ "github.com/denisenkom/go-mssqldb"
)

var db *sql.DB

const (
	server   = "localhost"
	port     = "1433"
	user     = "sa"
	password = "Password"
	database = "go-db"
)

func main() {
	connStr := fmt.Sprintf("server=%s;user id=%s;password=%s;port=%s;database=%s;",
		server, user, password, port, database)
	db, err := sql.Open("sqlserver", connStr)
	if err != nil {
		log.Fatalln(err.Error())
	}

	ctx := context.Background()

	err = db.PingContext(ctx)
	if err != nil {
		log.Fatalln(err.Error())
	}
	fmt.Println("Connected!")
}
```

> **note**
* Open() 函数并不会连接到数据库，甚至不会验证参数，它只是把后续连接到数据库锁必须的struct给设置好
* 而真正的连接时在被需要的时候才进行懒设置的
* sql.DB 不需要进行关闭
* 它就是用来连接数据库的， 而不是实际的连接
* 这个抽象保护了数据库连的池， 对此进行维护
* 使用sql.DB 的时候，可以定义它的全局变量进行使用， 也可以将它传递给函数/方法里


## 如何获取驱动
正常的做法是使用sql.Register() 函数， 数据库驱动的名称和一个实现了driver.Driver 接口的struct， 来注册数据的驱动
* sql.Register("sqlserver", &drv{})

> 为什么例子中没有这句话
> 因为 sql 驱动， 在这个包别引入的时候进行自我注册
```go
_ "github.com/denisenkom/go-mssqldb"
```
## 驱动自我注册
* 在go-mssqldb包被引入的时候， 它的init函数将会运行并进行自我注册
* 在引入go-mssqldb 包的时候， 把该包的名称设置为下划线_，这是因为我们不能直接使用数据库驱动
    * 未来升级驱动， 无需改变代码
* Go语言没有提供官方的数据库驱动， 所有的数据库驱动都是第三方驱动，都遵循sql.driver包里面定义的接口

## func (db *DB) PingContext(ctx context.Context) 
* PingContext函数用来验证与数据库的连接是否仍然生效， ，如有必要则建立一个连接
* 这个韩式需要一个Context（上下文）类型的参数， 这种类型可以携带 截止时间。取消信号和其他请求范围的值，并且可以横跨API边界和进程
* 上例中， 创建context使用的context.Background()函数， 该函数返回一个非nil的空context，它不会被取消，它没有值，没有截止时间
* 通常在main函数， 初始化或测试中 作为传入请求的context

## 查询
* sql.Db 类型用于查询的方法有
    * Query
    * QueryRow
    * QueryContext
    * QueryRowContext

### Rows
* 返回类型
```go
type Rows struct {
	// contains filtered or unexported fields
}
```
* Rows的方法
    * func (*Rows) Close error 
    * func (rs *Rows) ColumnTypes() ([]*ColumnType, error)
    * func (rs *Rows) Columns() ([]string, error)
    * func (rs *Rows) Err() error
    * func (rs *Rows) Next() bool
    * func (rs *Rows) NextResultSet() bool
    * func (rs *Rows) Scan(dest ...any) error

### QueryRow
* 返回类型是
```go
type Row struct {
	// contains filtered or unexported fields
}
```
* Row的方法
    * func (r *Row) Err() error
    * func (r *Row) Scan(dest ...any) error

### 单条查询
```go
func getone(id int)(a app,err error) {
	a = app{}
	err = db.QueryRow("SELECT Id, Name, Status, Level, Order From dbo.App WHERE Id=@Id",
		sql.Named("Id", id)).
		Scan(&a.ID, &a.name, &a.status, &a.level, &a.order)
	return 
}
```
### 多条查询
```go
func getMany(id int) (apps []app, err error) {
	rows, err := db.Query("SELECT Id, Name, Status, Level, Order From dbo.App WHERE Id>@Id",
		sql.Named("Id", id))
	for rows.Next() {
		a := app{}
		err = rows.Scan(&a.ID, &a.name, &a.status, &a.level, &a.order)
		if err != nil {
			log.Fatalln(err.Error())
		}
		apps = append(apps, a)
	}
	return
}
```

## 更新
* sql.DB 类型上用于更新的方法
    * Exec
    * ExecContext

```go
func (a *app) update() (err error) {
	_, err = db.Exec("UPDATE dbo.App SET Name=@Name, Order=@Order WHERE Id=@Id",
		sql.Named("Name", a.name), sql.Named("Order", a.order), sql.Named("Id", a.ID))
	if err != nil {
		log.Fatalln(err.Error())
	}
	return 
}
```

## 删除

```go
func (a *app) delete() (err error) {
	_, err = db.Exec("DELETE FROM dbo.App WHERE Id=@Id", sql.Named("Id", a.ID))
	if err != nil {
		log.Fatalln(err.Error())
	}
	return 
}
```

## 其他
* Prepare
* prepareContext
* Transactions
    * Begin
    * BeginTx

### Prepare
```go
// 预处理查询示例
func prepareQueryDemo() {
	sqlStr := "select id, name, age from user where id > ?"
	stmt, err := db.Prepare(sqlStr)
	if err != nil {
		fmt.Printf("prepare failed, err:%v\n", err)
		return
	}
	defer stmt.Close()
	rows, err := stmt.Query(0)
	if err != nil {
		fmt.Printf("query failed, err:%v\n", err)
		return
	}
	defer rows.Close()
	// 循环读取结果集中的数据
	for rows.Next() {
		var u user
		err := rows.Scan(&u.id, &u.name, &u.age)
		if err != nil {
			fmt.Printf("scan failed, err:%v\n", err)
			return
		}
		fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
	}
}
```


### Transactions
```go
// 事务操作示例
func transactionDemo() {
	tx, err := db.Begin() // 开启事务
	if err != nil {
		if tx != nil {
			tx.Rollback() // 回滚
		}
		fmt.Printf("begin trans failed, err:%v\n", err)
		return
	}
	sqlStr1 := "Update user set age=30 where id=?"
	ret1, err := tx.Exec(sqlStr1, 2)
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Printf("exec sql1 failed, err:%v\n", err)
		return
	}
	affRow1, err := ret1.RowsAffected()
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Printf("exec ret1.RowsAffected() failed, err:%v\n", err)
		return
	}

	sqlStr2 := "Update user set age=40 where id=?"
	ret2, err := tx.Exec(sqlStr2, 3)
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Printf("exec sql2 failed, err:%v\n", err)
		return
	}
	affRow2, err := ret2.RowsAffected()
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Printf("exec ret1.RowsAffected() failed, err:%v\n", err)
		return
	}

	fmt.Println(affRow1, affRow2)
	if affRow1 == 1 && affRow2 == 1 {
		fmt.Println("事务提交啦...")
		tx.Commit() // 提交事务
	} else {
		tx.Rollback()
		fmt.Println("事务回滚啦...")
	}

	fmt.Println("exec trans success!")
}
```
