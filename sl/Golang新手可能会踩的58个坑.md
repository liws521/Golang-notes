
- 左大括号{不能单独放一行
    编译器会在每行代码尾部特定的分隔符后加分号来分隔多条语句, func main(), 不在同一行写{, 就会被添加;, 导致miss function body
- 函数体代码中未使用的变量会编译报错, 全局变量不使用是可以的
- import未使用的包会编译报错, 可以用_来解决, 此时会执行包中的init()
- := 短变量声明只能在函数内部使用, 全局变量不可
- := 左侧至少有一个未被声明的变量, 才可以
- struct的field不能作为 := 的左值
```go
a.name, err := "abc", nil
// err未被声明过, 满足至少有一个未被声明的变量
// 但是结构体变量名.字段名不能作为:=的左值
// 通过提前声明err解决
var err error
a.name, err = "abc", nil
```
- 值为nil的slice, map的必须make分配空间后才能使用
- 创建map的时候可以指定容量, 但是不能像slice一样使用cap()来检测分配空间的大小, cap()不接收map类型的参数
- string的零值是空串"", 不要把nil和string扯上关系
- 数组做函数参数, 在C/C++中传数组名是地址传递, Golnag中数组是值类型, 做参数是值传递, 生成一份新拷贝, 想修改传数组类型对应的指针类型或者使用slice
- Golang的for-range返回i, v两个值
- 区分Golang的数组和slice
    - 二维数组内存连续分布, 和C一样
    - 二维切片, 切片的切片, 不连续







- a, b = b, a, 交换两个变量的值可以这样么, 哈哈

7.不小心覆盖了变量
对从动态语言转过来的开发者来说，简短声明很好用，这可能会让人误会 := 是一个赋值操作符。如果你在新的代码块中像下边这样误用了 :=，编译不会报错，但是变量不会按你的预期工作：

func main() {
    x := 1
    println(x)        // 1
    {
        println(x)    // 1
        x := 2
        println(x)    // 2    // 新的 x 变量的作用域只在代码块内部
    }
    println(x)        // 1
}
复制代码
这是 Go 开发者常犯的错，而且不易被发现。可使用 vet工具来诊断这种变量覆盖，Go 默认不做覆盖检查，添加 -shadow 选项来启用：

    > go tool vet -shadow main.go
    main.go:9: declaration of "x" shadows declaration at main.go:5
复制代码
注意 vet 不会报告全部被覆盖的变量，可以使用 go-nyet 来做进一步的检测：

    > $GOPATH/bin/go-nyet main.go
    main.go:10:3:Shadowing variable `x`


    8.显式类型的变量无法使用 nil 来初始化
nil 是 interface、function、pointer、map、slice 和 channel 类型变量的默认初始值。但声明时不指定类型，编译器也无法推断出变量的具体类型。

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

6.关闭 HTTP 的响应体
使用 HTTP 标准库发起请求、获取响应时，即使你不从响应中读取任何数据或响应为空，都需要手动关闭响应体。新手很容易忘记手动关闭，或者写在了错误的位置：

// 请求失败造成 panic
func main() {
    resp, err := http.Get("https://api.ipify.org?format=json")
    defer resp.Body.Close()    // resp 可能为 nil，不能读取 Body
    if err != nil {
        fmt.Println(err)
        return
    }

    body, err := ioutil.ReadAll(resp.Body)
    checkError(err)

    fmt.Println(string(body))
}

func checkError(err error) {
    if err != nil{
        log.Fatalln(err)
    }
}
复制代码
上边的代码能正确发起请求，但是一旦请求失败，变量 resp 值为 nil，造成 panic：

panic: runtime error: invalid memory address or nil pointer dereference

应该先检查 HTTP 响应错误为 nil，再调用 resp.Body.Close() 来关闭响应体：

// 大多数情况正确的示例
func main() {
    resp, err := http.Get("https://api.ipify.org?format=json")
    checkError(err)

    defer resp.Body.Close()    // 绝大多数情况下的正确关闭方式
    body, err := ioutil.ReadAll(resp.Body)
    checkError(err)

    fmt.Println(string(body))
}
复制代码
输出：

Get https://api.ipify.org?format=...: x509: certificate signed by unknown authority
复制代码
绝大多数请求失败的情况下，resp 的值为 nil 且 err 为 non-nil。但如果你得到的是重定向错误，那它俩的值都是 non-nil，最后依旧可能发生内存泄露。2 个解决办法：

可以直接在处理 HTTP 响应错误的代码块中，直接关闭非 nil 的响应体。
手动调用 defer 来关闭响应体：
// 正确示例
func main() {
    resp, err := http.Get("http://www.baidu.com")

    // 关闭 resp.Body 的正确姿势
    if resp != nil {
        defer resp.Body.Close()
    }

    checkError(err)
    defer resp.Body.Close()

    body, err := ioutil.ReadAll(resp.Body)
    checkError(err)

    fmt.Println(string(body))
}
复制代码
resp.Body.Close() 早先版本的实现是读取响应体的数据之后丢弃，保证了 keep-alive 的 HTTP 连接能重用处理不止一个请求。但 Go 的最新版本将读取并丢弃数据的任务交给了用户，如果你不处理，HTTP 连接可能会直接关闭而非重用，参考在 Go 1.5 版本文档。

如果程序大量重用 HTTP 长连接，你可能要在处理响应的逻辑代码中加入：

    _, err = io.Copy(ioutil.Discard, resp.Body) // 手动丢弃读取完毕的数据
复制代码
如果你需要完整读取响应，上边的代码是需要写的。比如在解码 API 的 JSON 响应数据：

    json.NewDecoder(resp.Body).Decode(&data)
复制代码
#37.关闭 HTTP 连接
一些支持 HTTP1.1 或 HTTP1.0 配置了 connection: keep-alive 选项的服务器会保持一段时间的长连接。但标准库 "net/http" 的连接默认只在服务器主动要求关闭时才断开，所以你的程序可能会消耗完 socket 描述符。解决办法有 2 个，请求结束后：

直接设置请求变量的 Close 字段值为 true，每次请求结束后就会主动关闭连接。
设置 Header 请求头部选项 Connection: close，然后服务器返回的响应头部也会有这个选项，此时 HTTP 标准库会主动断开连接。
// 主动关闭连接
func main() {
    req, err := http.NewRequest("GET", "http://golang.org", nil)
    checkError(err)

    req.Close = true
    //req.Header.Add("Connection", "close")    // 等效的关闭方式

    resp, err := http.DefaultClient.Do(req)
    if resp != nil {
        defer resp.Body.Close()
    }
    checkError(err)

    body, err := ioutil.ReadAll(resp.Body)
    checkError(err)

    fmt.Println(string(body))
}
复制代码
你可以创建一个自定义配置的 HTTP transport 客户端，用来取消 HTTP 全局的复用连接：

func main() {
    tr := http.Transport{DisableKeepAlives: true}
    client := http.Client{Transport: &tr}

    resp, err := client.Get("https://golang.google.cn/")
    if resp != nil {
        defer resp.Body.Close()
    }
    checkError(err)

    fmt.Println(resp.StatusCode)    // 200

    body, err := ioutil.ReadAll(resp.Body)
    checkError(err)

    fmt.Println(len(string(body)))
}
复制代码
根据需求选择使用场景：

若你的程序要向同一服务器发大量请求，使用默认的保持长连接。
若你的程序要连接大量的服务器，且每台服务器只请求一两次，那收到请求后直接关闭连接。或增加最大文件打开数 fs.file-max 的值。
#38.将 JSON 中的数字解码为 interface 类型
在 encode/decode JSON 数据时，Go 默认会将数值当做 float64 处理，比如下边的代码会造成 panic：

func main() {
    var data = []byte(`{"status": 200}`)
    var result map[string]interface{}

    if err := json.Unmarshal(data, &result); err != nil {
        log.Fatalln(err)
    }

    fmt.Printf("%T\n", result["status"])    // float64
    var status = result["status"].(int)    // 类型断言错误
    fmt.Println("Status value: ", status)
}
复制代码
panic: interface conversion: interface {} is float64, not int

如果你尝试 decode 的 JSON 字段是整型，你可以：

将 int 值转为 float 统一使用
将 decode 后需要的 float 值转为 int 使用
// 将 decode 的值转为 int 使用
func main() {
    var data = []byte(`{"status": 200}`)
    var result map[string]interface{}

    if err := json.Unmarshal(data, &result); err != nil {
        log.Fatalln(err)
    }

    var status = uint64(result["status"].(float64))
    fmt.Println("Status value: ", status)
}
复制代码
使用 Decoder 类型来 decode JSON 数据，明确表示字段的值类型
// 指定字段类型
func main() {
    var data = []byte(`{"status": 200}`)
    var result map[string]interface{}

    var decoder = json.NewDecoder(bytes.NewReader(data))
    decoder.UseNumber()

    if err := decoder.Decode(&result); err != nil {
        log.Fatalln(err)
    }

    var status, _ = result["status"].(json.Number).Int64()
    fmt.Println("Status value: ", status)
}

 // 你可以使用 string 来存储数值数据，在 decode 时再决定按 int 还是 float 使用
 // 将数据转为 decode 为 string
 func main() {
     var data = []byte({"status": 200})
      var result map[string]interface{}
      var decoder = json.NewDecoder(bytes.NewReader(data))
      decoder.UseNumber()
      if err := decoder.Decode(&result); err != nil {
          log.Fatalln(err)
      }
    var status uint64
      err := json.Unmarshal([]byte(result["status"].(json.Number).String()), &status);
    checkError(err)
       fmt.Println("Status value: ", status)
}
复制代码
使用 struct 类型将你需要的数据映射为数值型

// struct 中指定字段类型
func main() {
      var data = []byte(`{"status": 200}`)
      var result struct {
          Status uint64 `json:"status"`
      }

      err := json.NewDecoder(bytes.NewReader(data)).Decode(&result)
      checkError(err)
    fmt.Printf("Result: %+v", result)
}
复制代码
可以使用 struct 将数值类型映射为 json.RawMessage 原生数据类型 适用于如果 JSON 数据不着急 decode 或 JSON 某个字段的值类型不固定等情况：
// 状态名称可能是 int 也可能是 string，指定为 json.RawMessage 类型
func main() {
    records := [][]byte{
        []byte(`{"status":200, "tag":"one"}`),
        []byte(`{"status":"ok", "tag":"two"}`),
    }

    for idx, record := range records {
        var result struct {
            StatusCode uint64
            StatusName string
            Status     json.RawMessage `json:"status"`
            Tag        string          `json:"tag"`
        }

        err := json.NewDecoder(bytes.NewReader(record)).Decode(&result)
        checkError(err)

        var name string
        err = json.Unmarshal(result.Status, &name)
        if err == nil {
            result.StatusName = name
        }

        var code uint64
        err = json.Unmarshal(result.Status, &code)
        if err == nil {
            result.StatusCode = code
        }

        fmt.Printf("[%v] result => %+v\n", idx, result)
    }
复制代码
#39.struct、array、slice 和 map 的值比较
可以使用相等运算符 == 来比较结构体变量，前提是两个结构体的成员都是可比较的类型：

type data struct {
    num     int
    fp      float32
    complex complex64
    str     string
    char    rune
    yes     bool
    events  <-chan string
    handler interface{}
    ref     *byte
    raw     [10]byte
}

func main() {
    v1 := data{}
    v2 := data{}
    fmt.Println("v1 == v2: ", v1 == v2)    // true
}
复制代码
如果两个结构体中有任意成员是不可比较的，将会造成编译错误。注意数组成员只有在数组元素可比较时候才可比较。

type data struct {
    num    int
    checks [10]func() bool        // 无法比较
    doIt   func() bool        // 无法比较
    m      map[string]string    // 无法比较
    bytes  []byte            // 无法比较
}

func main() {
    v1 := data{}
    v2 := data{}

    fmt.Println("v1 == v2: ", v1 == v2)
}
复制代码
invalid operation: v1 == v2 (struct containing [10]func() bool cannot be compared)

Go 提供了一些库函数来比较那些无法使用 == 比较的变量，比如使用 "reflect" 包的 DeepEqual() ：

// 比较相等运算符无法比较的元素
func main() {
    v1 := data{}
    v2 := data{}
    fmt.Println("v1 == v2: ", reflect.DeepEqual(v1, v2))    // true

    m1 := map[string]string{"one": "a", "two": "b"}
    m2 := map[string]string{"two": "b", "one": "a"}
    fmt.Println("v1 == v2: ", reflect.DeepEqual(m1, m2))    // true

    s1 := []int{1, 2, 3}
    s2 := []int{1, 2, 3}
       // 注意两个 slice 相等，值和顺序必须一致
    fmt.Println("v1 == v2: ", reflect.DeepEqual(s1, s2))    // true
}
复制代码
这种比较方式可能比较慢，根据你的程序需求来使用。DeepEqual() 还有其他用法：

func main() {
    var b1 []byte = nil
    b2 := []byte{}
    fmt.Println("b1 == b2: ", reflect.DeepEqual(b1, b2))    // false
}
复制代码
注意：

DeepEqual() 并不总适合于比较 slice
func main() {
    var str = "one"
    var in interface{} = "one"
    fmt.Println("str == in: ", reflect.DeepEqual(str, in))    // true

    v1 := []string{"one", "two"}
    v2 := []string{"two", "one"}
    fmt.Println("v1 == v2: ", reflect.DeepEqual(v1, v2))    // false

    data := map[string]interface{}{
        "code":  200,
        "value": []string{"one", "two"},
    }
    encoded, _ := json.Marshal(data)
    var decoded map[string]interface{}
    json.Unmarshal(encoded, &decoded)
    fmt.Println("data == decoded: ", reflect.DeepEqual(data, decoded))    // false
}
复制代码
如果要大小写不敏感来比较 byte 或 string 中的英文文本，可以使用 "bytes" 或 "strings" 包的 ToUpper() 和 ToLower() 函数。比较其他语言的 byte 或 string，应使用 bytes.EqualFold() 和 strings.EqualFold()

如果 byte slice 中含有验证用户身份的数据（密文哈希、token 等），不应再使用 reflect.DeepEqual()、bytes.Equal()、 bytes.Compare()。这三个函数容易对程序造成 timing attacks，此时应使用 "crypto/subtle" 包中的 subtle.ConstantTimeCompare() 等函数

reflect.DeepEqual() 认为空 slice 与 nil slice 并不相等，但注意 byte.Equal() 会认为二者相等：
func main() {
    var b1 []byte = nil
    b2 := []byte{}

    // b1 与 b2 长度相等、有相同的字节序
    // nil 与 slice 在字节上是相同的
    fmt.Println("b1 == b2: ", bytes.Equal(b1, b2))    // true
}
复制代码
#40.从 panic 中恢复
在一个 defer 延迟执行的函数中调用 recover() ，它便能捕捉 / 中断 panic

// 错误的 recover 调用示例
func main() {
    recover()    // 什么都不会捕捉
    panic("not good")    // 发生 panic，主程序退出
    recover()    // 不会被执行
    println("ok")
}

// 正确的 recover 调用示例
func main() {
    defer func() {
        fmt.Println("recovered: ", recover())
    }()
    panic("not good")
}
复制代码
从上边可以看出，recover() 仅在 defer 执行的函数中调用才会生效。

// 错误的调用示例
func main() {
    defer func() {
        doRecover()
    }()
    panic("not good")
}

func doRecover() {
    fmt.Println("recobered: ", recover())
}
复制代码
recobered: panic: not good

#41.在 range 迭代 slice、array、map 时通过更新引用来更新元素
在 range 迭代中，得到的值其实是元素的一份值拷贝，更新拷贝并不会更改原来的元素，即是拷贝的地址并不是原有元素的地址：

func main() {
    data := []int{1, 2, 3}
    for _, v := range data {
        v *= 10        // data 中原有元素是不会被修改的
    }
    fmt.Println("data: ", data)    // data:  [1 2 3]
}
复制代码
如果要修改原有元素的值，应该使用索引直接访问：

func main() {
    data := []int{1, 2, 3}
    for i, v := range data {
        data[i] = v * 10    
    }
    fmt.Println("data: ", data)    // data:  [10 20 30]
}
复制代码
如果你的集合保存的是指向值的指针，需稍作修改。依旧需要使用索引访问元素，不过可以使用 range 出来的元素直接更新原有值：

func main() {
    data := []*struct{ num int }{{1}, {2}, {3},}
    for _, v := range data {
        v.num *= 10    // 直接使用指针更新
    }
    fmt.Println(data[0], data[1], data[2])    // &{10} &{20} &{30}
}
复制代码
#42.slice 中隐藏的数据
从 slice 中重新切出新 slice 时，新 slice 会引用原 slice 的底层数组。如果跳了这个坑，程序可能会分配大量的临时 slice 来指向原底层数组的部分数据，将导致难以预料的内存使用。

func get() []byte {
    raw := make([]byte, 10000)
    fmt.Println(len(raw), cap(raw), &raw[0])    // 10000 10000 0xc420080000
    return raw[:3]    // 重新分配容量为 10000 的 slice
}

func main() {
    data := get()
    fmt.Println(len(data), cap(data), &data[0])    // 3 10000 0xc420080000
}
复制代码
可以通过拷贝临时 slice 的数据，而不是重新切片来解决：

func get() (res []byte) {
    raw := make([]byte, 10000)
    fmt.Println(len(raw), cap(raw), &raw[0])    // 10000 10000 0xc420080000
    res = make([]byte, 3)
    copy(res, raw[:3])
    return
}

func main() {
    data := get()
    fmt.Println(len(data), cap(data), &data[0])    // 3 3 0xc4200160b8
}
复制代码
#43.Slice 中数据的误用
举个简单例子，重写文件路径（存储在 slice 中）

分割路径来指向每个不同级的目录，修改第一个目录名再重组子目录名，创建新路径：

// 错误使用 slice 的拼接示例
func main() {
    path := []byte("AAAA/BBBBBBBBB")
    sepIndex := bytes.IndexByte(path, '/') // 4
    println(sepIndex)

    dir1 := path[:sepIndex]
    dir2 := path[sepIndex+1:]
    println("dir1: ", string(dir1))        // AAAA
    println("dir2: ", string(dir2))        // BBBBBBBBB

    dir1 = append(dir1, "suffix"...)
       println("current path: ", string(path))    // AAAAsuffixBBBB

    path = bytes.Join([][]byte{dir1, dir2}, []byte{'/'})
    println("dir1: ", string(dir1))        // AAAAsuffix
    println("dir2: ", string(dir2))        // uffixBBBB

    println("new path: ", string(path))    // AAAAsuffix/uffixBBBB    // 错误结果
}
复制代码
拼接的结果不是正确的 AAAAsuffix/BBBBBBBBB，因为 dir1、 dir2 两个 slice 引用的数据都是 path 的底层数组，第 13 行修改 dir1 同时也修改了 path，也导致了 dir2 的修改

解决方法：

重新分配新的 slice 并拷贝你需要的数据
使用完整的 slice 表达式：input[low:high:max]，容量便调整为 max - low
// 使用 full slice expression
func main() {

    path := []byte("AAAA/BBBBBBBBB")
    sepIndex := bytes.IndexByte(path, '/') // 4
    dir1 := path[:sepIndex:sepIndex]        // 此时 cap(dir1) 指定为4， 而不是先前的 16
    dir2 := path[sepIndex+1:]
    dir1 = append(dir1, "suffix"...)

    path = bytes.Join([][]byte{dir1, dir2}, []byte{'/'})
    println("dir1: ", string(dir1))        // AAAAsuffix
    println("dir2: ", string(dir2))        // BBBBBBBBB
    println("new path: ", string(path))    // AAAAsuffix/BBBBBBBBB
}
复制代码
第 6 行中第三个参数是用来控制 dir1 的新容量，再往 dir1 中 append 超额元素时，将分配新的 buffer 来保存。而不是覆盖原来的 path 底层数组

#44.旧 slice
当你从一个已存在的 slice 创建新 slice 时，二者的数据指向相同的底层数组。如果你的程序使用这个特性，那需要注意 "旧"（stale） slice 问题。

某些情况下，向一个 slice 中追加元素而它指向的底层数组容量不足时，将会重新分配一个新数组来存储数据。而其他 slice 还指向原来的旧底层数组。

// 超过容量将重新分配数组来拷贝值、重新存储
func main() {
    s1 := []int{1, 2, 3}
    fmt.Println(len(s1), cap(s1), s1)    // 3 3 [1 2 3 ]

    s2 := s1[1:]
    fmt.Println(len(s2), cap(s2), s2)    // 2 2 [2 3]

    for i := range s2 {
        s2[i] += 20
    }
    // 此时的 s1 与 s2 是指向同一个底层数组的
    fmt.Println(s1)        // [1 22 23]
    fmt.Println(s2)        // [22 23]

    s2 = append(s2, 4)    // 向容量为 2 的 s2 中再追加元素，此时将分配新数组来存

    for i := range s2 {
        s2[i] += 10
    }
    fmt.Println(s1)        // [1 22 23]    // 此时的 s1 不再更新，为旧数据
    fmt.Println(s2)        // [32 33 14]
}
复制代码
#45.类型声明与方法
从一个现有的非 interface 类型创建新类型时，并不会继承原有的方法：

// 定义 Mutex 的自定义类型
type myMutex sync.Mutex

func main() {
    var mtx myMutex
    mtx.Lock()
    mtx.UnLock()
}
复制代码
mtx.Lock undefined (type myMutex has no field or method Lock)...

如果你需要使用原类型的方法，可将原类型以匿名字段的形式嵌到你定义的新 struct 中：

// 类型以字段形式直接嵌入
type myLocker struct {
    sync.Mutex
}

func main() {
    var locker myLocker
    locker.Lock()
    locker.Unlock()
}
复制代码
interface 类型声明也保留它的方法集：

type myLocker sync.Locker

func main() {
    var locker myLocker
    locker.Lock()
    locker.Unlock()
}
复制代码
#46.跳出 for-switch 和 for-select 代码块
没有指定标签的 break 只会跳出 switch/select 语句，若不能使用 return 语句跳出的话，可为 break 跳出标签指定的代码块：

// break 配合 label 跳出指定代码块
func main() {
loop:
    for {
        switch {
        case true:
            fmt.Println("breaking out...")
            //break    // 死循环，一直打印 breaking out...
            break loop
        }
    }
    fmt.Println("out...")
}
复制代码
goto 虽然也能跳转到指定位置，但依旧会再次进入 for-switch，死循环。

#47.for 语句中的迭代变量与闭包函数
for 语句中的迭代变量在每次迭代中都会重用，即 for 中创建的闭包函数接收到的参数始终是同一个变量，在 goroutine 开始执行时都会得到同一个迭代值：

func main() {
    data := []string{"one", "two", "three"}

    for _, v := range data {
        go func() {
            fmt.Println(v)
        }()
    }

    time.Sleep(3 * time.Second)
    // 输出 three three three
}
复制代码
最简单的解决方法：无需修改 goroutine 函数，在 for 内部使用局部变量保存迭代值，再传参：

func main() {
    data := []string{"one", "two", "three"}

    for _, v := range data {
        vCopy := v
        go func() {
            fmt.Println(vCopy)
        }()
    }

    time.Sleep(3 * time.Second)
    // 输出 one two three
}
复制代码
另一个解决方法：直接将当前的迭代值以参数形式传递给匿名函数：

func main() {
    data := []string{"one", "two", "three"}

    for _, v := range data {
        go func(in string) {
            fmt.Println(in)
        }(v)
    }

    time.Sleep(3 * time.Second)
    // 输出 one two three
}
复制代码
注意下边这个稍复杂的 3 个示例区别：

type field struct {
    name string
}

func (p *field) print() {
    fmt.Println(p.name)
}

// 错误示例
func main() {
    data := []field{{"one"}, {"two"}, {"three"}}
    for _, v := range data {
        go v.print()
    }
    time.Sleep(3 * time.Second)
    // 输出 three three three 
}


// 正确示例
func main() {
    data := []field{{"one"}, {"two"}, {"three"}}
    for _, v := range data {
        v := v
        go v.print()
    }
    time.Sleep(3 * time.Second)
    // 输出 one two three
}

// 正确示例
func main() {
    data := []*field{{"one"}, {"two"}, {"three"}}
    for _, v := range data {    // 此时迭代值 v 是三个元素值的地址，每次 v 指向的值不同
        go v.print()
    }
    time.Sleep(3 * time.Second)
    // 输出 one two three
}
复制代码
#48.defer 函数的参数值
对 defer 延迟执行的函数，它的参数会在声明时候就会求出具体值，而不是在执行时才求值：

// 在 defer 函数中参数会提前求值
func main() {
    var i = 1
    defer fmt.Println("result: ", func() int { return i * 2 }())
    i++
}
复制代码
result: 2

#49.defer 函数的执行时机
对 defer 延迟执行的函数，会在调用它的函数结束时执行，而不是在调用它的语句块结束时执行，注意区分开。

比如在一个长时间执行的函数里，内部 for 循环中使用 defer 来清理每次迭代产生的资源调用，就会出现问题：


// 命令行参数指定目录名
// 遍历读取目录下的文件
func main() {

    if len(os.Args) != 2 {
        os.Exit(1)
    }

    dir := os.Args[1]
    start, err := os.Stat(dir)
    if err != nil || !start.IsDir() {
        os.Exit(2)
    }

    var targets []string
    filepath.Walk(dir, func(fPath string, fInfo os.FileInfo, err error) error {
        if err != nil {
            return err
        }

        if !fInfo.Mode().IsRegular() {
            return nil
        }

        targets = append(targets, fPath)
        return nil
    })

    for _, target := range targets {
        f, err := os.Open(target)
        if err != nil {
            fmt.Println("bad target:", target, "error:", err)    //error:too many open files
            break
        }
        defer f.Close()    // 在每次 for 语句块结束时，不会关闭文件资源

        // 使用 f 资源
    }
}
复制代码
先创建 10000 个文件：

#!/bin/bash
for n in {1..10000}; do
    echo content > "file${n}.txt"
done
复制代码
运行效果：

img

解决办法：defer 延迟执行的函数写入匿名函数中：

// 目录遍历正常
func main() {
    // ...

    for _, target := range targets {
        func() {
            f, err := os.Open(target)
            if err != nil {
                fmt.Println("bad target:", target, "error:", err)
                return    // 在匿名函数内使用 return 代替 break 即可
            }
            defer f.Close()    // 匿名函数执行结束，调用关闭文件资源

            // 使用 f 资源
        }()
    }
}
复制代码
当然你也可以去掉 defer，在文件资源使用完毕后，直接调用 f.Close() 来关闭。

#50.失败的类型断言
在类型断言语句中，断言失败则会返回目标类型的“零值”，断言变量与原来变量混用可能出现异常情况：

// 错误示例
func main() {
    var data interface{} = "great"

    // data 混用
    if data, ok := data.(int); ok {
        fmt.Println("[is an int], data: ", data)
    } else {
        fmt.Println("[not an int], data: ", data)    // [isn't a int], data:  0
    }
}


// 正确示例
func main() {
    var data interface{} = "great"

    if res, ok := data.(int); ok {
        fmt.Println("[is an int], data: ", res)
    } else {
        fmt.Println("[not an int], data: ", data)    // [not an int], data:  great
    }
}
复制代码
#51.阻塞的 gorutinue 与资源泄露
在 2012 年 Google I/O 大会上，Rob Pike 的 Go Concurrency Patterns 演讲讨论 Go 的几种基本并发模式，如 完整代码 中从数据集中获取第一条数据的函数：

func First(query string, replicas []Search) Result {
    c := make(chan Result)
    replicaSearch := func(i int) { c <- replicas[i](query) }
    for i := range replicas {
        go replicaSearch(i)
    }
    return <-c
}
复制代码
在搜索重复时依旧每次都起一个 goroutine 去处理，每个 goroutine 都把它的搜索结果发送到结果 channel 中，channel 中收到的第一条数据会直接返回。

返回完第一条数据后，其他 goroutine 的搜索结果怎么处理？他们自己的协程如何处理？

在 First() 中的结果 channel 是无缓冲的，这意味着只有第一个 goroutine 能返回，由于没有 receiver，其他的 goroutine 会在发送上一直阻塞。如果你大量调用，则可能造成资源泄露。

为避免泄露，你应该确保所有的 goroutine 都能正确退出，有 2 个解决方法：

使用带缓冲的 channel，确保能接收全部 goroutine 的返回结果：
func First(query string, replicas ...Search) Result {  
    c := make(chan Result,len(replicas))    
    searchReplica := func(i int) { c <- replicas[i](query) }
    for i := range replicas {
        go searchReplica(i)
    }
    return <-c
}
复制代码
使用 select 语句，配合能保存一个缓冲值的 channel default 语句： default 的缓冲 channel 保证了即使结果 channel 收不到数据，也不会阻塞 goroutine
func First(query string, replicas ...Search) Result {  
    c := make(chan Result,1)
    searchReplica := func(i int) { 
        select {
        case c <- replicas[i](query):
        default:
        }
    }
    for i := range replicas {
        go searchReplica(i)
    }
    return <-c
}
复制代码
使用特殊的废弃（cancellation） channel 来中断剩余 goroutine 的执行：
func First(query string, replicas ...Search) Result {  
    c := make(chan Result)
    done := make(chan struct{})
    defer close(done)
    searchReplica := func(i int) { 
        select {
        case c <- replicas[i](query):
        case <- done:
        }
    }
    for i := range replicas {
        go searchReplica(i)
    }

    return <-c
}
复制代码
Rob Pike 为了简化演示，没有提及演讲代码中存在的这些问题。不过对于新手来说，可能会不加思考直接使用。
52.使用指针作为方法的 receiver
只要值是可寻址的，就可以在值上直接调用指针方法。即是对一个方法，它的 receiver 是指针就足矣。

但不是所有值都是可寻址的，比如 map 类型的元素、通过 interface 引用的变量：

type data struct {
    name string
}

type printer interface {
    print()
}

func (p *data) print() {
    fmt.Println("name: ", p.name)
}

func main() {
    d1 := data{"one"}
    d1.print()    // d1 变量可寻址，可直接调用指针 receiver 的方法

    var in printer = data{"two"}
    in.print()    // 类型不匹配

    m := map[string]data{
        "x": data{"three"},
    }
    m["x"].print()    // m["x"] 是不可寻址的    // 变动频繁
}
复制代码
cannot use data literal (type data) as type printer in assignment: data does not implement printer (print method has pointer receiver)

cannot call pointer method on m["x"] cannot take the address of m["x"]

#53.更新 map 字段的值
如果 map 一个字段的值是 struct 类型，则无法直接更新该 struct 的单个字段：

// 无法直接更新 struct 的字段值
type data struct {
    name string
}

func main() {
    m := map[string]data{
        "x": {"Tom"},
    }
    m["x"].name = "Jerry"
}
复制代码
cannot assign to struct field m["x"].name in map

因为 map 中的元素是不可寻址的。需区分开的是，slice 的元素可寻址：

type data struct {
    name string
}

func main() {
    s := []data{{"Tom"}}
    s[0].name = "Jerry"
    fmt.Println(s)    // [{Jerry}]
}
复制代码
注意：不久前 gccgo 编译器可更新 map struct 元素的字段值，不过很快便修复了，官方认为是 Go1.3 的潜在特性，无需及时实现，依旧在 todo list 中。

更新 map 中 struct 元素的字段值，有 2 个方法：

使用局部变量
// 提取整个 struct 到局部变量中，修改字段值后再整个赋值
type data struct {
    name string
}

func main() {
    m := map[string]data{
        "x": {"Tom"},
    }
    r := m["x"]
    r.name = "Jerry"
    m["x"] = r
    fmt.Println(m)    // map[x:{Jerry}]
}
复制代码
使用指向元素的 map 指针
func main() {
    m := map[string]*data{
        "x": {"Tom"},
    }

    m["x"].name = "Jerry"    // 直接修改 m["x"] 中的字段
    fmt.Println(m["x"])    // &{Jerry}
}
复制代码
但是要注意下边这种误用：

func main() {
    m := map[string]*data{
        "x": {"Tom"},
    }
    m["z"].name = "what???"     
    fmt.Println(m["x"])
}
复制代码
panic: runtime error: invalid memory address or nil pointer dereference

#54.nil interface 和 nil interface 值
虽然 interface 看起来像指针类型，但它不是。interface 类型的变量只有在类型和值均为 nil 时才为 nil

如果你的 interface 变量的值是跟随其他变量变化的（雾），与 nil 比较相等时小心：

func main() {
    var data *byte
    var in interface{}

    fmt.Println(data, data == nil)    // <nil> true
    fmt.Println(in, in == nil)    // <nil> true

    in = data
    fmt.Println(in, in == nil)    // <nil> false    // data 值为 nil，但 in 值不为 nil
}
复制代码
如果你的函数返回值类型是 interface，更要小心这个坑：

// 错误示例
func main() {
    doIt := func(arg int) interface{} {
        var result *struct{} = nil
        if arg > 0 {
            result = &struct{}{}
        }
        return result
    }

    if res := doIt(-1); res != nil {
        fmt.Println("Good result: ", res)    // Good result:  <nil>
        fmt.Printf("%T\n", res)            // *struct {}    // res 不是 nil，它的值为 nil
        fmt.Printf("%v\n", res)            // <nil>
    }
}


// 正确示例
func main() {
    doIt := func(arg int) interface{} {
        var result *struct{} = nil
        if arg > 0 {
            result = &struct{}{}
        } else {
            return nil    // 明确指明返回 nil
        }
        return result
    }

    if res := doIt(-1); res != nil {
        fmt.Println("Good result: ", res)
    } else {
        fmt.Println("Bad result: ", res)    // Bad result:  <nil>
    }
}
复制代码
#55.堆栈变量
你并不总是清楚你的变量是分配到了堆还是栈。

在 C++ 中使用 new 创建的变量总是分配到堆内存上的，但在 Go 中即使使用 new()、make() 来创建变量，变量为内存分配位置依旧归 Go 编译器管。

Go 编译器会根据变量的大小及其 "escape analysis" 的结果来决定变量的存储位置，故能准确返回本地变量的地址，这在 C/C++ 中是不行的。

在 go build 或 go run 时，加入 -m 参数，能准确分析程序的变量分配位置：

img

#56.GOMAXPROCS、Concurrency（并发）and Parallelism（并行）
Go 1.4 及以下版本，程序只会使用 1 个执行上下文 / OS 线程，即任何时间都最多只有 1 个 goroutine 在执行。

Go 1.5 版本将可执行上下文的数量设置为 runtime.NumCPU() 返回的逻辑 CPU 核心数，这个数与系统实际总的 CPU 逻辑核心数是否一致，取决于你的 CPU 分配给程序的核心数，可以使用 GOMAXPROCS 环境变量或者动态的使用 runtime.GOMAXPROCS() 来调整。

误区：GOMAXPROCS 表示执行 goroutine 的 CPU 核心数，参考文档

GOMAXPROCS 的值是可以超过 CPU 的实际数量的，在 1.5 中最大为 256

func main() {
    fmt.Println(runtime.GOMAXPROCS(-1))    // 4
    fmt.Println(runtime.NumCPU())    // 4
    runtime.GOMAXPROCS(20)
    fmt.Println(runtime.GOMAXPROCS(-1))    // 20
    runtime.GOMAXPROCS(300)
    fmt.Println(runtime.GOMAXPROCS(-1))    // Go 1.9.2 // 300
}
复制代码
#57.读写操作的重新排序
Go 可能会重排一些操作的执行顺序，可以保证在一个 goroutine 中操作是顺序执行的，但不保证多 goroutine 的执行顺序：

var _ = runtime.GOMAXPROCS(3)

var a, b int

func u1() {
    a = 1
    b = 2
}

func u2() {
    a = 3
    b = 4
}

func p() {
    println(a)
    println(b)
}

func main() {
    go u1()    // 多个 goroutine 的执行顺序不定
    go u2()    
    go p()
    time.Sleep(1 * time.Second)
}
复制代码
运行效果：

img

如果你想保持多 goroutine 像代码中的那样顺序执行，可以使用 channel 或 sync 包中的锁机制等。

#58.优先调度
你的程序可能出现一个 goroutine 在运行时阻止了其他 goroutine 的运行，比如程序中有一个不让调度器运行的 for 循环：

func main() {
    done := false

    go func() {
        done = true
    }()

    for !done {
    }

    println("done !")
}
复制代码
for 的循环体不必为空，但如果代码不会触发调度器执行，将出现问题。

调度器会在 GC、Go 声明、阻塞 channel、阻塞系统调用和锁操作后再执行，也会在非内联函数调用时执行：

func main() {
    done := false

    go func() {
        done = true
    }()

    for !done {
        println("not done !")    // 并不内联执行
    }

    println("done !")
}
复制代码
可以添加 -m 参数来分析 for 代码块中调用的内联函数：

img

你也可以使用 runtime 包中的 Gosched() 来 手动启动调度器：

func main() {
    done := false

    go func() {
        done = true
    }()

    for !done {
        runtime.Gosched()
    }

    println("done !")
}
