


# map与工厂模式
- map的value可以是一个函数func
- 与Golang的Dock type接口方式一起, 可以方便的实现单一方法对象的工厂模式
```go
func main() {
	m := map[int]func(op int) int{}
	m[1] = func(op int) int { return op }
	m[2] = func(op int) int { return op * op }
	fmt.Println(m[1](3), m[2](3))
}
```

# 实现set
- Golang的内置集合中没有set的实现, 可以map[type]bool
- 元素的唯一性
- 判断一个元素是否存在就可以直接 if set['a'] {}
