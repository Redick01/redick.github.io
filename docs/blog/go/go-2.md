# 自定义包的目录结构、go install 和 go test<!-- {docsify-ignore-all} -->

&nbsp; &nbsp;为了示范，我们创建了一个名为 uc 的简单包，它含有一个 UpperCase 函数将字符串的所有字母转换为大写。当然这并不值得创建一个自定义包，同样的功能已
被包含在 strings 包里，但是同样的技巧也可以应用在更复杂的包中。

## 自定义包的目录结构

&nbsp; &nbsp;下面的结构给了你一个好的示范(uc 代表通用包名, 名字为粗体的代表目录，斜体代表可执行文件):

```Shell
/home/user/goprograms
	ucmain.go	(uc包主程序)
	Makefile (ucmain的makefile)
	ucmain
	src/uc	 (包含uc包的go源码)
		uc.go
	 	uc_test.go
	 	Makefile (包的makefile)
	 	uc.a
	 	_obj
			uc.a
		_test
			uc.a
	bin		(包含最终的执行文件)
		ucmain
	pkg/linux_amd64
		uc.a	(包的目标文件)
```
将你的项目放在 goprograms 目录下(你可以创建一个环境变量 GOPATH，详见第 2.2/3 章节：在 .profile 和 .bashrc 文件中添加 
export GOPATH=/home/user/goprograms)，而你的项目将作为 src 的子目录。uc 包中的功能在 uc.go 中实现。

## 通过Makefile构建应用

```Make
.PHONY: all macos linux windows build help

# application name
BINARY_FILE=uc
# RESOURCE=conf/*.json
MAIN_FILE=ucmain.go

# target
TARGET=bin
# release
# RELEASE=release

all: help

.PHONY: macos
## macos: 生成 MacOS 平台可执行程序
macos:
	@echo "构建 MacOS 平台可执行程序..."
	mkdir -p ${TARGET}/mac
	CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build -gcflags="all=-trimpath=${GOPATH}" -asmflags="all=-trimpath=${GOPATH}" -ldflags "-w -s" -o ${TARGET}/mac/${BINARY_FILE} ${MAIN_FILE}

	# zip -q -r ${TARGET}/mac/${BINARY_FILE}.zip ${BINARY_FILE} ${RESOURCE}

	@go clean
	@echo "Done.\n"

.PHONY: linux
## linux: 生成 Linux 平台可执行程序
linux:
	@echo "构建 Linux 平台可执行程序..."
	mkdir -p ${TARGET}/${RELEASE}/linux
	CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -gcflags="all=-trimpath=${GOPATH}" -asmflags="all=-trimpath=${GOPATH}" -ldflags "-w -s" -o ${TARGET}/linux/${BINARY_FILE} ${MAIN_FILE}
	# zip -q -r ${TARGET}/${RELEASE}/linux/${BINARY_FILE}.zip ${BINARY_FILE} ${RESOURCE}
	@go clean
	@echo "Done.\n"

.PHONY: windows
## windows: 生成 Windows 平台可执行程序
windows:
	@echo "构建 Windows 平台可执行程序..."
	mkdir -p ${TARGET}/${RELEASE}/windows
	CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build -gcflags="all=-trimpath=${GOPATH}" -asmflags="all=-trimpath=${GOPATH}" -ldflags "-w -s" -o ${TARGET}/windows/${BINARY_FILE} ${MAIN_FILE}
	# zip -q -r ${TARGET}/${RELEASE}/windows/${BINARY_FILE}.zip ${BINARY_FILE} ${RESOURCE}
	@go clean
	@echo "Done."

.PHONY: build
## build: 生成 MacOS、Linux、Windows 平台可执行程序
build: clean macos linux windows

.PHONY: run
## run: 运行程序
run:
	@go run ucmain.go

.PHONY: clean
## clean: 移除已构建的二进制文件
clean:
	@go clean
	@rm -rf ${TARGET}

.PHONY: help
## help: 帮助
help:
	@echo "Usage: \n"
	@echo "  make <command> \n"
	@echo "The commands are: \n"
	@sed -n 's/^##//p' makefile | column -t -s ':' | sed -e 's/^/ /'
```
## 示例代码

[示例代码](https://github.com/Redick01/go-code/tree/master/uc)