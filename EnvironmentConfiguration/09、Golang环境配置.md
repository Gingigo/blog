# 09、golang 环境配置

### 下载

- [官网下载](https://golang.org/dl/)

> 可能需要翻墙

### 安装

- 正常安装即可（可调整安装路径）

### 配置

- 调整 GOPATH 的位置（如：调整为 `E:\gin\go`）
- 更新环境变量中 path 路径为 go 的安装路径下的 bin 目录 (如：`D:\software\Go\bin`)
- 在 GOPATH 路径下新建三个目录，分别为 `bin`、`pkg`、`src`

### 知识点

#### 系统

- GOROOT ：GO SDK 的目录
- GOPATH ：当前工作目录

#### 项目

- src ：存放源代码 (项目源代码)
- pkg ：编译时生成的中间文件
- bin ：编译后生成的可执行文件

### Dome

文件 helloworld.go

```go
package main

import "fmt"

func main() {
	fmt.Println("hello world")
}
```

