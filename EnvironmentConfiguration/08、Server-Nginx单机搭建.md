# Nginx单机搭建

## 环境

- CentOS 8
- [nginx-1.18.0](http://nginx.org/download/nginx-1.18.0.tar.gz)

## 安装

- 安装必要的依赖

  ```shell
  yum install -y gcc pcre pcre-devel openssl openssl-devel gd gd-devel
  ```

- 下载并解压

  ```shell
  wget http://nginx.org/download/nginx-1.18.0.tar.gz
  ```

  ```shell
  tar zxfv nginx-1.18.0.tar.gz -C /usr/local/
  ```

- 编译

  ```shell
  cd /usr/local/java/nginx-1.18.0
  ```

  ```shell
  ./configure
  ```

  ```shell
  make&make install
  ```

- 配置 vim 高亮

  ```shell
  cp -r contrib/vim/* ~/.vim
  ```

  

## 启动

- 进入安装目录

  ```shell
  /usr/local/nginx
  ```

- 配置 `nginx.conf`

  ```shell
  vim conf/nginx.conf
  ```

  ```shell
  listen       80; # 修改配置的端口
  ```

- 启动

  ```shell
  cd usr/local/nginx/sbin
  ```

  ```shell
  ./nginx
  ```

- 验证

  ```shell
  ps -ef | grep nginx
  ```

  ```log
  root     21128     1  0 23:08 ?        00:00:00 nginx: master process ./sbin/nginx
  nobody   21129 21128  0 23:08 ?        00:00:00 nginx: worker process
  root     21148  1319  0 23:27 pts/0    00:00:00 grep --color=auto nginx
  ```

- 浏览器访问

  ```htt
  http://ip:port
  ```

  ```tx
  Welcome to nginx!
  ```

  