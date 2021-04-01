### Gin web

web是基于Http协议进行交互的应用网络

web就是通过使用浏览器访问的各种资源

request - > response



go get -u github.com/gin-gonic/gin



基础小demo

1 创建路由  e:= gin.defalut() 使用默认路由

2  创建方法和方法函数  e.GET PUT DELETE POST 

​	GET(relativePath string, handlers ...HandlerFunc)

```GO
engine.GET("/hello", func(context *gin.Context) {
   context.JSON(http.StatusOK,gin.H{
      "method":"get",
   })
})
```

3 e .run(":9090")



gin 模板渲染 

加载模板 e.loadhtmlfiles(files...) 

​					e.loadhtmlglob(pattern...) 使用正则匹配所有模板文件



gin框架返回json

1 gin.H{}

2 结构体  字段要公有  或使用tag json



gin querystring   可以查询多个  

context.query

contex.defaultquery

context.getquery



gin 获取form表单

context.postform

context.defaultpostform

context.getpostform



gin 获取url path参数

context.param



gin 参数绑定  将请求的数据和后端的结构体字段进行绑定

 shouldbind(&u)



gin 文件上传

从请求中读取文件并保存到本地

file ，_ := contecxt.formfile("f1") // f1是前端规定的文件名

c.saveuoloadfile(file,path)



gin 重定向

c.redirect(301,"http://www.baidu.com")



gin 路由和路由组

路由表示请求对应一组方法，一个路径的解析，根据客户端提交的路径，将请求解析到相应的控制器上

get post put delet head patch trace connect option



路由组 一组拥有相同前缀的路由



gin 中间件

允许开发者在处理请求的过程中，加入用户自己的钩子函数，这个函数就叫做中间件，适合处理一些公共逻辑。