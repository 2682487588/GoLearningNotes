### 模板引擎的使用

### 定义模板文件

![image-20220218112347528](C:%5CUsers%5C%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20220218112347528.png)

### 解析模板文件

上面定义好了模板文件之后，可以使用下面的常用方法去解析模板文件，得到模板对象：

```go
func (t *Template) Parse(src string) (*Template, error)
func ParseFiles(filenames ...string) (*Template, error)
func ParseGlob(pattern string) (*Template, error)
```

当然，你也可以使用`func New(name string) *Template`函数创建一个名为`name`的模板，然后对其调用上面的方法去解析模板字符串或模板文件。

### 模板渲染

渲染模板简单来说就是使用数据去填充模板，当然实际上可能会复杂很多。

```go
func (t *Template) Execute(wr io.Writer, data interface{}) error
func (t *Template) ExecuteTemplate(wr io.Writer, name string, data interface{}) error
```

#### 定义模板文件

我们按照Go模板语法定义一个`hello.tmpl`的模板文件，内容如下：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Hello</title>
</head>
<body>
    <p>Hello {{.}}</p>
</body>
</html>
```

#### 解析和渲染模板文件

然后我们创建一个`main.go`文件，在其中写下HTTP server端代码如下：

```go
// main.go

func sayHello(w http.ResponseWriter, r *http.Request) {
	// 解析指定文件生成模板对象
	tmpl, err := template.ParseFiles("./hello.tmpl")
	if err != nil {
		fmt.Println("create template failed, err:", err)
		return
	}
	// 利用给定数据渲染模板，并将结果写入w
	tmpl.Execute(w, "沙河小王子")
}
func main() {
	http.HandleFunc("/", sayHello)
	err := http.ListenAndServe(":9090", nil)
	if err != nil {
		fmt.Println("HTTP server failed,err:", err)
		return
	}
}
```

## 模板语法

### {{.}}

模板语法应该包含{{和}}之间，其中{{.}},Go的模板语法中支持使用**管道符号`|`链接多个命令**，**`|`前面的命令会将运算结果(或返回值)传递给后一个命令的最后一个位置。**

### 变量

```template
$obj := {{.}}
```

其中`$obj`是变量的名字，在后续的代码中就可以使用该变量了

### 移除空格

`{{-`语法去除模板内容左侧的所有空白符号， 使用`-}}`去除模板内容右侧的所有空白符号。

```template
{{- .Name -}}
```

### 条件判断

```template
{if pipeline}} T1 {{end}}

{{if pipeline}} T1 {{else}} T0 {{end}}

{{if pipeline}} T1 {{else if pipeline}} T0 {{end}}
```

### range

Go的模板语法中使用**`range`关键字进行遍历**，有以下两种写法，其中`pipeline`的值必须是**数组、切片、字典或者通道**。

### with

- 主要是判断为空不为空，如果**pipeline不为空就输出T1**否则不输出

```template
{{with pipeline}} T1 {{end}}
如果pipeline为empty不产生输出，否则将dot设为pipeline的值并执行T1。不修改外面的dot。

{{with pipeline}} T1 {{else}} T0 {{end}}
如果pipeline为empty，不改变dot并执行T0，否则dot设为pipeline的值并执行T1。
```

### 预定义函数

and
    函数返回它的**第一个empty参数或者最后一个参数**；
    就是说"and x y"等价于"if x then y else x"；所有参数都会执行；

```vue
<p>{{and $v1 $v2 $p1}} </p> // v1 100 v2 50 p1 30  //30
```

> or 

​	返回第一个**非empty参数**或者**最后一个参数**；
​    亦即"or x y"等价于"if x then x else y"；所有参数都会执行

~~~html
<p>{{and $v1 $v2 $p1}} </p>  //100
~~~

```bash
and
    函数返回它的第一个empty参数或者最后一个参数；
    就是说"and x y"等价于"if x then y else x"；所有参数都会执行；
or
    返回第一个非empty参数或者最后一个参数；
    亦即"or x y"等价于"if x then x else y"；所有参数都会执行；
not
    返回它的单个参数的布尔值的否定
len
    返回它的参数的整数类型长度
index
    执行结果为第一个参数以剩下的参数为索引/键指向的值；
    如"index x 1 2 3"返回x[1][2][3]的值；每个被索引的主体必须是数组、切片或者字典。
print
    即fmt.Sprint
printf
    即fmt.Sprintf
println
    即fmt.Sprintln
html
    返回与其参数的文本表示形式等效的转义HTML。
    这个函数在html/template中不可用。
urlquery
    以适合嵌入到网址查询中的形式返回其参数的文本表示的转义值。
    这个函数在html/template中不可用。
js
    返回与其参数的文本表示形式等效的转义JavaScript。
call
    执行结果是调用第一个参数的返回值，该参数必须是函数类型，其余参数作为调用该函数的参数；
    如"call .X.Y 1 2"等价于go语言里的dot.X.Y(1, 2)；
    其中Y是函数类型的字段或者字典的值，或者其他类似情况；
    call的第一个参数的执行结果必须是函数类型的值（和预定义函数如print明显不同）；
    该函数类型值必须有1到2个返回值，如果有2个则后一个必须是error接口类型；
    如果有2个返回值的方法返回的error非nil，模板执行会中断并返回给调用模板执行者该错误；
```

### 比较函数

```template
eq      如果arg1 == arg2则返回真		equal
ne      如果arg1 != arg2则返回真	 	not equal
lt      如果arg1 < arg2则返回真			less than
le      如果arg1 <= arg2则返回真		less equal
gt      如果arg1 > arg2则返回真			great than
ge      如果arg1 >= arg2则返回真		great queal
```

eq（**只有eq**）可以**接受2个或更多个参数**，它会将**第一个参数和其余参数依次比较**，返回下式的结果

