#  Spring5 源码环境配置

### 下载 spring5 源码

- 源码的版本是 **v5.2.6.RELEASE**

- 下载地址 ：[点击跳转](https://github.com/spring-projects/spring-framework/tree/v5.2.6.RELEASE)

### 下载 Gradle 

- 查看 spring 构建需要哪个版本: 

  ```
  spring-framework-5.2.6.RELEASE\gradle\wrapper\gradle-wrapper.properties
  ```

- 获取到的版本是：gradle-5.6.4 [下载地址](https://gradle.org/releases/)

- 配置 Gradle 环境,(改成maven获取和阿里镜像)

- 参考环境配置 [参考链接](https://www.cnblogs.com/NyanKoSenSei/p/11458953.html)

###  构建spring 

- 不要先用idea打开

- 进入spring-framework-5.2.6.RELEASE 目录，执行`gradlew.bat`

  ```shell
  E:\gin\source-code\spring-framework-5.2.6.RELEASE>gradlew.bat
  ```

- 如果报错`org.gradle.process.internal.ExecException: Process 'command 'git'' finished with non-zero exit value 128`

  解决方法：将项目添加到git

- 正确构建 `BUILD SUCCESSFUL in 1s`

### idea 导入项目

- 用idea 打开项目，自动构建
- 如果测试dome无法引用项目中的模块，删除当前模块，重新添加即可



> 参考文章 
>
> http://shangdixinxi.com/detail-1419798.html
>
> https://www.cnblogs.com/zhangfengxian/p/11072500.html





