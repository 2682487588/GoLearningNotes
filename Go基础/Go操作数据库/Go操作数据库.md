## 1.Go操作MySQL

### 1.1连接

#### 1.1.1下载依赖

```bash
go get -u github.com/go-sql-driver/mysql
```

#### 1.1.2使用MySQL驱动

```go
func Open(driverName, dataSourceName string) (*DB, error)
```

**Open**打开一个**dirverName指定的数据库**，**dataSourceName指定数据源**，一般至少包括数据库文件名和其它连接必要的信息。

```go
import (
	"database/sql"

	_ "github.com/go-sql-driver/mysql"
)

func main() {
   // DSN:Data Source Name
	dsn := "user:password@tcp(127.0.0.1:3306)/dbname"
	db, err := sql.Open("mysql", dsn)
	if err != nil {
		panic(err)
	}
	defer db.Close()  // 注意这行代码要写在上面err判断的下面
}
```

#### 1.1.3  初始化连接

**Open函数可能只是验证其参数格式是否正确**，实际上并不创建与数据库的连接。如果要检查**数据源的名称是否真实有效**，应该**调用Ping方法**



```go
package main

import (
	"database/sql"
	"fmt"
	_ "github.com/go-sql-driver/mysql"
)

//需要注意 这里需要引用自己的mysql文件

var db *sql.DB
func initDB()(err error)  {
	//账号 密码 端口号(tcp)127.0.0.1:3306 表名 字符集  校验时间
	 dsn := "root:123456@tcp(127.0.0.1:3306)/gomysql?charset=utf8mb4&parseTime=true"
	 //加载驱动
	 //这里需要是=而不是:=因为我们是给全局变量(db)赋值
	 db,err = sql.Open("mysql",dsn)
	if err!=nil {
		return err
	}
	//尝试和数据库建立连接(校验dsn正确)
	//然后用了ping命令
	err=db.Ping()
	if err!=nil {
		return err
	}
	return nil
}
func main() {
	err := initDB()
	if err!=nil {
		fmt.Printf("connect failed,err:%v\n",err)
		return
	}
}
```

#### 1.1.4SetMaxOpenConns

`SetMaxOpenConns`设置与数**据库建立连接的最大数目**。 如果n大于0且小于最大**闲置连接数**，会将**最大闲置连接数减小到匹配最大开启连接数的限制**。 如果n<=0，不会限制最大开启连接数，**默认为0（无限制）**

#### 1.1.5SetMaxIdleConns

```go
func (db *DB) SetMaxIdleConns(n int)
```

连接池中的**最大闲置连接数**

 如果n**大于最大开启连接数**，则新的最大闲置连接数会减小到匹配**最大开启连接数的限制**。 如果**n<=0，不会保留闲置连接**。

### 1.2CRUD

#### 1.2.1 建库建表

我们先在MySQL中创建一个名为`sql_test`的数据库

```sql
CREATE DATABASE sql_test;
```

进入该数据库:

```sql
use sql_test;
```

执行以下命令创建一张用于测试的数据表：

```sql
CREATE TABLE `user` (
    `id` BIGINT(20) NOT NULL AUTO_INCREMENT,
    `name` VARCHAR(20) DEFAULT '',
    `age` INT(11) DEFAULT '0',
    PRIMARY KEY(`id`)
)ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4;
```

#### 1.2.2 查询

##### 单行查询

单行查询`db.QueryRow()`执行一次查询，并期望返回最多一行结果（即Row）。QueryRow总是返回非nil的值，直到返回值的Scan方法被调用时，才会返回被延迟的错误。（如：未找到结果）

```go
func (db *DB) QueryRow(query string, args ...interface{}) *Row
```

```go
func queryRowDemo() {
   sqlStr := "select id, name, age from user where id=?"
   var u user
   // 非常重要：确保QueryRow之后调用Scan方法，否则持有的数据库链接不会被释放
   err := db.QueryRow(sqlStr, 1).Scan(&u.id, &u.name, &u.age)
   if err != nil {
      fmt.Printf("scan failed, err:%v\n", err)
      return
   }
   fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
}
```

##### 多行查询

```go
func (db *DB) Query(query string, args ...interface{}) (*Rows, error)
```

