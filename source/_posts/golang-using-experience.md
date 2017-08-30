title: golang使用感受
toc: true
date: 2017-08-29 21:05:45

categories: Go

tags: Go

---

使用Golang有一段时间了，说一下感受。
### 语法方面
1. 语法比较简洁，有点像Python，这一点比Java要好。
2. 多返回值。这一点使得go语言程序不需要像用单返回值语言写的程序那样将多个返回值封装到一个对象里，一定程度上减少了代码量。
3. 函数是一级公民，可以将函数作为参数来传递。
4. 语言级别支持并发，减少程序员心智负担。
5. 比较完善的工具包，比如net/http，用简单的几行代码便可以实现一个http server服务。
6. ……

除了上述之外还有其他好多不错的地方，比如支持交叉编译，运行速度快，占用资源少等。下面说一下在开发过程中遇到的与其他语言开发习惯不太一样的地方。

### error处理

作为一门simple language，go对error的处理也十分的简单。我们知道在Java语言里使用try-catch-finally来处理错误和异常，而C语言则以返回错误码的方式来对错误作处理。go在这方面继承了C的风格，但与C不同的是go不是用整型来作为返回值而是用error这个interface类来作为返回值。
但是go的error处理方式从一开始就成为被人吐槽的点，有的人认为go的这种处理方式太古老了，使得处理错误的代码冗长且重复，例如：

```golang
err := func1()
if err != nil {
    //handle error...
}

err = func2()
if err != nil {
    //handle error...
}

err = func3()
if err != nil {
    //handle error...
}
```
Go 作者之一，Russ Cox对于这种观点进行过驳斥：当初选择返回值这种错误处理机制而不是try-catch这种机制，主要是考虑前者适用于大型软件，后者更适合小程序。当程序变大，try-catch会让错误处理更加冗长繁琐易出错。不过Russ Cox也承认Go的错误处理机制对于开发人员的确有一定的心智负担。

所以在开发过程中对于error有以下几个trick：
- checkError


```golang
func checkError(err error) {
    if err != nil {
        fmt.Println("Error is ", err)
        panic(err)
    }
}

func echo() {
    err := func1()
    checkError(err)

    err = func2()
    checkError(err)

    err = func3()
    checkError(err)
}
```
可以在需要的地方进行recover处理，防止由于某个panic导致了整个程序退出。当然了，在开发阶段也不要忘记“让其崩溃，crash is awesome!!!”的思想。

- Errors are values

标准库里就用了这种思想，例如bufio的Writer：

```golang
b := bufio.NewWriter(fd)
    b.Write(p0[a:b])
    b.Write(p1[c:d])
    b.Write(p2[e:f])
    .....
    if b.Flush() != nil {
            return b.Flush()
        }
    }
    
```

```golang
type Writer struct {
    err error
    buf []byte
    n   int
    wr  io.Writer
}

func (b *Writer) Write(p []byte) (nn int, err error) {
    for len(p) > b.Available() && b.err == nil {
        ... ...
    }
    if b.err != nil {
        return nn, b.err
    }
    ......
    return nn, nil
}

```
error作为一个状态值封装到了Writer里面，并且在Write方法入口处对Writer进行状态检查，如果error!=nil则直接return。

- exported Error变量

在开发过程中会遇到根据不同的error来进行不同的处理，所以一种做法就是exporte error:

```goalng
var (
    ErrInvalid    = errors.New("invalid argument")
    ErrPermission = errors.New("permission denied")
    ErrExist      = errors.New("file already exists")
    ErrNotExist   = errors.New("file does not exist")
)
```
这样就可以直接使用判断是否相等的方式来做处理：

```golang
if err == os.ErrInvalid {
    //handle invalid
}
```
或者使用type switch：

```golang
switch t:=err.(type)  {
    case ErrInvalid:
        //handle invalid
    case ErrPermission:
        //handle no permission
    ... ...
}
```

- 自定义error

通过自定义error可以增加更丰富的context，例如net包里，除了实现error interface的方法外还实现了Timeout，Temporary方法，可以通过type switch和type assertion来进行类型转换：

```golang
switch  err := err.(type) {
   case *url.Error:
	  err2, ok := err.Err.(net.Error)
	  if ok && err2.Timeout() {
	  // handle timeout
	  }
	  ...
}
```

- error OR bool?

当函数失败的原因只有一个，所以返回值的类型应该为bool，而不是error。

- 错误还是异常？

什么是错误？什么是异常？什么时候使用错误？什么时候使用异常？错误和异常如何转换？
个人理解，错误是指出现的问题我们意料之中的，例如：文件打开错误，连接失败等。异常是出现了意料之外的错误，例如：空指针，下标越界等。对于错误程序可以进行自动处理，对于异常我们可以在上游进行recover，避免服务终止。

错误转异常，例如尝试请求某个URL，最多尝试三次，尝试三次的过程中请求失败是错误，尝试三次还不成功则失败就被提升为异常。

异常转错误，例如panic触发的异常被recover恢复后，对函数返回值中error类型变量进行赋值并返回，以此告诉上游本次调用发生了错误，上游程序走错误处理流程。

### 依赖包管理

go的包管理跟Java的maven不太一样，go使用GOPATH来管理依赖，无论是自己写的代码还是go get下来的代码都在一个GOPATH下面。这样就存在很大的问题：

1. 如果项目的依赖包发生了修改就可能会影响到自己的代码。
2. 无法满足GOPATH下的两个工程分别依赖同一个包的不同版本。

从go1.6版本开始正式引入了vendor机制，使得在编译时先从源码根目录下的vendor中寻找依赖，如果没找到再去GOPATH中寻找。这样可以解决上述的两个问题。但使用vendor还存在以下问题：

1. 缺少外部依赖包的版本信息，无法进行版本管理和升级。
2. 无法使用指定版本的依赖包
3. 无法列出项目所有依赖的外部包

为了解决上述问题，社区里在vendor机制基础上开发了多个包管理工具，例如：govendor,godep,glide等。go也在开发官方的[dep](https://github.com/golang/dep) 使用这些管理工具可以较好的解决上述问题。
