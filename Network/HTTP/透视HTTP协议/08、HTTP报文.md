# 08、HTTP报文

HTTP 协议的<font color='orange'>请求报文</font>和<font color='orange'>响应报文</font>的结构基本相同，由三大部分组成：

## 结构

1. 起始行（start line）：描述请求或响应的基本信息；
2. 头部字段集合（header）：使用 key-value 形式更详细地说明报文；
3. 消息正文（entity）：实际传输的数据，它不一定是纯文本，可以是图片、视频等二进制数据。

> - 请求头或者响应头由**起始行**和**头部字段**组成；
> - 消息正文又称为“**实体**”很多时候就直接称为“**body**”。

HTTP 协议规定报文必须有 header，但可以没有 body，而且在 header 之后必须要有一个“空行”，也就是“CRLF”，十六进制的“0D0A”

一个完整的 HTTP 报文基本结构如下图：

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/network/http/01/09_02.png)

<center>图 01、HTTP 报文基本结构 </center>

下图是结合实际情况 GET 请求的 HTTP 报文

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/network/http/01/09_03.png)

<center>图 02、GET 请求 HTTP 报文信息 </center>

> 其中各个 Web 服务器都不允许过大的请求头，因为头部太大可能会占用大量的服务器资源，影响运行效率

### 起始行

#### 请求行（request line）

- 功能 ：

  - 它简要地描述了**客户端想要如何操作服务器端的资源**

- 构成:

  1. 请求方法：是一个动词，如 GET/POST，表示对资源的操作；
  2. 请求目标：通常是一个 URI，标记了请求方法要操作的资源；
  3. 版本号：表示报文使用的 HTTP 协议版本。

这三个部分通常使用空格（space）来分隔，最后要用 CRLF 换行表示结束。如图03

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/network/http/01/09_04png.png)

<center>图 03、请求行结构图 </center>

#### 状态行（status line）

- 功能;
  - 返回**服务器响应的状态**
- 构成：
  1. 本号：表示报文使用的 HTTP 协议版本；
  2. 状态码：一个三位数，用代码的形式表示处理的结果；
  3. 原因：作为数字状态码补充，是更详细的解释文字。

这三个部分通常使用空格（space）来分隔，最后要用 CRLF 换行表示结束。如图04

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/network/http/01/09_05png.png)

<center>图 04、状态行结构图 </center>

### 头部字段

请求头和响应头的结构是基本一样的，唯一的区别是起始行

- 构成：
  - 头部字段是 key-value 的形式，key 和 value 之间用“:”分隔
  - 最后用 CRLF 换行表示字段结束( 每个 key-value 后都有 CRLF ）
- 扩展
  - 可以自定义头
- 特点：
  - key  不区分大小写;
  - key 不允许空格、下划线“_”，但是可以使用连字符“-”;
  - key 后面必须紧接着“:”;
  - 无顺序
  - 段原则上不能重复，除非这个字段本身的语义允许，例如 Set-Cookie
- 例子:
  - 比如在“Host: 127.0.0.1”这一行里 key 就是“Host”，value 就是“127.0.0.1”。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/network/http/01/09_06.png)

<center>图 05、请求头和响应头的结构图 </center>

#### 常用头字段（四大类）

1. 通用字段：在请求头和响应头里都可以出现；
2. 请求字段：仅能出现在请求头里，进一步说明请求信息或者额外的附加条件；
3. 响应字段：仅能出现在响应头里，补充说明响应报文的信息；
4. 实体字段：它实际上属于通用字段，但专门描述 body 的额外信息。

- **Host**
  - 功能/意义 ：Host 字段告诉服务器这个请求应该由哪个主机来处理；
  - 请求字段；
  - HTTP/1.1 规范里要求**必须出现**的字段。
- **User-Agent**
  - 功能：它使用一个字符串来描述发起 HTTP 请求的客户端；
  - 请求字段
- **Date**
  - 意义：表示 HTTP 报文创建的时间，客户端可以使用这个时间再搭配其他字段决定缓存策略；
  - 通用字段，但通常出现在响应头里。
- **Server**
  - 意义：告诉客户端当前正在提供 Web 服务的软件名称和版本号（但是容易暴露服务信息）
  - 响应字段