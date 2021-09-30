## 使用new关键字分配内存
- Go提供了两种分配原语，即内建函数 new 和 make。 它们所做的事情不同，所应用的类型也不同。它们可能会引起混淆，但规则却很简单。 让我们先来看看 new。这是个用来分配内存的内建函数， 但与其它语言中的同名函数不同，它不会初始化内存，只会将内存置零。 也就是说，new(T) 会为类型为 T 的新项分配已置零的内存空间， 并返回它的地址，也就是一个类型为 *T 的值。用Go的术语来说，它返回一个指针， 该指针指向新分配的，类型为 T 的零值。
- 既然 new 返回的内存已置零，那么当你设计数据结构时， 每种类型的零值就不必进一步初始化了，这意味着该数据结构的使用者只需用 new 创建一个新的对象就能正常工作。例如，bytes.Buffer 的文档中提到“零值的 Buffer 就是已准备就绪的缓冲区。" 同样，sync.Mutex 并没有显式的构造函数或 Init 方法， 而是零值的 sync.Mutex 就已经被定义为已解锁的互斥锁了。

- “零值属性”可以带来各种好处。考虑以下类型声明。
```go
type SyncedBuffer struct {
    lock    sync.Mutex
    buffer  bytes.Buffer
}
// SyncedBuffer 类型的值也是在声明时就分配好内存就绪了。后续代码中， p 和 v 无需进一步处理即可正确工作。
p := new(SyncedBuffer)  // type *SyncedBuffer
var v SyncedBuffer      // type  SyncedBuffer
```
## 构造函数和复合字面量
有时零值还不够好，这时就需要一个初始化构造函数，如来自 os 包中的这段代码所示。
```go
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := new(File)
    f.fd = fd
    f.name = name
    f.dirinfo = nil
    f.nepipe = 0
    return f
}
// 这里显得代码过于冗长。我们可通过复合字面来简化它， 该表达式在每次求值时都会创建新的实例。
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := File{fd, name, nil, 0}
    return &f
}
```
- 请注意，返回一个局部变量的地址完全没有问题，这点与C不同。该局部变量对应的数据 在函数返回后依然有效。实际上，每当获取一个复合字面的地址时，都将为一个新的实例分配内存， 因此我们可以将上面的最后两行代码合并：
    - return &File{fd, name, nil, 0}
- 复合字面的字段必须按顺序全部列出。但如果以 字段:值 对的形式明确地标出元素，初始化字段时就可以按任何顺序出现，未给出的字段值将赋予零值。 因此，我们可以用如下形式：
    - return &File{fd: fd, name: name}
- 少数情况下，若复合字面不包括任何字段，它将创建该类型的零值。表达式 new(File) 和 &File{} 是等价的。
- 复合字面同样可用于创建数组、切片以及映射，字段标签是索引还是映射键则视情况而定。 在下例初始化过程中，无论 Enone、Eio 和 Einval 的值是什么，只要它们的标签不同就行。
```go
a := [...]string   {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
s := []string      {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
m := map[int]string{Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
```

## 使用make分配
- 再回到内存分配上来。内建函数 make(T,args) 的目的不同于 new(T)。它只用于创建切片、映射和信道，并返回类型为 T（而非 *T）的一个已初始化 （而非置零）的值。 出现这种用差异的原因在于，这三种类型本质上为引用数据类型，它们在使用前必须初始化。 例如，切片是一个具有三项内容的描述符，包含一个指向（数组内部）数据的指针、长度以及容量， 在这三项被初始化之前，该切片为 nil。对于切片、映射和信道，make 用于初始化其内部的数据结构并准备好将要使用的值。例如，
    - make([]int, 10, 100)
- 会分配一个具有100个 int 的数组空间，接着创建一个长度为10， 容量为100并指向该数组中前10个元素的切片结构。（生成切片时，其容量可以省略，更多信息见切片一节。） 与此相反，new([]int) 会返回一个指向新分配的，已置零的切片结构， 即一个指向 nil 切片值的指针。
- 下面的例子阐明了 new 和 make 之间的区别：
```go
var p *[]int = new([]int)       // 分配切片结构；*p == nil；很少用到
var v  []int = make([]int, 100) // 切片 v 现在引用了一个具有 100 个 int 元素的新数组

// 没必要的复杂用法:
var p *[]int = new([]int)
*p = make([]int, 100, 100)
// p是指向切片的指针, *p是这个切片

// 常规用法:
v := make([]int, 100)
```
- 请记住，make 只适用于映射、切片和信道且不返回指针。若要获得明确的指针，请使用 new 分配内存。

