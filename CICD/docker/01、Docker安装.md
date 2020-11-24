# 01、Docker安装与原理

## 安装

### 操作系统要求

- 系统：CentOS7+
- 存储驱动程序：overlay2

### 卸载已有的 Docker

```shell
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

### 安装 Docker

- 配置安装源

  ```shell
  $ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
  ```

- 安装最新版

  ```shell
  $ sudo yum install docker-ce docker-ce-cli containerd.io
  ```

- 指定版本

  - 查找版本

    ```shell
    $ sudo yum list docker-ce --showduplicates | sort -r
    ```

  - 带版本安装

    ```shell
    $ sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
    ```

- 启动

  ```shell
  $ sudo systemctl start docker
  ```

  - 设置自启动

    ```shell
    $ sudo systemctl enable docker.service
    ```

- 验证

  ```shell
  $ sudo docker run hello-world
  ```

## 原理

### chroot

- 概念

  chroot 是在 Unix 和 Linux 系统的一个操作，针对正在运作的软件行程和它的子进程，改变它外显的根目录。chroot 就是可以改变某进程的根目录，使这个程序不能访问目录之外的其他目录

- 作用

  目录资源隔离

### Namespace

- 概念

  Namespace 是 Linux 内核的一项功能，该功能对内核资源进行隔离，使得容器中的进程都可以在单独的命名空间中运行，并且只可以访问当前容器命名空间的资源。

- 作用

  Namespace 可以隔离进程 ID、主机名、用户 ID、文件名、网络访问和进程间通信等相关资源。

### Cgroups

- 概念

  Cgroups 是一种 Linux 内核功能，可以限制和隔离进程的资源使用情况（CPU、内存、磁盘 I/O、网络等）

- 作用

  在容器的实现中，Cgroups 通常用来限制容器的 CPU 和内存等资源的使用。

### UnionFS

- 概念

   UnionFS，是一种通过创建文件层进程操作的文件系统。

- 作用

  Docker 使用联合文件系统为容器提供构建层，使得容器可以实现写时复制以及镜像的分层构建和存储。

## 参考

> **Docker 安装：入门案例带你了解容器技术原理**