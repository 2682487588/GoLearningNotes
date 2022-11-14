### 1.1.4.Go主要特征

1. **自动立即回收**
2. 更丰富的内置类型
3. 函数多返回值
4. 错误处理
5. **匿名函数和闭包**
6. 类型和接口
7. **并发编程**
8. **反射**
9. **语言交互性**

### 1.1.6.Go语言命名

> 首字母  —— Unicode字符  或者  下划线

> 其他字母——  Unicode字符 /下划线/数字

> 字符长度不限

#### GO关键字

1. break
2. default
3. func   -- js 
4. interface
5. select
6. case
7. defer  --新
8. go--新
9. map
10. struct--  c
11. chan -- 新
12. else
13. goto -- c
14. package
15. switch
16. const -- js
17. fallthrough -- 新
18. if
19. range
20. type
21. continue
22. for
23. import
24. return
25. var -- js

#### Go保留字

##### Constant

1. true
2. false
3. iota  -- 新
4. nil  -- 新

##### Types:

1. int
2. int8
3. int16
4. int32
5. int64
6. uint     -- 无符号整形
7. uint8
8. uint16
9. uint32
10. uint64
11. uintptr --新
12. float32
13. float64
14. complex128 --新
15. bool --新
16. byte
17. rune --新
18. string
19. error

##### Function

1. make
2. len
3. cap
4. new
5. append
6. copy
7. close
8. delete
9. complex
10. real  --新
11. imag 
12. panic --新
13. recover --新

#### 可见性

1.声明在**函数内部**，是**函数本地值**，类似private

2.函数外部，当前包(包内所有.go)可见的全局值,类似Protect

3.函数外部&&**首字母大写**是**所有包可见的全局值** 类似Public



### 1.1.7.Go声明

~~~go
var //变量
const //常量
type  //类型
func //函数
~~~

### 1.1.8.项目结构

~~~go
src  //源代码
pkg  //包
bin  //相关Bin
~~~

1.  建立工程文件夹 goproject

2. 在工程文件夹中建立src,pkg,bin文件夹

3.  在GOPATH中添加projiect路径 例 e:/goproject

4. 如工程中有自己的包examplepackage，那在src文件夹下建立以包名命名的文件夹 例 examplepackage

5. 在src文件夹下编写主程序代码代码 goproject.go

6. 在examplepackage文件夹中编写 examplepackage.go 和 包测试文件 examplepackage_test.go

7. 编译调试包

   go build examplepackage

   go test examplepackage

   go install examplepackage

8. 编译主程序

   go build goproject.go

   成功后会生成goproject.exe文件

   至此一个Go工程编辑成功。

### 1.1.9Go编译

bin——可执行文件

pkg——.a文件

golang中的**import name**，实际是到**GOPATH中去寻找name.a,** 使用时是该name.a的**源码中生命的package名字**