```go
// 查询多条数据示例
func queryMultiRowDemo() {
	sqlStr := "select id, name, age from user where id > ?"
	rows, err := db.Query(sqlStr, 0)
	if err != nil {
		fmt.Printf("query failed, err:%v\n", err)
		return
	}
	// 非常重要：关闭rows释放持有的数据库链接
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

#### 1.2.3 插入数据

插入、更新和删除操作都使用`Exec`方法。

```go
func (db *DB) Exec(query string, args ...interface{}) (Result, error)
```

Exec执行一次命令（包括查询、删除、更新、插入等），返回的Result是对已执行的SQL命令的总结。参数args表示query中的占位参数。

具体插入数据示例代码如下：

```go
// 插入数据
func insertRowDemo() {
	sqlStr := "insert into user(name, age) values (?,?)"
	ret, err := db.Exec(sqlStr, "王五", 38)
	if err != nil {
		fmt.Printf("insert failed, err:%v\n", err)
		return
	}
	theID, err := ret.LastInsertId() // 新插入数据的id
	if err != nil {
		fmt.Printf("get lastinsert ID failed, err:%v\n", err)
		return
	}
	fmt.Printf("insert success, the id is %d.\n", theID)
}
```

#### 1.2.4更新数据

具体更新数据示例代码如下：

```go
// 更新数据
func updateRowDemo() {
	sqlStr := "update user set age=? where id = ?"
	ret, err := db.Exec(sqlStr, 39, 3)
	if err != nil {
		fmt.Printf("update failed, err:%v\n", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Printf("get RowsAffected failed, err:%v\n", err)
		return
	}
	fmt.Printf("update success, affected rows:%d\n", n)
}
```

#### 1.2.5删除数据

具体删除数据的示例代码如下：

```go
// 删除数据
func deleteRowDemo() {
	sqlStr := "delete from user where id = ?"
	ret, err := db.Exec(sqlStr, 3)
	if err != nil {
		fmt.Printf("delete failed, err:%v\n", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Printf("get RowsAffected failed, err:%v\n", err)
		return
	}
	fmt.Printf("delete success, affected rows:%d\n", n)
}
```

#### 总体

~~~go
package main

import (
	"database/sql"
	"fmt"
	_ "github.com/go-sql-driver/mysql"
)

// 定义一个全局对象db
var db *sql.DB

// 定义一个初始化数据库的函数
func initDB() (err error) {
	// DSN:Data Source Name
	dsn := "root:123456@tcp(127.0.0.1:3306)/sql_test?charset=utf8&parseTime=True"
	// 不会校验账号密码是否正确
	// 注意！！！这里不要使用:=，我们是给全局变量赋值，然后在main函数中使用全局变量db
	db, err = sql.Open("mysql", dsn)
	if err != nil {
		return err
	}
	// 尝试与数据库建立连接（校验dsn是否正确）
	err = db.Ping()
	if err != nil {
		return err
	}
	return nil
}
type user struct {
	id   int
	age  int
	name string
}
func queryRowDemo() {
	sqlStr := "select id, name, age from user where id=?"
	var u user
	// 非常重要：确保QueryRow之后调用Scan方法，否则持有的数据库链接不会被释放
	err := db.QueryRow(sqlStr, 1).Scan(&u.id, &u.name, &u.age)
	if err != nil {
		fmt.Printf("scan failed, err:%v\n", err)
		return
	}
	fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
}
// 查询多条数据示例
func queryMultiRowDemo() {
	sqlStr := "select id, name, age from user where id > ?"
	rows, err := db.Query(sqlStr, 0)
	if err != nil {
		fmt.Printf("query failed, err:%v\n", err)
		return
	}
	// 非常重要：关闭rows释放持有的数据库链接
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
func insertRowDemo()  {
	 sqlStr := "insert into user(name,age) value (?,?)"
	 //
	ret,err := db.Exec(sqlStr,"王五",40)
	if err!=nil {
		fmt.Printf("inserf failed,err:%v\n",err)
		return
	}
	//插入成功之后需要返回这个id
	theID,err:=ret.LastInsertId()
	if err != nil{
		fmt.Printf("get the last insertid failed,err:%v\n",theID)
		return
	}
	fmt.Printf("insert success,theID is:%v\n",theID)

}
func updateRowDemo()  {

	sqlStr := "update user set name =? where id = ?"
	//执行含有sqlStr参数的语句
	ret,err:=db.Exec(sqlStr,"赵四",4)
	if err!=nil {
		fmt.Printf("update failed,err:%v\n",err)
		return
	}
	AnoID,err:=ret.RowsAffected()
	if err!=nil {
		fmt.Printf("updateRowAffected failed,err:%v\n",err)
		return
	}
	fmt.Printf("update success AnoID:%v\n",AnoID)

}
// 删除数据
func deleteRowDemo() {
	sqlStr := "delete from user where id = ?"
	ret, err := db.Exec(sqlStr, 5)
	if err != nil {
		fmt.Printf("delete failed, err:%v\n", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Printf("get RowsAffected failed, err:%v\n", err)
		return
	}
	fmt.Printf("delete success, affected rows:%d\n", n)
}
func main() {
	err := initDB() // 调用输出化数据库的函数
	if err != nil {
		fmt.Printf("init db failed,err:%v\n", err)
		return
	}
	//queryRowDemo()
	//insertRowDemo()
    //updateRowDemo()
    deleteRowDemo()
	queryMultiRowDemo()
}

~~~

### 1.3MySQL预处理

#### 1.3.1什么是预处理？

普通SQL语句执行过程：

1. 客户端对SQL语句**进行占位符替换**得到完整的SQL语句。
2. 客户端发送**完整SQL语句到MySQL服务端**
3. MySQL服务端执行完整的SQL语句并将结果**返回给客户端**。

预处理执行过程：

1. 把SQL语句分成两部分，**命令部分与数据部分**。
2. 先把**命令部分发送给MySQL服务端**，**MySQL服务端进行SQL预处理**。
3. 然后把**数据部分**发送给MySQL服务端，MySQL服务端对SQL语句进行**占位符替换**。
4. MySQL服务端执行完整的SQL语句并将结果返回给客户端

#### 1.3.2为什么要预处理？

1. 优化MySQL服务器重复执行SQL的方法，可以提升服务器性能，提前让服务器编译，**一次编译多次执行**，节省后续编译的成本。
2. 避免**SQL注入**问题。

#### 1.3.3 Go实现MySQL预处理

```go
func (db *DB) Prepare(query string) (*Stmt, error)
```

查询操作的预处理示例代码如下：

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

插入、更新和删除操作的预处理十分类似，这里以插入操作的预处理为例：

```go
// 预处理插入示例
func prepareInsertDemo() {
	sqlStr := "insert into user(name, age) values (?,?)"
	stmt, err := db.Prepare(sqlStr)
	if err != nil {
		fmt.Printf("prepare failed, err:%v\n", err)
		return
	}
	defer stmt.Close()
	_, err = stmt.Exec("小王子", 18)
	if err != nil {
		fmt.Printf("insert failed, err:%v\n", err)
		return
	}
	_, err = stmt.Exec("沙河娜扎", 18)
	if err != nil {
		fmt.Printf("insert failed, err:%v\n", err)
		return
	}
	fmt.Println("insert success.")
}
```

#### 总结 其实就多了一个db.Prepare(sqlStr)

#### 1.3.4 SQL注入问题

**我们任何时候都不应该自己拼接SQL语句！**

```go
// sql注入示例
func sqlInjectDemo(name string) {
	sqlStr := fmt.Sprintf("select id, name, age from user where name='%s'", name)
	fmt.Printf("SQL:%s\n", sqlStr)
	var u user
	err := db.QueryRow(sqlStr).Scan(&u.id, &u.name, &u.age)
	if err != nil {
		fmt.Printf("exec failed, err:%v\n", err)
		return
	}
	fmt.Printf("user:%#v\n", u)
}
```

此时以下输入字符串**都可以引发SQL注入问题**：

```go
sqlInjectDemo("xxx' or 1=1#")
sqlInjectDemo("xxx' union select * from user #")
sqlInjectDemo("xxx' and (select count(*) from user) <10 #")
```

|   数据库   |  占位符语法  |
| :--------: | :----------: |
|   MySQL    |     `?`      |
| PostgreSQL | `$1`, `$2`等 |
|   SQLite   |  `?` 和`$1`  |
|   Oracle   |   `:name`    |