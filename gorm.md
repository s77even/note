### GORM

Object Relational Mapping （将一个结构体数据类型映射为数据库中的一条数据）

数据表 <-> 结构体

数据行 <-> 结构体实例

字段 <->结构体字段



优点：提高了开发效率

缺点：牺牲执行性能，牺牲灵活性，弱化SQL能力



安装 go get -u github.com/jinzhu/gorm



gorm中为不同数据库包装了不同的驱动

mysql : github.com/jinzhu/gorm/dialects/mysql

```go
// import _ "github.com/jinzhu/gorm/dialects/postgres"
// import _ "github.com/jinzhu/gorm/dialects/sqlite"
// import _ "github.com/jinzhu/gorm/dialects/mssql"
```



连接mysql

```go
gorm.open("mysql","root:seven7777777@tcp(127.0.0.1:3306)/user?charset=utf8mb4&parseTime=true&loc=Local")
```

想要正确的处理 `time.Time` ，您需要带上 `parseTime` 参数



gorm允许通过一个现有的数据连接来初始化gorm.db

```go
import (  "database/sql"  "gorm.io/gorm")

sqlDB, err := sql.Open("mysql", "mydb_dsn")
gormDB, err := gorm.Open(mysql.New(
    mysql.Config{  
    Conn: sqlDB,
}), 
    &gorm.Config{})
```



AutoMigrate 自动迁移

 AutoMigrate 会创建表，创建缺少的外键，约束，列和索引，并且会更改现有列的类型（如果其大小、精度、是否为空可更改）。

但 **不会删除**未使用的列，以保护数据。

```go
db.AutoMigrate(&User{})
db.AutoMigrate(&User{}, &Product{}, &Order{})
```



##### 简单操作

创建记录     db.Create(&u1)

查询第一条记录	db.First(u)

更新记录	db.Model(&u).Update("hobby", "双色球")   //更新u中的hobby字段为双色球

删除记录		db.Delete(&u)      //u 为查询到的记录  



##### gorm Model

在代码中定义model与数据表进行映射

gorm内置了一个gorm.Model结构体，包含了ID, CreatedAt(创建时间),UpdatedAt(更新时间), DeletedAt(删除时间)四个字段

其中 gorm中的delete 并不会真正将数据删除，而是将删除时间设置为过期时间，达到软删除的目的



##### Tag

使用结构体声明模型时，可给字段增加标记

`gorm:""`![image-20201222164534606](C:\Users\wwwwwwl\AppData\Roaming\Typora\typora-user-images\image-20201222164534606.png)

gorm支持以下标记

![image-20201222164548069](C:\Users\wwwwwwl\AppData\Roaming\Typora\typora-user-images\image-20201222164548069.png)



##### gorm 主键 表名 列名 的约定和设定

###### 主键

默认使用ID字段作为主键   也可增加“gorm：“primary_key”“标签来自定义主键

###### 表名

默认表名是结构体名称的复数,可关闭复数模式   			db.SingularTable(true)

可通过指定表名创建指定数据表        db.Table("pro-user").CreateTable(&User{})

可更改默认表名规则

```go
gorm.DefaultTableNameHandler = func(db *gorm.DB, defaultTableName string) string {
   return "pro-"+defaultTableName
}
```

###### 列名

列名由结构体字段的小写名称组成 ，当结构体字段为驼峰命名法时，由下划线进行分割

可以使用结构体tag指定列名				  `gorm:"column:beast_id"` 



#### 增删改查

##### 创建

db.creat()

在创建时，通过tag定义的有默认值的字段，在创建记录时会排除没有值或为零值的字段，并使用默认值

var user = User{name：“”，age：99}    // name 有默认值

db.creat(&user)     // SQL: INSERT INTO users("age") values('99');  排除了零值字段name 

所有字段的零值, 比如`0`, `""`,`false`或者其它`零值`，都不会保存到数据库内



避免以上情况，可以将默认字段的类型改为对应类型的指针（*string）或实现scanner/valuer接口

使用指针 创建时 var user = User{name：new（string）age：18}

使用接口  

```go
// 使用 Scanner/Valuer
//引入database/sql
type User struct {
	ID int64
	Name sql.NullString `gorm:"default:'小王子'"` // sql.NullString 实现了Scanner/Valuer接口
	Age  int64
}
// type NullString struct {
//	String string
//	Valid  bool  // Valid is true if String is not NULL 为true时表示使用string 可为空“”，为false表示空值
//}
user := User{Name: sql.NullString{"", true}, Age:18}
db.Create(&user)  // 此时数据库中该条记录name字段的值就是''
```



##### 删除

当struct中含有gorm.model字段时，不加Unscoped的删除都是软删除，使用gorm查询不到，但是数据依然在数据表中

db.Delete（&u） 会根据U的主键去删除对应数据

*如果u没有主键值，delete会删除所有记录*

###### 链式操作删除 