## 数组
- 在详细规划内存布局时，数组是非常有用的，有时还能避免过多的内存分配， 但它们主要用作切片的构件。这是下一节的主题了，不过要先说上几句来为它做铺垫。
- 以下为数组在Go和C中的主要区别。在Go中，
    - 数组是值。将一个数组赋予另一个数组会复制其所有元素。
    _ 特别地，若将某个数组传入某个函数，它将接收到该数组的一份副本而非指针。
    - 数组的大小是其类型的一部分。类型 [10]int 和 [20]int 是不同的。
    - 数组为值的属性很有用，但代价高昂；若你想要C那样的行为和效率，你可以传递一个指向该数组的指针。
    - 但这并不是Go的习惯用法，切片才是。

## 切片
- 切片通过对数组进行封装，为数据序列提供了更通用、强大而方便的接口。 除了矩阵变换这类需要明确维度的情况外，Go中的大部分数组编程都是通过切片来完成的。
- 切片保存了对底层数组的引用，若你将某个切片赋予另一个切片，它们会引用同一个数组。 若某个函数将一个切片作为参数传入，则它对该切片元素的修改对调用者而言同样可见， 这可以理解为传递了底层数组的指针。因此，Read 函数可接受一个切片实参 而非一个指针和一个计数；切片的长度决定了可读取数据的上限。以下为 os 包中 File 类型的 Read 方法签名:

func (f *File) Read(buf []byte) (n int, err error)
该方法返回读取的字节数和一个错误值（若有的话）。若要从更大的缓冲区 b 中读取前32个字节，只需对其进行切片即可。

    n, err := f.Read(buf[0:32])
这种切片的方法常用且高效。若不谈效率，以下片段同样能读取该缓冲区的前32个字节。

    var n int
    var err error
    for i := 0; i < 32; i++ {
        nbytes, e := f.Read(buf[i:i+1])  // Read one byte.
        n += nbytes
        if nbytes == 0 || e != nil {
            err = e
            break
        }
    }
只要切片不超出底层数组的限制，它的长度就是可变的，只需将它赋予其自身的切片即可。 切片的容量可通过内建函数 cap 获得，它将给出该切片可取得的最大长度。 以下是将数据追加到切片的函数。若数据超出其容量，则会重新分配该切片。返回值即为所得的切片。 该函数中所使用的 len 和 cap 在应用于 nil 切片时是合法的，它会返回 0。

func Append(slice, data []byte) []byte {
    l := len(slice)
    if l + len(data) > cap(slice) {  // 重新分配
        // 为未来的增长,双重分配所需的内容.
        newSlice := make([]byte, (l+len(data))*2)
        // copy函数是预先声明的，适用于任何切片类型。
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0:l+len(data)]
    copy(slice[l:], data)
    return slice
}
最终我们必须返回切片，因为尽管 Append 可修改 slice 的元素，但切片自身（其运行时数据结构包含指针、长度和容量）是通过值传递的。

向切片追加东西的想法非常有用，因此有专门的内建函数 append。 要理解该函数的设计，我们还需要一些额外的信息，我们将稍后再介绍它。

## 二维切片
- Go的数组和切片都是一维的。要创建等价的二维数组或切片，就必须定义一个数组的数组， 或切片的切片，就像这样：
    - type Transform [3][3]float64  // 一个 3x3 的数组，其实是包含多个数组的一个数组。
    - type LinesOfText [][]byte     // 包含多个字节切片的一个切片。
    - 由于切片长度是可变的，因此其内部可能拥有多个不同长度的切片。这是种常见的情况：每行都有其自己的长度。
