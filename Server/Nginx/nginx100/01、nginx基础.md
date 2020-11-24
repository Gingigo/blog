# 01、nginx 基础

## 一、优点

- 高并发，高性能
- 可扩展性好
- 高可靠性
- 热部署
- BSD 许可证

## 二、组成

- 汽车：Nginx 二进制可执行文件；
- 驾驶员：Nginx.conf 配置文件；
- 行驶轨迹：access.log 访问日志；
- 黑匣子：error.log 错误日志。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/server/nginx/nginx100/01-01.jpg)

<center>图01、nginx 四大组成部分</center>

## 三、安装

参考：[Nginx单机搭建](../../../环境配置/Nginx单机搭建.md)

## 四、语法

1. 配置文件由指令与指令块构成
2. 每条指令以；分号结尾，指令与参数间以空格符号分隔
3. 指令块以 {} 大括号将多条指令组织在一起
4. include 语句允许组合多个配置文件以提升可维护性
5. 使用 # 符合添加注释，提高可读性
6. 使用 $ 符号使用变量
7. 部分指令的参数支持正则表达式

### 时间单位

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/server/nginx/nginx100/01-02.jpg)

<center>图02、nginx 配置参数时间单位</center>

### 空间单位

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/server/nginx/nginx100/01-03.jpg)

<center>图03、nginx 配置参数空间单位</center>

### http 指令块

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/server/nginx/nginx100/01-04.jpg)

<center>图04、nginx http 指令块

