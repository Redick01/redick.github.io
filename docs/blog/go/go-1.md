# go mod命令作用<!-- {docsify-ignore-all} -->





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