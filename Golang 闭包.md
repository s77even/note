Golang中函数是头等对象，可以当做返回值，可以做参数传递，可以绑定到变量，这样的函数为fanction value，是一个指针，但是并不直接直接指向函数入口，指向一个fancval struct。函数体内只有一个fn uintptr，指向函数的入口地址。

为什么使用一个二级指针？ 解决闭包，不同的捕获变量



Golang 闭包

闭包= 函数+函数外变量的引用(捕获变量) DX寄存器保存addr和变量偏移

闭包就是有捕获列表的fanction value

(保证捕获变量在外层函数与闭包函数中的一致性)

```go
package main

import "fmt"

func app() func(string) string {
	t := "Hi"
	c := func(b string) string {
		t = t + " " + b
		fmt.Println(&t)
		return t
	}
	return c
}

func main() {
	a := app()
	b := app()
	a("go")
	fmt.Println(a("go"),)
	fmt.Println(b("All"))
}
```



在http包的proxyURL中有一个函数

```go
func ProxyURL(fixedURL *url.URL) func(*Request) (*url.URL, error) {
 return func(*Request) (*url.URL, error) {
  return fixedURL, nil
 }
}
```



返回值是一个函数，引用了父函数环境内部的参数，因此是闭包



一般来说，一个函数返回另外一个函数，这个被返回的函数可以引用外层函数的局部变量，这形成了一个闭包。通常，闭包通过一个结构体来实现，它存储一个函数和一个关联的上下文环境。但 Go 语言中，匿名函数就是一个闭包，它可以直接引用外部函数的局部变量