- 有时必须分配一个二维数组，例如在处理像素的扫描行时，这种情况就会发生。 我们有两种方式来达到这个目的。一种就是独立地分配每一个切片；而另一种就是只分配一个数组， 将各个切片都指向它。采用哪种方式取决于你的应用。若切片会增长或收缩， 就应该通过独立分配来避免覆盖下一行；若不会，用单次分配来构造对象会更加高效。 以下是这两种方法的大概代码，仅供参考。首先是一次一行的：
```go
// 分配底层切片.
picture := make([][]uint8, YSize) // y每一行的大小
//循环遍历每一行
for i := range picture {
	picture[i] = make([]uint8, XSize)
}

// 现在是一次分配，对行进行切片：
// 分配底层切片
picture := make([][]uint8, YSize) //  每 y 个单元一行。
// 分配一个大一些的切片以容纳所有的元素
pixels := make([]uint8, XSize*YSize) // 指定类型[]uint8, 即便图片是 [][]uint8.
//循环遍历图片所有行，从剩余像素切片的前面对每一行进行切片。
for i := range picture {
	picture[i], pixels = pixels[:XSize], pixels[XSize:]
}
```

## 映射
- 映射是方便而强大的内建数据结构，它可以关联不同类型的值。其键可以是任何相等性操作符支持的类型， 如整数、浮点数、复数、字符串、指针、接口（只要其动态类型支持相等性判断）、结构以及数组。 切片不能用作映射键，因为它们的相等性还未定义。与切片一样，映射也是引用类型。 若将映射传入函数中，并更改了该映射的内容，则此修改对调用者同样可见。
- 映射可使用一般的复合字面语法进行构建，其键-值对使用冒号分隔，因此可在初始化时很容易地构建它们。
```go
var timeZone = map[string]int{
    "UTC":  0*60*60,
    "EST": -5*60*60,
    "CST": -6*60*60,
    "MST": -7*60*60,
    "PST": -8*60*60,
}
```
- 赋值和获取映射值的语法类似于数组，不同的是映射的索引不必为整数。
- 若试图通过映射中不存在的键来取值，就会返回与该映射中项的类型对应的零值。 例如，若某个映射包含整数，当查找一个不存在的键时会返回 0。 集合可实现成一个值类型为 bool 的映射。将该映射中的项置为 true 可将该值放入集合中，此后通过简单的索引操作即可判断是否存在。
```go
attended := map[string]bool{
    "Ann": true,
    "Joe": true,
    ...
}

if attended[person] { // person不在集合中，返回 false
    fmt.Println(person, "was at the meeting")
}
```
- 有时你需要区分某项是不存在还是其值为零值。如对于一个值本应为零的 "UTC" 条目，也可能是由于不存在该项而得到零值。你可以使用多重赋值的形式来分辨这种情况。
    - seconds, ok := timeZone[tz]
    - 若仅需判断映射中是否存在某项而不关心实际的值，可使用空白标识符 （_）来代替该值的一般变量。
- 要删除映射中的某项，可使用内建函数 delete，它以映射及要被删除的键为实参。 即便对应的键不在该映射中，此操作也是安全的。
    - delete(timeZone, "PDT")  // 现在是标准时间
## 打印
- Go采用的格式化打印风格和C的 printf 族类似，但却更加丰富而通用。 这些函数位于 fmt 包中，且函数名首字母均为大写：如 fmt.Printf、fmt.Fprintf，fmt.Sprintf 等。 字符串函数（Sprintf 等）会返回一个字符串，而非填充给定的缓冲区。
- 你无需提供一个格式字符串。每个 Printf、Fprintf 和 Sprintf 都分别对应另外的函数，如 Print 与 Println。 这些函数并不接受格式字符串，而是为每个实参生成一种默认格式。Println 系列的函数还会在实参中插入空格，并在输出时追加一个换行符，而 Print 版本仅在操作数两侧都没有字符串时才添加空白。以下示例中各行产生的输出都是一样的。
- fmt.Fprint 一类的格式化打印函数可接受任何实现了 io.Writer 接口的对象作为第一个实参；变量os.Stdout 与 os.Stderr 都是人们熟知的例子。
- 从这里开始，就与C有些不同了。首先，像 %d 这样的数值格式并不接受表示符号或大小的标记， 打印例程会根据实参的类型来决定这些属性。

var x uint64 = 1<<64 - 1
fmt.Printf("%d %x; %d %x\n", x, x, int64(x), int64(x))
打印结果

18446744073709551615 ffffffffffffffff; -1 -1
复制代码
若你只想要默认的转换，如使用十进制的整数，你可以使用通用的格式 %v（对应“值”）；其结果与 Print 和 Println 的输出完全相同。此外，这种格式还能打印任意值，甚至包括数组、结构体和映射。 以下是打印上一节中定义的时区映射的语句。

