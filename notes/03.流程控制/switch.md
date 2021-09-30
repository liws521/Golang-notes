# meta
```go
switch x {
case v1:
    statement
    fallthrough
case v2, v3:
    statement
default:
    statement
}
```
- golang中x的可以是任意类型, 不限于常量
- 编译器默认为每个case自动添加break, 所以是非贯穿的, 若想贯穿加fallthrough抑制编译器行为
- case后的v时和x同类型的任意值/表达式, 注意类型必须相同, golang的类型概念很强
- 可以同时测试多个值, 用逗号分割 case v1, v2:
- 当switch后面什么也没写时, 相当于写了个true, 此时case后面可以写condition, 此时该switch起到了if-else if-else的效果

# switch

## type-switch
- switch语句还可以被用于type-switch来判断某个interfase变量中实际存储的变量类型.
```go
switch x.(type) {
    case type:
        statement
    case type:
        statement
    default:
        statement
}
```