```
db.Where("name=?","xx").Delete(&User{})
```

###### 在Delete函数后指定参数

```go
db.Delete(User{}, "name like ?", "seven")
```

###### 物理删除 真正的删除 Unscoped

```go
db.Unscoped().Where("name=?", "seven").Delete(&User{})
```

###### 查询软删除后的数据

```go
db.Unscoped().Where("name=?", "seven").Find(&user)
```



##### 更新

###### 更新所有字段

```go
db.Save(&user)
```

###### 更新指定字段 update或updates

```go
db.Model(&user).Update("name", "hello")
```

```go
db.Model(&user).Where("active = ?", true).Update("name", "hello")
```

```go
db.Model(&user).Updates(map[string]interface{}{"name": "hello", "age": 18, "active": false})
```

```go
db.Model(&user).Updates(User{Name: "hello", Age: 18})
```

```go
//当使用 struct 更新时，GORM只会更新那些非零值的字段
// 对于下面的操作，不会发生任何更新，"", 0, false 都是其类型的零值
db.Model(&user).Updates(User{Name: "", Age: 0, Active: false})
```

###### 想更新或忽略某些字段，你可以使用 `Select`，`Omit`

```go
db.Model(&user).Select("name")..Updates(map[string]interface{}{"name": "hello", "age": 18, "active": false})
```

```go
db.Model(&user).Omit("name").Updates(map[string]interface{}{"name": "hello", "age": 18, "active": false})
```

######  不使用hook更新

在上述更新操作中，会调用model的钩子函数，更新updateAt时间戳，如果不想更新updateAt ，可以使用updatecolumn，updatecolumns进行更新操作，类似于update

另，批量更新是不会调用钩子函数

```go
// 使用 `RowsAffected` 获取更新记录总数
db.Model(User{}).Updates(User{Name: "hello", Age: 18}).RowsAffected
```





#### 查询

###### 一般查询

```go
// 根据主键查询第一条记录
db.First(&user)

// 随机获取一条记录
db.Take(&user)

// 根据主键查询最后一条记录
db.Last(&user)

// 查询所有的记录
db.Find(&users)

// 查询指定的某条记录(仅当主键为整型时可用)
db.First(&user, 10)
```



###### where 条件查询

```go
// Get first matched record
db.Where("name = ?", "jinzhu").First(&user)
// Get all matched records
db.Where("name = ?", "jinzhu").Find(&users)
// <> 不等于
db.Where("name <> ?", "jinzhu").Find(&users)
// IN
db.Where("name IN (?)", []string{"jinzhu", "jinzhu 2"}).Find(&users)
// LIKE
db.Where("name LIKE ?", "%jin%").Find(&users)
```

###### struct和map 查询

```go
// Struct
db.Where(&User{Name: "jinzhu", Age: 20}).First(&user)
// Map
db.Where(map[string]interface{}{"name": "jinzhu", "age": 20}).Find(&users)
```



当通过**结构体**进行查询时，GORM将会只通过非零值字段查询，这意味着如果你的字段值为`0`，`''`，`false`或者其他`零值`时，将不会被用于构建查询条件,同可以使用指针或实现 Scanner/Valuer 接口来避免这个问题.

```go
db.Where(&User{Name: "jinzhu", Age: 0}).Find(&users)
//// SELECT * FROM users WHERE name = "jinzhu"; SQL语句条件中没有age
```



###### 内联条件

当内联条件与多个立即执行方法一起使用时，内联条件不会传递给后面的立即执行方法

```go
db.Find(&user, "name = ?", "jinzhu")
```

立即执行方法：会立即生成SQL语句的方法，一般是指crud方法



###### FirstOrInit  - FirstOrCreat

firstorinit：获取匹配的第一条记录，否则根据给定的条件初始化一个结构体对象，未找到并不会将数据插入到数据库中

firstorcreat：获取匹配的第一条记录, 否则根据给定的条件创建一个新的记录，未找到会将记录插入到数据库中内 (仅支持 struct 和 map 条件)



attrs条件： 如果记录未找到，会使用attrs条件中的参数创建结构体，找到则条件无效

assign条件：不管是否找到， 都使用条件创建，并不会改变数据库中的数据



##### 高级查询

选择查询字段 select

```go
db.Select("name, age").Find(&users) 
```

Scan 扫描结果至一个新的结构体 （搭配select）

```go
type Result struct {
  Name string
  Age  int
}

var result Result
db.Table("users").Select("name, age").Where("name = ?", "Antonio").Scan(&result)

var results []Result
db.Table("users").Select("name, age").Where("id > ?", 0).Scan(&results)
```

获取记录总数 count

`Count` 必须是链式查询的最后一个操作 ，因为它会覆盖前面的 `SELECT`，但如果里面使用了 `count` 时不会覆盖

数量限制 指定从数据库检索出的最大记录数。Limit

```go
db.Limit(3).Find(&users)
```

