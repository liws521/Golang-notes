- 多核CPU的环境 -》 一些编程语言的框架在不断地提高多核资源使用效率，如Java的Netty等 -》 Go语言在多核并发上拥有原生的设计优势

## http例子
```go
package main
import "net/http" // 这个包的作用是http的基础封装和访问
func main() {
    // 使用http.FileServer文件服务器将当前目录作为根目录（“/”）的处理器，访问根目录，就会进入当前目录
    http.Handle("/", http.FileServer(http.Dir(".")))
    // 默认的HTTP服务侦听在本机8080端口
    http.ListenAndServe(":8080", nil)
}
```
- 运行代码, 在浏览器里输入 http://127.0.0.1:8080 即可浏览文件，这些文件正是当前目录在HTTP服务器上的映射目录

- Go语言不仅可以输出可执行文件，还可以编译输出能导入C语言的静态库、动态库
    - 这句的意思是不仅能生成可执行文件, 还可以生成一个库
- 同时从Go 1.7版本开始，Go语言支持将代码编译为插件。使用插件可以动态加载需要的模块，而不是一次性将所有的代码编译为一个可执行文件

## 使用Go开发的项目
- Docker
    - 网址为https://github.com/docker/docker
    - 介绍：Docker是一种操作系统层面的虚拟化技术，可以在操作系统和应用程序之间进行隔离，也可以称之为容器。Docker可以在一台物理服务器上快速运行一个或多个实例。例如，启动一个CentOS操作系统，并在其内部命令行执行指令后结束，整个过程就像自己在操作系统一样高效。
- golang
    - 网址为https://github.com/golang/go
    - 介绍：Go语言的早期源码使用C语言和汇编语言写成。从Go 1.5版本自举后，完全使用Go语言自身进行编写。Go语言的源码对了解Go语言的底层调度有极大的参考意义，建议希望对Go语言有深入了解的读者读一读
- Kubernetes项目
    - 网址为https://github.com/kubernetes/kubernetes
    - 介绍：Google公司开发的构建于Docker之上的容器调度服务，用户可以通过Kubernetes集群进行云端容器集群管理。
- etcd项目
    - 网址为https://github.com/coreos/etcd
    - 介绍：一款分布式、可靠的KV存储系统，可以快速进行云配置
- beego项目
    - 网址为https://github.com/astaxie/beego
    - 介绍：beego是一个类似Python的Tornado框架，采用了RESTFul的设计思路，使用Go语言编写的一个极轻量级、高可伸缩性和高性能的Web应用框架。
- martini项目
    - 网址为https://github.com/go-martini/martini
    - 介绍：一款快速构建模块化的Web应用的Web框架
- codis项目
    - 网址为https://github.com/CodisLabs/codis
    - 介绍：国产的优秀分布式Redis解决方案
- delve项目
    - 网址为https://github.com/derekparker/delve
    - 介绍：Go语言强大的调试器，被很多集成环境和编辑器整合。

- zh-CN修改为en-US
- 在C语言中，变量在声明时，并不会对变量对应内存区域进行清理操作。此时，变量值可能是完全不可预期的结果。
    - 开发者需要习惯在使用C语言进行声明时要初始化操作，稍有不慎，就会造成不可预知的后果。
    - 在网络上只有程序员才能看懂的“烫烫烫”和“屯屯屯”的梗，就来源于C/C++中变量默认不初始化。
    - 微软的VC编译器会将未初始化的栈空间以16进制的0xCC填充，而未初始化的堆空间使用0xCD填充，而0xCC和0xCD在中文的GB2312编码中刚好对应“烫”和“屯”字
    - 因此，如果一个字符串没有结束符“\0”，直接输出的内存数据转换为字符串就刚好对应“烫烫烫”和“屯屯屯”。

- 变量交换
    - 位运算法, a^=b, b^=a, a^=b
    - 利用Go的多重赋值特性

- 在二进制传输、读写文件的结构描述时，为了保持文件的结构不会受到不同编译目标平台字节长度的影响，不要使用int和uint

- export GO111MODULE=on
    - mac下用export

- 输出Sin函数图像
```go
package main
import (
	"math"
	"image"
	"image/color"
	"image/png"
	"os"
	"fmt"
)
func main() {
	// 图片大小
	const size = 300
	// 根据给定大小创建灰度图
	// 灰度图是一种常见的图片格式，
	// 一般情况下颜色由8位组成，灰度范围为0～255，0表示黑色，255表示白色
	// 初始化好的灰度图对象内存区域默认值都是0，对应全是黑色，
	// 考虑到显示效果和习惯，将所有像素设置为255，也就是白色
	pic := image.NewGray(image.Rect(0, 0, size, size))
	// 遍历每个像素
	for x := 0; x < size; x++ {
		for y := 0; y < size; y++ {
			// 填充为白色, 将每一个像素的灰度设为255，也就是白色
			pic.SetGray(x, y, color.Gray{255})
		}
	}
	// 图片的坐标轴原点在左上角, y轴是向下的
	for x := 0; x < size; x++ {
		s := float64(x) * 2 * math.Pi / size
		y := size / 2 - math.Sin(s) * size / 2
		pic.SetGray(x, int(y), color.Gray{0})
	}
	// 将图片写入文件
	file, err := os.Create("sin.png")
	if err != nil {
		fmt.Println(err)
	}
	png.Encode(file, pic)
	file.Close()
}
```

- 在C++、C#语言中，字符串以类的方式进行封装
    - C#语言中在使用泛型匹配约束类型时，字符串是以Class的方式存在，而不是String，因为并没有“字符串”这种原生数据类型。
    - 在C++语言中使用模板匹配类型时，为了使字符串与其他原生数据类型一样支持赋值操作，需要对字符串类进行操作符重载
    - 而Golang中的string是一种基本数据类型

- 用反引号扩起字符串, 可以跨行, 转义字符无效, 原样被赋值到string中

- UTF-8和Unicode有何区别？
    - Unicode是字符集。ASCII也是一种字符集, 字符集为每个字符分配一个唯一的ID，我们使用到的所有字符在Unicode字符集中都有唯一的一个ID对应
    - UTF-8是编码规则，将Unicode中字符的ID以某种方式进行编码

- go run -gcflags "-m -l" hello.go
- 使用go run运行程序时，-gcflags参数是编译参数。其中-m表示进行内存分配分析，-l表示避免程序内联，也就是避免进行程序优化

- 字符串不可变有很多好处，
    - 如天生线程安全，大家使用的都是只读对象，无须加锁；
    - 再者，方便内存共享，而不必使用写时复制（Copy On Write）等技术
    - 字符串hash值也只需要制作一份

- Go中也有StringBuilder类似的东西, bytes.Buffer, 比+拼接更加高效

- C, int* a = (int*)malloc(10)
- C++, int * b = new int(4)

- error是一个接口类型, 实现了Error() string即可实现该接口
- var err = errors.New("string")
- 还可以自定义包含复杂信息的struct, 实现error接口, 在绑定的Error()方法中输出想要的格式

- panic(), 手动触发panic, 处理defer栈中的信息, 然后over

- export GOPATH=`pwd`, 反引号把指令执行的结果返回