# [返回](/)

> 参考资料：[Go圣经](https://www.k8stech.net/gopl/chapter0/)

# 包和构建

- 所有导入的包必须在每个文件的开头显式声明，这样的话编译器就没有必要读取和分析整个源文件来判断包的依赖关系
- 禁止包的环状依赖，因为没有循环依赖，包的依赖关系形成一个有向无环图，每个包可以被独立编译，而且很可能是被并发编译
- 编译后包的目标文件不仅仅记录包本身的导出信息，目标文件同时还记录了包的依赖关系。因此，在编译一个包的时候，编译器只需要读取每个直接导入包的目标文件，而不需要遍历所有依赖的的文件（很多都是重复的间接依赖） 

**重命名避免冲突**

```go
import (
	"crypto/rand"
	mrand "math/rand" // 包重命名
)
```

**匿名导入**

如果只是导入一个包而并不使用导入的包将会导致一个编译错误。但是有时候我们只是想利用导入包而产生的其他作用：它会计算包级变量的初始化表达式和执行导入包的init初始化函数。这时候我们需要抑制“unused import”编译错误，我们可以用下划线`_`来重命名导入的包。像往常一样，下划线`_`为空白标识符，并不能被访问

```go
import _ "image/png" // register PNG decoder
```

**命名规范**

-  一般要用短小的包名 
- 尽可能让命名有描述性且无歧义
- 一般采用单数的形式
- 避免包名有其它的含义

## 构建

goroot/gopath/gomodule

### gopath

对于大多数的Go语言用户，只需要配置一个名叫GOPATH的环境变量，用来指定当前工作目录即可。当需要切换到不同工作区的时候，只要更新GOPATH就可以了

> 建议随项目设置gopath，尽量不要使用全局的

```bash
$ export GOPATH=$HOME/gobook
$ go get gopl.io/...
```

举例，工作区的目录结构可能是这样的：

```bash
GOPATH/
	src/
		gopl.io/
			.git/
			ch1/
				helloworld/
					main.go
				dup/
					main.go
				...
		golang.org/x/net/
			.git/
			html/
				parse.go
				node.go
				...
	bin/
		helloworld
		dup
	pkg/
		darwin_amd64/
		...
```

GOPATH对应的工作区目录有三个子目录

- src子目录用于存储源代码。每个包被保存在与$GOPATH/src的相对路径为包导入路径的子目录中，例如gopl.io/ch1/helloworld相对应的路径目录。我们看到，一个GOPATH工作区的src目录中可能有多个独立的版本控制系统，例如gopl.io和golang.org分别对应不同的Git仓库
- pkg子目录用于保存编译后的包的目标文件
- bin子目录用于保存编译后的可执行程序，例如helloworld可执行程序

### goroot

GOROOT用来指定Go的安装目录，还有它自带的标准库包的位置。GOROOT的目录结构和GOPATH类似，因此存放fmt包的源代码对应目录应该为$GOROOT/src/fmt。用户一般不需要设置GOROOT，默认情况下Go语言安装工具会将其设置为安装的目录路径

其中`go env`命令用于查看Go语言工具涉及的所有环境变量的值，包括未设置环境变量的默认值。GOOS环境变量用于指定目标操作系统（例如android、linux、darwin或windows），GOARCH环境变量用于指定处理器的类型，例如amd64、386或arm等。虽然GOPATH环境变量是唯一必须要设置的，但是其它环境变量也会偶尔用到

```bash
$ go env
GOPATH="/home/gopher/gobook"
GOROOT="/usr/local/go"
GOARCH="amd64"
GOOS="darwin"
...
```

`go env -w`可以修改

### gomodule

**为什么弃用 GOPATH 模式**

在 GOPATH 的 $GOPATH/src 下进行 .go 文件或源代码的存储，我们可以称其为 GOPATH 的模式，这个模式，看起来好像没有什么问题，那么为什么我们要弃用呢，参见如下原因：

- 在执行go get的时候，你无法传达任何的版本信息的期望，也就是说你也无法知道自己当前更新的是哪一个版本，也无法通过指定来拉取自己所期望的具体版本
- 在运行 Go 应用程序的时候，你无法保证其它人与你所期望依赖的第三方库是相同的版本，也就是说在项目依赖库的管理上，你无法保证所有人的依赖版本都一致
- 没办法处理 v1、v2、v3 等等不同版本的引用问题，因为 GOPATH 模式下的导入路径都是一样的，都是`github.com/foo/bar`

在 Go modules 中，我们能够使用如下命令进行操作：

> go mod init 生成 go.mod 文件
> go mod download 下载 go.mod 文件中指明的所有依赖
> go mod tidy 整理现有的依赖
> go mod graph 查看现有的依赖结构
> go mod edit 编辑 go.mod 文件
> go mod vendor 导出项目所有的依赖到vendor目录
> go mod verify 校验一个模块是否被篡改过
> go mod why 查看为什么需要依赖某模块

所提供的环境变量
在 Go modules 中有如下常用环境变量，我们可以通过 go env 命令来进行查看，如下：

```go
$ go env
GO111MODULE="auto"
GOPROXY="https://proxy.golang.org,direct"
GONOPROXY=""
GOSUMDB="sum.golang.org"
GONOSUMDB=""
GOPRIVATE=""
...
```

#### GO111MODULE

Go语言提供了 GO111MODULE 这个环境变量来作为 Go modules 的开关，其允许设置以下参数：

auto：只要项目包含了 go.mod 文件的话且在gopath外部，启用 Go modules，目前在 Go1.11 至 Go1.14 中仍然是默认值

- on：启用 Go modules，推荐设置，将会是未来版本中的默认值
- off：禁用 Go modules，不推荐设置

#### GOPROXY

这个环境变量主要是用于设置 Go 模块代理（Go module proxy），其作用是用于使 Go 在后续拉取模块版本时能够脱离传统的 VCS 方式，直接通过镜像站点来快速拉取

GOPROXY 的默认值是：`https://proxy.golang.org,direct`，这有一个很严重的问题，就是 proxy.golang.org 在国内是无法访问的，因此这会直接卡住你的第一步，所以你必须在开启 Go modules 的时，同时设置国内的 Go 模块代理，执行如下命令

> $ go env -w GOPROXY=https://goproxy.cn,direct
> GOPROXY 的值是一个以英文逗号 “,” 分割的 Go 模块代理列表，允许设置多个模块代理，假设你不想使用，也可以将其设置为 “off” ，这将会禁止 Go 在后续操作中使用任何 Go 模块代理

**direct是什么**

而在刚刚设置的值中，我们可以发现值列表中有 “direct” 标识，它又有什么作用呢？

实际上 “direct” 是一个特殊指示符，用于指示 Go 回源到模块版本的源地址去抓取（比如 GitHub 等），场景如下：当值列表中上一个 Go 模块代理返回 404 或 410 错误时，Go 自动尝试列表中的下一个，遇见 “direct” 时回源，也就是回到源地址去抓取，而遇见 EOF 时终止并抛出类似 “invalid version: unknown revision…” 的错误

#### GOSUMDB

它的值是一个 Go checksum database，用于在拉取模块版本时（无论是从源站拉取还是通过 Go module proxy 拉取）保证拉取到的模块版本数据未经过篡改，若发现不一致，也就是可能存在篡改，将会立即中止

GOSUMDB 的默认值为：sum.golang.org，在国内也是无法访问的，但是 GOSUMDB 可以被 Go 模块代理所代理（详见：Proxying a Checksum Database）

因此我们可以通过设置 GOPROXY 来解决，而先前我们所设置的模块代理 goproxy.cn 就能支持代理 sum.golang.org，所以这一个问题在设置 GOPROXY 后，你可以不需要过度关心

也可以将其设置为“off”，也就是禁止 Go 在后续操作中校验模块版本

#### GONOPROXY/GONOSUMDB/GOPRIVATE

这三个环境变量都是用在当前项目依赖了私有模块，例如像是你公司的私有 git 仓库，又或是 github 中的私有库，都是属于私有模块，都是要进行设置的，否则会拉取失败

更细致来讲，就是依赖了由 GOPROXY 指定的 Go 模块代理或由 GOSUMDB 指定 Go checksum database 都无法访问到的模块时的场景

而一般建议直接设置 GOPRIVATE，它的值将作为 GONOPROXY 和 GONOSUMDB 的默认值，所以建议的最佳姿势是直接使用 GOPRIVATE

并且它们的值都是一个以英文逗号 “,” 分割的模块路径前缀，也就是可以设置多个，例如：

```go
$ go env -w GOPRIVATE="git.example.com,github.com/eddycjy/mquote"
```

设置后，前缀为 git.xxx.com 和 github.com/eddycjy/mquote 的模块都会被认为是私有模块

如果不想每次都重新设置，我们也可以利用通配符，例如：

```go
$ go env -w GOPRIVATE="*.example.com"
```

这样子设置的话，所有模块路径为 example.com 的子域名（例如：git.example.com）都将不经过 Go module proxy 和 Go checksum database，需要注意的是不包括 example.com 本身

#### 使用 Go Modules

目前 Go modules 并不是默认开启，因此Go语言提供了 GO111MODULE 这个环境变量来作为 Go modules 的开关，其允许设置以下参数：

- auto：只要项目包含了 go.mod 文件的话启用 Go modules，目前在Go1.11至 Go1.14 中仍然是默认值。
- on：启用 Go modules，推荐设置，将会是未来版本中的默认值。
- off：禁用 Go modules，不推荐设置。

**初始化项目**

在完成 Go modules 的开启后，我们需要创建一个示例项目来进行演示，执行如下命令：

```go
$ mkdir -p $HOME/eddycjy/module-repo 
$ cd $HOME/eddycjy/module-repo
```

然后进行 Go modules 的初始化，如下：

```go
$ go mod init github.com/eddycjy/module-repo
go: creating new go.mod: module github.com/eddycjy/module-repo
12
```

在执行 `go mod init`命令时，我们指定了模块导入路径为 `github.com/eddycjy/module-repo`。接下来我们在该项目根目录下创建 main.go 文件，如下：

```go
package main
 
import (
    "fmt"
    "github.com/eddycjy/mquote"
)
 
 
func main() {
	fmt.Println(mquote.GetHello())
}
```

然后在项目根目录执行 `go get github.com/eddycjy/mquote` 命令，如下：

```go
$ go get github.com/eddycjy/mquote 
go: finding github.com/eddycjy/mquote latest
go: downloading github.com/eddycjy/mquote v0.0.0-20200220041913-e066a990ce6f
go: extracting github.com/eddycjy/mquote v0.0.0-20200220041913-e066a990ce6f
```

**查看 go.mod 文件**

在初始化项目时，会生成一个 go.mod 文件，是启用了 Go modules 项目所必须的最重要的标识，同时也是 GO111MODULE 值为 auto 时的识别标识，它描述了当前项目（也就是当前模块）的元信息，每一行都以一个动词开头

在我们刚刚进行了初始化和简单拉取后，我们再次查看 go.mod 文件，基本内容如下：

```go
module github.com/eddycjy/module-repo
 
go 1.13
 
require (
	github.com/eddycjy/mquote v0.0.0-20200220041913-e066a990ce6f
)
```

为了更进一步的讲解，我们模拟引用如下：

```go
module github.com/eddycjy/module-repo

go 1.13
 
require (
    example.com/apple v0.1.2
    example.com/banana v1.2.3
    example.com/banana/v2 v2.3.4
    example.com/pear // indirect
    example.com/strawberry // incompatible
)
 
 
exclude example.com/banana v1.2.4
replace example.com/apple v0.1.2 => example.com/fried v0.1.0 
replace example.com/banana => example.com/fish
```

- module：用于定义当前项目的模块路径
- go：用于标识当前模块的 Go 语言版本，值为初始化模块时的版本，目前来看还只是个标识作用
- require：用于设置一个特定的模块版本
- exclude：用于从使用中排除一个特定的模块版本
- replace：用于将一个模块版本替换为另外一个模块版本

另外你会发现 `example.com/pear` 的后面会有一个 indirect 标识，
indirect 标识表示该模块为间接依赖，也就是在当前应用程序中的 import 语句中，并没有发现这个模块的明确引用，有可能是你先手动 go get 拉取下来的，也有可能是你所依赖的模块所依赖的，情况有好几种

**查看 go.sum 文件**

在第一次拉取模块依赖后，会发现多出了一个 go.sum 文件，其详细罗列了当前项目直接或间接依赖的所有模块版本，并写明了那些模块版本的 SHA-256 哈希值以备 Go 在今后的操作中保证项目所依赖的那些模块版本不会被篡改

**查看全局缓存**

我们刚刚成功的将 github.com/eddycjy/mquote 模块拉取了下来，其拉取的结果缓存在 GOPATH/pkg/mod和 ​GOPATH/pkg/sumdb 目录下，而在mod目录下会以 github.com/foo/bar 的格式进行存放，如下：

```
mod
├── cache
├── github.com
├── golang.org
├── google.golang.org
├── gopkg.in
...
```

需要注意的是同一个模块版本的数据只缓存一份，所有其它模块共享使用。如果你希望清理所有已缓存的模块版本数据，可以执行 `go clean -modcache`

#### **Go Modules 下的 go get 行为**

在拉取项目依赖时，你会发现拉取的过程总共分为了三大步，分别是 finding（发现）、downloading（下载）以及 extracting（提取）

- go get 拉取依赖，会进行指定性拉取（更新），并不会更新所依赖的其它模块
- go get -u 更新现有的依赖，会强制更新它所依赖的其它全部模块，不包括自身
- go get -u -t ./… 更新所有直接依赖和间接依赖的模块版本，包括单元测试中用到的
  那么我想选择具体版本应当如何执行呢，如下：

| 命令                            | 作用                                                  |
| ------------------------------- | ----------------------------------------------------- |
| go get golang.org/x/text@latest | 拉取最新的版本，若存在tag，则优先使用                 |
| go get golang.org/x/text@master | 拉取 master 分支的最新 commit                         |
| go get golang.org/x/text@v0.3.2 | 拉取 tag 为 v0.3.2 的 commit                          |
| go get golang.org/x/text@342b2e | 拉取 hash 为 342b231 的 commit，最终会被转换为 v0.3.2 |

**go get 的版本选择**

我们回顾一下我们拉取的 go get github.com/eddycjy/mquote，其结果是 v0.0.0-20200220041913-e066a990ce6f，对照着上面所提到的 go get 行为来看，你可能还会有一些疑惑，那就是在 go get 没有指定任何版本的情况下，它的版本选择规则是怎么样的，也就是为什么 go get 拉取的是 v0.0.0，它什么时候会拉取正常带版本号的 tags 呢。实际上这需要区分两种情况，如下：

- 所拉取的模块有发布 tags：
  - 如果只有单个模块，那么就取主版本号最大的那个tag
  - 如果有多个模块，则推算相应的模块路径，取主版本号最大的那个tag（子模块的tag的模块路径会有前缀要求）
- 所拉取的模块没有发布过 tags：
  - 默认取主分支最新一次 commit 的 commithash。

#### Go Modules 的导入路径说明

### 不同版本的导入路径

在前面的模块拉取和引用中，你会发现我们的模块导入路径就是 github.com/eddycjy/mquote 和 github.com/eddycjy/mquote/module/tour，似乎并没有什么特殊的。

其实不然，实际上 Go modules 在主版本号为 v0 和 v1 的情况下省略了版本号，而在主版本号为v2及以上则需要明确指定出主版本号，否则会出现冲突，其tag与模块导入路径的大致对应关系如下：

tag 模块导入路径

```
v0.0.0	github.com/eddycjy/mquote
v1.0.0	github.com/eddycjy/mquote
v2.0.0	github.com/eddycjy/mquote/v2
v3.0.0	github.com/eddycjy/mquote/v3
```

简单来讲，就是主版本号为 v0 和 v1 时，不需要在模块导入路径包含主版本的信息，而在 v1 版本以后，也就是 v2 起，必须要在模块的导入路径末尾加上主版本号，引用时就需要调整为如下格式：

```go
import (
    "github.com/eddycjy/mquote/v2/example"
)
123
```

另外忽略主版本号 v0 和 v1 是强制性的（不是可选项），因此每个软件包只有一个明确且规范的导入路径

**为什么忽略 v0 和 v1 的主版本号**

导入路径中忽略 v1 版本的原因是：考虑到许多开发人员创建一旦到达 v1 版本便永不改变的软件包，这是官方所鼓励的，不认为所有这些开发人员在无意发布 v2 版时都应被迫拥有明确的 v1 版本尾缀，这将导致 v1 版本变成“噪音”且无意义

导入路径中忽略了 v0 版本的原因是：根据语义化版本规范，v0的这些版本完全没有兼容性保证。需要一个显式的 v0 版本的标识对确保兼容性没有多大帮助

[更多](https://blog.csdn.net/u011069013/article/details/110114319)

## 包管理

### 下载包

使用命令`go get`可以下载一个单一的包或者用`...`下载整个子目录里面的每个包。Go语言工具箱的go命令同时计算并下载所依赖的每个包，这也是前一个例子中golang.org/x/net/html自动出现在本地工作区目录的原因。

一旦`go get`命令下载了包，然后就是安装包或包对应的可执行的程序。我们将在下一节再关注它的细节，现在只是展示整个下载过程是如何的简单。第一个命令是获取golint工具，它用于检测Go源代码的编程风格是否有问题。第二个命令是用golint命令对2.6.2节的gopl.io/ch2/popcount包代码进行编码风格检查。它友好地报告了忘记了包的文档：

```bash
$ go get github.com/golang/lint/golint
$ $GOPATH/bin/golint gopl.io/ch2/popcount
src/gopl.io/ch2/popcount/main.go:1:1:
  package comment should be of the form "Package popcount ..."
```

`go get`命令支持当前流行的托管网站GitHub、Bitbucket和Launchpad，可以直接向它们的版本控制系统请求代码。对于其它的网站，你可能需要指定版本控制系统的具体路径和协议，例如 Git或Mercurial。运行`go help importpath`获取相关的信息

`go get`命令获取的代码是真实的本地存储仓库，而不仅仅只是复制源文件，因此你依然可以使用版本管理工具比较本地代码的变更或者切换到其它的版本。例如golang.org/x/net包目录对应一个Git仓库：

```bash
$ cd $GOPATH/src/golang.org/x/net
$ git remote -v
origin  https://go.googlesource.com/net (fetch)
origin  https://go.googlesource.com/net (push)
```

需要注意的是导入路径含有的网站域名和本地Git仓库对应远程服务地址并不相同，真实的Git地址是go.googlesource.com。这其实是Go语言工具的一个特性，可以让包用一个自定义的导入路径，但是真实的代码却是由更通用的服务提供，例如googlesource.com或github.com。因为页面 https://golang.org/x/net/html 包含了如下的元数据，它告诉Go语言的工具当前包真实的Git仓库托管地址：

```bash
$ go build gopl.io/ch1/fetch
$ ./fetch https://golang.org/x/net/html | grep go-import
<meta name="go-import"
      content="golang.org/x/net git https://go.googlesource.com/net">
```

如果指定`-u`命令行标志参数，`go get`命令将确保所有的包和依赖的包的版本都是最新的，然后重新编译和安装它们。如果不包含该标志参数的话，而且如果包已经在本地存在，那么代码将不会被自动更新

`go get -u`命令只是简单地保证每个包是最新版本，如果是第一次下载包则是比较方便的；但是对于发布程序则可能是不合适的，因为本地程序可能需要对依赖的包做精确的版本依赖管理。通常的解决方案是使用vendor的目录用于存储依赖包的固定版本的源代码，对本地依赖的包的版本更新也是谨慎和持续可控的。在Go1.5之前，一般需要修改包的导入路径，所以复制后golang.org/x/net/html导入路径可能会变为gopl.io/vendor/golang.org/x/net/html。最新的Go语言命令已经支持vendor特性，但限于篇幅这里并不讨论vendor的具体细节。不过可以通过`go help gopath`命令查看Vendor的帮助文档

### 构建包

`go build`命令编译命令行参数指定的每个包

也可以偷懒，直接`go run *.go`

`go install`命令和`go build`命令很相似，但是它会保存每个包的编译成果，而不是将它们都丢弃。被编译的包会被保存到GOPATH/pkg目录下，目录路径和 src目录路径对应，可执行程序被保存到GOPATH/bin目录

### 内部包

在Go语言程序中，包是最重要的封装机制。没有导出的标识符只在同一个包内部可以访问，而导出的标识符则是面向全宇宙都是可见的

有时候，一个中间的状态可能也是有用的，标识符对于一小部分信任的包是可见的，但并不是对所有调用者都可见。例如，当我们计划将一个大的包拆分为很多小的更容易维护的子包，但是我们并不想将内部的子包结构也完全暴露出去。同时，我们可能还希望在内部子包之间共享一些通用的处理包，或者我们只是想实验一个新包的还并不稳定的接口，暂时只暴露给一些受限制的用户使用

为了满足这些需求，Go语言的构建工具对包含internal名字的路径段的包导入路径做了特殊处理。这种包叫internal包，一个internal包只能被和internal目录有同一个父目录的包所导入。例如，net/http/internal/chunked内部包只能被net/http/httputil或net/http包导入，但是不能被net/url包导入。不过net/url包却可以导入net/http/httputil包

```go
net/http
net/http/internal/chunked
net/http/httputil
net/url
```

### 查询包

`go list`命令可以查询可用包的信息。其最简单的形式，可以测试包是否在工作区并打印它的导入路径：

```bash
$ go list github.com/go-sql-driver/mysql
github.com/go-sql-driver/mysql
```

`go list`命令的参数还可以用`"..."`表示匹配任意的包的导入路径。我们可以用它来列出工作区中的所有包：

```bash
$ go list ...
archive/tar
archive/zip
bufio
bytes
cmd/addr2line
cmd/api
...many more...
```

或者是特定子目录下的所有包：

```bash
$ go list gopl.io/ch3/...
gopl.io/ch3/basename1
gopl.io/ch3/basename2
gopl.io/ch3/comma
gopl.io/ch3/mandelbrot
gopl.io/ch3/netflag
gopl.io/ch3/printints
gopl.io/ch3/surface
```

或者是和某个主题相关的所有包:

```bash
$ go list ...xml...
encoding/xml
gopl.io/ch7/xmlselect
```

`go list`命令还可以获取每个包完整的元信息，而不仅仅只是导入路径，这些元信息可以以不同格式提供给用户。其中`-json`命令行参数表示用JSON格式打印每个包的元信息。

```bash
$ go list -json hash
{
	"Dir": "/home/gopher/go/src/hash",
	"ImportPath": "hash",
	"Name": "hash",
	"Doc": "Package hash provides interfaces for hash functions.",
	"Target": "/home/gopher/go/pkg/darwin_amd64/hash.a",
	"Goroot": true,
	"Standard": true,
	"Root": "/home/gopher/go",
	"GoFiles": [
			"hash.go"
	],
	"Imports": [
		"io"
	],
	"Deps": [
		"errors",
		"io",
		"runtime",
		"sync",
		"sync/atomic",
		"unsafe"
	]
}
```

# go语法

## 基本语法

### 基础

编译型、强类型（隐式类型）、动态类型

- 非驼峰命名，首字母大写

- 类型声明在后

- 短变量声明：`a := 1`，只能用在函数内

- 命令行参数：os包，os.Args，是一个切片。[flag包相关](https://www.k8stech.net/gopl/chapter2/ch2-03/)

- 返回值可命名

  ```go
  func foo(x int) int{
      return x + 1
  }
  
  // 命名返回值
  func foo(x int) (ret int){
      ret = x + 1
      // 可以直接返回，返回的是ret
      return
  }
  ```

- 基本数据类型

  ```go
  bool
  
  string
  
  int  int8  int16  int32  int64
  uint uint8 uint16 uint32 uint64 uintptr
  
  byte // uint8 的别名
  
  rune // int32 的别名
      // 表示一个 Unicode 码点
  
  float32 float64
  
  complex64 complex128
  ```

  **rune**

  - 7 bits ASCII → 固定32 bits Unicode → Unicode标准形式，变长的UTF-8

  - rune代表一个UTF-8码点

    - “unicode/utf8”：utf8.RuneCountInString(s)，数utf-8字符数

    - 解码器

      ```go
      for i := 0; i < len(s); {
      	r, size := utf8.DecodeRuneInString(s[i:])
      	fmt.Printf("%d\t%c\n", i, r)
      	i += size
      }
      ```

    - 更简洁的方式：range，会自动解码对应长度并令索引跳转相应长度

      ```go
      for i, r := range "Hello, 世界" {
      	fmt.Printf("%d\t%q\t%d\n", i, r, r)
      }
      ```

- defer关键字：推迟执行，defer的函数先完成求值，但直到外层函数返回时才会被调用

  > panic之后也会调用
  >
  > 在Go的panic机制中，延迟函数的调用在释放堆栈信息之前

  ```go
  package main
  
  import "fmt"
  
  func main() {
  	defer fmt.Println("world")
  
  	fmt.Println("hello")
  }
  // answer is “hello world”
  ```

  本质是压入函数栈

  ```go
  package main
  
  import "fmt"
  
  func main() {
  	fmt.Println("counting")
  
  	for i := 0; i < 10; i++ {
  		defer fmt.Println(i)
  	}
  
  	fmt.Println("done")
  }
  // 9 8 7 6 5 4 3 2 1 done
  ```

- 一个原生的字符串面值形式是，使用反引号``代替双引号。在原生的字符串面值中，没有转义操作；全部的内容都是字面的意思，包含退格和换行，因此一个程序中的原生字符串面值可能跨越多行

  > 在原生字符串面值内部是无法直接写字符的，可以用八进制或十六进制转义或连接字符串常量完成）。唯一的特殊处理是会删除回车以保证在所有平台上的值都是一样的，包括那些把回车也放入文本文件的系统（Windows系统会把回车和换行一起放入文本文件中）

### 简单io

> - [https://blog.csdn.net/wohu1104/article/details/106433545](https://blog.csdn.net/wohu1104/article/details/106433545)
> - [https://blog.csdn.net/qq_25100257/article/details/120937548](https://blog.csdn.net/qq_25100257/article/details/120937548)

- fmt：Scanf、Sprintf等等

- bufio包：
  - `input := bufio.NewScanner(os.Stdin)`
    - `input.Scan()，读下一行，返回是否有下一行`
      - `input.Text()，显示读取内容`

  ```go
  func main() {
  	cache := make(map[string]int)
  	f, err := os.Open("test.txt")
  	if err != nil {
  		return
  	}
  	input := bufio.NewScanner(f)
  	input.Split(bufio.ScanWords)  // 按单词读取，而不是按行
  	for {
  		if !input.Scan() {
  			break
  		}
  		word := input.Text()
  		cache[word]++
  	}
  	fmt.Println("word\tcount")
  	for w, c := range cache {
  		fmt.Printf("%v\t%v\n", w, c)
  	}
  }
  ```

- `f, err := os.Open(fileName)`

- `data, err := ioutil.ReadFile(filename)`：一次性读取全部文件

### 格式化

```go
%d          十进制整数
%x, %o, %b  十六进制，八进制，二进制整数。
%f, %g, %e  浮点数： 3.141593 3.141592653589793 3.141593e+00
%t          布尔：true或false
%c          字符（rune） (Unicode码点)
%s          字符串
%q          带双引号的字符串"abc"或带单引号的字符'c'
%v          变量的自然形式（natural format）
%T          变量的类型
%%          字面上的百分号标志（无操作数）
```

### 常量

- iota常量生成器

  常量声明可以使用iota常量生成器初始化，它用于生成一组以相似规则初始化的常量，但是不用每行都写一遍初始化表达式。在一个const声明语句中，在第一个声明的常量所在的行，iota将会被置为0，然后在每一个有常量声明的行给iota加1

  ```go
  const (
  	_ = 1 << (10 * iota)
  	KiB // 1024
  	MiB // 1048576
  	GiB // 1073741824
  	TiB // 1099511627776             (exceeds 1 << 32)
  	PiB // 1125899906842624
  	EiB // 1152921504606846976
  	ZiB // 1180591620717411303424    (exceeds 1 << 64)
  	YiB // 1208925819614629174706176
  )
  ```

- 无类型常量
  - 一个常量可以有任意一个确定的基础类型，例如int或float64，或者是类似time.Duration这样命名的基础类型，但是许多常量并没有一个明确的基础类型
  - 编译器为这些没有明确基础类型的数字常量提供比基础类型更高精度的算术运算；你可以认为至少有256bit的运算精度
  - 有六种未明确类型的常量类型，分别是无类型的布尔型、无类型的整数、无类型的字符、无类型的浮点数、无类型的复数、无类型的字符串

  通过延迟明确常量的具体类型，无类型的常量不仅可以提供更高的运算精度，而且可以直接用于更多的表达式而不需要显式的类型转换。例如，上面例子中的ZiB和YiB的值已经超出任何Go语言中整数类型能表达的范围，但是它们依然是合法的常量，而且像下面的常量表达式依然有效（译注：YiB/ZiB是在编译期计算出来的，并且结果常量是1024，是Go语言int变量能有效表示的）：

  ```go
  fmt.Println(YiB/ZiB) // "1024"
  ```

  另一个例子，math.Pi无类型的浮点数常量，可以直接用于任意需要浮点数或复数的地方：

  ```go
  var x float32 = math.Pi
  var y float64 = math.Pi
  var z complex128 = math.Pi
  ```

  只有常量可以是无类型的。当一个无类型的常量被赋值给一个变量的时候，就像下面的第一行语句，或者出现在有明确类型的变量声明的右边，如下面的其余三行语句，无类型的常量将会被隐式转换为对应的类型，如果转换合法的话

  ```go
  var f float64 = 3 + 0i // untyped complex -> float64
  f = 2                  // untyped integer -> float64
  f = 1e123              // untyped floating-point -> float64
  f = 'a'                // untyped rune -> float64
  ```

  上面的语句相当于:

  ```go
  var f float64 = float64(3 + 0i)
  f = float64(2)
  f = float64(1e123)
  f = float64('a')
  ```

  无类型整数常量转换为int，它的内存大小是不确定的，但是无类型浮点数和复数常量则转换为内存大小明确的float64和complex128

  > 如果不知道浮点数类型的内存大小是很难写出正确的数值算法的，因此Go语言不存在整型类似的不确定内存大小的浮点数和复数类型

## 特殊结构

### 指针

指针存放数据的地址，可通过指针对数据进行访问&修改

```go
package main

import "fmt"

func main() {
	i, j := 42, 2701

	p := &i         // 指向 i
	fmt.Println(*p) // 通过指针读取 i 的值
	*p = 21         // 通过指针设置 i 的值
	fmt.Println(i)  // 查看 i 的值

	p = &j         // 指向 j
	*p = *p / 37   // 通过指针对 j 进行除法运算
	fmt.Println(j) // 查看 j 的值
}
```

可看作**“间接引用”或“重定向”**

> Java引用赋新值是指向了另一个地方，地址值改变
>
> Go的指针赋新值是指针所指的地址处值的改变，指针指向的地址没有变**（p没有变，*p改变）**

调用内建的new函数。表达式new(T)将创建一个T类型的匿名变量，初始化为T类型的零值，然后返回变量地址，返回的指针类型为`*T`

如果两个类型都是空的，也就是说类型的大小是0，例如`struct{}`和`[0]int`，有可能有相同的地址（依赖具体的语言实现）

> 请谨慎使用大小为0的类型，因为如果类型的大小为0的话，可能导致Go语言的自动垃圾回收器有不同的行为，具体请查看`runtime.SetFinalizer`函数相关文档

### 结构体

一组字段（field）

```go
package main

import "fmt"

type Vertex struct {
	X int
	Y int
}

func main() {
	fmt.Println(Vertex{1, 2})
}

```

> 如果我们有一个指向结构体的指针 `p`，那么可以通过 `(*p).X` 来访问其字段 `X`。不过这么写太啰嗦了，所以语言也允许我们使用隐式间接引用，直接写 `p.X` 就可以

结构体成员名字是以大写字母开头的，那么该成员就是导出的；这是Go语言导出规则决定的。一个结构体可能同时包含导出和未导出的成员。**但如果struct嵌套了，那么即使被嵌套在内部的struct名称首字母小写，其他包也能访问到它里面首字母大写的字段**

结构体全体成员都是用==可比较的，那么结构体也可比较，比较的是每个成员的值*（不同于 Java比较地址值）*

> 匿名成员语法糖：匿名的嵌套结构体，访问时可以直接访问成员
>
> 例如：
>
> ```go
> // Charcount computes counts of Unicode characters.
> package main
> 
> import (
> 	"fmt"
> )
> 
> type S struct {
> 	X, Y int
> 	SS
> }
> 
> type SS struct {
> 	Z int
> }
> 
> func main() {
> 	t := S{1, 2, SS{3}}
> 	t.Z = 33
> 	fmt.Println(t)
> }
> ```

**赋值**

```go
package main

import "fmt"

type Vertex struct {
	X, Y int
}

var (
	v1 = Vertex{1, 2}  // 创建一个 Vertex 类型的结构体
	v2 = Vertex{X: 1}  // Y:0 被隐式地赋予
	v3 = Vertex{}      // X:0 Y:0
	p  = &Vertex{1, 2} // 创建一个 *Vertex 类型的结构体（指针）
)

func main() {
	fmt.Println(v1, p, v2, v3)
}

```

[导出和暴露问题](https://blog.csdn.net/weixin_33709219/article/details/86023269)

### 数组和切片

#### 数组

```go
package main

import "fmt"

func main() {
	var a [2]string
	a[0] = "Hello"
	a[1] = "World"
	fmt.Println(a[0], a[1])
	fmt.Println(a)

	primes := [6]int{2, 3, 5, 7, 11, 13}
	fmt.Println(primes)
}
```

#### 切片

[https://www.k8stech.net/gopl/chapter4/ch4-02/](https://www.k8stech.net/gopl/chapter4/ch4-02/)

```go
package main

import "fmt"

func main() {
	primes := [6]int{2, 3, 5, 7, 11, 13}

	var s []int = primes[1:4]
	fmt.Println(s)
}
```

> 就像数组这一段的引用，并不实际存储这一段为另一个数组

- 切片文法类似于没有长度的数组文法
  - 这是一个数组文法：

    ```go
    [3]bool{true, true, false}
    ```

  - 下面这样则会创建一个和上面相同的数组，然后**构建一个引用了它的切片**：

    ```go
    []bool{true, true, false}
    ```

- 切片的长度和容量可通过表达式 `len(s)` 和 `cap(s)` 来获取，len就是切片长度，cap是从第一个开始数，数到数组末尾*（从0开始的切片，cap都为数组长度）*

- 切片的零值是 `nil`，nil 切片的长度和容量为 0 且没有底层数组

- 和数组不同的是，slice之间不能比较，因此我们不能使用==操作符来判断两个slice是否含有全部相等元素*（除了bytes.Equal函数来判断两个字节型 slice）*

  > [https://www.k8stech.net/gopl/chapter4/ch4-02/](https://www.k8stech.net/gopl/chapter4/ch4-02/)

**make**

切片可以用内建函数 `make` 来创建，也是创建动态数组的方式

`make` 函数会分配一个元素为零值的数组并返回一个引用了它的切片：

```
a := make([]int, 5)  // len(a)=5
```

要指定它的容量，需向 `make` 传入第三个参数：

```
b := make([]int, 0, 5) // len(b)=0, cap(b)=5

b = b[:cap(b)] // len(b)=5, cap(b)=5
b = b[1:]      // len(b)=4, cap(b)=4
```

**append**

向切片追加元素，底层数组大小不够时会扩容

> 可以追加切片，写成`append(x, y...)`的形式

**range**

`for` 循环的 `range` 形式可遍历切片或映射

当使用 `for` 循环遍历切片时，每次迭代都会返回两个值。第一个值为当前元素的下标，第二个值为该下标所对应元素的一份副本*（副本存放在同一个地址空间）*

可以将下标或值赋予 `_` 来忽略它

```go
for i, _ := range pow
for _, value := range pow
```

若你只需要索引，忽略第二个变量即可

```go
for i := range pow
```

#### 总结

数组和切片：数组是固定容量的，切片不是，切片是动态的可扩容的，本质上是一个struct

### 两个实验

- 参数传递实验

  > ```go
  > package main
  > 
  > import "fmt"
  > 
  > func main() {
  > 	a1 := [3]int{1, 2, 3}
  > 	fmt.Println("-----before change arr-----")
  > 	fmt.Printf("数组原地址: %p\n", &a1)
  > 	fmt.Println("a1: ", a1)
  > 	fmt.Println("-----after change arr-----")
  > 	changeArray(a1)
  > 	fmt.Println("a1: ", a1)
  > 	
  > 	fmt.Println("**************************\n")
  > 	
  > 	s1 := a1[:]
  > 	fmt.Println("-----before change slice-----")
  > 	fmt.Printf("切片原地址: %p\n", &s1)
  > 	fmt.Println("s1: ", s1)
  > 	fmt.Println("-----after change slice-----")
  > 	changeSlice(s1)
  > 	fmt.Println("s1: ", s1)
  > 	fmt.Println("a1: ", a1)
  > 	
  > 	fmt.Println("**************************\n")
  > 	
  > 	fmt.Println("-----before append slice-----")
  > 	fmt.Printf("切片原地址: %p\n", &s1)
  > 	fmt.Println("s1: ", s1)
  > 	fmt.Println("-----after append slice-----")
  > 	appendAndChangeSlice(s1)
  > 	fmt.Println("s1: ", s1)
  > 	fmt.Println("a1: ", a1)
  > 
  > }
  > 
  > func changeArray(arr [3]int){
  > 	arr[0] = 999
  > 	fmt.Printf("数组参数地址: %p\n", &arr)
  > }
  > 
  > func changeSlice(slice []int){
  > 	slice[0] = 999
  > 	fmt.Printf("切片参数地址: %p\n", &slice)
  > 	fmt.Printf("切片首地址: %p\n", &slice[0])
  > }
  > 
  > func appendAndChangeSlice(slice []int){
  > 	slice = append(slice, 4, 5, 6, 7, 8)
  > 	slice[1] = 9999
  > 	fmt.Printf("切片参数地址: %p\n", &slice)
  > 	fmt.Printf("切片首地址: %p\n", &slice[0])
  > }
  > ```
  >
  > 结果为：
  >
  > ```
  > -----before change arr-----
  > a1:  [1 2 3]
  > 数组原地址: 0xc000016018
  > -----after change arr-----
  > 数组参数地址: 0xc000016048
  > a1:  [1 2 3]
  > **************************
  > 
  > -----before change slice-----
  > s1:  [1 2 3]
  > 切片原地址: 0xc00000c030
  > -----after change slice-----
  > 切片参数地址: 0xc00000c060
  > 切片首地址: 0xc000016018
  > s1:  [999 2 3]
  > a1:  [999 2 3]
  > **************************
  > 
  > -----before append slice-----
  > s1:  [999 2 3]
  > 切片原地址: 0xc00000c030
  > -----after append slice-----
  > 切片参数地址: 0xc00000c0a8
  > 切片首地址: 0xc000104040
  > s1:  [999 2 3]
  > a1:  [999 2 3]
  > ```

- 赋值传递实验

  > ```go
  > package main
  > 
  > import "fmt"
  > 
  > func main() {
  > 	a1 := [3]int{1, 2, 3}
  > 	fmt.Println("a1: ", a1)
  > 	fmt.Printf("a1地址: %p\n", &a1)
  > 	a2 := a1
  > 	fmt.Println("a2: ", a2)
  > 	fmt.Printf("a2地址: %p\n", &a2)
  > 	a2[0] = 999
  > 	fmt.Println("修改a2后, a2: ", a2, "  a1: ", a1)
  > 	
  > 	fmt.Println("**************************\n")
  > 	
  > 	s1 := a1[:]
  > 	fmt.Println("s1: ", s1)
  > 	fmt.Printf("s1地址: %p\n", &s1)
  > 	s2 := s1
  > 	fmt.Println("s2: ", s2)
  > 	fmt.Printf("s2地址: %p\n", &s2)
  > 	fmt.Printf("s1首地址: %p   s2首地址: %p\n", &s1[0], &s2[0])
  > 	
  > }
  > ```
  >
  > 结果为：
  >
  > ```
  > a1:  [1 2 3]
  > a1地址: 0xc0000be000
  > a2:  [1 2 3]
  > a2地址: 0xc0000be030
  > 修改a2后, a2:  [999 2 3]   a1:  [1 2 3]
  > **************************
  > 
  > s1:  [1 2 3]
  > s1地址: 0xc0000b4018
  > s2:  [1 2 3]
  > s2地址: 0xc0000b4048
  > s1首地址: 0xc0000be000   s2首地址: 0xc0000be000
  > ```

总结：

- 数组传参/赋值时传的是数组真实数据的拷贝，而不是地址拷贝（与Java不同，Java的数组是引用类型，这里是值类型）

  > Go语言对待数组的方式和其它很多编程语言不同，其它编程语言可能会隐式地将数组作为引用或指针对象传入被调用的函数

- 切片传参/赋值时传的是切片的拷贝，生成新的切片，但指向的底层数组一致，也就是说，是浅拷贝（切片是引用类型）

- append后会新生成一个底层数组并令切片的Pointer指向它

- 切片首元素的地址==数组首元素的地址==数组地址

**在go函数中参数的传递可以是传值（对象的复制，需要开辟新的空间来存储该新对象）和传引用（指针的复制，和原来的指针指向同一个对象，但两个指针本身的内存地址不一致，只是指向的区域一致），建议使用指针，原因有两个：能够改变参数的值，避免大对象的复制操作节省内存** *（上述实验实际上都是值传递，都有复制过程）*

### go只有值传递

基于上面的实验得出结论：**go只有值传递**

首先明确值传递的定义：传递时会将数据复制一份；而引用传递：可以直接传递指针

区别在哪呢？假如有一个函数func foo(ptr *int)，我们有下面的函数：

```go
func main() {
    i := 123
    ptr := &i
	foo(ptr)
    fmt.Println(*ptr)
}

func foo(ptr *int) {
    i := 234
    ptr = &i
}
```

如果是引用传递，这里直接传指针，则会打印234，实际上可以尝试一下，打印的是123

说明传递的是指针本身的一个拷贝，也就是，**函数内外指针本身存放在不同的内存地址，但它们指向的内存地址是一致的**，这时去修改函数内的指针指向的内存地址，自然影响不到函数外的指针，所以是值传递

> - 浅拷贝的时候，指针本身也会被拷贝，区分开指针本身地址和它的指向地址
> - Java中同理，只有值传递，引用传递的时候会生成一个引用的副本，但指向地址一致

**for循环的一个注意点**

```go
var rmdirs []func()
for _, d := range tempDirs() {
	dir := d // NOTE: necessary!
	os.MkdirAll(dir, 0755) // creates parent directories too
	rmdirs = append(rmdirs, func() {
		os.RemoveAll(dir)
	})
}
// ...do some work…
for _, rmdir := range rmdirs {
	rmdir() // clean up
}
```

你可能会感到困惑，为什么要在循环体中用循环变量d赋值一个新的局部变量，而不是像下面的代码一样直接使用循环变量dir。需要注意，下面的代码是错误的

```go
var rmdirs []func()
for _, dir := range tempDirs() {
	os.MkdirAll(dir, 0755)
	rmdirs = append(rmdirs, func() {
		os.RemoveAll(dir) // NOTE: incorrect!
	})
}
```

问题的原因在于循环变量的作用域。在上面的程序中，for循环语句引入了新的词法块，循环变量dir在这个词法块中被声明。在该循环中生成的所有函数值都共享相同的循环变量。需要注意，函数值中记录的是循环变量的内存地址，而不是循环变量某一时刻的值。以dir为例，后续的迭代会不断更新dir的值**（修改的是dir对应的内存地址上面的值）**，当删除操作执行时，for循环已完成，dir中存储的值等于最后一次迭代的值。这意味着，每次对os.RemoveAll的调用删除的都是相同的目录

### 映射（Map）

在映射 `m` 中插入或修改元素：

```go
m[key] = elem
```

获取元素：

```go
elem = m[key]
```

删除元素：

```go
delete(m, key)
```

通过双赋值检测某个键是否存在：

```go
elem, ok = m[key]
```

若 `key` 在 `m` 中，`ok` 为 `true` ；否则，`ok` 为 `false`

若 `key` 不在映射中，那么 `elem` 是该映射元素类型的零值

同样的，当从映射中读取某个不存在的键时，结果是映射的元素类型的零值

**注** ：若 `elem` 或 `ok` 还未声明，可以使用短变量声明：

```
elem, ok := m[key]
```

常常以空结构体作为value来实现set，即：

map[string]struct{}{"key1":struct{}{}}

### 字符串和BYTE切片

标准库中有四个包对字符串处理尤为重要：bytes、strings、strconv和unicode包

#### strings

strings包提供了许多如字符串的查询、替换、比较、截断、拆分和合并等功能

#### bytes

bytes包也提供了很多类似功能的函数，但是针对和字符串有着相同结构的[]byte类型。因为字符串是只读的，因此逐步构建字符串会导致很多分配和复制。在这种情况下，使用bytes.Buffer类型将会更有效

bytes包提供了Buffer类型用于字节slice的缓存。一个Buffer开始是空的，但是随着string、byte或[]byte等类型数据的写入可以动态增长，一个bytes.Buffer变量并不需要初始化，因为零值也是有效的：

```go
// intsToString is like fmt.Sprint(values) but adds commas.
func intsToString(values []int) string {
	var buf bytes.Buffer
	buf.WriteByte('[')
	for i, v := range values {
		if i > 0 {
			buf.WriteString(", ")
		}
		fmt.Fprintf(&buf, "%d", v)
	}
	buf.WriteByte(']')
	return buf.String()
}

func main() {
	fmt.Println(intsToString([]int{1, 2, 3})) // "[1, 2, 3]"
}
```

当向bytes.Buffer添加任意字符的UTF8编码时，最好使用bytes.Buffer的WriteRune方法，但是WriteByte方法对于写入类似’[‘和’]‘等ASCII字符则会更加有效。

bytes.Buffer类型有着很多实用的功能，我们在第七章讨论接口时将会涉及到，我们将看看如何将它用作一个I/O的输入和输出对象，例如当做Fprintf的io.Writer输出对象，或者当作io.Reader类型的输入源对象

#### strconv

strconv包提供了布尔型、整型数、浮点数和对应字符串的相互转换，还提供了双引号转义相关的转换

- 将一个整数转为字符串，一种方法是用fmt.Sprintf返回一个格式化的字符串；另一个方法是用strconv.Itoa(“整数到ASCII”)：
- FormatInt和FormatUint函数可以用不同的进制来格式化数字

```go
fmt.Println(strconv.FormatInt(int64(x), 2)) // "1111011"
```

- fmt.Printf函数的%b、%d、%o和%x等参数提供功能往往比strconv包的Format函数方便很多

```go
s := fmt.Sprintf("x=%b", x) // "x=1111011"
```

- 如果要将一个字符串解析为整数，可以使用strconv包的Atoi或ParseInt函数，还有用于解析无符号整数的ParseUint函数：

```go
x, err := strconv.Atoi("123")             // x is an int
y, err := strconv.ParseInt("123", 10, 64) // base 10, up to 64 bits
```

ParseInt函数的第三个参数是用于指定整型数的大小；例如16表示int16，0则表示int。在任何情况下，返回的结果y总是int64类型，可以通过强制类型转换将它转为更小的整数类型

#### unicode

unicode包提供了IsDigit、IsLetter、IsUpper和IsLower等类似功能，它们用于给字符分类。每个函数有一个单一的rune类型的参数，然后返回一个布尔值。而像ToUpper和ToLower之类的转换函数将用于rune字符的大小写转换

> strings包也有类似的函数，它们是ToUpper和ToLower

### JSON

标准库中的encoding/json、encoding/xml、encoding/asn1等包提供支持

> Protocol Buffers的支持由 github.com/golang/protobuf 包提供

结构体slice转为JSON的过程叫编组（marshaling）。编组通过调用json.Marshal函数完成：

```go
data, err := json.Marshal(movies)
if err != nil {
	log.Fatalf("JSON marshaling failed: %s", err)
}
fmt.Printf("%s\n", data)
```

json.MarshalIndent函数将产生整齐缩进的输出。该函数有两个额外的字符串参数用于表示每一行输出的前缀和每一个层级的缩进

在编码时，默认使用Go语言结构体的成员名字作为JSON的对象（通过reflect反射技术）。**只有导出的结构体成员才会被编码**

#### 成员Tag

结构体成员Tag可以指定编码成json时以什么作为名字。一个结构体成员Tag是和在编译阶段关联到该成员的元信息字符串：

```bash
Year  int  `json:"released"`
Color bool `json:"color,omitempty"`
```

成员Tag中json对应值的第一部分用于指定JSON对象的名字，一个额外的omitempty选项，表示当Go语言结构体成员为空或零值时不生成该JSON对象（这里false为零值）

编码的逆操作是解码，对应将JSON数据解码为Go语言的数据结构，Go语言中一般叫unmarshaling，通过json.Unmarshal函数完成

```go
var titles []struct{ Title string }
if err := json.Unmarshal(data, &titles); err != nil {
	log.Fatalf("JSON unmarshaling failed: %s", err)
}
fmt.Println(titles) // "[{Casablanca} {Cool Hand Luke} {Bullitt}]"
```

#### web服务中的 json

许多web服务都提供JSON接口，通过HTTP接口发送JSON格式请求并返回JSON格式的信息。为了说明这一点，可通过Github的issue查询服务来演示类似的用法。首先，我们要定义合适的类型和常量：

```go
// Package github provides a Go API for the GitHub issue tracker.
// See https://developer.github.com/v3/search/#search-issues.
package github

import "time"

const IssuesURL = "https://api.github.com/search/issues"

type IssuesSearchResult struct {
	TotalCount int `json:"total_count"`
	Items          []*Issue
}

type Issue struct {
	Number    int
	HTMLURL   string `json:"html_url"`
	Title     string
	State     string
	User      *User
	CreatedAt time.Time `json:"created_at"`
	Body      string    // in Markdown format
}

type User struct {
	Login   string
	HTMLURL string `json:"html_url"`
}
```

SearchIssues函数发出一个HTTP请求，然后解码返回的JSON格式的结果。因为用户提供的查询条件可能包含类似`?`和`&`之类的特殊字符，为了避免对URL造成冲突，用url.QueryEscape来对查询中的特殊字符进行转义操作。

```go
package github

import (
	"encoding/json"
	"fmt"
	"net/http"
	"net/url"
	"strings"
)

// SearchIssues queries the GitHub issue tracker.
func SearchIssues(terms []string) (*IssuesSearchResult, error) {
	q := url.QueryEscape(strings.Join(terms, " "))
	resp, err := http.Get(IssuesURL + "?q=" + q)
	if err != nil {
		return nil, err
	}

	// We must close resp.Body on all execution paths.
	// (Chapter 5 presents 'defer', which makes this simpler.)
	if resp.StatusCode != http.StatusOK {
		resp.Body.Close()
		return nil, fmt.Errorf("search query failed: %s", resp.Status)
	}

	var result IssuesSearchResult
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		resp.Body.Close()
		return nil, err
	}
	resp.Body.Close()
	return &result, nil
}
```

使用了基于流式的解码器json.Decoder，它可以从一个输入流解码JSON数据。还有一个针对输出流的json.Encoder编码对象

```go
// Issues prints a table of GitHub issues matching the search terms.
package main

import (
	"fmt"
	"log"
	"os"

	"github"
)

func main() {
	result, err := github.SearchIssues(os.Args[1:])
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("%d issues:\n", result.TotalCount)
	for _, item := range result.Items {
		fmt.Printf("#%-5d %9.9s %.55s\n",
			item.Number, item.User.Login, item.Title)
	}
}
```

通过命令行参数指定检索条件。下面的命令是查询Go语言项目中和JSON解码相关的问题，还有查询返回的结果：

```bash
$ go build gopl.io/ch4/issues
$ ./issues repo:golang/go is:open json decoder
13 issues:
#5680    eaigner encoding/json: set key converter on en/decoder
#6050  gopherbot encoding/json: provide tokenizer
#8658  gopherbot encoding/json: use bufio
#8462  kortschak encoding/json: UnmarshalText confuses json.Unmarshal
#5901        rsc encoding/json: allow override type marshaling
#9812  klauspost encoding/json: string tag not symmetric
#7872  extempora encoding/json: Encoder internally buffers full output
#9650    cespare encoding/json: Decoding gives errPhase when unmarshalin
#6716  gopherbot encoding/json: include field name in unmarshal error me
#6901  lukescott encoding/json, encoding/xml: option to treat unknown fi
#6384    joeshaw encoding/json: encode precise floating point integers u
#6647    btracey x/tools/cmd/godoc: display type kind of each named type
#4237  gjemiller encoding/base64: URLEncoding padding is optional
```

> [https://www.k8stech.net/gopl/chapter4/ch4-05/](https://www.k8stech.net/gopl/chapter4/ch4-05/)练习题：
>
> ```go
> // Package github provides a Go API for the GitHub issue tracker.
> // See https://developer.github.com/v3/search/#search-issues.
> package github
> 
> import "time"
> 
> const CreateURL = "https://api.github.com/repos/RickSanchezo137/RickaSanchezo137/issues"
> type CreateReq struct {
> 	Title string `json:"title"`
> 	Labels []string `json:"labels"`
> 	Body string `json:body`
> }
> type CreateRes struct {
> 	Number int `json:"number"`
> 	CreatedAt time.Time `json:"created_at"`
> 	ClosedAt time.Time `json:"closed_at"`
> 	UpdatedAt time.Time `json:"updated_at"`
> 	Title string `json:"title"`
> 	HTMLURL string `json:"html_url"`
> 	State string `json:"state"`
> 	Labels []map[string]interface{} `json:"labels"`
> }
> ```
>
> ```go
> package github
> 
> import (
> 	"encoding/json"
> 	"fmt"
> 	"net/http"
> 	"net/url"
> 	"strings"
> 	"bytes"
> )
> 
> func CreateIssue(labels []string, title []string, context string) (*CreateRes, error) {
> 	ql := url.QueryEscape(strings.Join(labels, " "))
> 	qt := url.QueryEscape(strings.Join(title, " "))
> 	body := CreateReq{qt, []string{ql}, context}
> 	reqData, err := json.Marshal(body)
> 	if err != nil {
> 		return nil, err
> 	}
> 	client := &http.Client{}
> 	req, err := http.NewRequest("POST", CreateURL, bytes.NewReader(reqData))
> 	req.Header.Set("Authorization", "token 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")
> 	req.Header.Set("Accept", "application/vnd.github.v3+json")
> 	req.Header.Set("Content-Type", "application/json;charset=UTF-8")
> 	resp, err := client.Do(req)
> 	defer resp.Body.Close()
> 	if err != nil {
> 		return nil, err
> 	}
> 	if resp.StatusCode != 201 {
> 		return nil, fmt.Errorf("create failure: %s", resp.Status)
> 	}
> 	fmt.Println("create success")
> 	var result CreateRes
> 	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
> 		return nil, err
> 	}
> 	return &result, nil
> }
> ```
>
> ```go
> // Issues prints a table of GitHub issues matching the search terms.
> package main
> 
> import (
> 	"fmt"
> 	"log"
> 	"github"
> 	"encoding/json"
> )
> 
> func main() {
> 	//body=Describe+the+problem
> 	labels := []string{"bug"}
> 	title := []string{"New", "bug", "report"}
> 	resp, err := github.CreateIssue(labels, title, "This is a bug")
> 	if err != nil {
> 		log.Fatal(err)
> 		return
> 	}
> 	data, err := json.MarshalIndent(resp, "", "\t")
> 	if err != nil {
> 		log.Fatalf("JSON marshaling failed: %s", err)
> 	}
> 	fmt.Printf("%s\n", data)
> }
> ```

#### 文本/HTML模板

[https://www.k8stech.net/gopl/chapter4/ch4-06/](https://www.k8stech.net/gopl/chapter4/ch4-06/)

### 函数值

**函数也是值**。它们可以像其它值一样传递

函数值可以用作函数的参数或返回值

```go
package main

import (
	"fmt"
	"math"
)

func compute(fn func(float64, float64) float64) float64 {
	return fn(3, 4)
}
func add(x, y float64) float64 {
	return x + y
}
func main() {
	hypot := func(x, y float64) float64 {
		return math.Sqrt(x*x + y*y)
	}
	fmt.Println(hypot(5, 12))

	fmt.Println(compute(hypot))
	fmt.Println(compute(add))
}
```

### 闭包

> [https://www.sulinehk.com/post/golang-closure-details/](https://www.sulinehk.com/post/golang-closure-details/)

- 闭包是一个函数值，它引用了其函数体之外的变量*（除自身局部变量之外的变量）*。该函数可以访问并赋予其引用的变量的值，换句话说，该函数被这些变量“绑定”在一起，被引用的变量将和这个函数一同存在
- 闭包是函数和相关引用环境组成的实体

```go
package main

import "fmt"

func adder() func(int) int {
	sum := 0
    /* 下面的函数，即adder()的返回值，为一个闭包；因为它引用了自由变量sum，与sum绑定，且能通过函数值作为一个实体存在
     */
	return func(x int) int {
		sum += x
		return sum
	}
}

func main() {
    // 两个闭包实体：pos、neg
	pos, neg := adder(), adder()
	for i := 0; i < 10; i++ {
		fmt.Println(
			pos(i),
			neg(-2*i),
		)
	}
}
```

> 类比Java中的内部类对象：引用某外部变量；与外部变量绑定；作为方法和引用环境组成的实体存在*（Java的方法不能作为类似函数值的东西来保存，因此可以采用匿名内部类的方式，如下，方法为add，看作num和add绑定）*
>
> ```java
> public class Main{
>     public static void main(String[] args){
>         Outer o = new Outer();
>         // in是一个闭包
>         Inner in = o.inner();
>         in.add();
>         in.add();
>         in.add();
>         System.out.println(o.num); // 3 
>         //
>         o = null;
>         System.out.println(o);
>         System.out.println(in);  // o==null, in!=null, 内存泄漏
>     }
> }
> interface Inner{
>     void add();
> }
> class Outer{
>     public int num;
>     public Inner inner(){
>         return new Inner() {
>             @Override
>             public void add() {
>                 num++;
>             }
>         };
>     }
> }
> ```

### init函数

包的初始化首先是解决包级变量的依赖顺序，然后按照包级变量声明出现的顺序依次初始化：

```go
var a = b + c // a 第三个初始化, 为 3
var b = f()   // b 第二个初始化, 为 2, 通过调用 f (依赖c)
var c = 1     // c 第一个初始化, 为 1

func f() int { return c + 1 }
```

如果包中含有多个.go源文件，它们将按照发给编译器的顺序进行初始化，Go语言的构建工具首先会将.go文件根据文件名排序，然后依次调用编译器编译

对于在包级别声明的变量，如果有初始化表达式则用表达式初始化，还有一些没有初始化表达式的，例如某些表格数据初始化并不是一个简单的赋值过程。在这种情况下，我们可以用一个特殊的init初始化函数来简化初始化工作。每个文件都可以包含多个init初始化函数

```go
func init() { /* ... */ }
```

这样的init初始化函数除了不能被调用或引用外，其他行为和普通函数类似。在每个文件中的init初始化函数，在程序开始执行时按照它们声明的顺序被自动调用

每个包在解决依赖的前提下，以导入声明的顺序初始化，每个包只会被初始化一次。因此，如果一个p包导入了q包，那么在p包初始化的时候可以认为q包必然已经初始化过了。初始化工作是自下而上进行的，main包最后被初始化。以这种方式，可以确保在main函数执行之前，所有依赖的包都已经完成初始化工作了

## 方法

Go 没有类。不过你可以为结构体类型定义方法

方法就是一类带特殊的 **接收者** 参数的函数

> 方法接收者在它自己的参数列表内，位于 `func` 关键字和方法名之间
>
> 在此例中，`Abs` 方法拥有一个名为 `v`，类型为 `Vertex` 的接收者
>
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"math"
> )
> 
> type Vertex struct {
> 	X, Y float64
> }
> 
> func (v Vertex) Abs() float64 {
> 	return math.Sqrt(v.X*v.X + v.Y*v.Y)
> }
> 
> func main() {
> 	v := Vertex{3, 4}
> 	fmt.Println(v.Abs())
> }
> ```

接收者的类型定义和方法声明必须在同一包内；不能为内建类型声明方法

> 非结构体
>
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"math"
> )
> 
> type MyFloat float64
> 
> func (f MyFloat) Abs() float64 {
> 	if f < 0 {
> 		return float64(-f)
> 	}
> 	return float64(f)
> }
> 
> func main() {
> 	f := MyFloat(-math.Sqrt2)
> 	fmt.Println(f.Abs())
> }
> ```

### 指针接收者

```go
package main

import (
	"fmt"
	"math"
)

type Vertex struct {
	X, Y float64
}

func (v Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

// 
func (v *Vertex) Scale(f float64) {
	v.X = v.X * f
	v.Y = v.Y * f
}

func main() {
	v := Vertex{3, 4}
	v.Scale(10)
	fmt.Println(v.Abs())  // 50
}
```

注意，如果将此处的*Vertex改成Vertex，结果为5，因为go的值传递和Java不一样，会新生成一个浅拷贝对象；引用传递则是拷贝地址

### 方法即函数&隐式的处理

- 方法即函数：方法都可以写成函数的形式

  ```go
  package main
  
  import (
  	"fmt"
  	"math"
  )
  
  type Vertex struct {
  	X, Y float64
  }
  
  func Abs(v Vertex) float64 {
  	return math.Sqrt(v.X*v.X + v.Y*v.Y)
  }
  
  func Scale(v *Vertex, f float64) {
  	v.X = v.X * f
  	v.Y = v.Y * f
  }
  
  func main() {
  	v := Vertex{3, 4}
  	Scale(&v, 10)
  	fmt.Println(Abs(v))
  }
  ```

  这里的Scale方法的Vertex必须写成*Vertex，否则就是值传递了，创建一个Vertex副本

  但在方法的写法中却不用，为什么？

- 隐式的处理：在方法的写法中，会隐式转换成指针的形式

  比较前两个程序，你大概会注意到带指针参数的函数必须接受一个指针：

  ```go
  var v Vertex
  ScaleFunc(v, 5)  // 编译错误！
  ScaleFunc(&v, 5) // OK
  ```

  而以指针为接收者的方法被调用时，接收者既能为值又能为指针：

  ```go
  var v Vertex
  v.Scale(5)  // OK
  p := &v
  p.Scale(10) // OK
  ```

  对于语句 `v.Scale(5)`，即便 `v` 是个值而非指针，带指针接收者的方法也能被直接调用。 也就是说，由于 `Scale` 方法有一个指针接收者，为方便起见，Go 会将语句 `v.Scale(5)` 编译为 `(&v).Scale(5)`

## 接口

```go
package main

import (
	"fmt"
	"math"
)

type Abser interface {
	Abs() float64
}

func main() {
	var a Abser
	f := MyFloat(-math.Sqrt2)
	v := Vertex{3, 4}

	a = f  // a MyFloat 实现了 Abser
	a = &v // a *Vertex 实现了 Abser



	fmt.Println(a.Abs())
}

type MyFloat float64

func (f MyFloat) Abs() float64 {
	if f < 0 {
		return float64(-f)
	}
	return float64(f)
}

type Vertex struct {
	X, Y float64
}

func (v *Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}
```

类型通过实现一个接口的所有方法来实现该接口。既然无需专门显式声明，也就没有“implements”关键字

隐式接口从接口的实现中解耦了定义，这样接口的实现可以出现在任何包中，无需提前准备

因此，也就无需在每一个实现上增加新的接口名称，这样同时也鼓励了明确的接口定义

```go
package main

import "fmt"

type I interface {
	M()
}

type T struct {
	S string
}

// 此方法表示类型 T 实现了接口 I，但我们无需显式声明此事。
func (t T) M() {
	fmt.Println(t.S)
}

func main() {
	var i I = T{"hello"}
	i.M()
}
```

### 接口值

接口也是值。它们可以像其它值一样传递；接口值可以用作函数的参数或返回值。

在内部，接口值可以看做包含值和具体类型的元组：`(value, type)`

接口值保存了一个具体底层类型的具体值；接口值调用方法时会执行其底层类型的同名方法。见describe方法：

```go
package main

import (
	"fmt"
	"math"
)

type I interface {
	M()
}

type T struct {
	S string
}

func (t *T) M() {
	fmt.Println(t.S)
}

type F float64

func (f F) M() {
	fmt.Println(f)
}

func main() {
	var i I

	i = &T{"Hello"}
	describe(i)
	i.M()

	i = F(math.Pi)
	describe(i)
	i.M()
}

func describe(i I) {
	fmt.Printf("(%v, %T)\n", i, i)
}
```

### 空接口

指定了零个方法的接口值被称为 *空接口：*

```go
interface{}
```

空接口可保存任何类型的值。（因为每个类型都至少实现了零个方法）

空接口被用来处理未知类型的值。例如，`fmt.Print` 可接受类型为 `interface{}` 的任意数量的参数

### 类型断言

断言接口变量是否为某个类型，有点像instanceof

```go
t := i.(T)
```

该语句断言接口值 `i` 保存了具体类型 `T`，并**将其底层类型为 `T` 的值赋予变量 `t`**

若 `i` 并未保存 `T` 类型的值，该语句就会触发一个恐慌（panic）

为了 **判断** 一个接口值是否保存了一个特定的类型，类型断言可返回两个值：其底层值以及一个报告断言是否成功的布尔值

```go
t, ok := i.(T)
```

若 `i` 保存了一个 `T`，那么 `t` 将会是其底层值，而 `ok` 为 `true`

否则，`ok` 将为 `false` 而 `t` 将为 `T` 类型的零值，程序并不会产生恐慌

> 请注意这种语法和读取一个映射时的相同之处

### 类型选择

```go
switch v := i.(type) {
case T:
    // v 的类型为 T
case S:
    // v 的类型为 S
default:
    // 没有匹配，v 与 i 的类型相同
}
```

### Stringer

> 类似Java重写toString

`fmt`包中定义的 `Stringer`是最普遍的接口之一

```go
type Stringer interface {
    String() string
}
```

`Stringer` 是一个可以用字符串描述自己的类型。`fmt` 包（还有很多包）都通过此接口来打印值

### 错误

Go 程序使用 `error` 值来表示错误状态

与 `fmt.Stringer` 类似，`error` 类型是一个内建接口：

```go
type error interface {
    Error() string
}
```

（与 `fmt.Stringer` 类似，`fmt` 包在打印值时也会满足 `error`）

通常函数会返回一个 `error` 值，调用的它的代码应当判断这个错误是否等于 `nil` 来进行错误处理。

```go
i, err := strconv.Atoi("42")
if err != nil {
    fmt.Printf("couldn't convert number: %v\n", err)
    return
}
fmt.Println("Converted integer:", i)
```

例子：

```go
package main

import (
	"fmt"
	"time"
)

type MyError struct {
	When time.Time
	What string
}

func (e *MyError) Error() string {
	return fmt.Sprintf("at %v, %s",
		e.When, e.What)
}

func run() error {
	return &MyError{
		time.Now(),
		"it didn't work",
	}
}

func main() {
	if err := run(); err != nil {
		fmt.Println(err)
	}
}
```

### Reader

`io` 包指定了 `io.Reader` 接口，它表示从数据流的末尾进行读取

Go 标准库包含了该接口的许多实现，包括文件、网络连接、压缩和加密等等

`io.Reader` 接口有一个 `Read` 方法：

```go
func (T) Read(b []byte) (n int, err error)
```

`Read` 用数据填充给定的字节切片并返回填充的字节数和错误值。在遇到数据流的结尾时，它会返回一个 `io.EOF` 错误

示例代码创建了一个 strings.Reader 并以每次 8 字节的速度读取它的输出

```go
package main

import (
	"fmt"
	"io"
	"strings"
)

func main() {
	r := strings.NewReader("Hello, Reader!")

	b := make([]byte, 8)
	for {
		n, err := r.Read(b)
		fmt.Printf("n = %v err = %v b = %v\n", n, err, b)
		fmt.Printf("b[:n] = %q\n", b[:n])
		if err == io.EOF {
			break
		}
	}
}
```

### 常用接口

- FLAG.VALUE

  [FLAG.VALUE接口](https://www.k8stech.net/gopl/chapter7/ch7-04/)

- sort.Interface

  [Sort接口](https://www.k8stech.net/gopl/chapter7/ch7-06/)

- http.Handler

  [Http相关接口](https://www.k8stech.net/gopl/chapter7/ch7-07/)

# goroutine

## goroutine

轻量级线程、协程、用户级线程

```go
go f(x, y, z)
```

会启动一个新的 Go 程并执行

```go
f(x, y, z)
```

`f`, `x`, `y` 和 `z` 的求值发生在当前的 goroutine中，而 `f` 的执行发生在新的 Go 程中

Go 程在相同的地址空间中运行，因此在访问共享的内存时必须进行同步。sync 包提供了这种能力，不过在 Go 中并不经常用到，因为还有其它的办法

### 多对多线程模型

#### 动态栈

每一个OS线程都有一个固定大小的内存块（一般会是2MB）来做栈，这个栈会用来存储当前正在被调用或挂起（指在调用其它函数时）的函数的内部变量。这个固定大小的栈同时很大又很小。因为2MB的栈对于一个小小的goroutine来说是很大的内存浪费，比如对于我们用到的，一个只是用来WaitGroup之后关闭channel的goroutine来说。而对于go程序来说，同时创建成百上千个goroutine是非常普遍的，如果每一个goroutine都需要这么大的栈的话，那这么多的goroutine就不太可能了。除去大小的问题之外，固定大小的栈对于更复杂或者更深层次的递归函数调用来说显然是不够的。修改固定的大小可以提升空间的利用率，允许创建更多的线程，并且可以允许更深的递归调用，不过这两者是没法同时兼备的

相反，一个goroutine会以一个很小的栈开始其生命周期，一般只需要2KB。一个goroutine的栈，和操作系统线程一样，会保存其活跃或挂起的函数调用的本地变量，但是和OS线程不太一样的是，一个goroutine的栈大小并不是固定的；栈的大小会根据需要**动态地伸缩**。而goroutine的栈的最大值有1GB，比传统的固定大小的线程栈要大得多，尽管一般情况下，大多goroutine都不需要这么大的栈

#### goroutine调度

OS线程会被操作系统内核调度。每几毫秒，一个硬件计时器会中断处理器，这会调用一个叫作scheduler的内核函数。这个函数会挂起当前执行的线程并将它的寄存器内容保存到内存中，检查线程列表并决定下一次哪个线程可以被运行，并从内存中恢复该线程的寄存器信息，然后恢复执行该线程的现场并开始执行线程

因为操作系统线程是被内核所调度，所以从一个线程向另一个“移动”需要完整的上下文切换，也就是说，保存一个用户线程的状态到内存，恢复另一个线程的到寄存器，然后更新调度器的数据结构。这几步操作很慢，因为其局部性很差需要几次内存访问，并且会增加运行的cpu周期。

Go的运行时**包含了其自己的调度器**，这个调度器使用了一些技术手段，比如m:n调度，因为其会在n个操作系统线程上多工（调度）m个goroutine。Go调度器的工作和内核的调度是相似的，但是这个调度器只关注单独的Go程序中的goroutine（调度器按程序独立）

和操作系统的线程调度不同的是，Go调度器并不是用一个硬件定时器，而是被Go语言“建筑”本身进行调度的。例如当一个goroutine调用了time.Sleep，或者被channel调用或者mutex操作阻塞时，调度器会使其进入休眠并开始执行另一个goroutine，直到时机到了再去唤醒第一个goroutine。因为这种调度方式不需要进入内核的上下文，所以重新调度一个goroutine比调度一个线程代价要低得多

> Go的调度器使用了一个叫做GOMAXPROCS的变量来决定会有多少个操作系统的线程同时执行Go的代码。其默认的值是运行机器上的CPU的核心数，所以在一个有8个核心的机器上时，调度器一次会在8个OS线程上去调度GO代码（GOMAXPROCS是前面说的m:n调度中的n）。在休眠中的或者在通信中被阻塞的goroutine是不需要一个对应的线程来做调度的。在I/O中或系统调用中或调用非Go语言函数时，是需要一个对应的操作系统线程的，但是GOMAXPROCS并不需要将这几种情况计算在内
>
> 可以用GOMAXPROCS的环境变量来显式地控制这个参数，或者也可以在运行时用runtime.GOMAXPROCS函数来修改它

#### goroutine没有ID号

在大多数支持多线程的操作系统和程序语言中，当前的线程都有一个独特的身份（id），并且这个身份信息可以以一个普通值的形式被很容易地获取到，典型的可以是一个integer或者指针值。这种情况下我们做一个抽象化的thread-local storage（线程本地存储，多线程编程中不希望其它线程访问的内容）就很容易，只需要以线程的id作为key的一个map就可以解决问题，每一个线程以其id就能从中获取到值，且和其它线程互不冲突

goroutine没有可以被程序员获取到的身份（id）的概念。这一点是设计上故意而为之，由于thread-local storage总是会被滥用

## 信道

信道是带有类型的管道，你可以通过它用信道操作符 `<-` 来发送或者接收值

```go
ch <- v    // 将 v 发送至信道 ch
v := <-ch  // 从 ch 接收值并赋予 v
```

> 箭头就是数据流的方向

和映射与切片一样，信道在使用前必须创建：

```go
ch := make(chan int)
// ch := make(chan int, 2) // 带缓冲的
```

默认情况下，发送和接收操作在另一端准备好之前都会**阻塞**。这使得 goroutine 可以在没有显式的锁或竞态变量的情况下进行同步

```go
package main

import "fmt"

func sum(s []int, c chan int) {
	sum := 0
	for _, v := range s {
		sum += v
	}
	c <- sum // 将和送入 c
}

func main() {
	s := []int{7, 2, 8, -9, 4, 0}

	c := make(chan int)
	go sum(s[:len(s)/2], c)
	go sum(s[len(s)/2:], c)
	x, y := <-c, <-c // 从 c 中接收

	fmt.Println(x, y, x+y)
}
```

> 主协程的阻塞会视为死锁

> **为什么卡住不显示？记得问**
>
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
> 	go func(){
> 		ch := make(chan int, 2)
> 		ch <- 1
> 		ch <- 2
> 		fmt.Println(<-ch)
> 		fmt.Println(<-ch)
> 	}()
> }
> ```

```go
make(chan<- int) // 定义只发送的channel
make(<-chan int) // 定义只接收的channel
```

### range 和 close

发送者可通过 `close` 关闭一个信道来表示没有需要发送的值了。**接收者**可以通过为接收表达式分配第二个参数来测试信道是否被关闭：若没有值可以接收且信道已被关闭，那么在执行完

```
v, ok := <-ch
```

之后 `ok` 会被设置为 `false`

循环 `for i := range c` 会不断从信道接收值，直到它被关闭

- 只有发送者才能关闭信道，而接收者不能。向一个已经关闭的信道发送数据会引发程序恐慌（panic）
- 信道与文件不同，通常情况下无需关闭它们。只有在必须告诉接收者不再有需要发送的值时才有必要关闭，例如终止一个 `range` 循环

### select

`select` 语句使一个 Go 程可以等待多个通信操作

`select` 会阻塞到某个分支可以继续执行为止，这时就会执行该分支。当多个分支都准备好时会随机选择一个执行

```go
package main

import "fmt"

func fibonacci(c, quit chan int) {
	x, y := 0, 1
	for {
		select {
		case c <- x:
			x, y = y, x+y
		case <-quit:
			fmt.Println("quit")
			return
		}
	}
}

func main() {
	c := make(chan int)
	quit := make(chan int)
	go func() {
		for i := 0; i < 10; i++ {
			fmt.Println(<-c)
		}
		quit <- 0
	}()
	fibonacci(c, quit)
}
```

当 `select` 中的其它分支都没有准备好时，`default` 分支就会执行

为了在尝试发送或者接收时不发生阻塞，可使用 `default` 分支：

```go
select {
case i := <-c:
    // 使用 i
default:
    // 从 c 中接收会阻塞时执行
}
```

下面这个例子更微妙。ch这个channel的buffer大小是1，所以会交替的为空或为满，所以只有一个case可以进行下去，无论i是奇数或者偶数，它都会打印0 2 4 6 8

```go
ch := make(chan int, 1)
for i := 0; i < 10; i++ {
	select {
	case x := <-ch:
		fmt.Println(x) // "0" "2" "4" "6" "8"
	case ch <- i:
	}
}
```

> 思考为什么？

下面的select语句会在abort channel中有值时，从其中接收值；无值时什么都不做。这是一个非阻塞的接收操作；反复地做这样的操作叫做“轮询channel”

```go
select {
case <-abort:
	fmt.Printf("Launch aborted!\n")
	return
default:
	// do nothing
}
```

信道是并发安全的，go有口头禅：“不要使用共享数据来通信；使用通信来共享数据”

## sync.Mutex

我们已经看到信道非常适合在各个 goroutine 间进行通信

但是如果我们并不需要通信呢？比如说，若我们只是想保证每次只有一个 goroutine 能够访问一个共享的变量，从而避免冲突？

Go 标准库中提供了 sync.Mutex 互斥锁类型及其两个方法：

- `Lock`
- `Unlock`

我们可以通过在代码前调用 `Lock` 方法，在代码后调用 `Unlock` 方法来保证一段代码的互斥执行。参见 `Inc` 方法

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// SafeCounter 的并发使用是安全的。
type SafeCounter struct {
	v   map[string]int
	mux sync.Mutex
}

// Inc 增加给定 key 的计数器的值。
func (c *SafeCounter) Inc(key string) {
	c.mux.Lock()
	// Lock 之后同一时刻只有一个 goroutine 能访问 c.v
	c.v[key]++
	c.mux.Unlock()
}

// Value 返回给定 key 的计数器的当前值。
func (c *SafeCounter) Value(key string) int {
	c.mux.Lock()
	// Lock 之后同一时刻只有一个 goroutine 能访问 c.v
	defer c.mux.Unlock()
	return c.v[key]
}

func main() {
	c := SafeCounter{v: make(map[string]int)}
	for i := 0; i < 1000; i++ {
		go c.Inc("somekey")
	}

	time.Sleep(time.Second)
	fmt.Println(c.Value("somekey"))
}
```

go里没有重入锁

一个通用的解决方案是将一个函数分离为多个函数，比如我们把函数分离成两个：一个不导出的，这个函数不加锁，假设锁总是会被保持并去做实际的操作；另一个是导出的函数，这个函数会调用刚刚的函数，但在调用前会先去获取锁

[读写锁](https://www.k8stech.net/gopl/chapter9/ch9-03/)

只要在go build，go run或者go test命令后面加上-race的flag，就会使编译器创建一个你的应用的“修改”版或者一个附带了能够记录所有运行期对共享变量访问工具的test，并且会记录下每一个读或者写共享变量的goroutine的身份信息。另外，修改版的程序会记录下所有的同步事件，比如go语句，channel操作，以及对`(*sync.Mutex).Lock`，`(*sync.WaitGroup).Wait`等等的调用。

竞争检查器会检查这些事件，会寻找在哪一个goroutine中出现了这样的case，例如其读或者写了一个共享变量，这个共享变量是被另一个goroutine在没有进行同步操作便直接写入的。这种情况也就表明了是对一个共享变量的并发访问，即数据竞争。这个工具会打印一份报告，内容包含变量身份，读取和写入的goroutine中活跃的函数的调用栈

竞争检查器会报告所有的已经发生的数据竞争。然而，它只能检测到运行时的竞争条件；并不能证明之后不会发生数据竞争。所以为了使结果尽量正确，请保证你的测试并发地覆盖到了你的包

由于需要额外的记录，因此构建时加了竞争检测的程序跑起来会慢一些，且需要更大的内存，即使是这样，这些代价对于很多生产环境的工作来说还是可以接受的

[一个缓存的设计](https://www.k8stech.net/gopl/chapter9/ch9-07/)

## sync.WaitGroup

sync.WaitGroup：等待子协程结束

- Add：++
- Wait：Wait Until all end
- Done：--

**注意！**：在开辟子协程之前就Add，如果在子协程内Add，即Add和Done都在子协程内，可能在Add之前主协程就会判断到等待协程数为0，这个子协程就失效了

> [https://tour.go-zh.org/concurrency/10](https://tour.go-zh.org/concurrency/10)

# goWeb

## net包

简单的使用：`resp, err := http.Get(url)`，resp为返回的响应

```go
// Fetch prints the content found at a URL.
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"os"
)

func main() {
	for _, url := range os.Args[1:] {
		resp, err := http.Get(url)
		if err != nil {
			fmt.Fprintf(os.Stderr, "fetch: %v\n", err)
			os.Exit(1)
		}
		b, err := ioutil.ReadAll(resp.Body)
		resp.Body.Close()
		if err != nil {
			fmt.Fprintf(os.Stderr, "fetch: reading %s: %v\n", url, err)
			os.Exit(1)
		}
		fmt.Printf("%s", b)
	}
}
```

- resp.Body：响应体
- resp.Status：状态码

## 简单web服务

Go语言的内置库使得写一个web服务器变得异常简单

```go
// Server1 is a minimal "echo" server.
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", handler) // each request calls handler
	log.Fatal(http.ListenAndServe("localhost:8000", nil))
}

// handler echoes the Path component of the request URL r.
func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "URL.Path = %q\n", r.URL.Path)
}
```

- main函数将所有发送到 **/** 路径下的请求和handler函数关联起来，**/** 开头的请求其实就是所有发送到当前站点上的请求，服务监听8000端口
- 发送到这个服务的“请求”是一个http.Request类型的对象，这个对象中包含了请求中的一系列相关字段，其中就包括我们需要的URL
- 当请求到达服务器时，这个请求会被传给handler函数来处理，这个函数会将/hello这个路径从请求的URL中解析出来，然后把其发送到响应中，这里我们用的是标准输出流的fmt.Fprintf
- 服务器每一次接收请求处理时都会另起一个goroutine，这样服务器就可以同一时间处理多个请求

> Linux/Mac OS下，go run ...的末尾加&进行后台运行

> 实际上，实现web服务需要实现http.Handler接口，有方法serverHTTP
>
> HandleFunc提供了一种转换机制，令普通方法转换成可以作为接口实现方法的格式
>
> ```go
> package http
> 
> type HandlerFunc func(w ResponseWriter, r *Request)
> 
> func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
> 	f(w, r)
> }
> ```
>
> [https://www.k8stech.net/gopl/chapter7/ch7-07/](https://www.k8stech.net/gopl/chapter7/ch7-07/)

[取参数](https://blog.csdn.net/kenkao/article/details/47857757)

不同的方法 → 注册到不同的ServerMux → 运行

go http包提供了默认的DefaultServerMux，监听时传入nil即可

[post请求](https://blog.csdn.net/YMY_mine/article/details/98496009)

## 练习题

### 并发的FTP服务器

> [https://www.k8stech.net/gopl/chapter8/ch8-02/](https://www.k8stech.net/gopl/chapter8/ch8-02/)

> 主要功能
>
> - cd命令来切换目录
> - ls来列出目录内文件
> - get和send来传输文件
> - close来关闭连接

结构：

myFTP

|_main

​	|_server.go

​	|_client.go

|_attach

​	|_serverFuncs.go

​	|_errors.go

|_go.mod

#### main

```go
// server.go
package main

import (
	"log"
	"net"
	"strings"
	"os"
	"fmt"
	"myFTP/attach"
)

type orderFmt struct {
	len int
	description string
	format string
}

var cmds = map[string]orderFmt{
	"doc": orderFmt{1, "查看指令文档", "doc [None]"},
	"pwd": orderFmt{1, "显示当前路径", "pwd [None]"},
	"cd": orderFmt{2, "进入/退出某路径", "cd [Path]"},
	"ls": orderFmt{1, "显示当前路径下内容", "ls [None]"},
	"get": orderFmt{2, "获取某文件到本地", "get [FilePath]"},
	"send": orderFmt{2, "向指定路径传输本地文件", "send [Path]"},
	"close": orderFmt{1, "关闭连接", "close [None]"}}


func main() {
	listener, err := net.Listen("tcp", "localhost:8000")
	if err != nil {
		// Fatal：打印日志；退出程序；不执行defer
		log.Fatal(err)
	}

	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Print(err) // e.g., connection aborted
			continue
		}
		go parseConn(conn) // handle one connection at a time
	}
}

func parseConn(c net.Conn) {
	defer c.Close()
	currPath, err := os.Getwd()
	if err != nil {
		c.Write([]byte("> 读取服务器文件目录异常"))
		return
	}
	for {
		buf := make([]byte, 64)
		_, err := c.Read(buf)
		// 读取错误，表示有可能是连接断开
		if err != nil {
			fmt.Println("疑似远程客户端", c.RemoteAddr(), "异常断开")
			return
		}
		// 指令拆分
		for i, _ := range buf {
			if buf[i] == 0 {
				buf = buf[:i]
				break
			}
		}
		ss := strings.Split(strings.Trim(string(buf), " "), " ")
		if len(ss) == 0 {
			c.Write([]byte("> 请输入指令"))
			continue
		}
		// 取指令头
		_, ok := cmds[string(ss[0])]
		if !ok {
			c.Write([]byte("> 无法解析该指令"))
			continue
		}
		// 处理指令
		ret, err := handler(ss, c, &currPath)
		c.Write([]byte("> " + ret))
		if attach.Errno(2) == err {
			fmt.Println("远程客户端", c.RemoteAddr(), "断开")
			break
		}
	}
}

func handler(ss []string, c net.Conn, path *string) (string, error) {
	head := ss[0]
	if cmds[head].len != len(ss) {
		return "请检查指令格式: " + cmds[head].format, attach.Errno(1)
	}
	
	switch head {
		case "pwd": {
			return *path, nil
		}
		case "ls": {
			s, err := attach.LsFunc(*path)
			if err != nil {
				log.Fatal(err)
				return "'ls': 指令运行异常", attach.Errno(3)
			}
			return s, nil
		}
		case "cd": {
			s, err := attach.CdFunc(path, ss[1])
			if err != nil {
				log.Fatal(err)
				return "'cd': 指令运行异常", attach.Errno(4)
			}
			return s, nil
		}
		case "doc": {
			str := "\n支持的指令有:\n"
			for k, _ := range cmds {
				p := cmds[k]
				str += k + "\t\t" + p.format + "\t\t" + p.description + "\n"
			}
			return str, nil
		}
		case "close": {
			return "FIN", attach.Errno(2)
		}
		default: return "无法处理的异常", attach.Errno(5)
	}
}
```

```go
// client.go
package main

import (
	"log"
	"net"
	"fmt"
	"bufio"
	"os"
)

func main() {
	conn, err := net.Dial("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}
	sendCommand(conn)
}

func sendCommand(c net.Conn) {
	defer c.Close()
	// 发送指令
	var command string
	for {
		buf := make([]byte, 65536)
		fmt.Print("\rmyFTP> ")
		// scanf加换行符很重要！
		// 否则会将上次你输入的回车放在缓冲，下次输入直接跳过
		// 默认不接收空格和回车，所以换方式：① scanf + %c ② NewReader或NewScanner读Stdin
		// fmt.Scanf("%s\n", &command)  
		scanner := bufio.NewScanner(os.Stdin)
		if scanner.Scan() {
			command = scanner.Text()
		}
		if len(command) == 0 {
			continue
		}
		_, err := c.Write([]byte(command))
		if err != nil {
			log.Fatal(err)
		}
		
		// 结果回显
		_, err = c.Read(buf)
		fmt.Println(string(buf))
		if string(buf)[:5] == "> FIN" {
			fmt.Println("> 连接已断开")
			break
		}
	}
}
```

#### attach

 ```go
// errors.go
package attach

import "fmt"

type Errno uintptr 

var errors = [...]string{
	1:   "指令格式异常",   
	2:   "连接断开", 
	3:   "ls指令运行异常",    
	4:   "cd指令运行异常", 
	5:   "无法处理的异常"}

func (e Errno) Error() string {
	if 0 <= int(e) && int(e) < len(errors) {
		return errors[e]
	}
	return fmt.Sprintf("未知异常:  errno %d", e)
}
 ```

```go
// serverFuncs.go
package attach

import (
	"strings"
	"os"
	"fmt"
	"io/ioutil"
)

func LsFunc(path string) (string, error) {
	ret := "\n"
	rd, err := ioutil.ReadDir(path)
	if err != nil {
		return ret, err
	}
	lineCounter := 0
	for _, fi := range rd {
		lineCounter++
		if lineCounter == 5 {
			ret += "\n"
			lineCounter = 0
		}
		if fi.IsDir(){
			// %c是开始和结束符标记，后面跟的是接下来的颜色
			ret += fmt.Sprintf("%c[1;32m%s%c[0m", 0x1B, fi.Name(), 0x1B) + "\t"
		} else {
			ret += fi.Name() + "\t"
		}
	}
	return ret, nil
}

func CdFunc(path *string, dst string) (string, error) {
	newPath := *path
	dst = strings.Trim(dst, "\\")
	ps := strings.Split(dst, "\\")
	if len(ps) == 1{
		if ps[0][0:2] == ".." {
			if len(ps[0]) != 2 {
				return "该路径不存在", nil
			}
			for i := len(newPath) - 1; i >= 0; i-- {
				if newPath[i] == '\\' && i != len(newPath) - 1 {
					*path = newPath[:i]
					if len(*path) == 2 {
						*path += "\\"
					}
					fmt.Println(newPath)
					return *path, nil
				}
			}
			fmt.Println(newPath)
			return "已到达根路径", nil
		}
	}
	for _, v := range ps {
		if newPath[len(newPath) - 1] == '\\' {
			newPath += v
		} else {
			newPath += "\\" + v
		}
		fmt.Println(newPath)
		fi, err := os.Stat(newPath)
		if err != nil {
			return "该路径不存在", nil
		}
		if fi.IsDir() {
			*path = newPath
			continue
		}		
		return "这不是一个目录", nil
	}
	return *path, nil
}
```

> todo: send & get

### 聊天服务器（群聊）

> https://www.k8stech.net/gopl/chapter8/ch8-10/

chat

|_main

​	|_main.go

​	|_client.go

|_utils

​	|_broadcaster.go

​	|_handler.go

```go
// main.go
package main

import (
	"net"
	"log"
	"chat/utils"
)

func main() {
	listener, err := net.Listen("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}
	go utils.Broadcaster()
	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Print(err)
			continue
		}
		go utils.HandleConn(conn)
	}
}
```

```go
package main

import (
	"log"
	"net"
	"fmt"
	"bufio"
	"os"
)

func main() {
	conn, err := net.Dial("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}
	handler(conn)
}

func handler(conn net.Conn) {
	defer conn.Close()
	go clientReader(conn)
	input := bufio.NewScanner(os.Stdin)
	for input.Scan() {
		_, err := fmt.Fprintln(conn, input.Text())
		if err != nil {
			log.Fatal(err)
		}
	}
}

func clientReader(conn net.Conn) {
	input := bufio.NewScanner(conn)
	for input.Scan() {
		fmt.Println(input.Text())
	}
}
```

```go
// broadcaster.go
package utils

type client chan<- string

var (
	entering = make(chan client)
	leaving = make(chan client)
	messages = make(chan string)
)

func Broadcaster() {
	clients := make(map[client]bool)
	for {
		select {
		case msg := <-messages:
			for cli := range clients {
				cli <- msg
			}
		case cli := <-entering: 
			clients[cli] = true
		case cli := <-leaving:
			delete(clients, cli)
			close(cli)
		}
		
	}
}
```

```go
// handler.go
package utils

import (
	"bufio"
	"net"
	"fmt"
)

func HandleConn(conn net.Conn) {
	ch := make(chan string)
	go clientWriter(conn, ch)

	who := "[User-" + conn.RemoteAddr().String() + "]"
	ch <- "You are" + who
	messages <- who + " is online"
	entering <- ch
	
	input := bufio.NewScanner(conn)
	for input.Scan() {
		messages <- who + ": " + input.Text()
	}
	
	leaving <- ch
	messages <- who + " is offline"
	conn.Close()
}

func clientWriter(conn net.Conn, ch <-chan string) {
	for msg := range ch {
		fmt.Fprintln(conn, msg) // NOTE: ignoring network errors
	}
}
```

# go测试

go test命令是一个按照一定的约定和组织来测试代码的程序。在包目录内，所有以`_test.go`为后缀名的源文件在执行go build时不会被构建成包的一部分，它们是go test测试的一部分

在`*_test.go`文件中，有三种类型的函数：测试函数、基准测试（benchmark）函数、示例函数

- 测试函数是以Test为函数名前缀的函数，用于测试程序的一些逻辑行为是否正确；go test命令会调用这些测试函数并报告测试结果是PASS或FAIL
- 基准测试函数是以Benchmark为函数名前缀的函数，它们用于衡量一些函数的性能；go test命令会多次运行基准测试函数以计算一个平均的执行时间
- 示例函数是以Example为函数名前缀的函数，提供一个由编译器保证正确性的示例文档

go test命令会遍历所有的`*_test.go`文件中符合上述命名规则的函数，生成一个临时的main包用于调用相应的测试函数，接着构建并运行、报告测试结果，最后清理测试中生成的临时文件

# 反射

有时候我们需要编写一个函数能够处理一类并不满足普通公共接口的类型的值，也可能是因为它们并没有确定的表示方式，或者是在我们设计该函数的时候这些类型可能还不存在。没有办法来检查未知类型的表示方式，我们被卡住了。这就是我们为何需要反射的原因

## REFLECT.TYPE 和 REFLECT.VALUE

反射是由 reflect 包提供的。它定义了两个重要的类型，Type 和 Value。一个 Type 表示一个Go类型。它是一个接口，有许多方法来区分类型以及检查它们的组成部分，例如一个结构体的成员或一个函数的参数等。唯一能反映 reflect.Type 实现的是接口的类型描述信息，也正是这个实体标识了接口值的动态类型

函数 reflect.TypeOf 接受任意的 interface{} 类型，并以 reflect.Type 形式返回其动态类型：

```go
t := reflect.TypeOf(3)  // a reflect.Type
fmt.Println(t.String()) // "int"
fmt.Println(t)          // "int"
```

其中 TypeOf(3) 调用将值 3 传给 interface{} 参数，将一个具体的值转为接口类型会有一个隐式的接口转换操作，它会创建一个包含两个信息的接口值：操作数的动态类型（这里是 int）和它的动态的值（这里是 3）

因为 reflect.TypeOf 返回的是一个动态类型的接口值，它总是返回具体的类型。因此，下面的代码将打印 “*os.File” 而不是 “io.Writer”。稍后，我们将看到能够表达接口类型的 reflect.Type

```go
var w io.Writer = os.Stdout
fmt.Println(reflect.TypeOf(w)) // "*os.File"
```

要注意的是 reflect.Type 接口是满足 fmt.Stringer 接口的。因为打印一个接口的动态类型对于调试和日志是有帮助的， fmt.Printf 提供了一个缩写 %T 参数，内部使用 reflect.TypeOf 来输出：

```go
fmt.Printf("%T\n", 3) // "int"
```

reflect 包中另一个重要的类型是 Value。一个 reflect.Value 可以装载任意类型的值。函数 reflect.ValueOf 接受任意的 interface{} 类型，并返回一个装载着其动态值的 reflect.Value。和 reflect.TypeOf 类似，reflect.ValueOf 返回的结果也是具体的类型，但是 reflect.Value 也可以持有一个接口值

```go
v := reflect.ValueOf(3) // a reflect.Value
fmt.Println(v)          // "3"
fmt.Printf("%v\n", v)   // "3"
fmt.Println(v.String()) // NOTE: "<int Value>"
```

和 reflect.Type 类似，reflect.Value 也满足 fmt.Stringer 接口，但是除非 Value 持有的是字符串，否则 String 方法只返回其类型。而使用 fmt 包的 %v 标志参数会对 reflect.Values 特殊处理

对 Value 调用 Type 方法将返回具体类型所对应的 reflect.Type：

```go
t := v.Type()           // a reflect.Type
fmt.Println(t.String()) // "int"
```

reflect.ValueOf 的逆操作是 reflect.Value.Interface 方法。它返回一个 interface{} 类型，装载着与 reflect.Value 相同的具体值：

```go
v := reflect.ValueOf(3) // a reflect.Value
x := v.Interface()      // an interface{}
i := x.(int)            // an int
fmt.Printf("%d\n", i)   // "3"
```

reflect.Value 和 interface{} 都能装载任意的值。所不同的是，一个空的接口隐藏了值内部的表示方式和所有方法，因此只有我们知道具体的动态类型才能使用类型断言来访问内部的值（就像上面那样），内部值我们没法访问。相比之下，一个 Value 则有很多方法来检查其内容

反射能够访问到结构体中未导出的成员

## 通过REFLECT.VALUE修改值

Go语言中类似x、x.f[1]和*p形式的表达式都可以表示变量，但是其它如x + 1和f(2)则不是变量。一个变量就是一个可寻址的内存空间，里面存储了一个值，并且存储的值可以通过内存地址来更新

对于reflect.Values也有类似的区别。有一些reflect.Values是可取地址的；其它一些则不可以。考虑以下的声明语句：

```go
x := 2                   // value   type    variable?
a := reflect.ValueOf(2)  // 2       int     no
b := reflect.ValueOf(x)  // 2       int     no
c := reflect.ValueOf(&x) // &x      *int    no
d := c.Elem()            // 2       int     yes (x)
```

其中a对应的变量不可取地址。因为a中的值仅仅是整数2的拷贝副本。b中的值也同样不可取地址。c中的值还是不可取地址，它只是一个指针`&x`的拷贝。实际上，所有通过reflect.ValueOf(x)返回的reflect.Value都是不可取地址的。但是对于d，它是c的解引用方式生成的，指向另一个变量，因此是可取地址的。我们可以通过调用reflect.ValueOf(&x).Elem()，来获取任意变量x对应的可取地址的Value

我们可以通过调用reflect.Value的CanAddr方法来判断其是否可以被取地址：

```go
fmt.Println(a.CanAddr()) // "false"
fmt.Println(b.CanAddr()) // "false"
fmt.Println(c.CanAddr()) // "false"
fmt.Println(d.CanAddr()) // "true"
```

每当我们通过指针间接地获取的reflect.Value都是可取地址的，即使开始的是一个不可取地址的Value。在反射机制中，所有关于是否支持取地址的规则都是类似的。例如，slice的索引表达式e[i]将隐式地包含一个指针，它就是可取地址的，即使开始的e表达式不支持也没有关系。以此类推，reflect.ValueOf(e).Index(i)对应的值也是可取地址的，即使原始的reflect.ValueOf(e)不支持也没有关系

要从变量对应的可取地址的reflect.Value来访问变量需要三个步骤：

1. 调用Addr()方法，它返回一个Value，里面保存了指向变量的指针
2. 在Value上调用Interface()方法，也就是返回一个interface{}，里面包含指向变量的指针
3. 知道变量的类型，我们可以使用类型的断言机制将得到的interface{}类型的接口强制转为普通的类型指针。这样我们就可以通过这个普通指针来更新变量了：

```go
x := 2
d := reflect.ValueOf(&x).Elem()   // d refers to the variable x
px := d.Addr().Interface().(*int) // px := &x
*px = 3                           // x = 3
fmt.Println(x)                    // "3"
```

或者，不使用指针，而是通过调用可取地址的reflect.Value的reflect.Value.Set方法来更新对应的值：

```go
d.Set(reflect.ValueOf(4))
fmt.Println(x) // "4"
```

Set方法将在运行时执行和编译时进行类似的可赋值性约束的检查。以上代码，变量和值都是int类型，但是如果变量是int64类型，那么程序将抛出一个panic异常，所以关键问题是要确保改类型的变量可以接受对应的值：

```go
d.Set(reflect.ValueOf(int64(5))) // panic: int64 is not assignable to int
```

同样，对一个不可取地址的reflect.Value调用Set方法也会导致panic异常：

```go
x := 2
b := reflect.ValueOf(x)
b.Set(reflect.ValueOf(3)) // panic: Set using unaddressable value
```

这里有很多用于基本数据类型的Set方法：SetInt、SetUint、SetString和SetFloat等。

```go
d := reflect.ValueOf(&x).Elem()
d.SetInt(3)
fmt.Println(x) // "3"
```

从某种程度上说，这些Set方法总是尽可能地完成任务。以SetInt为例，只要变量是某种类型的有符号整数就可以工作，即使是一些命名的类型、甚至只要底层数据类型是有符号整数就可以，而且如果对于变量类型值太大的话会被自动截断。但需要谨慎的是：对于一个引用interface{}类型的reflect.Value调用SetInt会导致panic异常，即使那个interface{}变量对于整数类型也不行

```go
x := 1
rx := reflect.ValueOf(&x).Elem()
rx.SetInt(2)                     // OK, x = 2
rx.Set(reflect.ValueOf(3))       // OK, x = 3
rx.SetString("hello")            // panic: string is not assignable to int
rx.Set(reflect.ValueOf("hello")) // panic: string is not assignable to int

var y interface{}
ry := reflect.ValueOf(&y).Elem()
ry.SetInt(2)                     // panic: SetInt called on interface Value
ry.Set(reflect.ValueOf(3))       // OK, y = int(3)
ry.SetString("hello")            // panic: SetString called on interface Value
ry.Set(reflect.ValueOf("hello")) // OK, y = "hello"
```

一个可取地址的reflect.Value会记录一个结构体成员是否是未导出成员，如果是的话则拒绝修改操作。因此，CanAddr方法并不能正确反映一个变量是否是可以被修改的。另一个相关的方法CanSet是用于检查对应的reflect.Value是否是可取地址并可被修改的：

```go
fmt.Println(fd.CanAddr(), fd.CanSet()) // "true false"
```

## 显示一个类型的方法集

使用reflect.Type来打印任意值的类型和枚举它的方法：

```go
// Print prints the method set of the value x.
func Print(x interface{}) {
	v := reflect.ValueOf(x)
	t := v.Type()
	fmt.Printf("type %s\n", t)

	for i := 0; i < v.NumMethod(); i++ {
		methType := v.Method(i).Type()
		fmt.Printf("func (%s) %s%s\n", t, t.Method(i).Name,
			strings.TrimPrefix(methType.String(), "func"))
	}
}
```

reflect.Type和reflect.Value都提供了一个Method方法。每次t.Method(i)调用将一个reflect.Method的实例，对应一个用于描述一个方法的名称和类型的结构体。每次v.Method(i)方法调用都返回一个reflect.Value以表示对应的值，也就是一个方法是帮到它的接收者的。使用reflect.Value.Call方法（我们这里没有演示），将可以调用一个Func类型的Value，但是这个例子中只用到了它的类型。

这是属于time.Duration和`*strings.Replacer`两个类型的方法：

```go
methods.Print(time.Hour)
// Output:
// type time.Duration
// func (time.Duration) Hours() float64
// func (time.Duration) Minutes() float64
// func (time.Duration) Nanoseconds() int64
// func (time.Duration) Seconds() float64
// func (time.Duration) String() string

methods.Print(new(strings.Replacer))
// Output:
// type *strings.Replacer
// func (*strings.Replacer) Replace(string) string
// func (*strings.Replacer) WriteString(io.Writer, string) (int, error)
```

