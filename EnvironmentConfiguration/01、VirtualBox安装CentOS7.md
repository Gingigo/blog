# VirtualBox 安装CentOS7 

### 下载

-  阿里云站点  [点击下载]( http://mirrors.aliyun.com/centos/7/isos/x86_64/ )
-  官网下载  [点击下载]( http://isoredirect.centos.org/centos/7.4.1708/isos/x86_64/ )

### 分配资源

1. 新建虚拟机

   ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/virtualbox/virtualbox_coentos7_01.png)配置虚拟机参数

   - 名称自己填写，最好能一眼看出是什么系统
   - 类型：Linux
   - 版本：Other Linux（64-bit），因为我下载的是CentOS7 64位的

   - 内存大小结合自己物理机的大小决定
   - 虚拟硬盘，我们选择“现在创建虚拟硬盘”

   ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/virtualbox/virtualbox_coentos7_02.png)

3. 创建虚拟硬盘

   - 文件位置，选择磁盘较大的分盘
   - 文件大小，根据磁盘的大小决定，一般大于等于8G
   - 虚拟硬盘文件类型，选择VDI
   - 存储在物理硬盘，选择动态分配

   ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/virtualbox/virtualbox_coentos7_03.png)

4. 导入ios 文件

   - 打开虚拟机设置

     ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/virtualbox/virtualbox_coentos7_04.png)

   - 添加 Centos镜像

     ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/virtualbox/virtualbox_coentos7_05.png)

5. 启动虚拟机

   ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/virtualbox/virtualbox_coentos7_06.png)

### 安装 CentOS7

1. 选择第一个 “Install CentOS 7”

   ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/virtualbox/virtualbox_coentos7_07.png)

2. 选择语言

   ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/virtualbox/virtualbox_coentos7_08.png)

3. 选择centos配置

   ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/virtualbox/virtualbox_coentos7_09.png)

4. 开始安装，并设置root 用户密码

   ![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/environment/configuration/virtualbox/virtualbox_coentos7_10.png)

### 更新yum

- 安装 wget

  `yum -y install wget`

- 备份

  `mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak`

- 下载yum

  ` wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo`

-  生成缓存 

  ` yum makecache `

- 更新

  ` yum -y update`