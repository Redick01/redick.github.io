# go 命令<!-- {docsify-ignore-all} -->





## go mod是什么



> 模块是相关Go包的集合。modules是源代码交换和版本控制的单元。
> go命令直接支持使用modules，包括记录和解析对其他模块的依赖性。modules替换旧的基于GOPATH的方法来指定在给定构建中使用哪些源文件。



## 命令详解



#### init

```shell
go mod init
```

生成 go.mod 文件，此命令会在当前目录中初始化和创建一个新的go.mod文件，手动创建go.mod文件再包含一些module声明也等同该命令，而go mod init命令便是帮我们简便操作，可以帮助我们自动创建。



#### download

```shell
go mod download
```

下载 go.mod 文件中指明的所有依赖，使用此命令来下载指定的模块，模块的格式可以根据主模块依赖的形式或者path@version形式指定。



#### tidy

```shell
go mod tidy
```

整理现有的依赖，使用此命令来下载指定的模块，并删除已经不用的模块



#### graph

```shell
go mod graph
```

查看现有的依赖结构，生成项目所有依赖的报告，但可读性太差，图形化更方便。



#### edit

```shell
go mod edit
```

编辑go.mod文件，之后通过download或edit进行下载。



#### vendor

```shell
go mod vendor
```

导出项目所有的依赖到vendor目录，从mod中拷贝到项目的vendor目录下，IDE可以识别这样的目录。



#### verify

```shell
go mod verify
```

校验一个模块是否被篡改过，查询某个常见的模块出错是否已被篡改。



#### why

```shell
go mod why
```

查看为什么需要依赖某模块，查询某个不常见的模块是否是哪个模块的引用。


## go build

go build 是 Go 语言中用于构建应用程序的命令。它会在当前目录中查找并编译 Go 语言源码文件，并生成可执行文件。

#### 基本用法：

```Shell
go build
```
这个简单的命令会在当前目录中查找 Go 源码文件，并编译生成可执行文件。如果当前目录中有多个 Go 文件，它们将被一同编译。

#### 指定输出文件名：
```Shell
go build -o myprogram
```
使用 -o 选项可以指定生成的可执行文件的名称。在上述例子中，生成的可执行文件将被命名为 myprogram。

#### 构建特定文件：
```Shell
go build mysource.go
```
你也可以直接指定要构建的特定源文件。这样只会编译并生成与该源文件对应的可执行文件。

#### 构建并安装：
```Shell
go build -o $GOPATH/bin/myprogram
```
如果你希望将生成的可执行文件安装到 $GOPATH/bin 目录中，可以使用 -o 选项指定输出路径。

#### 使用构建标签：
```Shell
go build -tags mytag
```
通过 -tags 选项，你可以指定使用特定的构建标签。这在处理有条件编译的代码时很有用。

#### 交叉编译：
```Shell
GOOS=linux GOARCH=amd64 go build
```
使用 GOOS 和 GOARCH 环境变量，你可以进行交叉编译，生成在其他操作系统和体系结构上运行的可执行文件。

#### 使用 Go Modules 构建：
如果你的项目使用 Go Modules，你可以在模块模式下使用 go build，而无需在 $GOPATH 中创建项目。确保你的项目目录包含一个 go.mod 文件，并执行以
下命令：
```Shell
go build
```
这将构建你的项目并生成可执行文件。

go build 命令提供了许多选项和用法，可以根据项目的需要进行调整。你可以在命令行中运行 go help build 查看所有选项和详细说明。


## go get

go get 是 Go 语言中用于获取、安装以及升级包的命令。主要功能有两个方面：一是从远程仓库获取并安装包，二是升级已安装的包。

#### 获取并安装包：
```Shell
go get example.com/package
```
这个命令会从指定的远程仓库（例如 GitHub）获取包，并将其安装到你的 $GOPATH 下的 src 目录中。这里的 example.com/package 应该是实际的包导入
路径。

如果包是一个可执行程序，go get 会将可执行文件安装到 $GOPATH/bin 目录下。

#### 获取并安装特定版本：
```Shell
go get example.com/package@v1.2.3
```
使用 @ 符号和版本号可以获取特定版本的包。这样可以确保你的代码在不同时间和环境中使用相同的包版本。

#### 获取并安装到特定目录：
```Shell
go get -d example.com/package
```
通过使用 -d 选项，go get 只会下载包，而不会安装。这可以用于获取包并查看其源码，而不将其安装到 $GOPATH。

#### 升级已安装的包：
```Shell
go get -u example.com/package
```
使用 -u 选项可以将已安装的包更新到最新版本。这个命令会下载并安装包的最新版本。

#### 获取私有仓库的包：
如果包存储在私有仓库，你可以使用 go get，并在导入路径前加上仓库地址：
```Shell
go get your_private_repo.com/your_username/your_package
```
在这里，your_private_repo.com 是私有仓库的地址，your_username 是你的用户名，your_package 是你的包名。

请注意，go get 默认会获取包的最新版本，如果你想要指定版本，请使用 @ 符号和版本号。此外，Go Modules 已经成为推荐的包管理工具，对于新项目，建
议使用 Go Modules 替代传统的 GOPATH 方式。

## go install
go install 是 Go 语言中用于安装 Go 程序的命令。它将编译并安装 Go 源代码，并将生成的二进制文件放在 $GOPATH/bin 目录下。以下是一些
go install 的详解：

不强制使用GOPAHT了，使用go Modules代替

#### 基本用法：
```Shell
go install
```
这个简单的命令会在当前目录中查找 Go 源码文件，并编译生成可执行文件，然后将可执行文件安装到 $GOPATH/bin 目录下。

#### 指定输出路径：
```Shell
go install -o /path/to/custom/directory
```
使用 -o 选项可以指定生成的可执行文件的输出路径。这样会将可执行文件安装到指定的自定义目录中。

#### 安装特定包：
```Shell
go install mypackage
```
你也可以直接指定要安装的特定包。这样只会编译并安装与该包对应的可执行文件。

#### 安装并执行：
```Shell
go install && myprogram
```
通过使用 && 运算符，你可以在 go install 成功后立即运行生成的可执行文件。

#### 使用 Go Modules 安装：
如果你的项目使用 Go Modules，你可以在模块模式下使用 go install，而无需在 $GOPATH 中创建项目。确保你的项目目录包含一个 go.mod 文件，并执行
以下命令：
```Shell
go install
```

这将编译并安装你的项目，并将生成的二进制文件放在 Go Modules 缓存中。

go install 命令提供了许多选项和用法，可以根据项目的需要进行调整。你可以在命令行中运行 go help install 查看所有选项和详细说明。