struct
===

## Why do we need struct?
一个程序就是一个世界, 如果我想维护两只猫的信息, 具有名字年龄颜色等属性, 利用传统技术需要很多变量来表示, 而且一只猫的两个变量之间也没有太多的相关性, 不利于数据的管理和维护, 如果希望对一只猫的属性进行操作, 或者绑定方法等也都不好处理, 这就引出了struct

## basic introduction
- Golang中的struct是值类型, 变量名指向的就是这块内存空间, 而非变量存储一个地址, 由地址指向数据空间
- 所以struct变量之间的赋值, 默认是值拷贝
- 声明struct的语法如下
```go
type 结构体名称 struct {
    字段名 type
    字段名 type
}
```
- 结构体名称和字段名的访问权限, 通过大写来控制, 大写代表着可以在其他包中使用
- 结构体变量 vs 对象
    - 怎么叫都行吧, 结构体就是一种模板, 并非一个真正的实体, 而结构体变量就和其他类型的变量一样, 代表着一块内存空间, 是一个真正的实体, 就像是Java中的类和对象的关系
- 结构体变量的几种创建语法及其使用
```
var cat1 Cat
var cat2 Cat = Cat{}
var cat3 Cat = Cat{"tom", 10}
var p4 *Cat = new(Cat) // 用new得到的是指针类型
var p5 *Cat = &Cat{}

(*p4).Name = "tom"
p4.Name = "tom"
// 因为p4是个指针, 所以标准的访问方式应该先加解引用符号得到结构体变量, 然后再用点操作符
// C/C++将先解引用再点操作符用 -> 替代
// Golang为了简洁没有沿用这种方式, 而是再编译器底层做了优化, 只写点操作符就可以, 编译器在底层做判断, 将之还原为先星再点的形式
```
    - 注意点的优先级是高于星的, (*p).name 与 *p.name 是完全不同的

## details
- 结构体的所有字段在内存中是连续分布的
- Golang的类型的概念特别强
    - 两个不同的类型之间不能直接赋值, 不能进行运算
    - type myInt int, 看似是取别名, 但是却是重新定义, 创造了一种新的类型, 两者之间不能赋值不能运算, 但是可以进行强制类型转换
    - 结构体是用户自定义的数据类型, 和其他类型进行转换时需要有完全相同的字段, 不仅字段的个数, 类型要完全相同, 甚至字段名都要相同
    - 同理, 对一个结构体类型进行type重定义, 得到的也是一种新类型, 可以与原来的类型进行强制类型转换, 但是不能直接赋值
    - 对引用类型重定义一个新类型, 两者之间是可以直接赋值的, 因为本质上传递的是个地址
    - 这里我记得有个什么东西, 新的定义了原来的就不能用了, 原来不是type重定义, 而是import包的时候取别名, 原来的名字就不能用了
- 结构体的字段后面可以写tag, 该tag可以通过反射机制获取, 常见的使用场景是json序列化的时候自定义字段名, `json: "name"`

Golang OOP
===

- 面向对象编程(OOP)只是一种思想, 每种语言的实现形式各不相同
- Golang也支持OOP, 但是和传统的面向对象编程有区别, 并不是纯粹的面向对象语言, 所以说Golang支持面向对象特性是更为准确
- Golang没有class, 是基于struct来实现OOP特性的
- Golang OOP非常简洁, 去掉了传统OOP语言的`继承, 方法重载, 构造函数和析构函数, 隐藏的this指针等`
- Golang仍然有OOP的继承, 封装, 多态的特性, 只是实现方式和其他语言不同