fmt.Printf("%v\n", timeZone)  // or just fmt.Println(timeZone)
- 对于映射, Printf 会自动对映射值按照键的字典顺序排序。
    当然，映射中的键可能按任意顺序输出。当打印结构体时，改进的格式 %+v 会为结构体的每个字段添上字段名，而另一种格式 %#v 将完全按照Go的语法打印值。
```go
type T struct {
    a int
    b float64
    c string
}
t := &T{ 7, -2.35, "abc\tdef" }
fmt.Printf("%v\n", t)
fmt.Printf("%+v\n", t)
fmt.Printf("%#v\n", t)
fmt.Printf("%#v\n", timeZone)
// 将打印

&{7 -2.35 abc   def}
&{a:7 b:-2.35 c:abc     def}
&main.T{a:7, b:-2.35, c:"abc\tdef"}
map[string]int{"CST":-21600, "EST":-18000, "MST":-25200, "PST":-28800, "UTC":0}
```
- （请注意其中的&符号）当遇到 string 或 []byte 值时， 可使用 %q 产生带引号的字符串；而格式 %#q 会尽可能使用反引号。（%q 格式也可用于整数和符文，它会产生一个带单引号的符文常量。） 此外，%x 还可用于字符串、字节数组以及整数，并生成一个很长的十六进制字符串， 而带空格的格式（% x）还会在字节之间插入空格。
- 另一种实用的格式是 %T，它会打印某个值的类型。
    - 若你想控制自定义类型的默认格式，只需为该类型定义一个具有 String() string 签名的方法。对于我们简单的类型 T，可进行如下操作。
```go
func (t *T) String() string {
    return fmt.Sprintf("%d/%g/%q", t.a, t.b, t.c)
}
fmt.Printf("%v\n", t)
// 会打印出如下格式：
7/-2.35/"abc\tdef"
```
-（如果你需要像指向 T 的指针那样打印类型 T 的值， String 的接收者就必须是值类型的；上面的例子中接收者是一个指针， 因为这对结构来说更高效而通用。更多详情见指针vs.值接收者一节）

- 我们的 String 方法也可调用 Sprintf， 因为打印例程可以完全重入并按这种方式封装。不过有一个重要的细节你需要知道： 请勿通过调用 Sprintf 来构造 String 方法，因为它会无限递归你的的 String 方法。如果 Sprintf 调用试图将接收器直接打印为字符串，而该字符串又将再次调用该方法，则会发生这种情况。这是一个常见的错误，如本例所示。

type MyString string

func (m MyString) String() string {
    return fmt.Sprintf("MyString=%s", m) // 错误：会无限递归
}
复制代码
要解决这个问题也很简单：将该实参转换为基本的字符串类型，它没有这个方法。

type MyString string
func (m MyString) String() string {
    return fmt.Sprintf("MyString=%s", string(m)) // 可以：注意转换
}
在 初始化 一节中，我们将看到避免这种递归的另一种技术。

另一种打印技术就是将打印例程的实参直接传入另一个这样的例程。Printf 的签名为其最后的实参使用了 ...interface{} 类型，这样格式的后面就能出现任意数量，任意类型的形参了。

