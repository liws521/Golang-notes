good articles
===

## [你真的懂string与[]byte的转换了么](https://mp.weixin.qq.com/s/g67GDIk5qz8naYPI4x7IdA)
- 标准转换就是我们常用的 b := []byte(str)
- 强转换(黑魔法)
```go
func String2Bytes(s string) []byte {
	sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
	bh := reflect.SliceHeader{
		Data: sh.Data,
		Len:  sh.Len,
		Cap:  sh.Len,
	}
	return *(*[]byte)(unsafe.Pointer(&bh))
}

func Bytes2String(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}
```
- 强转换的性能会明显优于标准转换
    - 标准转换本质是底层数组的拷贝
    - 强转换是直接替换指针的指向, 从而使string和[]byte指向同一个底层数组
    - 但标准转换更加安全, 安全的代价就是性能的妥协
- 标准转换的实现细节
    - 当长度大于32, Golang会调用mallocgc分配一块新的内存
    - 无论长度如何最后都会调用copy进行一次拷贝, 所以标准转换是逐字符的值拷贝
- 强转换的实现细节
    - 万能的unsafe.Pointer指针
        - 在Golang中任何类型的指针*T都可以转换为unsafe.Pointer类型的指针, 它可以存储任何类型变量的地址.
        - 同时, unsafe.Pointer类型的指针也可以转换回普通指针, 而且可以不必和之前的类型*T相同.
        - 另外, unsafe.Pointer类型还可以转换为uintptr类型, 该类型保存了指针所指向地址的数值, 从而可以使我们对地址进行数值运算.
    - string和slice在reflect包中, 对应的结构体是reflect.StringHeader和reflect.SliceHeader, 它们是string和slice的运行时表达.
    - 强转换直接把一个[]byte类型的变量b, 取它的地址&b, 这是一个指向切片的指针, 把它转换为unsafe.Pointer, 再转换为*string, 这就是一个指向string的指针了, 最后解引用得到一个string, 相当于直接把地址给它了, 没有逐字符的值拷贝, 所以效率好
- string为什么要设计为不可修改
    - string不可修改, 意味着它是只读属性, 这样的好处就是在并发场景下, 我们可以在不加锁的控制下, 多次使用同一字符串, 有效保证高效共享的情况下而不用担心安全问题