### reflect包 学习笔记

reflect使能够在运行期得知对象的类型信息和内存结构，全部信息来源自接口变量。

### Type 类型

首先区分 Type 和 Kind 。

Type：真实类型，静态类型。

Kind：基础结构类别，底层类型。

```go
type X int
func main(){
    var a X =100
    t := reflect.TypeOf(a) // func TypeOf(i interface{}) Type 类型
    v := reflect.ValueOf(a)// func ValueOf(i interface{}) Value 值
    fmt.Println(v,t.Name(),t.Kind())// 100 X int
}
```

传入对象区分基础结构和指针类型，TypeOf(x) != TypeOf(&x)



方法Elem()返回指针指向的对象类型，（数组，slice，map或channel）中的元素类型

遍历结构体的字段，必须获取结构体的基础类型。如果传人的是结构体指针，需要用Elem()返回类型。匿名字段需要再嵌套一层，type.Field(i).Anonymous可判断是否为匿名字段，type.Field(i).Field(x)获取匿名嵌套字段，可用多级索引直接访问匿名字段

```go
type user struct{
    name string
    age int
}
type manager struct{
    user
    name int
    titile string
}
func main() {
    // 声明一个空结构体
    var m manager
    t:= reflect.TypeOf(m)    
    //fmt.Println(t.Type)
    age:= t.FieldByIndex([]int{0,1}) //0表示最外层第0个字段 1表示嵌套第一层第一个字段
    fmt.Println(age.Name,age.Type) // age int  /{0，0}则表示user.name /{1}表示manager.name
    
    //可按名称直接查找字段  有同名遮蔽
    name, _ ：= t.FieldByName("name")
    fmt.Println(name.Name,name.Type) // name int  
}
```



方法Method()可输出方法集，区分指针和基类型

反射 能探知当前包或外包的非导出结构成员



反射可以提取stuct Tag

```go
type user struct{
    name string `field:"name" type:"varchar(50)"`
    age int `field:"age" type:"int"`
}
func main() {
    var u user
    t:= reflect.TypeOf(u)    
    for i:=0; i<t.NumField();i++{
        f:=t.Field(i)
        fmt.Println(f.Tag.Get("field"),f.Tag.Get("type"))
        //name varchar(50)
        //age int
    }
}
```

func (t *rtype) Implements(u reflect.Type) bool // 判断 t 类型是否实现了 u 接口。 

func (t *rtype) ConvertibleTo(u reflect.Type) bool // 判断 t 类型的值可否转换为 u 类型。 

func (t *rtype) AssignableTo(u reflect.Type) bool // 判断 t 类型的值可否赋值给 u 类型。 



### 值 value

要想修改目标对象，需要传入指针，传入指针就需要使用Elem()获取目标对象。

```go
a:=100
va ,vp := reflect.ValueOf(a),reflect.ValueOf(&a).Elem()
fmt.Println(va.CanAddr(),va.CanSet(),vp.CanAddr(),vp.CanSet())// false false true true
```

非导出字段不能直接进行设置操作，无论是当前包还是外包，可寻址时，可使用unsafa包进行设置。



可通过Interface()方法返回接口类型进行类型推断和转换，转换为原始类型可进行参数修改



接口有两种nil

``` go
var a interface{}  // a == nil
var b interface = (*int)(int)  // b!=nil   ***reflect.ValueOf(b).IsNil=true 判断值为nil
```



### 方法

动态调用方法

无法调用非导出方法。

 

### 反射性能损失严重

