# 01、git 基本操作

## 基本结构

### 个人基本结构

<img src="https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/cicd/git/git_01_01.jpg" style="zoom:80%;" />

<center>图01  git 基本结构图</center>

其中，工作区、暂存区、本地库 是属于个人开发环境下的，而 GITHUB 是共区域；

### 团队内部协作

<img src="https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/cicd/git/git_01_02.jpg" style="zoom:70%;" />

<center>图02  git 团队内部协作流程图</center>

基本流程是：

1. dev01 创建本地库，编写代码，经过 git add 、git commit  提交到 dev01本地库；
2. dev01 将本地库和远程 GITHUB 库绑定，并将 dev01 本地库的代码推到远程库中
3. dev02 通过 git clone 获取远程库的代码（会自动初始化 git init）;
4. dev02 修改代码，，经过 git add 、git commit  提交到 dev02 本地库;
5. dev 02 通过 git push 推送到远程库；

### 跨团队协作

<img src="https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/cicd/git/git_01_03.jpg" style="zoom:70%;" />

<center>图03  git 跨团队协作流程图</center>

假设现在有个功能需要第三方人员开发 

基本流程是：

1. 第三方人员 dev03 就可以 fork 项目；
2. dev03 git clone 项目进行开发，git add 、git commit 等提交到dev03 的本地库;
3. 通过git push 推到 dev03 fork的项目（project by dev03）；
4. dev03 通过 pull request 请求合并到原来的项目（project by dev01）;
5. dev01 通过审查看是否通过dev03 的合并请求，如果通过，dev01 通过 git pull 拉去合并的代码；

## 基本命令

- 系统命令

  | 命令                    | 说明                                | 例子                              |
  | ----------------------- | ----------------------------------- | --------------------------------- |
  | `git config user.name`  | 项目级别/仓库级别：开发人员的用户名 | `git config user.name gin`        |
  | `git config user.email` | 项目级别/仓库级别：开发人员的邮箱   | `git config fandunheng@gmail.com` |

> 如果需要配置**系统用户级别**，需要加参数`--global`,如`git config --global user.name gin`
>
> 配置存储的位置：
>
> - 项目级别/仓库级别：`./.git/config`  文件
> - 系统用户级别：`~/.gitconfig` 文件



- 本地库命令

| 命令         |               说明               |                          例子                           |
| :----------- | :------------------------------: | :-----------------------------------------------------: |
| `git init`   |          初始化本地仓库          |                                                         |
| `git add`    |          工作区->暂存区          |           `git add .` 或者 `git add filenzme`           |
| `git rm`     |          暂存区->工作区          |                    `git rm filenzme`                    |
| `git diff`   | 将工作区中的文件和暂存区进行比较 |         `git diff [本地库中历史版本] [文件名]`          |
| `git status` |         查看本地库的状态         |                                                         |
| `git commit` |          暂存区->本地库          |            `git commit -m "commit message"`             |
| `git log`    |      查看提交到本地库的历史      |    `git log --pretty=oneline` 或 `git log --oneline`    |
| `git reflog` |      查看提交到本地库的历史      |                                                         |
| `git reset`  |          本地库前进后退          | `--hard a6ace91` 或 `--hard HEAD^` 或者 `--hard HEAD~n` |

- 分支操作

| 命令                         | 说明     | 例子                  |
| ---------------------------- | -------- | --------------------- |
| `git branch [分支名]`        | 创建分支 | `git branch dev`      |
| `git branch -v`              | 查看分支 |                       |
| `git checkout [分支名]`      | 切换分支 | `git checkout master` |
| `git merge [有新内容分支名]` | 合并分支 | `git merge gin`       |

> 合并分支:
>
> 1. 第一步：切换到接受修改的分支（被合并，增加新内容）上`git checkout [被合并分支名]`
> 2. 第二步：执行 merge 命令 `git merge [有新内容分支名]`

- 解决冲突

  1. 第一步：编辑文件，删除特殊符号；

  2. 第二步：把文件修改到满意的程度，保存退出；

  3. 第三步：git add [文件名]；

  4. 第四步：git commit -m "日志信息"；

     - 注意：此时 commit 一定不能带具体文件名。


- 远程库操作

| 命令                                      | 说明                         | 例子                           |
| ----------------------------------------- | ---------------------------- | ------------------------------ |
| `git remote -v`                           | 查看当前所有远程地址别名     |                                |
| `git remote add [别名] [远程地址]`        | 远程库添加别名               | `git remote add origin http..` |
| `git push [别名] [分支名]`                | 本地库推送到远程库中         | `git push origin master`       |
| `git clone [远程地址]`                    | 克隆远程库到本地库中         | `git clone http..`             |
| `git fetch [远程库地址别名] [远程分支名]` | 拉取远程库                   | `git fetch origin master`      |
| `git merge [远程库地址别名/远程分支名]`   | 远程库的代码合并到当前分支中 | `git merge origin master`      |
| `git pull[远程库地址别名] [远程分支名]`   | 拉取并且合并远程库的代码到   | `git pull origin master`       |



## **参考：**

>[01、尚硅谷GitHub教程](https://www.bilibili.com/video/BV1pW411A7a5)

