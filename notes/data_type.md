data_type
===

- 数据类型
    - 基本数据类型
        - 数值型
            - 整数类型(int, int8, int16, int32, int64, uint, uint8, uint16, uint32, uint64, byte)
            - 浮点类型(float32, float64)
        - 字符型(没有专门的字符型, 使用byte保存单个字母字符)
        - 布尔型(bool)
        - 字符串(string)

## Why do we need data type?
- 从底层而言, 所有的数据都是由bit组成
- 数据类型是数据组织形式, 通过对数据的组织以表达更多的对象
- Golang数据类型分类
    - 基本类型
    - 复合类型
    - 引用类型
    - 接口类型

## 基本类型
### 整数类型
- 有符号整数类型, int8, int16, int32, in64
- 无符号整数类型, uint8, uint16, uint32, uint64
- int/uint OS相关, 编译器相关
- Unicode字符rune类型是和int32等价的类型, 通常用于表示一个Unicode码点
- byte类型适合uint8等价的类型, byte类型一般用于强调数值是一个原始的数据而不是一个小的整数
- uintptr机器相关, 足以容纳指针, 该类型只有在底层编程时才需要, 特别是Go语言和C语言函数库或操作系统接口相交互的地方
- 至于每个类型的表示范围, 与其他语言一样, 底层二进制补码表示, 老生常谈, 不再赘述
- Q: Golang有像Integer.MAX_VALUE这种东西么, A: 有, 在math包里, math.MaxInt64

### bool
- 用一个字节存储
- 只允许true/false, 不允许是数字, 不允许是空
- 和C语言不同, Golang的bool类型不能用0和非0来代替false和true

### string
- Golang的字符串是由单个字节拼接起来的
- Golang的字符串使用UTF-8编码标识Unicode文本
- Golang中字符串是不可变的, 也就是一旦赋值, 不能用过str[0] = 'a', 对其进行修改
- 双引号括起的字符串常量会识别转义字符, 反引号括起的字符串会以原生形式输出
- 字符串拼接用 +
- 当一个拼接操作很长时, 可以分行写, 但要记得把+留在上一行的结尾, 这里了解编译原理就了解为什么了, 当编译器识别到+结尾就不会给添加分号了

## 基本数据类型间的转换
- Golang和Java, C/C++体系不同, 类型的概念特别强, 不同类型的变量之间赋值时需要显示转换, 也就是说Golang不存在类型提升之类的自动类型转换
- 无论是从小范围到大范围还是从大范围到小范围, 都需显示写明, 把大的给小的, 如果溢出不会报错, 只是按溢出处理
- 类型转换的语法
    - Golang的强制类型转换比较特别, 别记混了, type(变量名), Java的则是(type)变量名
    - i, ok := a.(int), 类型断言也能实现强制类型转换

### 基本数据类型和string的转换
- 基本数据类型转string
    - fmt.Sprintf("format", a), 返回转换后的string
    - 用strconv包中的函数
        - FormatInt/FormatFloat等函数将一个变量转换为int/float并返回字符串
    - itoa, 可以把int转为string
- string转基本数据类型
    - strconv中ParseInt, ParseFloat等返回的都是各个类型的64位, 注意, 不能用int去接, 要用int64接然后转成你需要的
    - 把不合法的转成基本数据类型, 比如把"hello"转成int, Go会转成这个类型的零值, 而不会报错