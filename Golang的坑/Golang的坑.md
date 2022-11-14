#### 8.显式类型的变量无法使用 nil 来初始化

nil 是 **interface**、**function**、**pointer**、**map**、**slice** 和 **channel** 类型变量的默认初始值

但声明时**不指定类型**，编译器也**无法推断出变量的具体类型**

```go
// 错误示例
func main() {
    var x = nil    // error: use of untyped nil
    _ = x
}

// 正确示例
func main() {
    var x interface{} = nil
    _ = x
}
```

#### 9.直接使用值为 nil 的 slice、map