func Printf(format string, v ...interface{}) (n int, err error) {
复制代码
在 Printf 函数中，v 看起来更像是 []interface{} 类型的变量，但如果将它传递到另一个变参函数中，它就像是常规实参列表了。 以下是我们之前用过的 log.Println 的实现。它直接将其实参传递给 fmt.Sprintln 进行实际的格式化。

// Println 通过 fmt.Println 的方式将日志打印到标准记录器
func Println(v ...interface{}) {
    std.Output(2, fmt.Sprintln(v...))  // Output takes parameters (int, string)
}
复制代码
在该 Sprintln 嵌套调用中，我们将 ... 写在 v 之后来告诉编译器将 v 视作一个实参列表，否则它会将 v 当做单一的切片实参来传递。

还有很多关于打印知识点没有提及。详情请参阅 godoc 对 fmt 包的说明文档。

顺便一提，... 形参可指定具体的类型，例如从整数列表中选出最小值的函数 min，其形参可为 ...int 类型。

func Min(a ...int) int {
    min := int(^uint(0) >> 1)  // 最大的 int
    for _, i := range a {
        if i < min {
            min = i
        }
    }
    return min
}
复制代码

## 追加
- 现在我们要对内建函数 append 的设计进行补充说明。append 函数的签名不同于前面我们自定义的 Append 函数。大致来说，它就像这样：
    - func append(slice []T, elements ...T) []T
- 其中的 T 为任意给定类型的占位符。实际上，你无法在Go中编写一个类型 T, append 为内建函数的原因：它需要编译器的支持。
- append 会在切片末尾追加元素并返回结果。我们必须返回结果， 原因与我们手写的 Append 一样，即底层数组可能会被改变。
- 但如果我们要像 Append 那样将一个切片追加到另一个切片中呢？ 很简单：在调用的地方使用 ...
```go
x := []int{1,2,3}
y := []int{4,5,6}
x = append(x, y...)
fmt.Println(x)
```
- 如果没有 ...，它就会由于类型错误而无法编译，因为 y 不是 int 类型的。
Golang 内置方法append用于向切片中追加一个或多个元素，实际项目中比较常见。

其原型如下：

func append(slice []Type, elems ...Type) []Type
复制代码
本节不会对append的使用方式详细展开，而是重点介绍几个使用中常见的误区或者陷阱。

#热身
按照惯例，我们先拿几个小题目来检测一下对append的理解是否足够深刻。

#题目一
函数Validation()用于一些合法性检查，每遇到一个错误，就生成一个新的error并追加到切片errs中， 最后返回包含所有错误信息的切片。 为了简单起见，假定函数发现了三个错误，如下所示：

func Validation() []error {
    var errs []error

    append(errs, errors.New("error 1"))
    append(errs, errors.New("error 2"))
    append(errs, errors.New("error 3"))

    return errs
}
复制代码
请问函数Validation()有什么问题？

#题目二
函数ValidateName()用于检查某个名字是否合法，如果不为空则认为合法，否则返回一个error。 类似的，还可以有很多检查项，比如检查性别、年龄等，我们统称为子检查项。 函数Validations()用于收集所有子检查项的错误信息，将错误信息汇总到一个切片中返回。

请问函数Validations()有什么问题？

func ValidateName(name string) error {
    if name != "" {
        return nil
    }

    return errors.New("empty name")
}

func Validations(name string) []error {
    var errs []error

    errs = append(errs, ValidateName(name))

    return errs
}
复制代码
#陷阱
前面的热身题目均来源于实际项目（已经做了最大程度的精简），分别代表一个本节将要介绍的陷阱。

#陷阱一： append 会改变切片的地址
append的本质是向切片中追加数据，而随着切片中元素逐渐增加，当切片底层的数组将满时，切片会发生扩容， 扩容会导致产生一个新的切片（拥有容量更大的底层数组），更多关于切片的信息，请查阅切片相关章节。

append每个追加元素，都有可能触发切片扩容，也即有可能返回一个新的切片，这也是append函数声明中返回值为切片的原因。实际使用中应该总是接收该返回值。

上述题目一中，由于初始切片长度为0，所以实际上每次append都会产生一个新的切片并迅速抛弃（被gc回收）。 原始切片并没有任何改变。需要特别说明的是，不管初始切片长度为多少，不接收append返回都是有极大风险的。

另外，目前有很多的工具可以自动检查出类似的问题，比如GolandIDE就会给出很明显的提示。

#陷阱二： append 可以追加nil值
向切片中追加一个nil值是完全不会报错的，如下代码所示：

slice := append(slice, nil)
复制代码
经过追加后，slice的长度递增1。

实际上nil是一个预定义的值，即空值，所以完全有理由向切片中追加。

上述题目二中，就是典型的向切片中追加nil（当名字为空时）的问题。单纯从技术上讲是没有问题，但在题目二场景中就有很大的问题。

题目中函数用于收集所有错误信息，没有错误就不应该追加到切片中。因后，后续极有可能会根据切片的长度来判断是否有错误发生，比如：

func foo() {
    errs := Validations("")

    if len(errs) > 0 {
        println(errs)
        os.Exit(1)
    }
}
复制代码
如果向切片中追加一个nil元素，那么切片长度则不再为0，程序很可能因此而退出，更糟糕的是，这样的切片是没有内容会打印出来的，这无疑又增加了定位难度。