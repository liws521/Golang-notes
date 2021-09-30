# meta
```go
if init; condition {
    statement
}
```
- golang的if可以带一条初始化语句, 也可以省略
    - 初始化语句中可以声明变量, 作用域只在if结构内部
    - 初始化语句也可以做别的事情, 例如print等都可
- golang追求一个问题只有一种解决方案, 左括号只能写在同一行
- golang不支持三目运算符 condition ? a : b

# if