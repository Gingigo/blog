# 01、Maven 技巧

## 优化编译速度

### 优化一

1. 增加跳过测试代码的编译命令

   ```shell
   -Dmaven.test.skip=true
   ```

2. 指明多线程进行编译

   ```shell
   -Dmaven.compile.fork=true
   ```

3. Maven是3.×以上版本，可以增加 `-T 1C` 参数，表示每个CPU核心跑一个工程

   ```shell
   -T 1C
   ```

完整命令：

```shell
mvn clean package -T 1C -Dmaven.test.skip=true  -Dmaven.compile.fork=true
```

### 优化二

1、指定模块打包

```txt
-am:同时构所列模块的依赖模块
-amd:同时构建依赖于所列模块的模块
-pl <arg> :构建指定的模块，之间用逗号分隔
```

完整命令

```shell
mvn clena package -pl parent-module/child-module1,parent-module/child-module2 -am
```



